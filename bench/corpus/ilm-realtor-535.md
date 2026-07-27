# ilm-realtor-535 — snooze popover painted behind the next row

**Kind:** `historical` · **Status:** ground truth recorded; PR diffs and merge SHAs not yet pinned
(private repo, pending fetch).

```yaml
id: ilm-realtor-535
kind: historical
repo: kleintech/ilm-realtor
introduced_by: { pr: 535, title: "feat(today): per-row Done / Clear / Snooze on the What Needs You feed" }
fixed_by:      { pr: 545, title: "fix(today): snooze popover no longer painted behind the next row" }
promoted_by:   { pr: 549 }
review_target: introduced_by
severity: high            # feature is unusable, not merely ugly
reported_by: pilot user   # reached production
detectable_from_diff_alone: false
```

## The defect

Clicking **Snooze** on the `/today` feed opened a popover that rendered *behind the next row*,
making the duration impossible to select.

Each `/today` row carried a persistent `transform: translateY(0)`, left over from a slide-up
entrance animation. **A non-`none` transform creates a stacking context.** The snooze dialog's
`z-index: 20` could therefore only order *within its own row's* stacking context — it had no way
to rise above sibling rows painted later in document order.

The fix lifts the whole row while its dialog is open (`position: relative` + `z-30`), driven by an
`onOpenChange` callback in `RowActions`.

## Why this entry earns its place

**The causal code is not in the diff.** PR #535 added a popover with a perfectly ordinary
`z-index`. The `transform` that breaks it predates the PR. Nothing in the diff is wrong on its own
terms; the defect is an *interaction* between new code and an unchanged ancestor. A reviewer shown
only the diff cannot find this, no matter how capable it is — the evidence is absent, not
overlooked.

**The needed context is of a kind the design does not currently model.** crossexam resolves two
relationships: symbol → definition, and function → callers. Both follow the **module graph**. The
context required here is *"which elements wrap this one at render time, and what styles do they
apply"* — the **composition tree**, which is not the module graph and has no symbol to look up.
`RowActions` does not import the row's CSS class. No amount of depth-2 slicing reaches it.

**Knowledge is not the bottleneck, so the test is clean.** "Transform creates a stacking context"
is well-documented CSS that every frontier model knows cold. A model that misses this bug missed
it for lack of *context or attention*, never for lack of knowledge — which is exactly the variable
the bench is trying to isolate.

**It survived human review** and was found by a pilot user in production. That is the population
`historical` entries exist to sample, and it is why they beat seeded bugs.

## What it should measure

| Capability | Pass condition |
| --- | --- |
| **Packet: DOM ancestry** | Does the packet surface the row's `transform` to a reviewer looking at the popover? Baseline expectation: **no**, with the current design. This entry should *fail* at M0 and pass once composition-tree slicing lands. |
| **Persona coverage** | Does any reviewer own rendering/layout? Baseline: **no** — none of the seven personas covers stacking contexts, overflow clipping, portals, or focus containment. |
| **Verification reach** | Can T1 confirm it? jsdom cannot — it has no layout or paint. Needs a real browser (`elementFromPoint` at the popover's coordinates, or a visual snapshot). Tests whether the sandbox has a browser tier. |
| **Refutation behavior** | If a reviewer *does* raise it, do refuters uphold it? Stacking-context reasoning is verifiable from spec, so refuters should not be able to talk it down. A refuted true positive here is a serious signal about T2. |

## Expected trajectory

An honest bench entry predicts its own result. This one should be **missed at M0** and become
findable only after the packet learns composition-tree context. If it is found at M0, the
diff-only hypothesis is weaker than believed and that is worth knowing. If it is still missed
after composition slicing lands, the problem is persona coverage, not context — and the two
outcomes call for different work.

## To finish this entry

1. Pin merge SHAs for #535 and #545.
2. Record exact `files` and `lines` for the ground-truth region from #545's diff.
3. Extract the fix's assertion, if any, as a candidate T1 oracle. If #545 shipped no test — likely,
   since this needs a browser — note that: *"the fix for a production bug shipped without a
   regression test"* is itself a finding the `tests` persona should have made.
