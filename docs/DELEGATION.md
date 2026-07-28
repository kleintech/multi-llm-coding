# Delegation — can this architecture also hand work *out*?

Reviewing a diff and generating one are different tasks. This document works out which parts of
crossexam transfer, which invert, and what to do about it now versus later.

**Conclusion up front:** delegation and review are **adjacent stages, not the same stage**. Do not
build delegation into crossexam. Do extract the packet builder as a task-agnostic core now, because
that is the piece both need and the only decision that is expensive to defer.

---

## 1. What transfers

| Component | Transfers? | Notes |
| --- | --- | --- |
| **Packet builder** | **Completely, and matters more** | A delegate must *write* correct code, not merely judge it. Every context lesson learned here applies with more force. |
| Verification ladder | Strongly, and gets easier | T0 → does it compile/typecheck. T1 → do the tests pass. Both are *more* natural for generated code than for claims about code. |
| Provider layer | For model delegates only | See §3 — agent products are not `acompletion` calls. |
| Deterministic orchestration | Yes | Submit → poll → collect is a state machine. |
| Bench + offline replay | Yes, different metric | Task success rather than finding precision. |
| Cost control, families, config | Yes | Unchanged. |

The packet is the strongest transfer and the argument is the corpus. A delegate asked to add media
cataloging to `refresh-preview-content.ts`, given only the file, writes
[exactly the bug #525 shipped](../bench/corpus/ilm-realtor-525.md) — it cannot know that a consumer
deletes via `del(url)` with an ambient token. Write→reader slicing is not a review feature. It is a
**context** feature, and generation needs it at least as much.

## 2. What inverts

Three things flip sign, and each is load-bearing.

**Decorrelation stops being the objective.** For review, N independent opinions *are* the product —
you take the union and count agreement. For generation you want **one** good implementation.
N implementations demands a *selector* — judge panel, tournament, best-of-N — which is a different
pattern with different economics. Union versus select. Nothing in crossexam's adjudication layer
does select, and retrofitting it would distort the part that works.

**Recusal inverts, which is exactly why they compose.** In review the author recuses. In delegation
the delegate *is* the author — so crossexam is precisely the right thing to point at its output,
with the delegate's family excluded from the panel. Delegation upstream, review downstream, and the
existing `author_family` config already expresses it.

**The threat model gets much worse.** `SECURITY.md` rests on two structural properties: reviewers
have **no tools**, and their only output channel is a **constrained schema**. A prompt-injected
reviewer produces a bad finding — a quality bug. A delegate needs repo write, tool use, iteration,
and often network. A prompt-injected *delegate* produces malicious code in your repository. That is
not the same risk class, and none of crossexam's mitigations carry over. Delegation needs its own
security design, not an extension of this one.

## 3. Lovable specifically — the wrong example

As of July 2026, **Lovable exposes no public API for programmatic access**. The community
workaround is headless-browser automation of the UI
([lovable-automation](https://github.com/ai-tool-development/lovable-automation), an explicitly
experimental proof of concept). Driving a product's web UI with Playwright is not an integration
you want in CI.

Its two real integration surfaces are:

- **An MCP server** — build, deploy, and manage Lovable apps from an MCP-capable client. Interactive
  and session-shaped, not a job you submit and collect.
- **GitHub sync** (two-way) — the artifact path. Lovable writes to a branch; you review the branch.

So the achievable pattern today is not "call Lovable as a delegate." It is: **use Lovable through
its own surface, let it push a branch, and point crossexam at the resulting PR.** That works right
now and needs no new code — it is just a PR with a known author family.

Two further cautions on the "purpose-built" premise:

- Lovable is tuned for **greenfield** generation in its own stack (React/Vite/Tailwind/Supabase).
  `ilm-realtor` is Next.js + Prisma with established conventions. A tool optimized for building new
  apps from a prompt is not obviously better inside a mature codebase, and the specialist advantage
  may not survive the transfer. Worth measuring before assuming.
- A UI specialist is exactly the generator most likely to produce
  [`ilm-realtor-535`](../bench/corpus/ilm-realtor-535.md) — a popover with a correct-looking
  `z-index` defeated by an ancestor's stacking context — and a generalist reviewer is exactly what
  misses it. Specialist generation *raises* the value of a `rendering` persona from a different
  family, rather than removing the need for review.

### Agents that *are* delegatable

If the goal is programmatic delegation rather than Lovable in particular, several 2026 agents expose
real job interfaces: **Codex** (cloud agent — submit a task, get a PR), **Devin** (managed instances
in isolated VMs, opens PRs), **Google Jules**, **Grok Build** (`-p` headless), **Cline CLI 2.0**
(headless CI mode), and the headless modes of Claude Code, Codex CLI, and Gemini CLI.

Note the shape they share, and how it differs from a review call:

```
review:      request → response                    seconds,  cents,  stateless
delegation:  submit  → poll … → branch/PR          hours,    dollars, stateful
```

Different protocol, different failure modes, different cost envelope. A `Delegate` protocol would be
`submit()` / `poll()` / `collect()`, not `review()`.

## 4. The decision

**Not in crossexam v1.** The tool's thesis is decorrelated review with hard verification. Delegation
has a different objective function, a different protocol, and a materially worse threat model.
Bolting it on would blur the one thing the project is trying to prove.

**But make the packet task-agnostic now.** This is the single cheap-now, expensive-later call:

- `ReviewPacket` → **`ContextPacket`**, in a `context/` package with no review-specific types.
- Persona briefs, the finding schema, and the ladder stay in `review/`.
- `crossexam review` becomes one consumer of `context/`, not its owner.

That costs almost nothing at M0 — it is naming and a directory — and it means a delegation tool
(this repo, another repo, or someone else's) inherits symbol slicing, write→reader resolution,
string-literal cross-language slicing, intent-first ordering, and budget tiering for free. Those
were expensive to learn and are the genuinely reusable asset here.

**Compose today, without writing anything:** delegate through whatever surface a tool offers, have
it open a PR, set `author_family` to that tool's model family, and let the panel review it with that
family recused. That is the working answer to the original question, and it is available now.

**Revisit after v1**, if the bench shows crossexam catching specialist-generated defects reliably.
That result would justify a sibling tool sharing `context/`. The reverse result — that reviewing
specialist output is unreliable — is a good reason not to automate handing work to specialists.
