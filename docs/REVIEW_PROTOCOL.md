# Review protocol

This document specifies what reviewers are asked, what they must return, and how their output is
turned into a decision. It is the contract between the deterministic pipeline and the models.

---

## 1. Personas

A persona is a **lane assignment**, not a personality. Its job is to partition the search space so
that four reviewers cover four different failure classes instead of racing to report the same
null-check. Each brief ends with a hard instruction: *report only findings in your lane; if you
see something outside it, ignore it — another reviewer owns it.*

Personas live in `crossexam/personas/*.md`, are versioned, and are recorded by version hash on
every finding so a regression in a brief is traceable.

| ID | Lane | Hunts for |
| --- | --- | --- |
| `contract` | Inputs and boundaries | Empty/null/zero, off-by-one, overflow, unicode & encoding, timezone & DST, precision loss, unvalidated external input, undocumented preconditions |
| `concurrency` | Time and shared state | Races, TOCTOU, reentrancy, non-atomic read-modify-write, deadlock ordering, missing idempotency on retryable operations, async cancellation |
| `security` | Adversarial input | Injection (SQL/shell/template/prompt), authz gaps and IDOR, secret handling, unsafe deserialization, SSRF, path traversal, unsafe defaults |
| `failure` | What happens when things break | Swallowed exceptions, partial failure leaving inconsistent state, resource leaks, unbounded growth, missing timeouts, retry storms, error paths that are themselves broken |
| `compat` | The blast radius outside the diff | Breaking API/schema/wire changes, migration ordering, backward compatibility, config default changes, behavior changes to existing callers |
| `tests` | Whether the tests are real | Tests that pass regardless of the code, over-mocking, missing negative cases, assertions on the wrong thing, coverage that doesn't touch the changed branch |
| `intent` | Does it do what it says | Diff vs. stated purpose, scope creep, silent behavior change, dead or unreachable additions, TODOs shipped as done |

`intent` deserves special mention: it is the persona most likely to catch the characteristic
failure of AI-authored code, which is not "wrong" so much as "confidently solved a slightly
different problem." Give it to a model from a different family than the author, always.

### Persona brief structure

```
ROLE      one paragraph: your lane, and that you own only it
STANCE    assume the diff is wrong; your job is to show how
METHOD    lane-specific checklist, 5–12 items, phrased as questions
EVIDENCE  you must cite file:line from the provided packet and quote the code
REPRO     if the claim is testable, you MUST supply an executable repro (see §3)
LIMITS    what you may not report (other lanes; style; anything the linter output already shows)
OUTPUT    the schema
```

The `STANCE` line is doing real work. Neutral prompts ("review this code") produce
compliments. The bias toward fault-finding is deliberate and is counterbalanced downstream by the
refutation stage — we would rather over-generate and filter hard than under-generate and have
nothing to filter.

---

## 2. Finding schema

Returned by round-1 reviewers as strict JSON. See
[`PROVIDERS.md`](PROVIDERS.md#structured-output-ladder) for how this is enforced across providers
with different capabilities.

```jsonc
{
  "findings": [
    {
      "title":        "string, ≤ 90 chars, states the defect not the fix",
      "file":         "path as it appears in the diff",
      "line_start":   0,
      "line_end":     0,
      "quote":        "verbatim source line(s) from the packet — used for T0",
      "claim_type":   "mechanical | behavioral | design",
      "severity":     "critical | high | medium | low",
      "category":     "one of the persona lane IDs",
      "body":         "why this is wrong; ≤ 200 words; no restating the code",
      "failure_case": "concrete input/state → wrong output or crash. Required for behavioral.",
      "repro": {                       // null if not supplied
        "kind":    "pytest | jest | go_test | shell",
        "content": "a complete, runnable snippet",
        "expect":  "fails_before_fix"
      },

      // --- absence claims: see §2.3 ---
      "asserts_absence": false,        // "X is never validated / handled / closed / tested"
      "absence_check": {               // REQUIRED when asserts_absence is true
        "symbols":  ["validate_payload", "Schema.check"],   // what would falsify this claim
        "patterns": ["validat", "sanitiz", "assert_"],      // regex, searched repo-wide
        "scope":    "repo | call_path | file"
      },

      "confidence": 0.0                // self-reported; see §2.1
    }
  ]
}
```

### 2.1 Self-assessed fields are untrusted input

`confidence`, `severity`, `claim_type`, and `asserts_absence` are all **self-assigned by a model
that has an incentive to misreport them**, and this is observed behavior, not a worry: a reviewer
in the predecessor system relabeled a style preference as a correctness defect specifically to
evade a prompt ban on style nits ([`OBSERVED_FAILURES.md`](OBSERVED_FAILURES.md) class 4). Any
field that gates publication will be gamed by a model optimizing to be heard.

So they are treated as routing hints, not metadata:

- **`confidence`** carries no weight in the publication decision. Self-reported LLM confidence is
  poorly calibrated and, worse, differently-poorly calibrated per model — ranking across vendors on
  it imports each vendor's calibration bias into the output. Recorded for the bench, where it can
  be scored against ground truth and promoted later if a model proves calibrated.
- **`severity`** is capped, not trusted. A finding with no `failure_case` and no `repro` cannot be
  published above `medium` regardless of what the reviewer claimed.
- **`claim_type`** is cross-checked. A cheap classifier over the finding text flags likely
  misassignment — "would be safer to," "consider," "should instead" phrasing on something labeled
  `behavioral` is the signature of an inflated nit. Disagreements are recorded and reconciled
  toward the *less* severe reading, since the cost asymmetry (ADR-0005) runs that way.
- **`asserts_absence`** is likewise classifier-checked, because under-declaring it is the cheapest
  way to skip the `absence_check` requirement.

The general rule: **never let a model self-certify past a filter.** Where a field decides
publication, something outside the model must be able to check it.

### 2.2 `claim_type` drives everything downstream

| Type | Definition | Max verification tier |
| --- | --- | --- |
| `mechanical` | Checkable by looking: wrong variable, missing await, inverted condition, unreachable branch, resource not closed | T0 often suffices; T1 if testable |
| `behavioral` | The code runs but produces the wrong result for some input | T1 required to reach CONFIRMED_EXECUTABLE |
| `design` | Architecture, naming, coupling, "this should be a different pattern" | T3 only — never inline |

Reviewers self-assign `claim_type` and are told plainly that `design` findings will be collapsed
into a footnote. This is an explicit incentive against the most common LLM review output, which is
confident architectural opinion.

### 2.3 Absence claims — a hypothesized major false-positive class

`asserts_absence` is orthogonal to `claim_type` and exists because of a categorical problem that
more context does not fix.

**Negative existence claims** — *"this input is never validated," "this error is never handled,"
"nothing closes this handle," "no test covers this branch"* — are a natural output of a model
shown a diff, because a diff is precisely the view in which everything outside it is invisible.

The design **assumes** these account for a large share of false positives in diff-only review. That
assumption is untested: it is inferred from the mechanism, not measured on a corpus. It is the
premise of [ADR-0006](adr/0006-context-sufficiency.md) and the justification for T0.5, and
`EVALUATION.md` §5a exists to check it at M1. If the share turns out to be small, most of what
follows should be cut.

The problem is structural, not a matter of packet size. **A positive claim can be verified against
a partial packet; an absence claim cannot.** "Line 42 inverts this condition" is checkable from the
hunk. "Nothing validates this" is a claim about the *whole repository*, and no packet is the whole
repository. Enlarging the packet lowers the rate and never closes the category — there is always
more code outside the window. Every review tool that treats context size as the fix is chasing an
asymptote.

So absence claims are routed differently. A reviewer making one must declare **what would falsify
it** — the symbols and patterns whose existence anywhere in the repo would prove it wrong. That
declaration is not a courtesy; it converts an unfalsifiable assertion into a falsifiable one, and
the pipeline then performs the search the reviewer could not (§3, T0.5).

Reviewers that assert absence without a populated `absence_check` are `MALFORMED`. Being forced to
name a falsifier is itself a filter: a model that cannot say what would disprove its claim is
usually pattern-matching rather than reasoning about this code.

---

## 3. Verification ladder

### T0 — reference check (static, free, universal)

Runs on every finding. Pure Python, no models.

| Check | Failure verdict |
| --- | --- |
| `file` exists in the changed-file set (or in the packet at all) | `INVALID_REFERENCE` |
| `line_start..line_end` falls inside a hunk or an included file | `INVALID_REFERENCE` |
| `quote`, normalized for whitespace, appears at the cited location | `INVALID_REFERENCE` |
| finding is not a restatement of an entry already in packet `signals` (linter/type/CI output) | `REDUNDANT` |
| `behavioral` finding has a non-empty `failure_case` | `MALFORMED` |
| `asserts_absence` finding has a populated `absence_check` | `MALFORMED` |

`INVALID_REFERENCE` findings are **dropped silently and never surfaced**, not even in the collapsed
section. A reviewer that cannot cite the code it is complaining about did not read the code. This
one check is expected to remove the plurality of false positives; the bench (`docs/EVALUATION.md`)
tracks the per-model T0 kill rate, which is a direct measure of a panel member's usefulness.

### T0.5 — falsification search (static, free, absence claims only)

Runs on every finding with `asserts_absence: true`. Pure Python and ripgrep, no models. This is
the stage that answers "the reviewer couldn't see the guard."

The reviewer named what would falsify its claim. The pipeline now runs that search across the
**entire repository** — not the packet, not the diff, the repo — which is exactly the search the
reviewer was structurally unable to perform:

1. **Declared search.** `absence_check.symbols` and `.patterns`, repo-wide.
2. **Caller-path walk.** Resolve callers of the changed function to depth 2 and search their bodies
   too. Catches the single most common case by far: validation that happens one or two frames up
   from the diff.
3. **Sibling-convention search.** If the claim is "no test covers this," look where this repo
   actually puts tests, using the `repo_card`'s conventions, rather than assuming a layout.

Outcomes:

| Result | Verdict | Rationale |
| --- | --- | --- |
| No match anywhere | `ABSENCE_SUPPORTED` → proceed to T2 with the negative result attached | The reviewer's claim survived the search it proposed. Strong signal. |
| Match found **inside** the reviewer's packet | `MALFORMED` → drop | The falsifier was in front of it and it missed it. That is a reading failure, not a context failure. |
| Match found **outside** the packet | `ABSENCE_UNSUPPORTED` → T2 **with the matched code injected** | The context-starvation case. Do not drop — see below. |

The third row is the important one, and the reason this is a routing stage rather than a filter.
A match does **not** mean the finding is wrong. A validator that exists but is wrong, or that
covers a different case, or that runs on the wrong branch, is still a real bug — and it is a
*better* bug than the one originally reported. So the finding is not dropped. It is escalated to
refutation with the found code attached and the question reframed:

> A reviewer claimed X is never validated. It was wrong that nothing exists: here is
> `validate_payload()` at `api/guards.py:88`, called from `handle_request()` at `api/routes.py:41`.
> **Does that guard actually cover the case the reviewer described?** Answer only that.

This converts a false positive into a precise question, and it is the highest-value transformation
in the pipeline. The reviewer's context gap is repaired mechanically and for free, at the exact
moment it matters, without having shipped the whole repository to every reviewer up front.

Findings surviving T0.5 as `ABSENCE_SUPPORTED` are also strong T2 inputs: the refuter is told the
repo-wide search already came back empty, so "I bet there's a guard somewhere" is not available to
it as a lazy refutation.

### T1 — sandbox repro (execution, cheap, when available)

If `repro` is present, run it in the sandbox against the **pre-fix** tree (the base commit with the
diff applied, i.e. the PR head — the claim is that the *current* code is broken):

- Repro fails as predicted → `CONFIRMED_EXECUTABLE`. This is the gold verdict. Publish regardless
  of what any model thinks.
- Repro passes → `REFUTED_EXECUTABLE`. Drop.
- Repro errors (import error, syntax error, missing fixture) → **not a verdict**. Fall through to
  T2. One repair attempt is allowed: the erroring reviewer gets the error text back and one chance
  to fix its own snippet. Ceiling of one, because repair loops are where cost runs away.

Sandbox policy: ephemeral container, no network, no mounted secrets, wall-clock timeout (default
60s), memory cap, read-only mount of the worktree except a scratch dir. In CI this runs on the
Action runner; locally it uses the same container image so verdicts are portable. If no container
runtime is available, T1 is skipped entirely and the run is marked `t1_unavailable` — silently
degrading to model consensus without saying so would misrepresent every verdict in the run.

### T2 — refutation (models, expensive, the fallback)

For clusters still unresolved. Select two refuters:

- from model **families** that did **not** raise the finding (a family cannot refute itself either
  way — no self-defense, no self-corroboration),
- excluding the author's family (**recusal**),
- whose packet tier included the cited code (§2.1 of `ARCHITECTURE.md`),
- preferring families not yet used as refuters in this run, to spread the load.

**Refuters get more context than finders, never less.** The refuter packet is the finder's packet
*plus* the cited file in full, its symbol slice at depth 2, all callers of the changed function,
and any code T0.5 surfaced. This is deliberate and it is worth stating plainly, because the
tempting cost optimization is the opposite — refutation looks like a narrow question, so it looks
like it needs less. It does not. Refuting "nothing validates this" requires seeing the validator,
which is by construction outside the diff. Starving the refuter of context guarantees it reproduces
the finder's blind spot and rubber-stamps exactly the false positives this stage exists to kill.
Refutation is where the context budget should be spent, not saved.

Refuters receive that expanded packet, the single finding, and a prompt biased toward refutation:

> A reviewer claims the following defect. Your job is to **refute** it. Show that the code is
> correct as written, that the claimed failure case cannot occur, or that the reviewer misread the
> code. Cite specific lines. If after genuine effort you cannot refute it, say so — but the
> default answer is `refuted: true`, and you should only return `refuted: false` when you have
> checked the specific mechanism and it holds.

Return schema: `{ "refuted": bool, "reason": string, "misread": bool, "counter_repro": string|null }`.

- Both refute → `REFUTED_CONSENSUS`. Drop.
- Neither refutes → `CONFIRMED_CONSENSUS`.
- Split → `DISPUTED`, and a third refuter from a further family breaks the tie if the finding is
  `high`/`critical`; otherwise it stays `DISPUTED`.

If a refuter returns a `counter_repro` — a runnable snippet demonstrating the code is fine — it
goes through the sandbox too. A passing counter-repro produces `REFUTED_EXECUTABLE`, which
outranks any consensus. Ground truth beats votes in both directions.

**The asymmetry is intentional.** Round 1 is biased toward finding faults; round 2 is biased
toward dismissing them. A finding must clear both a prosecutor and a defense counsel, each doing
its job badly in opposite directions, and the errors substantially cancel. This is the core
mechanism of the whole tool.

### T3 — advisory (no verification)

`design` claims and anything that survives with severity `low`. Never inline. Collapsed into a
single summary section, capped at 5 entries, ranked by independent-family count. Their purpose is
to be skimmable, not actionable.

---

## 4. Publication decision table

Evaluated top to bottom; first match wins. `IF` = independent families that raised the cluster.

| # | Condition | Outcome |
| --- | --- | --- |
| 1 | verdict is `INVALID_REFERENCE`, `MALFORMED`, or `REDUNDANT` | **Drop silently** (recorded in run record only) |
| 2 | verdict is `REFUTED_EXECUTABLE` or `REFUTED_CONSENSUS` | **Drop** (listed in run record; visible with `--verbose`) |
| 3 | verdict is `CONFIRMED_EXECUTABLE` | **Inline comment**, any severity |
| 4 | `claim_type == design` | **Advisory section**, capped at 5 |
| 5 | `CONFIRMED_CONSENSUS` and (`IF ≥ 2` or severity ∈ {critical, high}) | **Inline comment** |
| 6 | `CONFIRMED_CONSENSUS` otherwise | **Summary section** |
| 7 | `DISPUTED` and severity ∈ {critical, high} | **Summary section**, marked *disputed*, with both arguments |
| 8 | anything else | **Run record only** |

### Global caps

- **10 inline comments per run**, default. Ranked by: `CONFIRMED_EXECUTABLE` first, then severity,
  then independent families, then file path for stable ordering. Overflow moves to the summary
  section with an explicit "N further findings not shown inline" line — never silently truncated.
- **3 inline comments per file**, to avoid burying one file in a wall of annotations.
- **5 advisory entries.**

The caps are the precision policy made concrete. A panel that finds thirty real problems in one PR
is describing a PR that should be split, and the right output is "this PR has thirty problems,
here are the ten worst," not thirty annotations.

### What is never blocking

By default crossexam posts `COMMENT`, never `REQUEST_CHANGES`. A bot that can block a merge on a
model's opinion will be disabled by the first person it blocks wrongly, and then it protects
nothing. Opt-in escalation exists for a narrow case: `block_on: confirmed_executable_critical`
blocks only on findings backed by a **failing test that the tool actually ran** — which is a fact
about the code, not a model's view of it.

---

## 5. Worked example

A PR adds a cache with a TTL.

1. **Round 1.** `concurrency` (deepseek-v4-pro) reports a check-then-set race on eviction.
   `contract` (gemini-3.1-pro) reports the same race in different words, plus a claim that
   `ttl=0` means "never expire" when the code treats it as "already expired."
   `failure` (gpt-5.6-sol) reports an unbounded dict, and cites `cache.py:88` — a line that does
   not exist in the packet.
2. **T0.** The `cache.py:88` finding is dropped: `INVALID_REFERENCE`. Three findings remain.
3. **Clustering.** The two race reports merge into one cluster, `independent_families = 2`.
4. **T1.** The `ttl=0` finding shipped a repro. It runs. It fails as predicted →
   `CONFIRMED_EXECUTABLE`. The race cluster has no repro (hard to write deterministically) → T2.
   The unbounded-dict finding has no repro → T2.
5. **T2.** Race cluster: refuters are drawn from gpt-5.6 and a local GLM-5.2 (neither raised it;
   Claude is recused as author). Neither refutes → `CONFIRMED_CONSENSUS`. Unbounded dict: gemini
   refutes, pointing at an eviction call the reporter missed; deepseek agrees → `REFUTED_CONSENSUS`,
   dropped.
6. **Adjudication.** Rule 3 → `ttl=0` inline. Rule 5 (`IF=2`) → race inline. Two inline comments.
7. **Render.** Two inline comments, plus a summary: panel roster, 4 raw findings → 2 published,
   1 dropped as fabricated, 1 refuted, $0.31 spent.

The useful property: the PR author sees two comments, both real, and does not see the two that
weren't.
