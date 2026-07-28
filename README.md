# crossexam

**Adversarial, cross-vendor code review.** A panel of LLMs from *different* vendors reviews a
diff independently, then attacks each other's findings. Only findings that survive — ideally by
being demonstrated with a failing test — reach the pull request.

> **Status: design phase.** This repository currently contains the design only. No implementation
> has been written yet. See [`docs/ROADMAP.md`](docs/ROADMAP.md) for the build order.

---

## Why

If the model that writes your code is also the model that reviews it, the review inherits the
author's blind spots. Same pretraining corpus, same post-training, same taste, same failure modes.
A model is a poor adversary to itself.

The fix is not "more review." It is **decorrelated** review: reviewers drawn from different weight
lineages, given different mandates, kept blind to each other, and kept blind to who wrote the code.

The second half of the problem is that LLM reviewers are prolific liars. Ask five frontier models
to review a diff and you will get sixty findings, of which perhaps six matter. A review bot with a
40% false-positive rate gets muted within a week. So crossexam is built precision-first: every
finding must survive a verification gauntlet before anyone sees it, and where a claim is
*checkable*, it gets checked rather than voted on.

Those false positives do not have one shape, which is the main thing a small corpus from this
project's predecessor established ([`OBSERVED_FAILURES.md`](docs/OBSERVED_FAILURES.md)). At least
four distinct mechanisms produce them, and they need four different answers:

| The model… | Fix |
| --- | --- |
| claims something is **missing** that exists outside its window | make it declare what would disprove the claim, then run that search repo-wide ([ADR-0006](docs/adr/0006-context-sufficiency.md)) |
| infers the **wrong purpose** for the change — reads step 1 of a migration as violating the design it implements | put change intent first in the packet, always |
| **fails to trace** context it was already given | no context fix exists; only cross-family refutation |
| dresses a **style nit** as a correctness defect to get past a filter | never let a model self-certify past a filter |

Note what the second and third rows mean: "send more context" is not the answer to most of this,
and for the third row more context is actively counterproductive.

## The one-paragraph design

Orchestration is deterministic Python, not an agent loop. The pipeline builds a **review packet**
from the diff, fans it out to a **panel** of (model × persona) reviewers that cannot see each
other, collects **structured findings**, clusters them, and pushes each cluster through a
**verification ladder**: does the cited code exist → can a sandbox reproduce it → can two rival
models refute it. Survivors are ranked and posted as inline PR comments, capped hard. Everything
else is collapsed into a summary or dropped. Provider access sits behind one interface with
[LiteLLM](https://docs.litellm.ai/) as the default backend, so the panel can be hosted APIs,
an OpenRouter key, your own LiteLLM Proxy, or local models on your own box.

## Design documents

| Document | What's in it |
| --- | --- |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System shape, pipeline stages, packet construction, data flow, repo layout |
| [`docs/REVIEW_PROTOCOL.md`](docs/REVIEW_PROTOCOL.md) | Personas, finding schema, verification ladder, adjudication rules, publication table |
| [`docs/PROVIDERS.md`](docs/PROVIDERS.md) | Provider abstraction, LiteLLM/OpenRouter/proxy/local, panel config, structured-output fallback, cost control |
| [`docs/GITHUB_ACTION.md`](docs/GITHUB_ACTION.md) | Workflow design, the fork-PR security problem, idempotency, spend caps |
| [`docs/EVALUATION.md`](docs/EVALUATION.md) | How we prove the panel works: the bench, metrics, reviewer reputation |
| [`docs/SECURITY.md`](docs/SECURITY.md) | Prompt injection, untrusted content fencing, sandbox model, secrets |
| [`docs/OBSERVED_FAILURES.md`](docs/OBSERVED_FAILURES.md) | **Real false positives from the predecessor system, classified.** The only evidence-backed page here. |
| [`docs/UX_REVIEW.md`](docs/UX_REVIEW.md) | Journey review as a second mode — why UX needs a different unit, and where verification stops working |
| [`docs/DELEGATION.md`](docs/DELEGATION.md) | Whether this architecture can also hand work *out* to specialist agents, and why the packet is extracted |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | Milestones from M0 to v1.0 |
| [`docs/adr/`](docs/adr/) | Architecture decision records for the load-bearing choices |

## Design principles

1. **Decorrelation is the product.** Two reviewers from the same weight lineage count as one voice.
2. **Ground truth beats consensus.** A failing test outranks any number of model votes.
3. **Precision over recall.** A missed bug costs one bug. A false positive costs the tool's credibility.
4. **Deterministic orchestration.** The pipeline is a state machine; LLMs only occupy the leaves.
   Same inputs, same panel, same seed → same run structure, reproducibly.
5. **No vendor is privileged.** Anthropic models are participants, not the runtime. The tool must
   run end to end with zero Anthropic keys configured.
6. **The author recuses.** Whoever wrote the code does not get a vote on whether it is correct.

## Non-goals

- Not a linter, formatter, or type checker. Those are cheaper, faster, and already correct — run
  them first and feed their output to the panel as context.
- Not a code generator. crossexam reviews; it does not fix. (Suggested patches are a possible
  post-v1 addition, gated behind human application.)
- Not a benchmark harness for models. See `docs/EVALUATION.md` for the narrow, purpose-built
  bench we do need.
- Not a replacement for human review. It is a filter that makes human review cheaper.

## License

Apache-2.0 (intended; not yet applied).
