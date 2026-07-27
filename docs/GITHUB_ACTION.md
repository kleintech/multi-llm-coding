# GitHub Action

The primary surface. Everything below assumes the CLI exists and the Action is a thin wrapper
around `crossexam review --base <sha> --head <sha> --render github`.

---

## 1. The fork problem, first

This is the constraint that determines the whole workflow shape, so it goes first.

crossexam needs **API keys** and **write access to the PR**. GitHub deliberately withholds both
from `pull_request` events on forks, because the alternative is that anyone who can open a PR can
exfiltrate your secrets. The obvious workaround — `pull_request_target` — grants secrets and write
access, but runs in the context of the *base* repo while the PR contains *attacker-controlled
code*. Checking out the PR head under `pull_request_target` and running anything from it (install
scripts, test suites, the repo's own config) hands your secrets to the contributor. This is the
best-known footgun in GitHub Actions and it has burned major projects.

crossexam is a review tool for open-source repos, so fork PRs are the main case, not the edge
case. The design is the **two-workflow split**:

```
┌─────────────────────────────────────────────────────────────┐
│ crossexam-collect.yml     on: pull_request                  │
│ permissions: contents: read     (no secrets, no write)      │
│                                                             │
│ Checks out PR head. Builds the review packet. Runs linters, │
│ types, tests. Uploads packet.json + PR number as artifact.  │
│ NEVER makes a model call. NEVER touches a secret.           │
└───────────────────────────┬─────────────────────────────────┘
                            │ workflow_run: completed
┌───────────────────────────▼─────────────────────────────────┐
│ crossexam-review.yml      on: workflow_run                  │
│ permissions: pull-requests: write                           │
│ secrets: available                                          │
│                                                             │
│ Downloads the artifact. NEVER checks out PR code.           │
│ Runs the panel over the packet. Posts the review.           │
└─────────────────────────────────────────────────────────────┘
```

The trusted job never executes untrusted code — it only *reads* it, as data, and sends it to
models. That is exactly the exposure crossexam is for.

Three rules that make the split actually safe:

1. **The trusted job never checks out the PR head.** Not for a script, not for the config. It
   loads `panel.yaml` from the *base* branch, at the ref that triggered `workflow_run`. A PR that
   edits `panel.yaml` does not get its edited panel used against the base repo's keys.
2. **The artifact is validated as untrusted input.** Size-capped, schema-validated, path-traversal
   checked on every file path it contains. It came from a job an attacker controlled.
3. **The PR number comes from the artifact but is cross-checked** against the `workflow_run`
   payload's head SHA before posting, so a malicious packet cannot redirect the review comment
   onto an unrelated PR.

Sandbox execution (T1) is the one thing that must run *near* untrusted code. It runs in the
**collect** job — no secrets present, network off, container-isolated — and the results ship in
the artifact as claims to be re-validated, not trusted. This costs a round trip: reviewers haven't
proposed repros yet when collect runs. Options, in rough preference order:

- **v1:** T1 runs in a third `workflow_run` job that re-invokes collect-style isolation with the
  reviewer-supplied repros. One extra hop, correct security posture.
- **Alternative:** T1 in the trusted job but inside a network-disabled container with no env
  passthrough. Simpler, and the isolation is real, but it puts attacker-authored test code on the
  same runner as live keys. Not the default.
- **Same-repo PRs** skip all of this and run single-workflow, since the code is already trusted.

## 2. Triggers and gating

Reviewing every push on every PR is how you spend $400 in a month and get the tool switched off.

| Gate | Default |
| --- | --- |
| Draft PRs | Skipped, unless labeled `crossexam` |
| Path filter | Configurable; skip lockfiles, generated code, vendored dirs, `.md`-only changes |
| Diff size | Skip below 10 changed lines; warn and review `focused`-only above 2000 |
| Debounce | On `synchronize`, wait 90s and cancel superseded runs (`concurrency` group per PR) |
| Manual | `/crossexam` comment, or the `crossexam` label, forces a run |
| Author allowlist | Optional `first_time_contributor` restriction for public repos with drive-by PR spam |

## 3. Incremental review

Re-reviewing the whole diff on every push produces the same comments again and reads as noise.

On a `synchronize` event, look up the last reviewed head SHA (stored in the summary comment as an
HTML-commented JSON blob — no external state store needed, which matters for an OSS tool people
should be able to adopt in five minutes):

- Diff for the panel is `last_reviewed_head...new_head`.
- The prior run's published findings ride along as *suppressed context*: "these were previously
  reported; do not re-report unless the new commits made them worse."
- Prior findings whose cited lines were touched by the new commits are re-verified and marked
  resolved / still-present. Findings on untouched lines carry forward unchanged.
- Full re-review on demand via `/crossexam --full`.

Findings are keyed by a **content hash** of `(file, normalized quote, title stem)` rather than by
line number, so unrelated edits that shift line numbers don't resurrect resolved comments — which
is the single most annoying failure mode of every review bot in existence.

## 4. Output format

**One review**, not N comments. Inline comments attach to lines in the diff via the review API.

Each inline comment:

```markdown
**`[concurrency]` Eviction check-then-set races under concurrent access**

`_evict()` reads `len(self._store)` and then deletes, with no lock held. Two threads
crossing here evict the same key twice and the second `del` raises `KeyError`, which
propagates out of `get()`.

Concrete case: two threads calling `get()` on a full cache with the same missing key.

<sub>✅ Confirmed by test · Found independently by 2 of 4 model families · gpt-5.6-sol, deepseek-v4-pro</sub>
```

The footer is not decoration. It is the honesty surface: it says whether this was *proved* or
*voted on*, and by whom. A reader who learns that `[2 families, consensus]` findings are usually
right and `[disputed]` ones usually aren't has learned to use the tool correctly, and can only
learn that if we show them.

The summary comment (updated in place, never duplicated) carries: verdict line, panel roster with
per-model finding counts and survival rates, the funnel (`31 raw → 4 published`), the collapsed
advisory and dropped sections, cost, and the state blob.

Explicitly showing the dropped findings under a `<details>` fold is a deliberate choice. It lets a
skeptical maintainer audit the filter, and it makes the tool's own failure modes visible rather
than hidden — which is the only way the thresholds ever get tuned.

## 5. Permissions and cost hygiene

```yaml
# collect
permissions: { contents: read }
# review
permissions: { pull-requests: write, contents: read }
```

Never `contents: write`. crossexam does not push commits. If suggested patches are added later
they go in as GitHub suggestion blocks that a human clicks, not as commits.

Spend controls in the Action layer, on top of the per-run budget in `panel.yaml`:

- `max_usd_per_pr` accumulated across incremental runs, tracked in the state blob.
- A monthly repo ceiling, tracked in a repo variable, that disables the panel with a notice when
  hit rather than silently continuing to bill.
- Cost is printed in every summary. Making the number visible on every PR is the most effective
  cost control there is.

## 6. Sketch

```yaml
# .github/workflows/crossexam-collect.yml
name: crossexam collect
on: pull_request
permissions: { contents: read }
concurrency: { group: crossexam-${{ github.event.pull_request.number }}, cancel-in-progress: true }
jobs:
  collect:
    if: '!github.event.pull_request.draft'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: astral-sh/setup-uv@v5
      - run: uvx crossexam collect --base ${{ github.event.pull_request.base.sha }}
                                   --head ${{ github.event.pull_request.head.sha }}
                                   --pr   ${{ github.event.pull_request.number }}
                                   --out  packet.json
      - uses: actions/upload-artifact@v4
        with: { name: crossexam-packet, path: packet.json }
```

```yaml
# .github/workflows/crossexam-review.yml
name: crossexam review
on:
  workflow_run: { workflows: [crossexam collect], types: [completed] }
permissions: { pull-requests: write, contents: read }
jobs:
  review:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # base repo only — never the PR head
      - uses: actions/download-artifact@v4
        with:
          name: crossexam-packet
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
      - uses: astral-sh/setup-uv@v5
      - run: uvx crossexam review --packet packet.json --panel panel.yaml --render github
        env:
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
          # or LITELLM_PROXY_URL + LITELLM_VIRTUAL_KEY for the proxy topology
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Shipping this as a **composite action** (`kleintech/crossexam@v1`) rather than raw `uvx` lines is
the v1 packaging goal — the two-workflow split is subtle enough that people will get it wrong if
they have to assemble it by hand, and getting it wrong leaks their keys.
