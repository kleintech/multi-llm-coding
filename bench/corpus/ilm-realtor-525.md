# ilm-realtor-525 — cataloged cross-store Blob URLs the env cannot manage

**Kind:** `historical` · **The most valuable entry in the corpus: a documented false negative by
the predecessor review system, with its own claims recorded in the PR body.**

```yaml
id: ilm-realtor-525
kind: historical
repo: kleintech/ilm-realtor
introduced_by: { pr: 525, sha: "61d82b7dffcca9813c94898f8c6909786e0a6314", merged: "2026-07-16T10:38:31Z" }
fixed_by:      { pr: 531, sha: "b728d1bc8e378c19d5f6794b7351c631219a1e09", merged: "2026-07-17T10:25:45Z" }
review_target: introduced_by
severity: medium
lifetime: ~24 hours
was_reviewed_by_predecessor: true      # and passed
detectable_from_diff_alone: false
```

## What happened

`refresh-preview-content.ts` copies prod content into a preview DB verbatim, so image URLs point
at the **production** Blob store. #525 fixed a real bug — those images were missing from the Media
Library — by adding `catalogMedia()`, which sweeps content and inserts `MediaAsset` rows for the
prod-store URLs.

That fix was itself defective, and was replaced **the next day** by #531:

> Cataloging the prod URLs (the prior approach) left the library full of un-manageable, cross-store
> references that break if prod prunes a blob. […] the media library's delete acts on the ambient
> store's token — so a preview env can't actually manage a prod-store asset (`del(url)` silently
> no-ops).

Verified: `src/lib/mcp/tools/mediaAssets.ts:468` calls `await del(current.url)` with **no explicit
token**, so it uses the ambient `BLOB_READ_WRITE_TOKEN` — the preview store's. Against a
prod-store URL that call quietly does nothing, and line 474 swallows failures anyway
(`"Blob del() failed; removing catalog row anyway"`). The catalog row disappears; the blob does
not. #531 re-homes each blob into the target store instead, making the env self-contained.

## Why this entry matters most

**The predecessor adversarial-review system reviewed #525 and passed it.** From #525's own body:

> Fresh-context **adversarial review** (no builder framing) could not refute correctness,
> tenant-safety, idempotency, typecheck, or fidelity to `backfill-media-assets.ts`. Its one
> low-severity note (uninformative dry-run media count on an empty target) is addressed by the
> dry-run summary wording.

Every prior data point in this project has been a **false positive**. This is the first documented
**false negative**, and it is unusually well-instrumented: the review's claimed coverage is written
down, so we know precisely what it checked and what it missed. It checked correctness,
tenant-safety, idempotency, typecheck, and sibling-script fidelity — and #525 is impeccable on all
five. It never asked the question that mattered: *what happens downstream to the rows this writes?*

Note what this does to the scoreboard. `OBSERVED_FAILURES.md` records that the adversary
contributed zero real findings across three runs on a clean PR, and attributes that to test
selection. Here is a run on a PR that **did** contain a defect, and the adversary still produced
nothing. That is a genuine miss, not an artifact — and it is one data point, on one model, with a
diff-only packet.

## The missing context: a fourth relation

The design models three relations. None connects #525's diff to the code that breaks:

| Relation | Reaches | Finds this? |
| --- | --- | --- |
| Symbol slice | what this code **calls** | ✗ — `mediaAssets.ts` is never called by the script |
| Caller walk | what **calls** this code | ✗ — nothing calls a `main()` ops script |
| Composition tree | what **encloses** this code | ✗ — not applicable |
| **Write → reader** | **who reads the rows this code writes** | ✓ |

`catalogMedia()` INSERTs into `MediaAsset`. Its correctness is determined entirely by the
assumptions of everything that *reads* `MediaAsset` — and the two are coupled through a **database
schema**, an edge that appears in no import graph and no call graph.

This is cheap to resolve and generalizes well beyond Prisma: for a write to table/queue/topic/cache/
file-format `T`, find the readers of `T`. With an ORM it is close to a grep — `grep -rl 'mediaAsset\.'`
returns seven files here, one of which is the consumer that matters. The general form covers message
queues, cache keys, shared file formats, and config written by one service and read by another.

## The part context cannot fix

Honest limit, and it should temper expectations for the entry above.

The invariant #525 violated — *"`MediaAsset.url` must live in the ambient Blob store"* — **is
written down nowhere.** Not in the schema, not in a comment, not in an ADR. It exists implicitly in
`mediaAssets.ts` calling `del(url)` without a token. A reviewer given the consumer file could still
only find it by reasoning: *this deletes using an ambient token → a URL from another store won't
match → the delete silently no-ops → so cataloging foreign URLs creates rows that look manageable
and aren't.*

That is a four-step inference across two files plus knowledge of Vercel Blob's `del()` semantics.
Context is **necessary and not sufficient**. Retrieval cannot surface an invariant that was never
recorded; it can only put the reviewer in a position to re-derive it.

## What it measures

| Capability | Pass condition | Baseline |
| --- | --- | --- |
| **Write→reader slicing** | `src/lib/mcp/tools/mediaAssets.ts` reaches the packet | **Fails** — no mechanism reaches it |
| **Persona coverage** | Some lane owns "does written data satisfy its consumers' assumptions" | Ambiguous. `compat` is closest but is framed around *schema* change; the schema here is fine and the *semantics* are violated. Needs sharpening. |
| **Panel lift** | ≥1 family finds it where the single predecessor adversary did not | Directly measurable — the baseline is recorded |
| **T1** | Unreachable — ops script, live-DB convention, no unit tests by design | Expect `t1_out_of_reach`; this is a **T2-only** entry |

## Expected trajectory

Missed at M0 and M1. Findable only with write→reader slicing plus a persona that asks the consumer
question. Even then, uncertain — the inference chain is long and the invariant is undocumented.

**If no configuration of crossexam ever finds this, that is a legitimate result worth publishing.**
It marks the boundary between "context starvation," which this project can fix, and "undocumented
invariant plus multi-hop domain inference," which it probably cannot.

## Companion entry

[`ilm-realtor-413`](ilm-realtor-413.md) — the bug #525 was fixing. Same file, and a true *absence*
claim, which makes the pair a natural A/B for the T0.5 machinery.
