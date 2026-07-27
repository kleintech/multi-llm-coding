# Roadmap

Ordered so that the riskiest assumptions are tested first and cheapest. There are two:

1. **Context sufficiency.** That a reviewer can be given enough of the surrounding codebase to
   judge a diff without false-positiving on things handled elsewhere. This is the failure mode
   that kills naive multi-model review, and it is addressed by the packet's symbol slice, the
   context negotiation pass, and T0.5 falsification search. See
   [ADR-0006](adr/0006-context-sufficiency.md).
2. **Panel lift.** That a cross-vendor panel finds materially more real bugs than the best single
   model. If small, nearly everything downstream should be simplified or cut.

Both are settled by M2. Neither should be argued about past that point.

---

## M0 — Walking skeleton *(~1.5 weeks)*

Prove the loop end to end, locally. Context handling is in from day one — it is not a refinement,
it is the difference between a working reviewer and a noise generator.

- `Finding` / `RunRecord` models, `panel.yaml` loader with family resolution and validation
- Packet builder: diff + full changed files + **symbol slice at depth 1** (tree-sitter/ctags)
- `litellm_backend` with the structured-output ladder (rungs 1–3 only)
- Fan-out to 2 models × 2 personas, blind
- T0 reference check; `asserts_absence` in the schema
- **T0.5 falsification search** — declared-symbol and pattern search, repo-wide. Cheap to build
  (it is ripgrep and a caller walk), and it targets the hypothesized dominant false-positive
  class directly. M1 checks whether that class really is dominant.
- Terminal renderer, run record persisted
- `crossexam review --base main` works on a local repo

**Exit:** a real diff produces findings; T0 kills a fabricated citation; T0.5 catches at least one
"X is never validated" claim where the validator exists elsewhere in the repo. No clustering, no
refutation, no sandbox.

## M1 — The bench *(~1 week)*

Deliberately before the interesting features, so every later choice is measured rather than
argued.

- Corpus: 40 seeded + 30 clean + **every historical pair available at the time**, Python and
  TypeScript. `ilm-realtor-535` is entry one and should be in from the start — historical pairs
  are the population that matters, and deferring them was a holdover from before we had any.
- `bench run` / `bench score`, LLM judge with cached judgments
- Offline replay over run records
- Symbol slice depth 2 + packet tiers, so slice depth becomes a sweepable parameter
- Context negotiation pass (pass A/B), behind a flag so it can be ablated
- **First numbers:** single-model baselines, T0 kill rate, **T0.5 kill rate**, **absence-claim
  share of all false positives**, and a first panel-lift reading

**Exit:** we can answer two questions with numbers instead of opinions — "does adding gemini to a
gpt-only panel help, and by how much," and "how much of our false-positive rate is context
starvation, and which of the three context mechanisms actually pays for itself." Ablations 5 and 7
(slice depth; negotiation pass on/off) run here rather than at M5, because their answers determine
how much of M5 is worth building at all.

## M2 — The adversarial core *(~2 weeks)*

The actual thesis of the project.

- Clustering (embedding + line overlap), `independent_families`
- T2 refutation with family exclusion and recusal
- Publication decision table + caps
- Personas: all 8, versioned, selectable per repo shape
- Ablations 1–4 from [`EVALUATION.md`](EVALUATION.md#5-ablations-worth-running-before-v1)

**Exit / go-no-go:** refutation demonstrably raises precision at acceptable cost, and panel lift
is positive at matched precision. If refutation costs more than it returns, it becomes opt-in
here rather than shipping as a default nobody benefits from.

## M3 — Ground truth *(~1.5 weeks)*

- `repro` field in the schema; personas ask for it
- Sandbox runner: container, no network, no env, timeouts
- Sandbox tiers (`unit` / `dom` / `browser`) with `t1_out_of_reach` when the runtime cannot
  observe the claimed effect — a jsdom "refutation" of a layout bug is the worst possible verdict
- T1 with the one-shot repair loop; counter-repros in T2
- `CONFIRMED_EXECUTABLE` as top rank; `block_on` config

**Exit:** a bug the models describe in prose is *demonstrated* by a failing test the tool ran
itself, and the bench shows the executable path has materially higher precision than consensus.
This is the feature that separates crossexam from a prompt.

## M4 — GitHub Action *(~2 weeks)*

- Two-workflow split, artifact schema validation, PR cross-check
- GitHub renderer: one review, inline comments, in-place summary, honesty footers
- Incremental review with content-hash finding identity and the state blob
- Gating, debounce, spend ceilings
- Composite action `kleintech/crossexam@v1`
- Historical bug-fix corpus added to the bench (starting from `bench/corpus/ilm-realtor-535`)

**Exit:** running on this repository's own PRs. Dogfooding is the acceptance test.

## M5 — Context refinement *(~1.5 weeks)*

The core context mechanisms shipped at M0–M1. What remains is the expensive tail, sized by what
M1's numbers said it was worth.

- LSP-backed slicing for languages where tree-sitter resolution is too coarse
- Non-code slice targets: schema, migrations, IDL/protobuf, OpenAPI, config
- Composition-tree context, level 1 (where a component is *rendered*, not imported) —
  `ARCHITECTURE.md` §2.2b. Levels 2–3 stay unscheduled until the bench justifies them.
- Persona presets per repo shape; warn when a repo's dominant language has no persona covering it
- `signals`: linter, type checker, test, coverage output
- Conventions ingestion (`CLAUDE.md`, `AGENTS.md`, lint config)
- Anonymization scrub
- Prompt-prefix ordering for cache hits
- Ablation 6 (anonymization)

## M6 — Release *(~1.5 weeks)*

- Secret pre-scan with hard abort; output sanitization
- Panel presets (`cheap` / `balanced` / `paranoid` / `local`); `panel explain`; `panel suggest`
- Local-model path documented (Ollama/vLLM) and LiteLLM Proxy topology guide
- SARIF renderer
- Docs, published bench results **including the ablations that came back negative**
- Apache-2.0, PyPI, `uvx crossexam`

## Post-v1, roughly in order of expected value

1. **MCP server.** Any MCP-capable agent calls the panel mid-task. This is the one that changes
   the tool's character most: review stops being a gate at the end and becomes something an agent
   consults while writing. It is #1 in value and was deferred only because the Action proves the
   pipeline against real PRs first.
2. **Suggested patches** as GitHub suggestion blocks — generated by a *different* family than the
   one that found the issue, and re-verified against the repro before being offered.
3. **Batch mode** for scheduled sweeps over open PRs at 50% batch pricing.
4. **Repo-tuned panels** from accumulated local feedback.
5. **Whole-file / architectural review** outside the PR flow.
6. **Claude Code skill** wrapping the CLI (trivial once the CLI is stable).

## Deliberately not planned

- Auto-fix commits. The tool has `pull-requests: write` and should never want more.
- A hosted service. The keys stay with the user; that is most of the point.
- A general model-eval harness. `bench/` answers one question: which panel to run.
- Non-git VCS support.

---

## Risk register

| Risk | Signal it's real | Response |
| --- | --- | --- |
| **Context starvation persists** | M1 shows absence-claim FP rate still high after slice + T0.5 | The premise of ADR-0006 is wrong. Escalate: full-file context for all reviewers, or restrict the tool to positive claims only and refuse to publish absence claims at all. |
| **Panel lift is small** | M2 ablation shows +2–3 points over best single model | Reposition: the value is T0/T0.5/T1 filtering, not the panel. Cut to 2 models, keep verification. |
| **Refutation not worth its cost** | M2 precision lift per dollar is poor | Make T2 opt-in; lean harder on T0/T1 and independent-family count. |
| **Persona set has more holes** | A real bug lands in no lane, as `rendering` did | Expected, not exceptional. Personas are config; add one and re-run the bench. Track per-persona unique contribution so dead lanes get dropped. |
| **Composition-tree context is expensive or unreliable** | Level-1 render-site resolution misses the ilm-realtor-535 case, or blows the token budget | Accept the gap and document the class as out of reach; do not escalate to levels 2–3 on hope. |
| Model-written repros are unreliable | M3 shows `CONFIRMED_EXECUTABLE` false positives | Require the repro to reference the cited symbol; add a repro-sanity metric; demote the verdict's rank. |
| Cost per PR too high for casual use | `balanced` exceeds ~$1/PR | Push `cheap` as default; lean on prompt caching, batch, and local models. |
| False positives still annoy maintainers | Dogfooding at M4 feels noisy | Lower caps before lowering thresholds. Fewer, better comments. |
| Provider churn breaks the panel | Schema/capability failures in CI | The `--probe` command and capability overrides already exist for this; keep the fallback ladder deep. |
| Structural: the tool reviews AI code as well as it reviews human code | Bench shows no difference on AI-authored vs. human-authored bugs | Not a failure — but it means the "decorrelated from your author model" framing is weaker than the "cheap parallel review" framing. Adjust the pitch to match the evidence. |
