# Reference sync loop

Language-neutral shape of one change-only sync job. Adapt it to the
project's HTTP client, storage and scheduler. Rule ids refer to
[track-caching-rules.md](track-caching-rules.md).

## Pricing or availability job (every 5 to 10 min)

```
KIND  = "pricing"                       # or "availability"
PARAM = "pricingUpdatedSince"           # or "availabilityUpdatedSince"  (C3, C4: one per request)

watermark = store.get_watermark(KIND)
if watermark is None:
    full_load(KIND)                     # C2: snapshot once
    return

next_watermark = now_utc_iso8601()      # C13: capture BEFORE the request
                                        # C6: e.g. 2026-07-31T05:00:00+00:00

resp = GET /pms/units/delta?{PARAM}={watermark}
for unit_id in resp.updated:            # C8: ids only
    data = fetch_one(KIND, unit_id)     # /pms/units/{id}/pricing or /v2/pms/units/{id}/availability
    store.upsert(KIND, unit_id, data)   # C13: idempotent

store.set_watermark(KIND, next_watermark)   # C13: only after everything succeeded
```

## Content job (every 15 to 30 min)

```
watermark      = store.get_watermark("content")
next_watermark = now_utc_iso8601()

resp = GET /pms/units/delta?contentUpdatedSince={watermark}

changed = resp.updated
for batch in chunks(changed):
    for page in paginate(GET /pms/units?id[]={batch}&includeDescriptions=1):   # C11, C12
        for unit in page:
            store.upsert_unit(unit)     # C11: absent field != null, do not overwrite

hidden = set(resp.removed) - set(resp.updated)   # C9: snapshot; updated wins
store.reconcile_hidden(hidden)                    # mark hidden; compare to what you already know,
                                                  # do not treat every id as a new removal

store.set_watermark("content", next_watermark)
```

## Reservations job

```
watermark      = store.get_watermark("reservations")
next_watermark = now_utc_iso8601()

for page in paginate(GET /v2/pms/reservations?updatedSince={watermark}):   # C12: follow _links.next,
    for r in page:                                                         # or use scroll for big sets
        store.upsert_reservation(r)

store.set_watermark("reservations", next_watermark)
```

## Paginate helper

```
def paginate(url):
    while url:
        body = GET url
        yield body._embedded.<collection>
        url = body._links.next.href if body._links.next else None
```

Stop and alert if `total_items` is above 10,000 on a paged (non-scroll)
collection: those results are unreachable by paging (C12).

## Nightly or weekly

```
full_load(every kind)                   # C15: repairs drift
```
