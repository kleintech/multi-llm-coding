# ilm-realtor-413 — content refresh never cataloged the media it copied

**Kind:** `historical` · **The T0.5 test case: a *true* absence claim whose falsifier search
returns matches.**

```yaml
id: ilm-realtor-413
kind: historical
repo: kleintech/ilm-realtor
introduced_by: { pr: 413, title: "feat(seed): KotH demo tenant + content refresh pipeline",
                 note: "created scripts/refresh-preview-content.ts; merged in release #408, 2026-06-10" }
fixed_by:      { pr: 525, sha: "61d82b7dffcca9813c94898f8c6909786e0a6314", merged: "2026-07-16" }
review_target: introduced_by
severity: medium
lifetime: ~5 weeks
detectable_from_diff_alone: false
```

## The defect

`refresh-preview-content.ts` copies prod content rows (BlogPost / Neighborhood / LocalGuide / …)
into a preview DB verbatim, including each row's `content` JSON and `featuredImage`. Those
reference the **production** Blob store. The images render fine — the URLs are public — but nothing
inserts `MediaAsset` rows, so the Media Library comes up empty for every refreshed page.

Measured at fix time: **29** distinct managed images referenced across 10 content pages, **2**
cataloged, **27 missing**, all prod-store URLs.

This is an **omission**, not a regression. The script never had the step. `introduced_by` is
therefore whichever PR created the file — #413.

## Why this is the T0.5 test case

The finding a reviewer must produce is a **negative existence claim**: *"this copies content
referencing managed images but never catalogs them."* Exactly the shape
[ADR-0006](../../docs/adr/0006-context-sufficiency.md) is built around — except here **the claim is
true**, which is the case that machinery has never been tested against.

Trace it through the ladder:

1. Reviewer sets `asserts_absence: true`, `absence_check.symbols: ["MediaAsset", "mediaAsset.create"]`,
   `patterns: ["mediaAsset", "backfill-media"]`.
2. **T0.5 searches repo-wide and finds matches** — `backfill-media-assets.ts` and
   `import-media-from-blob.ts` both catalog media, in files outside the reviewer's packet.
3. Verdict: `ABSENCE_UNSUPPORTED`.

**A design that dropped absence claims on any match would kill this true positive here.** The
"reframe, don't drop" rule ([ADR-0006](../../docs/adr/0006-context-sufficiency.md) decision 3) is
what saves it — the finding escalates to T2 with the found code attached and the question narrowed:

> Cataloging exists in `backfill-media-assets.ts` and `import-media-from-blob.ts`. Does either
> cover the refresh path?

The correct answer is **no**, and it is derivable from the code: `import-media-from-blob.ts` lists
only the store its `BLOB_READ_WRITE_TOKEN` points at, so it is *structurally* blind to the
cross-store URLs the copied content references; `backfill-media-assets.ts` is a separate manual
script the refresh never invokes.

That rule was argued for on principle, with no evidence. **This entry is the first evidence for
it,** and the first case where dropping-on-match would have been demonstrably wrong.

## What it measures

| Capability | Pass condition | Baseline |
| --- | --- | --- |
| **Absence-claim routing** | Reviewer emits the claim with a populated `absence_check` | Untested |
| **Reframe-don't-drop** | Survives `ABSENCE_UNSUPPORTED` and reaches T2 rather than being dropped | **The core assertion of this entry** |
| **T2 quality** | Refuters correctly conclude neither existing path covers the refresh | Hard — requires reading `import-media-from-blob.ts`'s token scoping |
| **T0.5 packet injection** | The two sibling scripts are attached to the refuter's prompt | Required for the above to be answerable |

Note the failure mode this entry guards against is *silent*: a dropped finding produces no output
and no signal. Without a corpus entry that specifically asserts survival, a regression that turns
reframe back into drop would never be noticed.

## Expected trajectory

Plausibly findable early — the packet includes the whole script, and "copies content with image
refs, never writes MediaAsset" is visible from the diff alone. The hard part is not *raising* it
but *surviving* T0.5 and then T2, where a refuter that finds `backfill-media-assets.ts` and stops
reading will wrongly refute it.

**Watch for exactly that.** A lazy refutation here — "cataloging exists, see
`backfill-media-assets.ts`" — is the T2 analogue of the class-3 failure in
[`OBSERVED_FAILURES.md`](../../docs/OBSERVED_FAILURES.md): having the data and not tracing it.

## Companion entry

[`ilm-realtor-525`](ilm-realtor-525.md) — the fix for *this* bug introduced the next one, and the
predecessor review passed it. The pair is a compact illustration of why review is hard: the
obvious fix to a real absence bug created a subtler defect one relation further out.
