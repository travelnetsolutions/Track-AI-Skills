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

resp = GET /pms/units/delta?{PARAM}={urlencode(watermark)}   # C6: offset, %2B-encoded
next_watermark = resp.headers["Date"] - few_minutes          # C13: Track's clock, run start, overlap
for unit_id in resp.updated:            # C8: ids only
    data = fetch_one(KIND, unit_id)     # /pms/units/{id}/pricing or /v2/pms/units/{id}/availability
    store.upsert(KIND, unit_id, data)   # C13: idempotent

store.set_watermark(KIND, next_watermark)   # C13: only after everything succeeded
```

## Content job (every 15 to 30 min)

```
watermark      = store.get_watermark("content")

resp = GET /pms/units/delta?contentUpdatedSince={urlencode(watermark)}
next_watermark = resp.headers["Date"] - few_minutes

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
next_watermark = None

for page, headers in paginate(GET /v2/pms/reservations?updatedSince={urlencode(watermark)}):  # C12
    next_watermark = next_watermark or headers["Date"] - few_minutes   # first response = run start
    for r in page:
        store.upsert_reservation(r)     # C18: Cancelled arrives as an update

store.set_watermark("reservations", next_watermark)
```

## Paginate helper

```
def paginate(url):
    while url:
        resp = GET url
        yield resp.body._embedded.<collection>, resp.headers
        url = resp.body._links.next.href if resp.body._links.next else None
```

Stop and alert if `total_items` is above 10,000 on a paged (non-scroll)
collection: those results are unreachable by paging (C12).

## Weekly (or as needed)

```
full_load(every kind)                   # C15: reconciliation repairs drift
```

Stagger the jobs so they do not all fire on the same tick (C17).
