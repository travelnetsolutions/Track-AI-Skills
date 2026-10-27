# Build paths

Each path lists the minimal endpoints, the key, and the call order. Paths
marked **Published** follow a guide on developer.trackhs.com. Paths marked
**Assembled** are put together from the API reference and the key matrix,
with no guide of their own: say so when you give one, and suggest the
developer confirm it with Track before building on it.

Every path that keeps a local copy of data follows
[Caching Data on a Local Server](https://developer.trackhs.com/docs/caching-data-on-local-server)
and [Change Detection and Incremental Sync](https://developer.trackhs.com/docs/change-detection-and-incremental-sync)
(see the `track-api-caching` skill). Every path that shows photos follows
[The Image Serving Process](https://developer.trackhs.com/docs/the-image-serving-process)
(see the `track-api-images` skill).

---

## Booking engine or booking website

**Published.** Sources: [Quick Start Guide](https://developer.trackhs.com/docs/quick-start-guide),
[Creating Reservations With Reservations V2](https://developer.trackhs.com/docs/creating-reservations-with-reservations-v2),
[Fraud Prevention](https://developer.trackhs.com/docs/enhancing-fraud-prevention-in-trackhs-api-requests).

**Key:** channel key. Add a server key from the property manager only if you
must confirm, check in or out, or read folios.

| Step | Endpoint | Key | Purpose |
| --- | --- | --- | --- |
| 1 | `GET /pms/units` (all pages) | Channel | Initial load of units enabled for the channel; flattened with node and inherited values |
| 2 | `GET /pms/nodes` | Channel | Locations / properties (needs **Enable Node API** on the channel) |
| 3 | `GET /pms/units/{unitId}/images` | Channel | Photos, served through the image proxy |
| 4 | `GET /v2/pms/units/{unitId}/availability` | Channel | Availability calendar, today to max booking window by default |
| 5 | `GET /pms/units/{unitId}/pricing` | Channel | Rate types and rates |
| 6 | `GET /pms/units/delta` (one `*UpdatedSince` per call) | Channel | Recurring: which units changed content, availability or pricing |
| 7 | `POST /v2/pms/quotes` | Channel | Price the stay; returns `guaranteePolicy` (with `type`), `guaranteePolicyId`, `paymentPlan` |
| 8 | `POST /v2/pms/reservations` | Channel | Book with the quote's `guaranteePolicyId`, plus `payment` or `holdCard` by policy type, plus fraud data |
| 9 | `GET /v2/pms/reservations?updatedSince=` | Channel | Recurring: watch status changes, especially holds with `holdExpiresAt` |
| 10 | `POST /pms/reservations/{reservationId}/cancel` | Channel | Guest cancels |

**Pitfalls:**
- A `201` is not a confirmation. Read `status` and `holdExpiresAt`. Holds can
  auto-cancel, with windows as short as 60 minutes.
- Choose `payment` for `Deposit` / `FullDeposit`, `holdCard` for `Guarantee`
  and `Hold`. Branch on `guaranteePolicy.type`, never on its `name`.
- Send the quote's `guaranteePolicyId` on the create so the policy cannot
  change between quote and booking. With a channel key, an active policy on
  the channel overrides it.
- In the channel's Request mode, every new booking stays on Hold.
- Send the Seon session data and the guest's IP (`clientIPAddress`) with the
  reservation request.
- Channel keys only see what the channel is set up to see. Empty or short
  results are often a channel setting, not a bug.

## Single-unit microsite

**Assembled** from the booking website path. Same endpoints and key, scoped
to one `unitId`: skip the full unit and node loads beyond that unit, and run
delta and availability for that unit only. The property manager still has to
enable the unit for the channel.

## Channel manager, OTA or CRS

**Published** in part (Quick Start Guide, key guide, Reservations V2 guide,
Change Detection). Same as the booking website path, at portfolio scale.
Lean on `GET /pms/units/delta` and `GET /pms/units/types/delta` rather than
re-reading units, and poll `GET /v2/pms/reservations?updatedSince=` every
few minutes. Unit types matter if the property manager books by type
(`unitTypeId` on create).

## Revenue management

**Published.** Source: [Revenue Management](https://developer.trackhs.com/docs/revenue-management).

**Read side (channel key is enough):** `GET /pms/units`, `GET /v2/pms/units/{unitId}/availability`,
`GET /pms/units/{unitId}/pricing`, `GET /pms/units/delta`, `GET /v2/pms/reservations`
(pace and occupancy).

**Write side:** push unit-date prices back through the daily pricing
endpoints, for example `PUT` / `PATCH /v2/pms/units/{unitId}/daily-pricing`
(PMS Units API, so a server key per the key matrix). Usually pricing only,
not content or photos.

**Cadence:** daily or twice daily is most common in vacation rentals; some
partners recalculate as often as every 30 minutes.

**Pitfalls:** make pricing writes safe to repeat (retries resend the same
data), and agree with the property manager what the partner may overwrite:
all rates, some rate types, or only certain dates.

## Guest messaging (AI agents, autoresponders)

**Published.** Source: [Guest Communications](https://developer.trackhs.com/docs/guest-communications).

**Key:** server key. **Direction:** outbound only. Track is the delivery
pipe; channels cannot inject guest-side messages into Track.

| Step | Endpoint | Purpose |
| --- | --- | --- |
| 1 | `GET /pms/guest-communications/threads` | Find the conversation |
| 2 | `GET /v2/pms/guest-communications/threads/{contextId}/events` or `GET /v2/pms/folios/{folio_id}/events` | Read history for a thread or a reservation's folio |
| 3 | `POST /v2/pms/guest-communications/threads/{thread_id}/send` or `POST /v2/pms/folios/{folio_id}/events/send` | Send a message or add a note |

## Reviews on a website

**Published.** Source: [Reviews Sync On Your Website](https://developer.trackhs.com/docs/reviews-sync-on-your-website).

`GET /channel-management/channel/reviews?updatedSince=`, paged. Channel keys
get body and ratings with only `unitId` and `channelId`; to show reviewer
first name, unit name or photo, use a server key with the
`reputation_management.reputation_management_view` permission (else `403`).

## Damage or insurance program

**Published.** Source: [Embedded Damage / Insurance Programs](https://developer.trackhs.com/docs/embedded-damage-insurance-programs).

**Key:** server key. Backfill with `GET /v2/pms/reservations?arrivalStart=&arrivalEnd=`
(or `scroll`), then poll `?updatedSince=`. `GET /v2/pms/reservations/{reservationId}`
for one stay. Insurance data is on `insuranceProducts[]`.

## Owner portal

**Assembled.** No published guide. Owners and statements are server-key only
per the key matrix. Relevant endpoints in the PMS Owners API:
`GET /pms/owners/{ownerId}`, `GET /pms/owners/{ownerId}/units`,
`GET /pms/owners/statements`, `GET /pms/owners/statements/{statementId}/pdf`,
`GET /pms/owners/{ownerId}/transactions`. Track has its own owner portal;
the developer should confirm with the property manager and Track that a
custom one is what they need.

## Guest portal

**Assembled.** No published guide. Reservation reads work with either key,
but folios, payments and guest communications need a server key per the key
matrix. Relevant: `GET /v2/pms/reservations/{reservationId}`,
`GET /pms/folios/{folioId}/transactions`, the guest communications endpoints
above, and the unit endpoints for stay details. Track has its own guest
portal; the developer should confirm with the property manager and Track
that a custom one is what they need.

## Back-office tools (accounting, housekeeping, CRM)

**Published** in part (key guide, Change Detection). Server key. Poll with
`updatedSince` on `GET /pms/accounting/charges`, `GET /pms/accounting/transactions`,
`GET /pms/housekeeping/work-orders`, `GET /pms/maintenance/work-orders`,
`GET /crm/contacts`, `GET /crm/companies`, `GET /crm/leads`,
`GET /pms/owners`.

## Before go-live (every path)

From the Quick Start Guide: test against the sandbox
`https://api-integration-example.tracksandbox.io`, then send Track an
integration overview and the list of API calls you use for certification.
