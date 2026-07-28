# Architecture

## 1. Shape of the system

```
                         ┌─────────────────────────────────────┐
   git diff  ──────────► │  1. PACKET BUILDER                  │
   repo state            │  diff + files + symbol slice +      │
   CI output             │  conventions + tool output          │
   PR metadata           │  → ReviewPacket (tiered by budget)  │
                         └──────────────┬──────────────────────┘
                                        │  (identical packet to every reviewer)
                         ┌──────────────▼──────────────────────┐
                         │  2. PANEL FAN-OUT                   │
                         │  N × (model, persona) reviewers      │
                         │  blind to each other, blind to author│
                         │  pass A → findings | context request │
                         │  pass B → findings (expanded packet) │
                         │  → Finding[] (structured)           │
                         └──────────────┬──────────────────────┘
                         ┌──────────────▼──────────────────────┐
                         │  3. CLUSTERING                      │
                         │  dedup across reviewers;             │
                         │  count independent model families    │
                         │  → FindingCluster[]                 │
                         └──────────────┬──────────────────────┘
                         ┌──────────────▼──────────────────────┐
                         │  4. VERIFICATION LADDER             │
                         │  T0  reference check   (static)      │
                         │  T0.5 falsification    (repo search) │
                         │  T1  sandbox repro     (execution)   │
                         │  T2  refutation panel  (rival models)│
                         │  → VerifiedCluster[]                │
                         └──────────────┬──────────────────────┘
                         ┌──────────────▼──────────────────────┐
                         │  5. ADJUDICATION & RANKING          │
                         │  publication decision table, caps    │
                         └──────────────┬──────────────────────┘
                         ┌──────────────▼──────────────────────┐
                         │  6. RENDERERS                       │
                         │  GitHub review │ terminal │ SARIF   │
                         │  JSON run record (always)            │
                         └─────────────────────────────────────┘
```

Stages 1, 3, 5, and 6 are deterministic Python. Stages 2 and 4 call models. This split is the
single most important structural decision in the project; see
[ADR-0001](adr/0001-deterministic-orchestration.md).

## 2. Stage 1 — the review packet

Reviewers hallucinate in proportion to what they cannot see. A bare unified diff is the worst
possible input: it invites confident claims about functions the model has never read. The packet
builder's job is to make the diff *legible*.

### Contents

| Section | Source | Notes |
| --- | --- | --- |
| `intent` | PR title + body, commit messages, linked issue | **Required, first position.** Untrusted and fenced (`SECURITY.md`), authorship-anonymized (§2.3) — but never omitted or summarized away. See §2.0. |
| `diff` | `git diff <base>...<head>`, unified, 10 lines context | Canonical. Per-file, with stable hunk IDs. |
| `changed_files` | Full post-image of each changed file | Truncated per budget tier; hunks always retained. |
| `symbol_slice` | Definitions of symbols referenced in the diff but defined outside it | The expensive, high-value part. See §2.2. |
| `conventions` | `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, lint/type config, `.editorconfig` | Tells reviewers what "correct style" means here so they stop guessing. |
| `signals` | Existing CI logs, test results, coverage delta, linter output | Prevents the panel from re-reporting what the linter already caught. |
| `repo_card` | Language, framework, package manager, test command, entrypoints | Cheap orientation. Generated once, cached. |

### 2.0 Intent comes first

The `intent` section leads the packet, before the diff, and is never dropped by a budget tier.
This is a direct consequence of an observed failure
([`OBSERVED_FAILURES.md`](OBSERVED_FAILURES.md) class 2): a reviewer flagged a diff as violating an
architecture decision record when the diff implemented **step 1 of a three-step migration that
same ADR prescribed**, and retaining the flagged shortcut was correct at step 1. The ADR text was
already in the model's context. The decisive missing input was one word in the PR title —
"expand."

The general shape: a diff shows *what changed*, never *what the change is for*. Reviewers asked to
judge correctness without knowing the purpose will infer one, and a wrong inference produces
findings that are internally coherent, well-cited, and entirely wrong. Multi-step migrations,
feature flags behind a rollout, deliberate temporary shims, and staged deprecations are all cases
where the code is correct only relative to an intent the diff cannot express.

Order matters, not just presence. The predecessor system *gathered* PR title and body for the
orchestrator's use and never passed them to the reviewer — the context existed and did not reach
the model. Packet assembly asserts intent is present and first; an empty intent section is a
recorded warning on the run, since it predicts this failure class.

This sits in tension with `SECURITY.md`, which treats the PR description as attacker-controlled.
The resolution is that intent is admissible as evidence about **scope and phase** and inadmissible
as **instructions** or as a claim that the code is correct — see
[`OBSERVED_FAILURES.md`](OBSERVED_FAILURES.md#the-intent-tension).

### 2.1 Budget tiers

Panel members differ by an order of magnitude in context window and by 10× in price. The builder
emits the packet at three tiers and each adapter takes the largest it can afford under its
per-call token cap:

- **`full`** — everything above, whole files, symbol slice depth 2.
- **`standard`** — whole files up to 1500 lines, symbol slice depth 1, signals truncated.
- **`focused`** — hunks with 40 lines of context, symbol slice depth 1 restricted to symbols
  appearing in hunks, no whole files.

Tier assignment is recorded on the run so a finding can always be traced to what its author
actually saw. **A reviewer that only got `focused` is not allowed to be a refuter for claims about
code outside its packet** — this is a correctness rule, not an optimization.

### 2.2 Symbol slice

For each identifier referenced in changed hunks but defined elsewhere, include its definition.
Implementation ladder, cheapest first:

1. **ctags / tree-sitter** — language-agnostic, fast, no project setup. Default.
2. **LSP** (`pylsp`, `tsserver`, `gopls`) via a short-lived client — accurate, slow, requires the
   project to be installable. Opt-in for local runs.
3. **grep fallback** — `rg '\b(def|class|func|function|const) <sym>'`. Last resort.

Depth 1 = direct references. Depth 2 = references of those definitions. Depth 2 blows up fast;
cap by token budget with a breadth-first frontier and record what was elided so the packet can say
"37 further definitions omitted" rather than silently lying by omission.

Slice targets are **not only code symbols**, and references are **not only symbol references.**
Two verified cases, both of which a symbol-only slicer misses entirely:

- The one confirmed absence-claim false positive in
  [`OBSERVED_FAILURES.md`](OBSERVED_FAILURES.md) was disproved by a **database schema**. Schema
  files, migrations, IDL/protobuf, OpenAPI specs, and config are first-class slice targets.
- [`ilm-realtor-535`](../bench/corpus/ilm-realtor-535.md) turns on a CSS `@keyframes` block reached
  from an **animation name embedded in a string literal** (`animate-[slide-up-in_…_both]` in a
  Tailwind class). tree-sitter finds no symbol, LSP resolves nothing, and the import graph has no
  such edge — but without that keyframe body the reviewer cannot know the animation terminates on
  a `transform`, which is the whole defect.

The second is a general class: **cross-language references keyed by string literals.** Tailwind and
CSS-module class names, i18n keys, DI container tokens, SQL in template strings, route names,
feature-flag keys, event names. All are real edges in the program that no language server models.

A tractable approach: harvest string literals from changed hunks, and for each, search the repo for
a *definition-shaped* occurrence in files of other types — `@keyframes <s>`, `.<s> {`, `"<s>":` in
a locale file, a route table entry. Cheap (ripgrep over a candidate set), high-precision when the
literal is distinctive, and it degrades to nothing when it is not. Worth bounding hard: common
short strings will match everywhere and must be dropped by a frequency threshold rather than
flooding the packet.

### 2.2a Write → reader — the relation that carries data coupling

Symbol slicing answers *what does this call*. The caller walk answers *what calls this*. Neither
answers **who reads what this writes** — and for any code whose output is consumed elsewhere, that
is where correctness actually lives.

Verified case: [`ilm-realtor-525`](../bench/corpus/ilm-realtor-525.md). A script inserted
`MediaAsset` rows holding Blob URLs from a different store than the environment's own. The rows are
structurally valid and the insert is correct in isolation. They are wrong because a *consumer* —
`src/lib/mcp/tools/mediaAssets.ts`, which deletes via `del(url)` with the ambient store token —
assumes every catalogued URL lives in that store. The two files never meet: no import, no call, no
shared module. **They are coupled through a database table.**

The predecessor review checked this PR's correctness, tenant-safety, idempotency, and typing, found
all four sound, and shipped the defect — because none of those is the question the code failed.

Resolution is unusually cheap. For a write to a named sink, find its readers:

| Sink | Find readers by |
| --- | --- |
| ORM model / table | `grep -rl '<model>\.'` (`mediaAsset.`) — near-exact with Prisma/ActiveRecord |
| Raw SQL | table name in query strings — falls back to the string-literal search of §2.2 |
| Queue / topic / event | subscriber registration for the topic name |
| Cache key | key prefix, usually a shared constant |
| File format / artifact | producer's extension or path convention |

Include readers at depth 1, capped, and prefer *readers of the specific fields written*. The
signal-to-noise is good because the set is naturally small — seven files matched `mediaAsset.` in a
repo of this size, and one was the consumer that mattered.

**Scheduled at M5, ranked above composition-tree work**, which remains unevidenced. This relation
has a verified corpus entry and a mechanism that is essentially a grep.

The honest limit, recorded on the entry itself: the invariant #525 violated — *"`MediaAsset.url`
must live in the ambient store"* — is written down nowhere. It exists only implicitly, in an
untokened `del()` call. Retrieval can put a reviewer in front of the consumer; it cannot supply an
invariant nobody recorded. Context here is **necessary and not sufficient**.

### 2.2b The composition tree — a further context relation

Symbol slicing and caller walks both traverse the **module graph**. Some defects live on a
relation the module graph does not express: *what wraps this element at render time, and what does
it impose on it.*

The class: anything where behavior is imposed by an **enclosing scope** rather than a called
dependency — React context providers and error boundaries, middleware chains, DI containers,
decorators and aspects, inherited fixtures in a test suite, framework-enforced guarantees. A
reviewer asking "is this request authorized?" cannot answer from a handler whose authorization is
applied by router configuration it never references.

This class also defeats T0.5, which is the sharper reason to care: a repo-wide search for a guard
that is *structurally* rather than *lexically* present returns nothing, and that emptiness reads as
corroboration of the reviewer's absence claim. The mechanism designed to kill false positives
would promote one.

> **Provenance, and a caution.** This section was originally motivated by
> [`ilm-realtor-535`](../bench/corpus/ilm-realtor-535.md), on the assumption that the offending
> `transform` sat on an ancestor component. Reading the actual diffs showed otherwise — it is on
> the *same element*, from a constant in the *same file*, and that entry's real gap is
> string-literal cross-language resolution (§2.2). **This class currently has no verified example
> in the corpus.** It is retained because the reasoning is sound and the failure mode is familiar,
> but it is unevidenced, and it should not be built before an entry demonstrates it. The
> now-corrected motivating example is itself a good argument for that caution: reasoning about
> which context a bug needs, without the bug in front of you, produces confident wrong answers.

Approach, cheapest first — and this is a design sketch, not a settled mechanism:

1. **Static composition edges.** For a changed component, find where it is *rendered* (JSX usage,
   template inclusion, route registration), not where it is imported, and include the enclosing
   element with its styles and props. One level up is likely most of the value.
2. **Enclosing-scope registry.** Framework-specific: middleware chains for a matched route,
   providers wrapping a subtree, decorators on an enclosing class. Requires per-framework
   knowledge and is where this stops being cheap.
3. **Runtime capture** — record the actual rendered tree from an existing test run and slice from
   it. Most accurate, most invasive, almost certainly post-v1.

Honest status: this is an identified gap with a sketched fix, not a solved problem. Level 1 is in
scope for M5 and is the part with a clear cost/benefit. Levels 2 and 3 need evidence from the bench
before they earn a milestone — which is exactly what the corpus entry above is for.

### 2.3 Anonymization

Before the packet leaves the builder, strip authorship signals:

- `Co-Authored-By:` trailers, `Generated with <tool>` footers, `Claude-Session:` lines
- Vendor/model names in the PR body and commit messages → replaced with `[tool]`
- Bot account names in the intent section → `[automation]`

Reason: models defer. A reviewer told the code came from a frontier model finds fewer problems
than one told nothing, and a reviewer told the code came from a *rival* model may find more than
it should. Neither is what we want. Reviewers are told only: *"This diff was written by an
anonymous contributor. Assume it is wrong until you can show it is right."*

This is a heuristic scrub, not a guarantee — code style and comment idiom leak authorship. It
removes the loud signals cheaply. Nothing downstream depends on it holding perfectly.

## 3. Stage 2 — panel fan-out

### 3.1 The panel is a matrix, not a list

A panel entry is a `(model, persona, tier)` triple. Running one model across all personas gives
prompt diversity without weight diversity; running all models on one generic "review this"
prompt gives weight diversity without lane separation, and you get five copies of the same three
findings. The matrix gives both, and lets you spend more on models that earn it.

```
                 contract   concurrency   security   failure   compat   tests   intent
  gpt-5.6-sol       ●            ●                      ●
  gemini-3.1-pro    ●                         ●                   ●
  deepseek-v4-pro                ●            ●                            ●
  glm-5.2 (local)                                       ●                          ●
```

Persona definitions, and why this set, are in [`REVIEW_PROTOCOL.md`](REVIEW_PROTOCOL.md). The set
is configuration, not a constant — it is a claim about a given repo's failure surface, and that
claim is repo-specific.

### 3.2 The context negotiation pass

The packet builder guesses what a reviewer needs. It will sometimes guess wrong, and a reviewer
that silently proceeds on an insufficient packet produces exactly the confident, context-starved
false positive this tool exists to suppress. So round 1 is two fixed passes rather than one call:

- **Pass A.** The reviewer receives the packet and returns *either* findings *or* a
  `context_request`: up to 8 symbols, files, or call sites it needs in order to judge. The prompt
  is explicit that requesting context is the correct move when the packet is insufficient, and
  that guessing is not — the failure mode being designed against is a model that would rather
  answer than admit it cannot see enough.
- **Pass B.** Always exactly one, and skipped entirely if pass A returned findings. The requested
  symbols are resolved by the slicer, appended to the packet, and the reviewer is re-invoked.
  Unresolvable requests come back marked as such — "you asked for `verify_token`; it does not
  exist in this repository" is itself a strong signal, and a reviewer that then still claims a
  guard is missing has just had its claim independently corroborated.

Two calls maximum per reviewer, decided by a fixed rule, so the run structure stays deterministic
and the worst-case cost stays computable before dispatch — the properties
[ADR-0001](adr/0001-deterministic-orchestration.md) exists to protect. This is the bounded,
in-leaf form of adaptive investigation that ADR-0001 anticipated; the orchestrator still does not
make judgment calls.

Whether pass B pays for itself is an open empirical question and an M1 ablation. The cheap
alternative — one pass, larger packet for everyone — may well win on cost per confirmed finding.
The measurement to watch is not false-positive rate alone but **false positives per dollar**,
since a bigger packet for every reviewer and a second pass for the few who need it buy the same
thing at very different prices.

### 3.3 Blindness

Round-1 reviewers receive: the packet, their persona brief, the output schema. They do **not**
receive: other reviewers' findings, the panel roster, prior runs on this PR, or any indication
that other reviewers exist. Blindness is what makes "three families found this independently" a
real signal rather than an echo.

### 3.4 Concurrency and failure

All round-1 calls are issued concurrently, bounded by a semaphore (default 8). A reviewer that
errors after retries, times out, or returns unparseable output is dropped from the run with a
recorded reason. **The pipeline never fails because one provider is down** — it proceeds with the
surviving panel and reports degraded composition in the summary. If fewer than two independent
families survive, the run is marked `INCONCLUSIVE` and publishes nothing but a notice.

### 3.5 Determinism

`temperature=0` where the provider honors it, fixed `seed` where supported, fixed persona order,
fixed packet serialization. This does not make LLM output deterministic — it makes the *run
structure* reproducible, which is what matters for debugging and for the eval bench. Every call's
request and response is written to the run record.

## 4. Stage 3 — clustering

Two reviewers describing the same bug in different words must collapse to one cluster, or the
"independent families" count is meaningless and the PR gets duplicate comments.

Candidate pairs are those overlapping in file and line range (±5 lines). Within candidates,
cluster by cosine similarity of finding embeddings (`title + body`) above a tuned threshold
(start at 0.82; calibrate on the bench). Embeddings via a small cheap model — this is one place
where a single-vendor dependency is acceptable because a clustering mistake is recoverable and
visible.

Deliberately *not* using an LLM to dedup: it is slow, nondeterministic, and it is the stage most
likely to silently merge two distinct bugs into one and drop the second. Threshold-based
clustering fails visibly (duplicate comments) rather than invisibly (dropped findings). Ties and
near-threshold pairs are recorded for bench calibration.

Each cluster carries `independent_families: int` — the number of distinct **model families**
(not models, not reviewers) that raised it. See [`PROVIDERS.md`](PROVIDERS.md#model-families) for
what counts as one family.

## 5. Stage 4 — verification ladder

Detailed rules in [`REVIEW_PROTOCOL.md`](REVIEW_PROTOCOL.md#verification-ladder). In summary:

- **T0 — reference check** (static, free, always). Does the cited file exist? Is the cited line in
  the diff? Do quoted code fragments actually appear in the packet? Kills fabricated citations.
- **T0.5 — falsification search** (static, free, absence claims only). For findings asserting that
  something is *missing* — a hypothesized major false-positive class — run the reviewer's declared
  falsifiers across the whole repository, plus a caller-path walk. Repairs the reviewer's context
  gap mechanically and reframes the finding rather than dropping it.
- **T1 — sandbox repro** (execution, cheap, when a repro is supplied). Reviewers are asked to
  attach an executable repro. It runs in a sandbox. Pass/fail is a *fact*, not an opinion.
- **T2 — refutation** (models, expensive, when T1 is unavailable). Two reviewers from families
  that did **not** raise the finding are asked to refute it, with a prompt biased toward refutation
  and a packet strictly larger than the finder's.
- **T3 — advisory** (no verification). Design and style claims, which are unverifiable by
  construction. Never inline, always collapsed, hard-capped.

## 6. Stage 5 — adjudication

A publication decision table, not a magic score. See
[`REVIEW_PROTOCOL.md`](REVIEW_PROTOCOL.md#publication-decision-table). Two structural rules:

- **Recusal.** If the diff's author is known to be model family F, no reviewer from F may vote in
  refutation. In the common case (Claude wrote the code) this means Claude models can participate
  in round 1 as finders but not in adjudication. Configured via `author_family` or inferred from
  commit trailers *before* anonymization.
- **One vote per family.** Two `gpt-5.6-*` reviewers agreeing is one vote, not two.

## 7. Stage 6 — renderers

The run record — a JSON document containing the packet hash, panel roster, every request/response,
every cluster with its verification trace, and the final decision for each — is always written.
Renderers are pure functions over it.

- `github` — a single PR review with inline comments plus a summary comment. See
  [`GITHUB_ACTION.md`](GITHUB_ACTION.md).
- `terminal` — for local runs.
- `sarif` — for GitHub code scanning / other consumers.
- `json` — the run record itself, the input to the eval bench.

Because renderers are pure over a persisted record, a run can be re-rendered without re-calling
any model. This matters: tuning the publication thresholds is a fast offline loop over saved runs,
not an expensive re-run.

## 8. Repository layout

```
crossexam/
  core/
    packet.py          # ReviewPacket, budget tiers, anonymization
    slicing.py         # symbol slice: ctags / tree-sitter / LSP / grep
    models.py          # Finding, FindingCluster, RunRecord, Verdict (pydantic)
    cluster.py         # embedding + overlap clustering
    ladder.py          # T0..T3 verification
    adjudicate.py      # decision table, ranking, caps
    pipeline.py        # the state machine
  providers/
    base.py            # Reviewer protocol
    litellm_backend.py # default
    openrouter.py      # thin config profile over litellm
    local.py           # ollama / vLLM
    capability.py      # structured-output ladder, per-model capability matrix
  personas/
    *.md               # one brief per persona, versioned
  sandbox/
    runner.py          # T1 execution, container/isolation policy
  render/
    github.py  terminal.py  sarif.py
  bench/
    corpus/            # seeded-bug PRs + clean PRs
    run.py  score.py
  cli.py
.github/workflows/
  crossexam-collect.yml   # untrusted, no secrets
  crossexam-review.yml    # trusted, workflow_run
panel.example.yaml
```

## 9. Deployment modes

| Mode | Where | Panel | Sandbox | Notes |
| --- | --- | --- | --- | --- |
| **CI** | GitHub Action | Hosted APIs | Container, network-off | The primary surface. |
| **Local** | Your PC | Hosted + local (Ollama/vLLM) | Native or container | Free reviewers for the noisy personas; full sandbox fidelity; no secret exposure. |
| **Proxied** | Either, via LiteLLM Proxy | Whatever the proxy exposes | — | One key for the Action, central spend caps and caching, per-model logs. Optional. |

The local mode is not a lesser mode. Running an open-weights model (GLM-5.2, DeepSeek-V4-Pro,
Qwen3.7) locally as a persistent panel member costs nothing per run and adds a genuinely
independent weight lineage, which is exactly the scarce resource. The natural topology for a
single developer is: LiteLLM Proxy on the PC holding all keys plus local models, CI and local CLI
both pointing at it.

## 10. Cross-cutting concerns

**Cost.** Every call is metered. Hard per-run and per-PR ceilings, enforced before dispatch, not
after. Exceeding the ceiling truncates the panel rather than failing the run. See
[`PROVIDERS.md`](PROVIDERS.md#cost-control).

**Caching.** Packets are content-addressed by hash. A re-run with an unchanged packet and
unchanged panel replays from the run record. Incremental PR updates review only the new commits'
diff against the previously reviewed head, with the prior findings as suppressed context.

**Observability.** The run record is the trace. Optional OpenTelemetry export (LiteLLM emits this
natively) for anyone running the proxy.

**Secrets.** Never in the packet, never in a prompt, never in a rendered comment. The packet
builder runs a secret scan over its own output before dispatch and aborts on a hit — a review tool
that ships your `.env` to five vendors is a worse outcome than any bug it could find.
