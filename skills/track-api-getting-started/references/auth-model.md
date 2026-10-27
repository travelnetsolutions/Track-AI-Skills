# Track auth model

Sources:

- [Which API Key Do I Need?](https://developer.trackhs.com/docs/which-api-key-do-i-need)
- [Channel keys vs Server keys](https://developer.trackhs.com/docs/channel-keys-vs-server-keys)
- [401: Unauthorized](https://developer.trackhs.com/docs/401-unauthorized)
- Spec descriptions on each API section ("When used externally, this API
  requires a server context key")

## Key types

| Key | Issued by | Built for |
| --- | --- | --- |
| **Server key** | The property manager, in Track under Configuration > Company Setup > API Keys | The property manager's own integrations, or a partner trusted with full access to their data |
| **Channel key** | The property manager, in Track under Configuration > PMS Setup > Distribution Channels | Distribution channels: channel managers, OTAs, CRS providers, booking websites |
| **Sales Link key** | The property manager, for a Sales Link integration | Pushing customer and ancillary data into Track CRM and Pulse |

Track support cannot create or hand out keys. The developer asks the property
manager they are integrating with, and says which type they need.

The spec descriptions also mention a **user context**, where "endpoints may
be restricted based on role". The key guide covers only server, channel and
Sales Link keys for external integrations; do not describe user context
beyond that.

Requests authenticate with HTTP Basic (key as username, secret as password)
or HMAC. The base URL is the property manager's tenant domain plus `/api`,
for example `https://{customer}.trackhs.com/api`. The sandbox used for
certification is `https://api-integration-example.tracksandbox.io`.

## The one rule to remember

**The Channel API section of the docs is the complete list of what a channel
key can call.** Plan on everything outside it needing a server key.

## Capability matrix

| Endpoint family | Server key | Channel key | Sales Link key |
| --- | --- | --- | --- |
| Units, unit types, nodes, amenities, images, custom fields | Yes | Yes, read only, for units and nodes enabled for the channel | No |
| Unit and unit type delta | Yes | Yes | No |
| Availability, pricing, unit policies | Yes | Yes, for units enabled for the channel | No |
| Quotes | Yes | Yes, for units enabled for the channel | No |
| Create and read reservations (V1 and V2) | Yes | Yes, with the differences below | No |
| Cancel a reservation | Yes | Yes | No |
| Confirm, check in, check out, undo check-in, undo check-out | Yes | **No** (`403`) | No |
| Channel reviews | Yes, full payload | Yes, reduced payload | No |
| Sales Link push and search | Yes | No | Yes |
| Everything else: folios and payments, accounting, owners and statements, housekeeping and maintenance, CRM, promo codes, rate types, reservation types, policy setup | Yes | No | No |

Guest Communications is not in the guide's matrix, but its spec states that
external use requires a server context key.

## Same endpoint, different result

A channel key sees the property through its channel's settings in Track. The
same call can return less than it does with a server key, with no error.

| Area | With a channel key |
| --- | --- |
| Units | Only units the channel is added to, unless the channel has **All Units Available** |
| Nodes | Node endpoints need **Enable Node API** on the channel |
| Reservations | Only reservations made on this channel, unless the channel has **All Reservations Available** |
| Contact details on reservations | Only when the channel has **Include Contact** |
| Custom (`breakdown`) pricing on create | Only when the channel has **Allow Custom Pricing** |
| Rate types on pricing | Only the rate types available to the channel for that unit |
| Guarantee policy on Reservations V2 | An active policy on the channel overrides `guaranteePolicyId` |
| Reservation type | Set by the channel; `typeId` is for server keys |
| Hold vs Confirmed | Follows the channel's reservation mode; in Request mode new reservations stay on Hold. A server key behaves as Confirmed mode |
| Payment methods on create | Electronic only (card, e-check). Server keys can also record cash, check and others |
| Cancelling | `unitStatus` is server-key only; a channel cancel always cancels the reservation's work orders |
| Reviews | Body and ratings without reservation, guest and unit context |

If a channel-key call returns less than expected and no error, the developer
should check these settings with the property manager before assuming a bug.

## 401 vs 403

| Response | Meaning | What to do |
| --- | --- | --- |
| `401 Unauthorized` | Track could not authenticate: wrong key or secret, or a key for a different tenant | Check key, secret and tenant domain. For a channel key, check the channel is active in Track |
| `403 Forbidden` | The key is valid but this key type cannot call this endpoint | Check the matrix. Usually a server key is needed |

A server key can also get `403` on a few endpoints that check a Track
permission, for example Get Reviews (needs
`reputation_management.reputation_management_view`). There the fix is the
permission, not the key type.

## Choosing

- **Distributing inventory or taking bookings for guests** (channel manager,
  OTA, booking site): channel key, Channel API only. Add a server key from
  the property manager if you also need to confirm, check in or out, or read
  folios and accounting.
- **The property manager, or back-office tools for one** (accounting,
  owners, housekeeping, CRM): server key. It reaches nearly everything, so
  store it like a password.
- **Pushing customer or ancillary data into Track CRM**: Sales Link key.

One integration can hold more than one key. A common pattern is a channel
key for distribution plus a server key for the back-office calls a channel
key cannot make.

## Rate limit

10,000 requests every 5 minutes, per IP. Further limits can be applied case
by case for abuse. Source: [Rate Limits](https://developer.trackhs.com/docs/rate-limits).
