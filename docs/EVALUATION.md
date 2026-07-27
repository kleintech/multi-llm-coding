# Evaluation

Without measurement, "adversarial multi-model review" is a vibe. Every design choice in this
repository — panel composition, clustering threshold, the publication table, whether refutation is
worth its cost — is an empirical question, and most of them have non-obvious answers. The bench is
not a nice-to-have that comes after v1; it is the thing that tells us whether v1 works.

It is also the cheapest part of the project to build, because the run record already contains
everything needed to score a run offline.

---

## 1. What we measure

The tool's promise is "high-precision findings from decorrelated reviewers." That decomposes into
four measurable claims:

| Claim | Metric |
| --- | --- |
| Findings we publish are real | **Precision @ published** — the headline number. Target ≥ 0.85. |
| We catch what matters | **Recall @ seeded critical** — of known-critical seeded bugs, what fraction is published. |
| The panel beats its best member | **Panel lift** — panel recall minus best-single-model recall, at matched precision. **If this is not clearly positive, the project has no reason to exist.** |
| Cross-vendor beats single-vendor | **Decorrelation lift** — mixed-family panel vs. same-family panel of equal size and cost. |

Secondary, per-model, and the numbers that actually drive panel composition:

- **T0 kill rate** — fraction of a model's findings that cite code that doesn't exist. A direct
  measure of whether a model is reading or confabulating.
- **Survival rate** — fraction reaching publication. Low survival at high volume still has value
  *if* the filter catches it cheaply; that is the case for a $0.43/M model and not for a $30/M one.
- **Unique contribution** — findings this model raised that *no other family* did, and which
  survived. The single most important panel-composition statistic, and the one that justifies
  keeping a lower-scoring model on the panel.
- **Refuter accuracy** — how often a model's refutations agree with an executable verdict. Some
  models refute everything; they are useless as refuters and must be excluded from T2.
- **Calibration** — self-reported confidence vs. outcome, per model. Currently unused in the
  decision table (see `REVIEW_PROTOCOL.md` §2.1); this measurement is how a model earns the right
  to have its confidence count.

## 1a. The clean-PR trap

Before any corpus detail, one rule that a real failure in the predecessor system makes concrete
([`OBSERVED_FAILURES.md`](OBSERVED_FAILURES.md#the-test-selection-trap)).

Three tuning rounds there were run against a **clean, already-merged PR**. On a clean PR a correct
adversary produces exactly one output: nothing. So the only reachable outcomes were false
positives and silence — **a true positive was structurally impossible**. The adversary contributed
zero real findings across all three rounds, which looks damning and is not evidence about the
adversary at all. Recall was never measured, not once.

Two consequences, both binding:

1. **Every evaluation set must contain known defects.** A corpus of clean PRs measures precision
   and nothing else, and precision alone cannot distinguish a good reviewer from a silent one.
2. **Never tune against clean-only inputs.** Every prompt change that suppresses output scores as
   an improvement, so the optimization gradient points directly at a model that says nothing. The
   defect-bearing half of the corpus is what makes the gradient point somewhere useful.

`bench score` refuses to report precision from a run whose corpus contains no seeded or historical
defects, and prints why. A number that can only go one direction should not be printed at all.

## 2. The corpus

Three sources, in ascending order of realism and cost:

**A. Seeded bugs (bootstrap).** Take merged, well-tested PRs from real repos. Programmatically
inject a defect: invert a condition, drop an `await`, off-by-one a slice bound, remove a lock,
widen an authz check, swap `and`/`or`. Ground truth is exact and free. The weakness is real and
must be stated: seeded bugs are shallow and syntactically local, so they overstate performance on
exactly the class of defect crossexam is supposed to be better at. Necessary for bootstrapping
and for regression-testing changes to the pipeline; never sufficient on its own.

**B. Historical bug-fix pairs (the real signal).** Mine repos for commits that fix a bug (linked
issue, `fix:` conventional commit, revert). The bug-introducing commit becomes the test case; the
fix tells you the ground truth and often ships a regression test that serves directly as the
T1 repro oracle. These are real bugs that survived real human review, which is precisely the
population that matters. Harder to assemble; worth the effort.

**C. Clean PRs (the precision denominator).** Merged PRs with no subsequent fix commit touching
the same lines within N months. Any finding here is *presumed* a false positive. Presumed, not
proven — some are real bugs nobody hit yet, and this is the honest weakness of the whole precision
number. Manually adjudicate a sample each release rather than pretending the presumption is exact.

Target for a first useful bench: 40 seeded, 40 historical, 60 clean, across ≥4 languages
(Python, TypeScript, Go, Rust) and both application and library code. Small enough to build in a
week, large enough that a 10-point precision change is visible.

Corpus lives in `bench/corpus/` as pinned `(repo, base_sha, head_sha, ground_truth)` tuples, not
vendored source. Repos change; pinned SHAs don't.

## 3. Scoring

A published finding matches a ground-truth bug if it cites a file and line range overlapping the
known-buggy region **and** an LLM judge from a family not on the panel agrees the described
mechanism is the same mechanism.

The judge is the weak link and gets treated accordingly: judgments are cached by content hash,
a fixed 15% sample is human-audited every release, and judge-vs-human agreement is itself
published as a bench metric. A bench whose own error rate is unmeasured cannot certify anything.

Line-overlap alone is not enough — a finding that says "this variable is poorly named" on the
buggy line is not a detection, and counting it as one would make the headline number a lie.

## 4. Offline replay

The high-leverage property of the architecture: **stages 3–6 are pure functions over the run
record.** So the expensive part (model calls) runs once per corpus item, and every subsequent
experiment is free:

```
crossexam bench run    --panel panel/balanced.yaml --out runs/     # costs money, once
crossexam bench score  runs/ --config publish.yaml                 # free, instant
crossexam bench sweep  runs/ --param cluster_threshold=0.70:0.95   # free, instant
crossexam bench ablate runs/ --drop-model gemini-3.1-pro           # free, instant
```

Threshold tuning, publication-table changes, cap tuning, and panel ablation (what does the panel
look like without model X?) are all offline sweeps over saved records. Only a change to the
personas, the packet builder, or the panel roster requires re-running the models.

This is why the run record must capture *everything*, including dropped findings and full raw
responses. A record that only kept published findings would make ablation impossible.

## 5. Ablations worth running before v1

These aren't hypotheticals — each one could reasonably come back negative and change the design.

1. **Panel size.** 2 vs. 3 vs. 4 vs. 6 families. Where does recall stop rising? If the answer is
   3, `paranoid.yaml` should be deleted rather than sold.
2. **Refutation on/off.** T2 is the most expensive stage. Precision lift per dollar decides
   whether it survives contact with reality, or becomes opt-in.
3. **Personas vs. generic.** The full persona set vs. every model given "review this diff." If lane
   separation doesn't beat the generic prompt, the persona system is complexity for nothing.
4. **Recusal.** Does excluding the author's family from refutation actually change outcomes, or is
   it a principle with no measurable effect? Run it both ways on Claude-authored seeded bugs. The
   answer is genuinely unknown and it is one of the load-bearing claims in the README.
5. **Symbol slice depth.** 0 vs. 1 vs. 2. Directly trades tokens for precision; the packet builder
   is the most expensive part of the prompt and the easiest to over-build. Run at M1, not M5 —
   its answer sizes the rest of the context work.
6. **Anonymization.** Does telling reviewers "written by Claude" measurably suppress findings? A
   cheap, interesting experiment, and the result is worth publishing regardless of direction.
7. **Context negotiation pass on/off.** Does letting a reviewer request more context beat simply
   giving every reviewer a bigger packet? Compare on **false positives per dollar**, not
   false-positive rate — the two approaches buy similar quality at very different prices, and rate
   alone would hide that. If a bigger uniform packet wins, delete the pass; it is the most complex
   of the three context mechanisms and should have to earn its place.

## 5a. The context-sufficiency question, specifically

Naive multi-model review fails primarily on **absence claims** — "this input is never validated,"
"nothing handles this error" — where the handling exists outside the reviewer's window. This is a
categorical problem, not a tuning problem: absence cannot be established from a subset (see
[`REVIEW_PROTOCOL.md`](REVIEW_PROTOCOL.md#23-absence-claims--the-dominant-false-positive-class)).

The design attacks it three ways, and the bench must attribute credit among them or we will not
know which to keep:

| Metric | Question it answers |
| --- | --- |
| **Absence-claim share of false positives** | Is this really the dominant class, as assumed? If it is not, the T0.5 machinery is over-built and should shrink. |
| **T0.5 kill/reframe rate** | How often does repo-wide falsification search find the context the reviewer lacked? |
| **T0.5 reframe → confirmed rate** | Of absence claims reframed as "the guard exists, does it cover this case," how many become *real* findings? A high rate validates escalating rather than dropping; a near-zero rate means just drop them and save the T2 spend. |
| **Slice-depth precision curve** | Where does added context stop paying? |
| **Negotiation-pass request rate** | How often do reviewers admit the packet was insufficient — and are the ones that ask more accurate than the ones that don't? |
| **Post-repair FP rate** | The bottom line: false-positive rate on absence claims *after* all three mechanisms, vs. a diff-only baseline. This is the number that says whether the context problem is solved. |

The diff-only baseline is worth running deliberately and keeping in the published results. It is
the configuration most people's first experiment lands in, and quantifying how bad it is makes the
case for everything above it.

## 6. Production feedback loop

The bench is a proxy. Real usage is ground truth, and it is free to collect:

- A published finding whose cited lines are **changed in a subsequent commit before merge** →
  weak-positive signal (someone acted on it).
- A finding whose thread is resolved with no code change → weak-negative.
- 👍/👎 reactions on the comment → explicit, sparse, high-quality.
- Fully local, opt-in, stored in the repo's own workflow artifacts. **No telemetry leaves the
  user's infrastructure**, ever. An OSS tool whose premise is "don't trust one vendor with your
  judgment" cannot ship a phone-home.

Accumulated per-model precision from real usage feeds `crossexam panel suggest`, which proposes a
composition tuned to *your* repository — because the right panel for a Rust systems crate and a
React app are not the same panel, and no published benchmark will ever tell you that.
