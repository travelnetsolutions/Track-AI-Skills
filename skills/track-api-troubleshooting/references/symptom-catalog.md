# Track symptom catalog

Every diagnosis here comes from a page on developer.trackhs.com, named on
each entry. "Who" says who has to act: **Dev** (the developer's code),
**PM** (the property manager, in Track), **Support** (Track support).

Sources:

- **Keys**: [Which API Key Do I Need?](https://developer.trackhs.com/docs/which-api-key-do-i-need), [Channel keys vs Server keys](https://developer.trackhs.com/docs/channel-keys-vs-server-keys)
- **401**: [401: Unauthorized](https://developer.trackhs.com/docs/401-unauthorized)
- **409**: [409: Invalid page provided](https://developer.trackhs.com/docs/status-code-409)
- **Empty**: [200: Empty Collection](https://developer.trackhs.com/docs/200-empty-collection)
- **Booking**: [Creating Reservations With Reservations V2](https://developer.trackhs.com/docs/creating-reservations-with-reservations-v2)
- **Errors**: [SINGLE_PAYMENT_TYPE](https://developer.trackhs.com/docs/single_payment_type), [PAYMENT_CHANNEL](https://developer.trackhs.com/docs/payment_channel), [INVALID_CANCEL_POLICY](https://developer.trackhs.com/docs/invalid_cancel_policy)
- **Sync**: [Change Detection and Incremental Sync](https://developer.trackhs.com/docs/change-detection-and-incremental-sync), [Caching Data on a Local Server](https://developer.trackhs.com/docs/caching-data-on-local-server)
- **Images**: [The Image Serving Process](https://developer.trackhs.com/docs/the-image-serving-process)
- **Reviews**: [Reviews Sync On Your Website](https://developer.trackhs.com/docs/reviews-sync-on-your-website)
- **Rate**: [Rate Limits](https://developer.trackhs.com/docs/rate-limits)
- **Webhooks**: [Does Track have any webhooks that push data out to channels?](https://developer.trackhs.com/docs/does-track-have-any-webhooks-that-push-data-out-to-channels)
- **Ref**: the API reference page for the endpoint involved

---

## By HTTP status

### 401 Unauthorized (401, Keys)
**Means:** Track could not authenticate the request.
**Causes:** wrong key or secret; a key for a different tenant (wrong
`{customer}` domain); outdated key; for a channel key, the channel is not
active in Track.
**Confirm:** check the key, secret and tenant domain together; ask the PM
whether the channel is active.
**Fix:** correct the credentials or domain; the PM can issue a new key.
**Who:** Dev, or PM for a new key or an inactive channel. Track support
cannot create or hand out keys.

### 403 Forbidden (Keys)
**Means:** the key is valid, but this key type cannot call this endpoint.
**Causes:**
1. Channel key on an endpoint outside the Channel API section. Common:
   confirm, check in, check out and their undos; folios and payments;
   accounting; owners; housekeeping; CRM; guest communications; promo codes.
2. Server key on an endpoint that also checks a Track permission, for
   example Get Reviews (needs `reputation_management.reputation_management_view`).
**Confirm:** is the endpoint listed under the Channel API section of the
docs? If not, a channel key cannot call it.
**Fix:** for (1), get a server key from the PM for those calls; one
integration can hold both. For (2), the PM grants the permission.
**Who:** PM.

### 409 Invalid page provided (409, Empty, Ref)
**Means:** the requested page is past the last page of results, even though
data exists. Expected control flow on paginated endpoints, not a failure.
**Causes:** paging past `page_count`; a page number computed from an older
total after results shrank; hardcoded page loops.
**Not a 409:** a query with no results returns `200` with `page: 0` and
`page_count: 0` (see 200 below).
**Fix:** treat it as recoverable: stop paging, or restart from page 1 with
current parameters. Follow `_links.next` instead of computing page numbers.
Pages start at 1 (`page` minimum 1 in the reference); `page=0` is not a
valid page.
**Who:** Dev.

### 200 with an empty collection (Empty, Keys)
**Means:** valid, authenticated, in range, and nothing matches the query.
Not an error.
**Causes:** filters too narrow; or, with a channel key, the channel is not
set up to see the data (see "Data is missing" below).
**Fix:** show "No results found" and let the user broaden the search. If
data should be there, check channel settings with the PM.
**Who:** Dev, or PM.

### 422 CONTACT_REQUIRED (Booking, Ref)
**Means:** a reservation create had both `contact` and `contactId`, or
neither.
**Fix:** send exactly one. An inline `contact` needs `firstName` and
`lastName`.
**Who:** Dev.

### 422 "Rate type is invalid or not active." (Ref: Create Reservation V2)
**Means:** the `rateTypeId` sent on create was not found or is not active.
**Fix:** omit `rateTypeId` to let Track pick the best available rate, or
use one returned by `GET /pms/units/{unitId}/pricing` for that unit.
**Who:** Dev, or PM if the rate type should be active.

### Validation error when sending a payment (Booking)
**Means:** usually both `payment` and `holdCard` were sent; they are
mutually exclusive.
**Fix:** send one, chosen by the quote's `guaranteePolicy.type`
(`payment` for `Deposit` / `FullDeposit`, `holdCard` for `Guarantee` / `Hold`).
**Who:** Dev. See `track-api-booking`.

## By error code

### SINGLE_PAYMENT_TYPE (Errors)
**Means:** the `payment` object had more than one payment method, or none.
With an `amount`, exactly one method must be sent.
**Fix:** send one method (credit card, e-check, net commission, ...).
**Who:** Dev.

### PAYMENT_CHANNEL (Errors, Keys)
**Means:** the payment method is not allowed for the Distribution Channel,
for example net commission on a channel set to card and e-check only.
Channel keys can only use electronic methods (card, e-check).
**Fix:** use an allowed method, or the PM changes the channel's payment
settings.
**Who:** Dev or PM (Configuration > PMS Setup > Distribution Channels).

### INVALID_CANCEL_POLICY (Errors)
**Means:** the cancellation policy sent is inactive or not available to the
unit, or the unit has no active cancellation policy.
**Fix:** stop hardcoding `cancellationPolicyId` (with a channel, it is set
from the unit); the PM activates or assigns a policy.
**Who:** PM (Configuration > PMS Setup > Cancellation Policies, or the unit
profile's Reservations tab).

## Booking outcomes (no error)

| Symptom | Likely cause | Fix | Who |
| --- | --- | --- | --- |
| Every reservation comes back `Hold` | `Guarantee` policy, card sent as `payment` | Send `holdCard` | Dev |
| `Hold` at one PM, `Confirmed` at another, same payload | Different policy types | Quote first, branch on `guaranteePolicy.type` | Dev |
| Reservation's policy differs from the quote's | Quote's `guaranteePolicyId` not sent on create | Send it, flat | Dev |
| Auto-cancelled about an hour after booking | `Deposit` policy with `holdCard`; nothing collected before `holdExpiresAt` | Send `payment` for `Deposit` / `FullDeposit` | Dev |
| `Deposit` booking on `Hold`, nothing charged | No installment due today | Check the quote's `paymentPlan` | Expected |
| Pinned `guaranteePolicyId` ignored | Channel key; channel has its own active policy | PM checks the channel's guarantee policy | PM |
| Still `Hold` with the right payment field | Channel in Request mode, or policy type `Hold` | Expected; PM confirms in Track | PM |
| Guest got "confirmed", PM says cancelled | Code did not read `status` / `holdExpiresAt` and did not poll | Read status; poll `updatedSince` | Dev |
| New contact created on every booking | Inline contact with no email or phone (cannot be matched) (Ref) | Send `primaryEmail` / phone, or `contactId` | Dev |
| Create fails on a duplicate contact | Matched contact updated with an email or phone another contact already has (Ref) | Look up with CRM Contacts (server key) and send `contactId` | Dev |

Source: Booking, plus the Create Reservation V2 reference for contacts.

## Data is missing (no error)

| Symptom | Likely cause | Fix | Who |
| --- | --- | --- | --- |
| Channel key sees fewer units than exist | Units not added to the channel, and **All Units Available** off (Keys) | PM adds units or enables the setting | PM |
| Node endpoints do not work with a channel key | **Enable Node API** off on the channel (Keys) | PM enables it | PM |
| Channel key sees only some reservations | Only this channel's bookings, unless **All Reservations Available** (Keys) | PM enables it if appropriate | PM |
| No contact details on reservations (channel key) | **Include Contact** off (Keys) | PM enables it | PM |
| Fewer rate types on pricing with a channel key | Only rate types available to the channel for that unit (Keys) | Expected; PM adjusts rate type setup | PM |
| Description fields absent on units with a server key | `includeDescriptions=1` not sent; fields are absent, not null (Ref: Units) | Add `includeDescriptions=1` | Dev |
| Availability returns no data for later dates | Rates or seasons not configured out to the max booking window (Quick Start) | PM extends rates / seasons | PM |
| Reviews have no guest, reservation or unit context | Channel key gets body and ratings only (Reviews, Keys) | Server key with the reputation permission | PM |
| A record seems to vanish with a channel key | PM changed a channel setting (Sync) | Check with the PM | PM |
| Paging stops returning results past 10,000 | `page * size` capped at 10,000 (Ref, Sync) | Narrow date windows, or `scroll` on Reservations V2 | Dev |

Source for "Quick Start": [Quick Start Guide](https://developer.trackhs.com/docs/quick-start-guide).

## Sync and delta (no error)

| Symptom | Likely cause | Fix | Who |
| --- | --- | --- | --- |
| Delta misses availability or pricing changes | More than one `*UpdatedSince` sent; only the first (pricing, content, availability order) is used (Ref: Unit Delta) | One parameter per call | Dev |
| Window shifted by hours | Timestamp without UTC offset, read in the account's timezone (Ref, Sync) | Send full ISO 8601 with offset | Dev |
| Offset ignored or request odd with `+` | `+` not URL-encoded, arrives as a space (Sync) | Encode as `%2B`; use a query encoder | Dev |
| New unit never appears in delta `updated` | Units are created inactive; channel keys also need the unit mapped and enabled (Ref: Unit Delta) | PM activates and maps it | PM |
| Same units in `removed` on every call | `removed` is a full snapshot, not changes since your cursor (Ref, Sync) | Diff it against what you show | Dev |
| A unit is in both `updated` and `removed` | Inconsistent channel mapping (Ref: Unit Delta) | Trust `updated`; report the mapping | Dev, PM |
| Changes missed between runs | Cursor from local clock, max `updatedAt`, no overlap, or advanced after a failed run (Sync) | Cursor = run start from Track's `Date` header minus a few minutes, saved only on success | Dev |
| Cancelled reservations "disappear" from the mirror | Code treats absence as cancel; cancellations arrive as updates (Sync) | Handle `status: Cancelled` as an update | Dev |
| Waiting for a webhook that never comes | Track has no webhooks for data changes (Webhooks) | Poll with `updatedSince` / delta | Dev |
| Reservation search order looks random | `sortColumn` value not in the documented list falls back to internal id (Ref: Reservations V2) | Use a documented `sortColumn` | Dev |
| Reservation `search` misses a substring | `search` is a prefix match at the start of a word; a leading `+` searches phone only (Ref: Reservations V2) | Use `contactId` / `unitId` to filter exactly | Dev |

See `track-api-caching` for a full review of the sync code.

## Images (no error)

| Symptom | Likely cause | Fix | Who |
| --- | --- | --- | --- |
| Images break about an hour later | Signed URLs (expire after 60 minutes) stored and reused (Images) | Store the image or refetch the URL; or load through the proxy at once | Dev |
| Images blocked or unreliable | Hotlinking S3 directly, which is unsupported and can be blocked (Images) | Use `img.trackhs.com/<size>/<s3url>` or self-host | Dev |
| First page load of images is slow | Proxy cold cache on first request (Images) | Expected; later loads are cached | Expected |

See `track-api-images` for a full review.

## Volume

### Requests start failing under load (Rate)
**Means:** possibly the rate limit: 10,000 requests every 5 minutes, per
IP. Further limits can be applied case by case for abuse.
**Fix:** cache and use delta endpoints instead of re-reading; stagger jobs;
back off and retry later.
**Who:** Dev.
The docs do not state the status code or headers returned when the limit is
hit. Do not claim them.

---

## Not in the docs: do not claim

If the symptom is one of these, say the docs do not cover it and suggest
Track support:

- 404 or 5xx causes for a specific endpoint
- The status code, headers or lockout window when the rate limit is hit
- Whether a request that timed out was processed
- Anything about Track outages or maintenance
