# ilm-realtor-535 — snooze popover painted behind the next row

**Kind:** `historical` · **Status:** verified against both diffs. Merge SHAs still to pin.

```yaml
id: ilm-realtor-535
kind: historical
repo: kleintech/ilm-realtor
introduced_by: { pr: 535, title: "feat(today): per-row Done / Clear / Snooze on the What Needs You feed" }
fixed_by:      { pr: 545, title: "fix(today): snooze popover no longer painted behind the next row" }
promoted_by:   { pr: 549 }
review_target: introduced_by
severity: high            # feature unusable, not merely ugly
reported_by: pilot user   # reached production; tracked as ClickUp 86badh0hx
detectable_from_diff_alone: false

ground_truth:
  files: ["src/app/(realtor)/today/page.tsx"]
  mechanism: >
    A snooze popover is added with `absolute … z-20` inside a row whose class
    constant applies a CSS animation with fill-mode `both`, ending on
    `transform: translateY(0)`. The retained transform makes every row a
    stacking context, so the popover's z-index can only order within its own
    row and is painted under later sibling rows.
```

## The causal chain, verified

Three hops across two files. **Only step 4 is in the diff.**

| # | Where | What | In #535's diff? |
| --- | --- | --- | --- |
| 1 | `today/page.tsx:80` | `const ENTER = "motion-safe:animate-[slide-up-in_0.36s_cubic-bezier(0.16,1,0.3,1)_both]"` | **No** — pre-existing |
| 2 | `app/globals.css:505` | `@keyframes slide-up-in { from { …translateY(12px) } to { …translateY(0) } }` | **No** — different file, untouched |
| 3 | — | fill-mode `both` retains the `to` state ⇒ every row keeps a non-`none` transform ⇒ a stacking context per row | inference, not code |
| 4 | `today/page.tsx` | `+ className="absolute right-0 top-full mt-2 z-20 …"` | **Yes** — the whole diff |

The added line is **correct in isolation**. `z-20` on an absolutely-positioned popover is exactly
what you would write. Everything that makes it wrong lives outside the diff.

The decisive token is `_both` — `animation-fill-mode: both` — inside a Tailwind arbitrary-value
string. Drop that one word and the transform is discarded when the animation ends, no stacking
context forms, and there is no bug.

Verified by grepping #535's patch for `ENTER` and `slide-up-in`: **zero matches**, added or
context. The only hit in the entire `page.tsx` patch is the `z-20` line above.

## Correction to this entry's first draft

The first version of this entry claimed the required context was the **composition tree** — "what
wraps this element at render time" — on the assumption that the transform sat on an ancestor
component elsewhere. **That was wrong.** `ENTER` is defined at `page.tsx:80` and consumed by
`TodayRow` at `page.tsx:393` — the *same file*. A `full`-tier packet including whole changed files
would surface it.

The real gap is narrower and more interesting: **`globals.css`**. A reviewer that sees
`animate-[slide-up-in_…_both]` still cannot know the animation terminates on a transform without
the `@keyframes` body, which lives in a CSS file the diff neither imports nor mentions. Reaching it
requires resolving an **animation name embedded in a string literal** to a `@keyframes` block in
another language.

That is not a code symbol. tree-sitter/ctags will not find it, LSP will not find it, and the
import graph does not contain the edge. The general class — *cross-language references keyed by
string literals* — also covers Tailwind classes, CSS modules, i18n keys, DI tokens, SQL in
template strings, and route names.

Kept as a lesson, not deleted: reasoning about which context a bug needs, without the diff in
front of you, produces a confident and wrong answer. Which is the thing this whole project is
about.

## What it measures

| Capability | Pass condition | Baseline expectation |
| --- | --- | --- |
| **Whole-file context** | Packet includes `page.tsx` in full so `ENTER` is visible | Passes at `full`/`standard` tier; **fails at `focused`** |
| **String-literal cross-language slicing** | `globals.css`'s `@keyframes slide-up-in` reaches the packet | **Fails** — no mechanism proposed resolves this |
| **Persona coverage** | Some reviewer owns stacking/paint order | Fails until `rendering` ships |
| **T1 reach** | A repro that fails before the fix | **Impossible in jsdom.** See below. |
| **Refutation** | Refuters uphold it — the mechanism is spec-verifiable | Untested |

## The T1 finding — a prediction that came back wrong

This entry predicted #545 would ship **no** regression test, since catching the bug needs a
browser. **It shipped one.** `page.test.tsx` gained a 31-line test — but read what it asserts:

```js
// jsdom has no layout engine, so we assert the elevation
// marker that drives the z-index, on the correct row only.
expect(document.querySelectorAll('[data-row-elevated="true"]')).toHaveLength(1);
```

It asserts `data-row-elevated`, an attribute **the fix introduced**. So it is a **regression
guard** — it proves the fix is still present — and not a **bug demonstrator** — it cannot show the
bug exists. On #535's tree the attribute does not exist and the occlusion is invisible to jsdom, so
**no fails-before-fix test is writable in jsdom for this bug at all.**

This is a sharper version of the T1 hazard already in the design, and a plausible path to its worst
outcome. A reviewer asked for a repro here will likely write a jsdom test asserting *something* —
and whatever it asserts will pass on the buggy code. T1 then returns `REFUTED_EXECUTABLE`, the
highest-trust label in the system, killing a true positive that two models had correctly found.

The mitigation is not "trust jsdom less." It is that a runtime which cannot observe the claimed
effect must return `t1_out_of_reach` and never a verdict — see
[`REVIEW_PROTOCOL.md`](../../docs/REVIEW_PROTOCOL.md#t1-reach-what-execution-cannot-settle).
This entry is the regression test for that rule.

## Expected trajectory

Missed at M0 (`focused` tier, no `rendering` persona, no CSS resolution). Findable once whole-file
context plus a `rendering` reviewer land — a model that can see both `ENTER` and the popover has a
real shot, since "fill-mode `both` retains the transform" is standard CSS knowledge. Reliable only
with `globals.css` in the packet.

If a model finds it **without** `globals.css`, that is worth examining rather than celebrating: it
likely pattern-matched "animation + z-index" rather than reasoning from the keyframe, and would
fire the same way on a correct diff.

## Remaining

1. Pin merge SHAs for #535 and #545.
2. Record exact ground-truth line ranges from #545's `page.tsx` hunks.
3. Write the browser-tier repro this bug needs, as the first `browser` sandbox test case.
