# ADR-0002: LiteLLM as the default backend, behind a provider interface

**Status:** Accepted (design phase)

## Context

We need to call models across OpenAI, Google, DeepSeek, xAI, Zhipu, Moonshot, and local runtimes.
Options: native SDK per vendor; OpenRouter as the sole gateway; LiteLLM SDK; require a LiteLLM
Proxy deployment.

## Decision

Define a small `Reviewer` protocol. Ship `litellm_backend` as the default implementation.
OpenRouter is a config profile over it (`openrouter/<model>`); LiteLLM Proxy is the same backend
with `api_base` set; local models are the same backend pointed at Ollama/vLLM. Native per-vendor
backends are permitted but not built until something demands them.

## Rationale

- LiteLLM covers 100+ providers behind one call shape, and its Router already implements retries,
  cooldowns, timeouts, and cross-provider fallback — all needed, none interesting to rewrite.
- One backend, three topologies: direct, gateway, proxy. The user's deployment choice is a config
  change, not a code path.
- **The interface is the point.** Making OpenRouter mandatory would replace "captive to one model
  vendor's judgment" with "captive to one gateway's uptime and release cadence," which is a
  peculiar way to honor this project's premise. Requiring the Proxy would put a service deployment
  between a new user and their first review.

## Consequences

- A dependency on LiteLLM's release cadence for new-model support. Acceptable: it moves fast, and
  the escape hatch is a ~100-line native backend against a protocol with two methods.
- Provider-specific features (prompt caching semantics, batch endpoints) are exposed unevenly. The
  capability matrix in `providers/capability.py` and the `--probe` command absorb this.
- Structured-output support varies by provider, so the fallback ladder in
  [`PROVIDERS.md`](../PROVIDERS.md#structured-output-ladder) is mandatory, not an optimization.
