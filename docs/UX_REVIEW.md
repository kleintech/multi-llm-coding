# Journey review — UX as a second mode

`crossexam review` judges a **diff**. UX quality is not a property of a diff. It is a property of a
**journey**: a sequence of screens and decisions, in the head of a specific person trying to do a
specific thing, over time. No amount of context packet makes a diff reviewer see it.

So this is a second mode, `crossexam journey`, sharing `context/` and the panel machinery but with
a different unit, a different verification story, and a different output. **It is post-v1** — the
correctness thesis has to hold first. This document exists so the design is not invented ad hoc
later.

---

## 1. The honest limit, first

crossexam's value proposition is *verify, don't vote*. Findings survive because a test failed or a
repo-wide search came back empty, not because three models agreed.

**Most UX findings cannot be verified.** "This label is confusing," "a first-timer would hesitate
here," "this flow feels heavy" have no ground truth to check against. For those, the machinery
degenerates to voting — precisely the mode this project argues is weak.

That is not a reason to skip it. It is a reason to **split the output**, because a meaningful
fraction of UX claims *are* mechanically checkable:

| Class | Examples | Settled by |
| --- | --- | --- |
| **Measurable** | Clicks-to-goal, decisions before primary action, missing empty/error/loading state, keyboard-unreachable control, contrast failure, layout shift, dead-end screen with no forward affordance, focus lost after action | **The trace and the DOM.** No model opinion involved. |
| **Judgment** | Label clarity, information hierarchy, whether the primary action is the *right* primary action, tone, "does this feel like one product" | **A human.** Models generate candidates; they cannot adjudicate. |

Measurable findings are gated and automated. Judgment findings are **collected and triaged, never
gated** — they are input to a person's backlog, not a merge check. Conflating the two is how a UX
bot becomes noise.

## 2. The journey packet

A different packet from `ContextPacket`, though it reuses the budget tiering and intent-first rules.

```yaml
journey:
  id: realtor-close-a-deal
  persona: "Solo realtor, 3rd week using the product, on a laptop"
  goal: "Move a deal from Under Contract to Closed"
  entry: "/today"
  steps:                       # captured, not authored
    - screenshot: 001-today.png
      action: "clicked 'Open deal' on the Grace Bennett row"
      elapsed_ms: 340
    - screenshot: 002-deal-detail.png
      action: "scrolled to Parties card; clicked 'Mark closed'"
      …
  variants:                    # the same journey under different conditions
    - empty        # no deals exist
    - error        # the API fails at step 3
    - mobile       # 390px viewport
```

**Capture piggybacks on e2e, exactly as `renders` does.** `ilm-realtor`'s `deals-journeys.spec.ts`
already declares `Landing journey`, `Today → action journey`, `Deals → detail journey`, and
`Deals search journey` — real navigation through real flows. Adding capture at existing
`click()`/`goto()` boundaries yields a journey packet with no new harness, and a Playwright trace
already carries the screenshots, timings, and DOM snapshots the measurable tier needs.

The `variants` field is doing quiet work. Most UX rot lives in states nobody demos: the empty
account, the failed request, the narrow viewport. Capturing them is cheap once the happy path is
scripted, and it is where the measurable tier finds most of its hits.

## 3. Personas, again — different lanes

The persona insight transfers; the lanes do not. Correctness lanes partition *failure modes*. UX
lanes partition *who is on the other side of the screen*:

| Lane | Asks |
| --- | --- |
| `first-run` | No data, no habits, no vocabulary. What is on screen when nothing exists yet? What does the user do first, and is that what we want? |
| `recovery` | Something failed mid-task. Can the user tell what happened, what they lost, and what to do? Is anything unrecoverable that should not be? |
| `returning` | Back after a week. What changed? Where were they? What needs attention, without reading everything? |
| `power` | Tenth time today. Is it still N clicks? What is repeated that could be remembered, batched, or skipped? |
| `one-handed` | 390px, thumb reach, on the move. What is below the fold, what is too small, what assumes hover? |
| `assistive` | Keyboard only, screen reader. Reachable, announced, ordered, escapable. |

Established heuristic frameworks — Nielsen's ten, cognitive walkthrough — work well as lane briefs
because they are *checklists*, which is what turns "rate this" into something falsifiable.

## 4. Prompt discipline

The same rule that governs correctness personas governs these, and it matters more here because
there is no verification stage to catch a lazy answer: **neutral prompts produce compliments.**

Ask for observable, countable, quotable things:

- ✅ "At screen 3, list every element a first-time user could plausibly click to accomplish the
  goal. Which are wrong, and what happens if they click one?"
- ✅ "Count the decisions required before the primary action. Name each."
- ✅ "Quote the exact copy shown when this list is empty. If there is none, say so."
- ✅ "Name every screen where the user can reach a state with no visible way forward."
- ❌ "Is this good UX?" / "How would you improve this?" / "Rate the design."

Findings carry `file:line` where a code location exists, and a **screenshot + element reference**
otherwise — the same evidence requirement as `T0`, adapted. A finding that cannot point at a
specific screen and element is dropped, for the same reason a fabricated citation is.

## 5. Calibration — the substitute for verification

Judgment findings have no oracle. But you can *build* one.

Record a real user attempting the same journeys. Their observed friction — hesitations, wrong
clicks, backtracks, questions asked aloud — is ground truth. Score each model and each lane against
it: **which lanes predicted real friction, and which produced plausible noise?**

This is the bench, transposed. Without it, a UX panel is unfalsifiable opinion at scale, and the
temptation is to act on whatever sounds most confident. With even one recorded session per journey,
lanes and models earn or lose their place on evidence.

It also inverts a limit into an advantage: the pilot user who found
[`ilm-realtor-535`](../bench/corpus/ilm-realtor-535.md) is a better UX detector than any panel.
Models are worth using to decide *what to watch for* and to *process* what you saw — not to replace
watching.

## 6. What this shares with `review`

| Shared | Not shared |
| --- | --- |
| `context/` packet builder, budget tiers, intent-first | The unit — journey, not diff |
| Panel config, families, recusal, cost control | The verification ladder — T1 is meaningless here |
| Persona architecture and prompt discipline | Publication caps — judgment findings go to a backlog, not a PR |
| Evidence requirement and T0-style reference checks | Blocking — journey mode never gates a merge |
| Run record and offline replay | Decorrelation's purpose: coverage of opinion, not consensus |

That last row deserves emphasis. In correctness review, agreement across families is *evidence*. In
judgment-class UX findings it is not — three models sharing a training-data aesthetic agreeing that
a layout is fine tells you about the training data. Here a panel is worth running for **coverage**:
surfacing the union of what different priors notice, for a human to arbitrate.
