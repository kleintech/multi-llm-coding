# Provider layer

## 1. The abstraction

One protocol. Everything else is a backend.

```python
class Reviewer(Protocol):
    id: str                  # "gpt-5.6-sol@contract"
    family: str              # "openai.gpt-5"   ← the decorrelation unit
    max_tier: PacketTier     # full | standard | focused

    async def review(self, packet: ReviewPacket, persona: Persona) -> ReviewerResult: ...
    async def refute(self, packet: ReviewPacket, finding: Finding) -> Refutation: ...
```

`ReviewerResult` carries findings plus call metadata: tokens, cost, latency, retries, which rung
of the structured-output ladder succeeded, and the raw response. All of it lands in the run record.

Backends implement `Reviewer`:

| Backend | When |
| --- | --- |
| `litellm_backend` | **Default.** Direct to any provider, or to a LiteLLM Proxy by setting `api_base`. |
| `openrouter` | A config profile over `litellm_backend` (`openrouter/<model>`), for one-key setup. |
| `local` | Ollama / vLLM / llama.cpp, also via LiteLLM's OpenAI-compatible path. |
| `anthropic_native`, `openai_native`, … | Escape hatches for vendor features LiteLLM doesn't expose yet. Not built unless needed. |

Why LiteLLM as the default rather than direct SDKs: it covers 100+ providers behind one call
shape, and its Router already implements retries, cooldowns, timeouts, and cross-provider fallback
— all of which we need and none of which is interesting to write again.
([LiteLLM docs](https://docs.litellm.ai/), [Router](https://docs.litellm.ai/docs/routing))

Why it is a *default* and not a *requirement*: this is an OSS project whose whole premise is not
being captive to one vendor's judgment. Being captive to one gateway's release cadence instead
would be an odd way to honor that. The `Reviewer` protocol is small enough that a contributor who
needs a vendor feature LiteLLM lags on can write a 100-line native backend.

## 2. Model families

The decorrelation unit. Two reviewers in the same family count as **one voice** in every quorum,
refutation eligibility check, and independent-family count.

```yaml
families:
  openai.gpt-5:     [gpt-5.6-sol, gpt-5.6, gpt-5.6-mini, gpt-5.5*]
  anthropic.claude: [claude-opus-5, claude-sonnet-5, claude-fable-5, claude-haiku-4-5]
  google.gemini:    [gemini-3.1-pro, gemini-3.5-flash]
  deepseek:         [deepseek-v4-pro*, deepseek-v4*]
  zhipu.glm:        [glm-5.2*]
  qwen:             [qwen3.7*]
  moonshot.kimi:    [kimi-k2.6*]
  xai.grok:         [grok-4*]
  minimax:          [minimax-m3*]
```

Families are declared, not inferred, and shipped as a maintained file (`families.yaml`) with a
glob-matching resolver. Two judgment calls worth stating:

- **Distillation is not independence.** A small model distilled from a large one in the same lab
  is the same family. When a model's lineage is genuinely unclear — an open-weights model
  fine-tuned from another lab's base — the rule is to merge the families rather than split them.
  Wrongly merging costs a little diversity; wrongly splitting silently inflates every
  independent-family count in the tool, which is worse.
- **An unknown model gets its own family and a warning.** Never silently folded into an existing
  one.

## 3. Panel configuration

`panel.yaml`, the file users actually edit.

```yaml
version: 1

author_family: anthropic.claude    # recused from refutation; or "auto" to infer from trailers

defaults:
  backend: litellm
  temperature: 0
  timeout_s: 180
  max_output_tokens: 8000
  retries: 2

reviewers:
  - model: gpt-5.6-sol
    personas: [contract, concurrency, failure]
    tier: full

  - model: gemini-3.1-pro
    personas: [contract, security, compat]
    tier: full

  - model: deepseek-v4-pro
    personas: [concurrency, security, tests]
    tier: standard

  - model: glm-5.2
    backend: local                 # ollama on the workstation
    api_base: http://192.168.1.20:11434
    personas: [failure, intent]
    tier: focused

refutation:
  refuters_per_finding: 2
  tiebreak_on: [critical, high]

budget:
  max_usd_per_run: 2.00
  max_usd_per_pr: 6.00
  on_exceed: truncate_panel        # truncate_panel | fail | proceed_and_warn

sandbox:
  enabled: true
  image: ghcr.io/kleintech/crossexam-sandbox:1
  timeout_s: 60
  network: none

publish:
  max_inline: 10
  max_inline_per_file: 3
  max_advisory: 5
  block_on: never                  # never | confirmed_executable_critical
```

Presets ship for people who don't want to think about it: `panel/cheap.yaml` (~$0.15/PR),
`panel/balanced.yaml` (~$0.60/PR), `panel/paranoid.yaml` (~$3/PR), `panel/local.yaml` ($0, local
models only, for private code that must not leave the building).

### Validation at load

Refuse to start on: fewer than two families; an author family that is the only family; a persona
with no reviewer assigned; a model with no resolvable family; a budget ceiling below the estimated
floor cost of the configured panel. Failing loudly at config load is much better than producing a
confident-looking review from a panel that was structurally incapable of disagreeing with itself.

## 4. Structured-output ladder

Providers differ in how strictly they can be made to emit a schema, and the differences move.
Rather than a per-model special case, one descending ladder per call, with the successful rung
recorded:

1. **Native structured output / strict JSON schema** — response is schema-guaranteed.
2. **Tool calling** — expose a single `submit_findings` tool; take the arguments.
3. **JSON mode** — valid JSON guaranteed, schema not; validate with pydantic.
4. **Prompted JSON + tolerant parse** — fenced-block extraction, trailing-comma and smart-quote
   repair, then validate.
5. **One repair round-trip** — hand back the validation error, ask for a corrected object. Once.

Failure after rung 5 drops that reviewer from the run with reason `schema_failure`. It does not
fail the run.

A static capability matrix picks the entry rung (`capability.py`), with a `--probe` command that
tests a configured panel against a live endpoint and writes an override file. Capabilities drift
weekly; a hand-maintained table that is wrong is worse than a cheap probe that is right.

One constraint on the schema itself: it must stay **flat and shallow**. Deeply nested schemas
degrade output quality on mid-tier and open-weights models — models that would otherwise be
useful, cheap panel members. The `Finding` object in
[`REVIEW_PROTOCOL.md`](REVIEW_PROTOCOL.md#2-finding-schema) is one level deep except for `repro`,
deliberately.

## 5. Reaching the models

Three topologies, all supported; the config differs only in `api_base` and key handling.

**A. Direct to providers.** Each vendor's key in the environment. Lowest latency and cost, gives
access to provider-specific features (prompt caching, batch pricing). Costs you N secrets in the
GitHub Actions secret store.

**B. OpenRouter.** One key, every model, `openrouter/<model>` model strings. Best for getting
started and for reaching models you don't have accounts with. As of July 2026 OpenRouter passes
through provider token rates without markup, charging a 5.5% fee on credit purchases (minimum
$0.80; 5.0% flat for crypto), and supports BYOK — your own provider keys routed through their
platform, free for the first 1M requests/month, then 5% of the equivalent platform cost.
([pricing breakdown](https://ofox.ai/blog/openrouter-pricing-hidden-markup-breakdown-2026/),
[BYOK terms](https://flo2.com/blog/openrouter-pricing-explained)) The real cost is a single point
of failure and one more party seeing your source. Verify current terms before relying on them.

**C. LiteLLM Proxy.** You run the gateway; the Action and your local CLI both point at it with one
virtual key. You get central spend limits, per-model logging, caching, and OpenTelemetry traces,
and provider keys never enter GitHub. Cost: you now operate a service, and if it is on your home
PC it must be reachable from GitHub's runners — a tunnel with an auth layer, not an open port.

**Recommendation for your setup.** Start with (B) for the first working panel, because one key and
zero infrastructure gets you to a real review fastest. Move to (C) once the panel composition
stabilizes and you want spend caps and caching — and at that point the proxy is also the natural
place to expose the local open-weights models on your PC to CI runs, which is the highest-leverage
part of the whole topology: a free, genuinely independent weight lineage in every review.

## 6. Cost control

**Estimate before dispatch.** Packet token count is known exactly; output is capped by
`max_output_tokens`. Worst-case run cost is therefore computable before the first call. If it
exceeds the ceiling, `on_exceed` decides: `truncate_panel` drops reviewers from the bottom of a
priority order until it fits (and says so in the summary), `fail` refuses, `proceed_and_warn`
proceeds.

**Where the money actually goes.** Round 1 is N_reviewers × packet. Refutation is
N_findings × 2 × packet, and *that* is the term that runs away — thirty raw findings on a large
packet is sixty full-context calls. Three mitigations, in order of importance:

1. T0 runs before T2 and is free. Every fabricated finding it kills is two refutation calls saved.
2. Refuters get a **reduced packet**: the finding, the cited file, and its symbol slice — not the
   whole diff. Refutation is a narrow question.
3. Hard cap on findings entering T2 (default 25, ranked by severity × independent families).
   Overflow is recorded and reported, not silently dropped.

**Prompt caching.** The packet is a large shared prefix across every reviewer using the same tier.
Providers with prompt caching (Anthropic, OpenAI, Gemini implicit caching) cut the dominant cost
term substantially — OpenAI cached input runs $0.50/M against $5.00/M standard for `gpt-5.6-sol`,
a 10× reduction on the packet, which is most of the bill. Ordering the prompt as
`[packet][persona][schema]` rather than `[persona][packet][schema]` is what makes the prefix
shared, and is worth getting right early. Note this pulls against topology (B): OpenRouter's
caching behavior varies by upstream provider, so it is another reason the proxy or direct access
wins once you care about cost.

**Batch pricing.** Gemini batch is 50% off. Reviews aren't latency-critical the way an IDE
completion is — a PR review that lands in 20 minutes is fine. A `--batch` mode for scheduled
sweeps over open PRs is a natural post-v1 addition.

### Reference prices (verified July 2026 — drift constantly, treat as illustrative)

| Model | $/M in | $/M out | Context | Notes |
| --- | --- | --- | --- | --- |
| `gpt-5.6-sol` | 5.00 | 30.00 | 1.05M | $10/$45 above 272K input; cached input $0.50 |
| `gemini-3.1-pro` | 2.00 | 12.00 | — | $4/$18 above 200K; batch 50% off |
| `grok-4` | 3.00 | 15.00 | — | |
| `deepseek-v4-pro` | 0.43 | 0.87 | 1M | Open weights; strong price/performance |
| `glm-5.2` | local | local | — | Best open-weights on SWE-bench Pro (62.1%) |

Sources: [OpenAI](https://developers.openai.com/api/docs/models/gpt-5.6-sol),
[Gemini](https://benchlm.ai/google/api-pricing),
[comparisons](https://benchlm.ai/compare/deepseek-v4-pro-max-vs-grok-4-5),
[SWE-bench Pro](https://www.morphllm.com/swe-bench-pro). A `crossexam prices --refresh` command
that pulls current rates from the LiteLLM model-cost map keeps estimates honest without hardcoding
a table that rots.

## 7. Choosing a panel

Benchmark scores select *candidates*; the bench in [`EVALUATION.md`](EVALUATION.md) selects the
*panel*. They are different questions — the best reviewer is not the highest-scoring model, it is
the model that finds bugs the rest of your panel misses. A model that scores 80% and is wrong in
different places than your 96% model is worth more to a panel than a second 96% model that is
wrong in the same places.

Practical starting heuristics:

- **Two strong hosted models from different vendors** carry the run. As of July 2026 that is
  `gpt-5.6-sol` and `gemini-3.1-pro` for a Claude-authored codebase.
- **One cheap high-volume model** for the noisy lanes (`tests`, `intent`) where recall matters
  more than precision, because T0/T2 will filter it. `deepseek-v4-pro` at $0.43/M makes this
  nearly free.
- **One local open-weights model** as a permanent, free, independent voice. Also your fallback
  when a vendor has an outage.
- **The author's family may find, but never judges.** Claude in round 1 is useful — it knows the
  codebase conventions. Claude in refutation is defending its own work.
