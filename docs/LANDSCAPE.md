# Landscape — what exists now (October 2026)

Three months after this design was written, a lot of what it describes has shipped elsewhere. This
page maps existing tools against the design so we build only what is still missing.

**Short version:** the *panel* (send one diff to Claude, GPT, and Gemini and combine the answers) is
now a commodity. Nobody we found **checks** the findings: they vote. Verification, model-family
recusal, a real-bug benchmark, and doc↔code audit are still gaps.

---

## 1. What exists

### Cross-vendor panels (the closest matches)

| Tool | What it does | Fit |
| --- | --- | --- |
| **[Star Chamber](https://blog.mozilla.ai/the-star-chamber-multi-llm-consensus-for-code-quality/)** (Mozilla.ai, Apache-2.0, [`star-chamber` 0.9.0 on PyPI](https://pypi.org/project/star-chamber/)) | Sends code to several providers in parallel. Each returns structured JSON (severity, `file:line`, category). Findings are grouped by location and category and labeled **consensus / majority / individual**. Optional anonymous multi-round debate. Reads `CLAUDE.md` / `ARCHITECTURE.md`. CLI, Python library, and Claude Code skill. Direct keys, OpenRouter, or local models. | **Closest match to our M0–M2.** Same JSON-schema and grouping approach. No GitHub Action, no verification, explicitly advisory. |
| **[PAL MCP](https://github.com/BeehiveInnovations/pal-mcp-server)** (formerly Zen MCP, Apache-2.0, ~11.8k stars) | MCP server that lets Claude Code, Codex CLI, or Cursor call other models mid-session. Tools: `codereview`, `consensus` (stance-steered debate), `precommit`, `challenge`, and `clink` (bridges to Gemini/Codex CLIs). | **The interactive answer** — "keep my Claude Code experience but get other models' opinions." Not a CI/PR tool. |
| **[PR-Agent](https://github.com/marketplace/actions/the-pr-agent)** (Qodo, now community-owned, Apache-2.0) | Self-hosted PR review via GitHub Action, any model through LiteLLM. | Mature PR surface and model plumbing. One model per run, not a panel. |
| [Git AutoReview](https://gitautoreview.com/), [CodeMouse](https://codemouse.ai/blog/ai-code-review-for-github-scaling-quality-with-multi-model-consensus), claude-octopus | Commercial or small tools running Claude + GPT + Gemini on a PR and comparing. | Same idea, less open, unverified claims. |

### Single-vendor reviewers (strong, but not cross-vendor)

- **Claude Code Review** — several Claude agents per PR, a verification step before posting, comment-only and never blocking ([Help Net Security](https://www.helpnetsecurity.com/?p=360979)). Reported GA on Oct 15, 2026 for Team/Enterprise; we could not confirm this from a primary source.
- **OpenAI Codex Code Review** — launched at DevDay (Sep 29, 2026), GA on GitHub ([recap](https://codex.danielvaughan.com/2026/10/08/openai-devday-2026-recap-codex-cloud-agents-view-gpt61-sol-code-review-security-cloud/)).
- **Cursor Bugbot, Greptile, CodeRabbit, Graphite, Qodo 2.0** — established commercial reviewers.

All share the blind spot the project exists for: one vendor reviewing, often code written by the
same vendor.

### Doc↔code / architecture audit

No established tool. The landscape is **patterns, not products**: ADR conformance scripts in CI,
fitness functions, `AGENTS.md` constraints ([O'Reilly](https://www.oreilly.com/radar/architectural-guardrails-for-ai-generated-code/),
[Codex ADR article](https://codex.danielvaughan.com/2026/04/28/codex-cli-architecture-decision-records-adr-automated-governance/)).
The O'Reilly piece states the gap: LLM reviewers have no durable access to a team's recorded decisions.

### UX journey review

Nothing found.

## 2. What the benchmarks say about precision

Our central bet — precision is the problem — is confirmed independently:

- An [independent 2026 benchmark](https://entelligence.ai/code-review-benchmark-2026) puts Bugbot,
  Greptile, and CodeRabbit at **25–35% precision**, i.e. most comments are wrong or unhelpful.
  Graphite reaches 66.7% precision only by dropping to 7.5% recall.
- A July 2026 study of 31,073 CodeRabbit comments found developers **rejected 56.3%**, with false
  positives the top reason ([arXiv 2607.07980](https://arxiv.org/pdf/2607.07980)).
- Benchmarks disagree heavily with each other and with vendor claims
  ([comparison](https://medium.com/@codzee.io/we-compared-the-public-ai-code-review-benchmarks-from-2026-methodology-datasets-metrics-and-269dd35637a6)),
  which is the argument for our own corpus of real bugs from our own repo.

## 3. Gap analysis

| Design element | Exists elsewhere? | Status |
| --- | --- | --- |
| Cross-vendor panel, structured findings, grouping | ✅ Star Chamber, PAL, others | **Don't build.** |
| Provider plumbing (LiteLLM / OpenRouter / local) | ✅ LiteLLM, any-llm, PR-Agent | **Don't build.** |
| Interactive "ask another model" from Claude Code | ✅ PAL MCP, Star Chamber skill | **Don't build.** |
| PR comment bot | ✅ PR-Agent, every vendor | **Don't build** the surface; reuse one. |
| Debate between models | ✅ Star Chamber `--debate`, PAL `consensus` | **Don't build.** |
| **Citation check** (T0) — does the cited code exist | ❌ not found | Small, high value. |
| **Absence-claim falsification** (T0.5) — repo-wide search before believing "X is never validated" | ❌ not found | Our design's distinctive piece. |
| **Execution verification** (T1) — prove a finding with a failing test | ❌ not found in cross-vendor tools | Gap. Hard. |
| **Targeted refutation** (T2) — rival family tries to disprove a finding | ⚠️ partial (debate exists; not per-finding, not biased to refute) | Gap. |
| **Model-family counting and author recusal** | ❌ Star Chamber counts *providers*; nobody recuses the author | Small, cheap, and it is the core claim of "decorrelated." |
| **Context beyond the diff** (write→reader, string-literal, intent-first) | ⚠️ partial (Star Chamber reads `CLAUDE.md`, `--context-file`) | Gap, backed by our corpus. |
| **Real-bug benchmark from our repo** | ❌ public benchmarks exist, none for us | Gap, and it is how we'd judge any tool, including ours. |
| **Doc↔code drift audit** | ❌ patterns only | Clearest product gap. |
| **UX journey review** | ❌ | Gap, post-v1 as before. |

## 4. Recommendation

**Stop designing a review tool from scratch. Build a verification layer and an audit mode on top of
what exists.**

1. **Measure first (days, not weeks).** Run Star Chamber against the three `bench/corpus/` entries
   — `ilm-realtor-535`, `-413`, `-525`. They have known answers. If Star Chamber finds them with
   acceptable noise, most of the remaining design is unnecessary. If it doesn't, we know exactly
   which missing piece is responsible.
2. **Build the verification layer as a filter over someone else's findings.** Input: structured
   findings JSON (Star Chamber's format is already close to ours). Output: the same findings,
   marked confirmed / refuted / dropped, using T0 citation checks, T0.5 falsification search,
   family recusal, and per-finding refutation. Small, testable, and it attacks the 25–35%
   precision problem directly. Offer it upstream to Star Chamber if they want it.
3. **Build audit mode** (`AUDIT_MODE.md`). Nobody ships doc↔code drift checking, and the target
   repo has 24 ADRs waiting for it.
4. **Use PAL MCP today** for the interactive "ask Gemini/GPT from inside Claude Code" experience.
   No build required.
5. **Reuse an existing PR surface** (PR-Agent, or a thin Action around Star Chamber + the
   verification layer) instead of M4's bespoke GitHub Action.

What remains unique to this project: **verification, recusal, the corpus, and audit mode.** The
rest is a solved problem.

*Caveat: most sources here are vendor blogs or aggregators, and benchmark numbers conflict. The
recommendation's first step exists precisely so we stop relying on them.*
