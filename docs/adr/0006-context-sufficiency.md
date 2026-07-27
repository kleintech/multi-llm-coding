# ADR-0006: Absence claims are a routing problem, not a context-size problem

**Status:** Accepted (design phase)

**Supersedes nothing. Amends:** ADR-0001 (bounded adaptive depth inside a leaf), and the refuter
cost optimization previously described in `PROVIDERS.md` §6.

## Context

The characteristic failure of naive multi-model review — and the one observed in this project's
own prior experiments — is that reviewers given only a diff report defects that are already
handled elsewhere in the codebase. "This input is never validated," where a validator runs two
frames up. The result is a false-positive rate high enough to make the tool unusable.

The obvious response is "send more context." That response is insufficient, and understanding why
determines the design.

Sort review claims into two kinds:

- **Positive claims** — "line 42 inverts this condition." Verifiable against a partial packet. The
  evidence is present or it is not.
- **Absence claims** — "nothing validates this." A statement about the *entire repository*. No
  packet is the entire repository.

A bigger packet lowers the absence-claim error rate. It cannot fix the category, because there is
always more code outside the window. Treating context size as the solution is chasing an
asymptote while paying linearly in tokens for it — and paying that cost on *every* reviewer, for
*every* finding, to address a problem that affects a subset of both.

## Decision

Treat absence claims as a distinct, routed class rather than trying to out-context them.

1. **Declare the falsifier.** A finding with `asserts_absence: true` must populate `absence_check`
   with the symbols and patterns whose existence would disprove it. Missing → `MALFORMED`.
2. **T0.5 falsification search.** The pipeline runs that search repo-wide, plus a depth-2
   caller-path walk — the search the reviewer was structurally unable to perform. Deterministic,
   free, no model call.
3. **Reframe, don't drop.** A match outside the reviewer's packet does not falsify the finding; a
   guard that exists may still be wrong or cover a different case. The finding is escalated to
   refutation with the found code injected and the question narrowed to "does this guard cover the
   described case?"
4. **A match *inside* the reviewer's packet is `MALFORMED`.** That is a reading failure, not a
   context failure, and it must be scored differently or the metrics conflate two distinct
   problems with two distinct fixes.
5. **Refuters get more context than finders, never less**, and T0.5's findings are injected into
   their packet.
6. **Bounded context negotiation.** Round 1 is two fixed passes: return findings, or request up to
   8 symbols and get exactly one expanded retry.

## Rationale

- **It targets spend where the error is.** Repo-wide search costs nothing and runs only on the
  subset of findings that need it, rather than inflating every packet for every reviewer.
- **It converts an unfalsifiable claim into a falsifiable one.** Requiring the reviewer to name its
  own falsifier is independently valuable as a filter: a model that cannot say what would disprove
  its claim is usually pattern-matching rather than reading this code.
- **Reframing beats dropping.** "The validator exists — does it cover this case?" is a better
  question than the original finding, and dropping on any match would silently discard the real
  bugs where a guard is present but wrong.
- **The refuter inversion was a genuine defect.** Starving the stage whose job is catching
  context-starvation errors guarantees it reproduces them.

## Consequences

- **T0.5 needs repo-wide search infrastructure** (ripgrep + a caller-graph walk), which the packet
  builder needs anyway for slicing. Modest incremental cost.
- **Reviewer output schema grows**, and shallow schemas matter for open-weights panel members
  (`PROVIDERS.md` §4). `absence_check` is one level deep and only required when the flag is set.
- **T2 cost rises** — refuters get larger packets, and reframed absence claims that would otherwise
  have been dropped now enter T2. Partly offset by T0.5 killing malformed ones for free, and by
  prompt caching on the shared prefix. Net direction is unknown until M1 measures it.
- **The negotiation pass weakens ADR-0001's determinism slightly** — call count varies between one
  and two per reviewer. Bounded by a fixed rule, so worst-case cost stays computable before
  dispatch and offline replay still works. It is flagged and ablatable precisely because it is the
  weakest-justified of the three mechanisms.
- **Milestone reordering.** Symbol slice and T0.5 move from M5 to M0; slice depth and negotiation
  ablations move to M1. Context handling stops being a refinement and becomes a precondition.
- **Open risk:** reviewers may under-declare `asserts_absence`, phrasing an absence claim as a
  positive one to route around the requirement, and the flag is self-assigned. Mitigation is a
  cheap classifier over finding text as a cross-check at M2 if M1 shows the rate is material.
