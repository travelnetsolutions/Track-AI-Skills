# Capability map

Feature in plain language, to the endpoint(s) that deliver it. Key column
follows [auth-model.md](auth-model.md). Confirm parameters against the live
spec before quoting them.

| I want to... | Endpoint(s) | Key | Watch out for |
| --- | --- | --- | --- |
| List the properties I can sell | `GET /pms/units` | Channel or server | Channel keys see only units enabled for the channel. Server keys need `includeDescriptions=1` to get description fields |
| Show one property's details | `GET /pms/units/{unitId}` | Channel or server | Values may be inherited from the node or unit type |
| Group properties by location | `GET /pms/nodes`, `GET /pms/nodes/{nodeId}` | Channel or server | Channel keys need **Enable Node API** on the channel |
| Show amenities | `GET /pms/units/amenities`, unit `amenitiesIds[]` | Channel or server | |
| Show photos | `GET /pms/units/{unitId}/images`, `GET /pms/nodes/{nodeId}/images` | Channel or server | Never hotlink S3; use `img.trackhs.com` (see `track-api-images`) |
| Show a nightly rate calendar | `GET /v2/pms/units/{unitId}/availability` + `GET /pms/units/{unitId}/pricing` | Channel or server | Defaults to today through the max booking window; if rates or seasons are not configured out that far, no data is returned. Channel keys see only the channel's rate types |
| Search availability across a property | `GET /v2/pms/nodes/{nodeId}/availability` | Channel or server | |
| Show policies for a unit | `GET /pms/units/{unitId}/policies` | Channel or server | |
| Price a stay before booking | `POST /v2/pms/quotes` | Channel or server | Returns the guarantee policy `type` you need to choose `payment` vs `holdCard` |
| Take a booking | `POST /v2/pms/reservations` | Channel or server | A `201` can still be `Hold`; read `status` and `holdExpiresAt`. Send fraud data |
| Cancel a booking | `POST /pms/reservations/{reservationId}/cancel` | Channel or server | |
| Confirm, check in or check out | PMS Reservations API workflow actions | **Server only** | Channel keys get `403` |
| Know when a booking changes or is cancelled | `GET /v2/pms/reservations?updatedSince=` | Channel or server | No webhooks; poll. Cancelled reservations come back with `status: Cancelled` |
| Know which units changed | `GET /pms/units/delta` (and `/pms/units/types/delta`) | Channel or server | One `*UpdatedSince` param per call; `removed` is a full snapshot (see `track-api-caching`) |
| Pull more than 10,000 reservations | `GET /v2/pms/reservations` with `scroll`, or smaller date windows | Channel or server | `page * size` is capped at 10,000 |
| Show reviews | `GET /channel-management/channel/reviews?updatedSince=` | Channel (reduced) or server (full) | Server keys need the reputation management view permission |
| Message a guest | `POST /v2/pms/folios/{folio_id}/events/send` or `POST /v2/pms/guest-communications/threads/{thread_id}/send` | Server | Outbound only |
| Read a guest conversation | `GET /v2/pms/guest-communications/threads/{contextId}/events`, `GET /v2/pms/folios/{folio_id}/events` | Server | |
| Read a guest's payments and charges | `GET /pms/folios/{folioId}/transactions` | Server | The reservation's `guestBreakdown` gives totals without a folio call |
| Push dynamic pricing | `PUT` / `PATCH /v2/pms/units/{unitId}/daily-pricing` | Server | Make writes safe to repeat |
| Look up or create a guest | `GET` / `POST /crm/contacts` | Server | On reservation create you can send an inline `contact` instead (exactly one of `contact` or `contactId`) |
| Show owner statements | `GET /pms/owners/statements`, `.../{statementId}/pdf` | Server | |
| Sync accounting | `GET /pms/accounting/charges`, `GET /pms/accounting/transactions` with `updatedSince` | Server | |
| Sync housekeeping or maintenance | `GET /pms/housekeeping/work-orders`, `GET /pms/maintenance/work-orders` with `updatedSince` | Server | |
| Read custom fields | `GET /custom-fields` | Channel or server | |

If a feature is not in this table, search the live spec (the docs MCP
`search-endpoints` tool, or developer.trackhs.com) rather than guessing. If
nothing fits, say so and suggest Track support.
