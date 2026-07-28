# Bench corpus

Test cases for [`docs/EVALUATION.md`](../../docs/EVALUATION.md). Entries are **pinned SHA
references plus ground truth**, never vendored source — repos change, pinned commits don't.

## Kinds

| Kind | Ground truth | Weakness |
| --- | --- | --- |
| `seeded` | Exact, free — we injected the defect | Shallow and syntactically local. Overstates performance on the class crossexam is meant to be *better* at. |
| `historical` | The fix commit, and often its regression test | Expensive to assemble. **The population that matters** — these are bugs that survived real human review. |
| `clean` | Presumed none | Presumed, not proven. See the clean-PR trap below. |

## The rule

**No evaluation set may consist only of `clean` entries.** On a clean PR a correct reviewer emits
nothing, so the only reachable outcomes are false positives and silence — recall is unmeasurable
and every output-suppressing change scores as an improvement. `bench score` refuses to report
precision from a corpus with no known defects. See
[`OBSERVED_FAILURES.md`](../../docs/OBSERVED_FAILURES.md#the-test-selection-trap).

## Entry format

```yaml
id: org-repo-NNN
kind: historical | seeded | clean
repo: owner/name
introduced_by: { pr: 535, sha: "<merge sha>" }   # the commit under review
fixed_by:      { pr: 545, sha: "<merge sha>" }   # supplies ground truth
review_target: introduced_by.sha                  # what the panel is shown

ground_truth:
  files: [path/to/file.tsx]
  lines: [[40, 58]]            # the region a finding must overlap
  mechanism: >
    One paragraph. A finding matches only if it describes THIS mechanism —
    line overlap alone is not a detection.
  severity: high
  detectable_from_diff_alone: false   # if false, this entry tests context handling

tests:                          # what capability this entry exercises
  - context: dom-ancestry
  - persona: rendering
  - verification: browser-sandbox
```

`detectable_from_diff_alone: false` is the important field. Entries where the causal code sits
outside the diff are the ones that measure whether the packet builder works, and they should be a
deliberate majority of the `historical` set rather than an accident.

## Entries

| id | kind | tests |
| --- | --- | --- |
| [`ilm-realtor-535`](ilm-realtor-535.md) | historical | string-literal cross-language slicing, `rendering` persona, browser-tier T1, guard-vs-demonstrator |
| [`ilm-realtor-413`](ilm-realtor-413.md) | historical | a **true** absence claim; asserts reframe-don't-drop survives T0.5 |
| [`ilm-realtor-525`](ilm-realtor-525.md) | historical | write→reader slicing; **the first documented false negative** — predecessor review passed this PR |

### Entries that assert a *negative*

`ilm-realtor-413` exists to catch a silent regression: if reframe-don't-drop ever becomes
drop-on-match, a true positive vanishes with no error and no output. Entries whose pass condition is
"this finding still survives stage N" are as important as entries whose pass condition is "this bug
is found," and are much easier to forget to write.
