# Track data model

How the entities a new integration touches connect to each other. Every link
below is an ID field or a nested path documented in the API reference
(Channel API, PMS Units, PMS Nodes, PMS Reservations V2, PMS Folios, PMS
Owners, Guest Communications). Confirm field names against the live spec
before quoting them; this file is a map, not the spec.

## The graph

```
Node (location / property / building)
 │  parentId → parent Node              nodes nest into a hierarchy
 │
 └─ Unit  (nodeId)                      the bookable rental
     │  unitTypeId / typeId → Unit Type   optional grouping; can be booked by type
     │  lodgingTypeId → Lodging Type
     │  amenitiesIds[] → Amenities
     │  /pms/units/{unitId}/images         photos (nodes have /pms/nodes/{nodeId}/images)
     │  /pms/units/{unitId}/pricing        rate types and rates
     │  /v2/pms/units/{unitId}/availability
     │
     └─ Reservation  (unitId, unitTypeId)
          │  contactId → Contact (CRM)        the guest; embedded as _embedded.contact
          │  folioId → Folio                  money: charges, payments, balance
          │  channelId → Distribution Channel where it was booked
          │  guaranteePolicyId, cancellationPolicyId → Policies
          │  typeId → Reservation Type;  rateTypeId → Rate Type
          │
          ├─ Folio  (folioId)
          │    /pms/folios/{folioId}/transactions
          │    /v2/pms/folios/{folio_id}/events     guest comms events tied to the stay
          │
          └─ Review  (one per reservation; /channel-management/channel/reviews)

Guest Communication Thread
   a collection of events; may or may not be attached to a folio (reservation),
   and may or may not be attached to a contact
   /pms/guest-communications/threads, /v2/pms/guest-communications/threads/{contextId}/events

Owner
   /pms/owners/{ownerId}/units → the units they own
   /pms/owners/statements      → statements, with their reservations and transactions
```

## Things that surprise new developers

- **Units carry inherited values.** Many unit settings can come from the
  node or unit type. `GET /pms/units` returns data flattened with the node
  and inherited values; the `computed` and `inherited` query flags control
  this on the PMS Units API.
- **Reservations can target a unit type.** Create takes `unitId` or
  `unitTypeId`. A reservation may not have a specific unit yet
  (`isUnitAssigned`).
- **The folio is where money lives.** The reservation response carries a
  `guestBreakdown` summary, but transactions and payments are folio
  resources, and folios need a server key.
- **Policies are flat on create, nested on read.** Send `guaranteePolicyId`
  and `cancellationPolicyId`; the response returns `guaranteePolicy` and
  `cancellationPolicy` objects. Do not copy the response shape back into a
  request.
- **Cancelled reservations stay in the list.** Cancellation is a status
  change (`status: Cancelled`), not a deletion.
- **What you can see depends on your key.** With a channel key, units,
  nodes and reservations are limited to what the channel is set up to see.
  See [auth-model.md](auth-model.md).
