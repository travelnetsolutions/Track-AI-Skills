# Track caching rules

Every rule here comes from a page on developer.trackhs.com. The source is
named on each rule. Rules marked **General practice** are standard sync
engineering, not claims about how Track behaves. Present them that way.

Sources:

- **Guide**: [Caching Data on a Local Server](https://developer.trackhs.com/docs/caching-data-on-local-server)
- **Unit Delta**: API reference, `GET /pms/units/delta`
- **Unit Collection**: API reference, `GET /pms/units`
- **Reservations v2**: API reference, `GET /v2/pms/reservations`
- **Unit Types Delta**: API reference, `GET /pms/units/types/delta`

## The core pattern (Guide)

1. Pull a full snapshot of each data type once.
2. After that, ask only for what changed since your last sync.
3. Delta endpoints can be called as often as you need.

| Data type | Initial load | Change-only sync | Recommended interval |
| --- | --- | --- | --- |
| Unit content | `GET /pms/units` (all pages) | `GET /pms/units/delta?contentUpdatedSince=` or `GET /pms/units?contentUpdatedSince=` | 15 to 30 min |
| Availability | per unit, e.g. `GET /v2/pms/units/{unitId}/availability` | `GET /pms/units/delta?availabilityUpdatedSince=`, then refresh those units | 5 to 10 min |
| Pricing | per unit, e.g. `GET /pms/units/{unitId}/pricing` | `GET /pms/units/delta?pricingUpdatedSince=`, then refresh those units | 5 to 10 min |
| Reservations | `GET /pms/reservations` or `/v2/pms/reservations` (all pages) | same endpoint with `updatedSince=` | not stated; match how fresh you need it |
| Unit types | `GET /pms/units/types` | `GET /pms/units/types/delta` | not stated |

The Guide says intervals can be adjusted to fit your needs. Treat them as a
starting point, not a limit.

## Checklist

### C1. Cache exists (Guide)
Data that is read often (unit content, rates, availability) should be served
from a local store, not fetched from Track on every page view or request.
**Flag:** Track calls inside a request handler with no cache in front.

### C2. Full snapshot once, then changes only (Guide)
**Flag:** a scheduled job that re-downloads all units, all pricing, all
availability, or all reservations on every run.

### C3. Right parameter per data type (Guide, Unit Delta)
Content uses `contentUpdatedSince`, availability uses
`availabilityUpdatedSince`, pricing uses `pricingUpdatedSince`, reservations
use `updatedSince`.
**Flag:** one parameter used to detect every kind of change, for example only
`contentUpdatedSince`, then assuming pricing and availability are fresh.

### C4. One delta parameter per request (Unit Delta)
The delta endpoint uses only one of the three parameters per call. If more
than one is sent, it uses the first present in the order `pricingUpdatedSince`,
`contentUpdatedSince`, `availabilityUpdatedSince` and silently ignores the rest.
**Flag:** any delta request that sends two or three of them. That code is
missing changes without any error.

### C5. Deprecated parameter (Unit Collection)
On `GET /pms/units`, `updatedSince` is deprecated. Use `contentUpdatedSince`.
**Flag:** `/pms/units?updatedSince=`.

### C6. Fully-qualified timestamps (Unit Delta)
Send ISO 8601 with a UTC offset, for example `2026-07-31T00:00:00-05:00`.
Without an offset the value is resolved against the account's company
timezone and the window can shift by several hours. The response's `date`
field echoes your cutoff normalized to UTC.
**Flag:** timestamps built as `Y-m-d H:i:s`, `toLocaleString`, a bare date, or
any format with no offset.

### C7. Exclusive cutoff (Unit Delta)
A record whose timestamp exactly equals your cutoff is not returned.
**Flag:** logic that assumes the cutoff is inclusive.

### C8. Delta returns ids, not records (Unit Delta)
`updated` is a list of unit ids. Fetch the current data for those ids
(`GET /pms/units` accepts an `id` array filter) and upsert it.
**Flag:** code that treats `updated` as full records, or that re-fetches every
unit after seeing a non-empty `updated`.

### C9. `removed` is a snapshot, not a change list (Unit Delta)
`removed` appears only with `contentUpdatedSince`. It is not filtered by your
timestamp: every call returns the full current set of units mapped to you
that are not visible. Units never mapped to you appear in neither array. If
an id is in both `updated` and `removed`, `updated` wins: subtract `updated`
from `removed` before delisting anything.
**Flag:** code that treats `removed` as "removed since last sync" (for
example, firing a delist event for every id on every run), or that delists
ids also present in `updated`.

### C10. Only active units appear in `updated` (Unit Delta)
New units are created inactive and do not appear in `updated` until
activated. With a channel key, a unit also has to be mapped to that channel
with the mapping enabled (unless the channel has `allUnitsApi`). Server keys
are not subject to the mapping filter.
**Flag:** onboarding logic that expects a just-created unit to show up in the
next delta.

### C11. Description fields need `includeDescriptions` with server keys (Unit Collection)
With server keys, `shortDescription`, `longDescription`, `directions`,
`checkinDetails`, `houseRules`, `notes` and `amenityDescription` are returned
only when the request passes `includeDescriptions=1`. Otherwise they are
absent, not null. Channel keys always receive them.
**Flag:** a server-key content cache built without `includeDescriptions`, or
an upsert that writes null over a stored description because the field was
missing from the response.

### C12. Pagination and the 10,000 result cap (Unit Collection, Reservations v2)
Collections are paged with `page` and `size`, and limited to 10,000 total
results (`page * size`). Follow `_links.next` until it is absent. For large
reservation sets, `GET /v2/pms/reservations` supports `scroll` (start with
`1`, then follow `_links.next`; sorting is disabled while scrolling).
**Flag:** code that reads only the first page, assumes one page holds
everything, or pages past 10,000 results and silently gets nothing.

### C13. Durable watermark, advanced safely (General practice)
Store the "last synced" timestamp per data type somewhere that survives a
restart. Capture the new watermark before you send the request, and save it
only after every page has been fetched and stored. If a run fails, keep the
old watermark so the next run retries the same window. Upserts should be
idempotent so re-processing a record is harmless.
**Flag:** watermark kept only in memory, reset on deploy, set to "now" after
the request returns, or advanced even when the sync failed partway.

### C14. Separate jobs per data type (Guide, General practice)
Pricing and availability change faster than content (Guide intervals:
5 to 10 min vs 15 to 30 min). Running them as separate jobs with their own
watermarks lets each run at its own pace and fail on its own.
**Flag:** one job, one interval, one watermark for everything.

### C15. Periodic full resync (General practice)
A scheduled full reload (for example nightly or weekly) repairs anything a
missed run or a bug let drift.
**Flag:** no full resync path at all, so drift is permanent.

### C16. Poll interval sanity (Guide)
**Flag:** delta polling much tighter than the Guide suggests with no business
reason, or content polled every few hours when the integration needs fresh
listings. Recommend the Guide's ranges and let the developer adjust.

## Out of scope: do not claim

The published docs do not state these. Do not assert them. If the developer
asks, say the docs do not cover it and suggest they contact Track:

- Specific rate limits or throttling numbers
- Webhooks or push notifications for these changes
- Whether `updatedSince` on reservations is inclusive or exclusive, or how it
  treats timestamps without an offset (C6 and C7 are documented for the unit
  delta endpoint only)
- Anything about internal caching on Track's side
