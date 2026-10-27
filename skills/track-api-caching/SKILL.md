---
name: track-api-caching
description: Review and improve how a codebase caches and syncs data from the Track Hospitality API (developer.trackhs.com), following Track's "Caching Data on a Local Server" guide. Use when a developer integrating with Track wants to audit their sync jobs, cut API traffic, stop re-downloading every unit or reservation, move to delta or updatedSince syncing, pick polling intervals, or asks "is my Track integration caching correctly", "review my Track sync", "how often should I poll Track", "use the unit delta endpoint", "contentUpdatedSince / availabilityUpdatedSince / pricingUpdatedSince", or "keep a local copy of Track units, rates, availability or reservations". Works in any language.
---

# Track API caching review

Audit a codebase that calls the Track API and bring its caching in line with
Track's published guidance. The goal: take one full snapshot, then fetch only
what changed.

Source of truth is the public docs. Read [references/track-caching-rules.md](references/track-caching-rules.md)
before reviewing. It lists every rule this skill checks, each tied to a
published page. **Do not tell the developer about Track behavior that is not
in that file or on developer.trackhs.com.** If you are unsure how the API
behaves, say so and point them to the docs or to Track support.

## Workflow

### 1. Find the Track calls

Search for the API surface. Typical signals:

- Hosts: `*.trackhs.com`, `*.tracksandbox.io`, or a configured tenant domain ending in `/api`
- Paths: `/pms/units`, `/pms/units/delta`, `/pms/units/types/delta`, `/pms/reservations`, `/v2/pms/reservations`, `/pms/units/{id}/pricing`, `/v2/pms/units/{id}/availability`
- Params: `contentUpdatedSince`, `availabilityUpdatedSince`, `pricingUpdatedSince`, `updatedSince`, `page`, `size`, `scroll`

Build an inventory: for each call, note the file and line, the data type
(content, availability, pricing, reservations, unit types), what triggers it
(cron, queue, request handler, startup), how often, and where results are
stored.

If you find no Track calls, say so and stop.

### 2. Check each data type against the rules

Go through the checklist in [references/track-caching-rules.md](references/track-caching-rules.md).
For each data type, answer:

1. Is there a local cache at all, or does every request hit Track live?
2. Is there one initial full load, followed by change-only syncs?
3. Does the change-only sync use the right parameter for that data type?
4. Is the "last synced" watermark stored durably and advanced safely?
5. Is the polling interval in the recommended range?
6. Are pagination, timestamps and the delta response handled as documented?

### 3. Report

1. **Summary**: two or three lines (how many data types are cached correctly,
   the biggest win).
2. **Summary table**: one row per finding, most impactful first. Keep the
   fix to one short line; the detail goes in the blocks below.

   | # | Severity | Where | Rule | Fix |
   | --- | --- | --- | --- | --- |
   | 1 | High | `src/sync/pricing.ts:42` | C2 | Use `pricingUpdatedSince` delta, refresh only returned ids |
   | 2 | Medium | `src/sync/client.ts:17` | C6 | Send timestamps with a UTC offset |

   If there are no findings, say so instead of printing an empty table.

3. **Findings**: one block per row, in the same order and numbering:

```
1. [High] Pricing re-downloads every unit every 5 minutes
     Where: src/sync/pricing.ts:42
     Rule: C2 (full snapshot once, then changes only)
     Why it matters: every run pulls the whole portfolio instead of the handful
       of units whose pricing changed.
     Fix: call /pms/units/delta?pricingUpdatedSince=<watermark>, then refresh
       pricing only for the returned ids.
```

Severity guide:

- **High**: full reloads on a timer, no cache for data read on every request,
  watermark that can skip changes, delta calls that silently drop a parameter.
- **Medium**: intervals far outside the recommended range, timestamps without
  a UTC offset, pagination that stops early or ignores the result cap.
- **Low**: missing periodic full resync, logging and observability gaps.

Also list what the code already does well, so the developer knows what to keep.

### 4. Offer the fix

Ask before editing. When asked, implement fixes in the project's own language,
HTTP client, storage and scheduler. Do not introduce new frameworks. The
reference sync loop in [references/sync-loop.md](references/sync-loop.md) shows
the shape; adapt it, do not paste it.

Never put API keys in code, logs, or your report. If you see credentials
committed to the repo, flag that as a separate High finding.
