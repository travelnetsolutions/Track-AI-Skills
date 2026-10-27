# Track booking rules

Every rule here comes from a page on developer.trackhs.com. The source is
named on each rule. Rules marked **General practice** are standard
engineering, not claims about how Track behaves. Present them that way.

Sources:

- **Guide**: [Creating Reservations With Reservations V2](https://developer.trackhs.com/docs/creating-reservations-with-reservations-v2)
- **Reference**: API reference, Create Reservation V2 (`POST /v2/pms/reservations`)
- **Fraud**: [Fraud Prevention](https://developer.trackhs.com/docs/enhancing-fraud-prevention-in-trackhs-api-requests)
- **Keys**: [Which API Key Do I Need?](https://developer.trackhs.com/docs/which-api-key-do-i-need)
- **Errors**: [SINGLE_PAYMENT_TYPE](https://developer.trackhs.com/docs/single_payment_type), [PAYMENT_CHANNEL](https://developer.trackhs.com/docs/payment_channel), [INVALID_CANCEL_POLICY](https://developer.trackhs.com/docs/invalid_cancel_policy)
- **Change Detection**: [Change Detection and Incremental Sync](https://developer.trackhs.com/docs/change-detection-and-incremental-sync)

## How Track decides Hold vs Confirmed (Guide)

Every V2 reservation is created in `Hold`. It is promoted to `Confirmed` only
when both hold, checked in this order:

1. **The effective reservation mode is Confirmed.** Set per channel, can be
   overridden per unit for that channel. In Request mode the reservation
   stays `Hold` whatever else is sent. A server key has no channel and
   behaves as Confirmed mode.
2. **The guarantee policy is satisfied:**

| Policy `type` | Send | Promoted to Confirmed when |
| --- | --- | --- |
| `Deposit` / `FullDeposit` | `payment` | Payments collected meet or exceed the first payment plan installment |
| `Guarantee` | `holdCard` | A card is stored as the folio payment method (not charged) |
| `Hold` | `holdCard` | Never automatically; the property manager approves it in Track |

## Checklist

### B1. Quote before every create (Guide)
Call `POST /v2/pms/quotes` for the stay first. It is free, creates nothing,
works with channel and server keys, and returns the `guaranteePolicy` (with
`type`), `guaranteePolicyId`, cancellation policy and `paymentPlan` that
apply. The quote applies the same policy rules as the create.
**Flag:** a create built without a fresh quote for the same stay, or a quote
whose policy is thrown away.

### B2. Branch on `type`, never on `name` (Guide)
The policy `name` is free text chosen by the property manager and has no
effect. A policy named "Standard Guarantee Policy" can be type `Deposit`.
**Flag:** any condition on `guaranteePolicy.name`, a hardcoded policy name or
ID, or one payment field used for every property manager.

### B3. `payment` or `holdCard`, by type, never both (Guide)
`payment` for `Deposit` / `FullDeposit`; `holdCard` for `Guarantee` and
`Hold`. Sending both is rejected with a validation error. A `payment` object
is currently not processed under `Guarantee` or `Hold`: the card is neither
charged nor stored and the reservation stays `Hold`.
**Flag:** card always sent as `payment`; card always sent as `holdCard`
(a `Deposit` booking then collects nothing and can auto-cancel at
`holdExpiresAt`); both fields set.

### B4. Pin the quote's `guaranteePolicyId` (Guide, Reference)
Send the quote's `guaranteePolicyId` on the create. Without it Track selects
the policy again at booking time, and the policy can depend on how far out
the arrival is, so the booking can land under a different policy than the
one the code branched on. Policy fields are sent **flat**
(`guaranteePolicyId`, `cancellationPolicyId`); responses return them
**nested** (`guaranteePolicy`, `cancellationPolicy`).
With a channel key, an active guarantee policy configured on the channel
overrides `guaranteePolicyId` entirely, and a pinned policy must be active
and applied to or inherited by the unit. That is a Track configuration
matter for the property manager, not a code change.
**Flag:** no `guaranteePolicyId` on create; nested `guaranteePolicy` objects
copied from a response into a request.

### B5. Leave `payment.amount` out (Guide, Reference)
Track calculates what is due. With `amount` omitted it charges the payment
plan installments dated today, plus any travel insurance or damage waiver
due with the first payment. If the earliest installment is a future date,
nothing is charged today: the card is stored, no charge is attempted, the
reservation stays `Hold`, and the request still succeeds. The quote's
`paymentPlan` shows this before booking.
**Flag:** `amount` computed from the quote total, a hardcoded deposit
percentage, or any client-side math, without a documented reason.

### B6. A `201` is not a confirmation (Guide)
Read these on every create response before telling the guest they are
booked:

| Field | Meaning |
| --- | --- |
| `status` | `Confirmed` or `Hold`. Anything on `Hold` needs follow-up |
| `holdExpiresAt` | When set, the reservation is cancelled automatically at that time if still on `Hold` |
| `guaranteePolicy.type` | Which rule decided the outcome |
| `guestBreakdown.netPayments` | What was actually collected |

In Request mode, or under a `Hold` policy, `Hold` is expected: show the
guest "request received", not "confirmed".
**Flag:** success UI or confirmation email triggered by the HTTP status
alone; `status` never read; a `Hold` shown as confirmed.

### B7. Follow up on holds (Guide, Change Detection)
The auto-cancel window is set by each property manager per policy, not a
Track default, and windows as short as 60 minutes are in use. Channel
reservations are not exempt. There is no webhook for status changes: poll
`GET /v2/pms/reservations?updatedSince={timestamp}` often enough to see a
cancellation before the guest does. Cancellations come back as updates with
`status: Cancelled`.
**Flag:** no polling after create; polling too slow for the
`holdExpiresAt` windows seen; code waiting for a webhook.

### B8. Exactly one of `contact` or `contactId` (Guide, Reference)
Sending both, or neither, returns `422` with `CONTACT_REQUIRED`. An inline
`contact` needs `firstName` and `lastName`.
**Flag:** both set, neither set, or an inline contact without both names.

### B9. Contacts that match (Reference)
Track tries to find an existing contact, matching in order on `id`,
`primaryEmail`, `secondaryEmail`, `cellPhone`, `homePhone`, `otherPhone`,
`proxyEmail`. A contact sent with no email or phone cannot be matched and
creates a new contact every time. When a match is found, Track updates it
with the attributes sent, which can make a unique email or phone collide
with a second contact and fail. Emails and phones must be unique across
contacts. To handle duplicates yourself, look the guest up with the CRM
Contacts API (server keys only) and send `contactId`.
**Flag:** inline contacts with no email or phone; no handling for a
duplicate-contact failure.

### B10. Fraud data on every reservation request (Fraud, Reference)
Send `session` (the Seon v6 JavaScript agent payload from
`seon.getSession()`) and `clientIPAddress` (the guest's IP) with each
reservation request. Behind a load balancer or proxy, take the leftmost IP
in `X-Forwarded-For`. Still send the request if the session is missing
because the script did not run; the guide suggests a timeout on the JS.
**Flag:** no `session` or `clientIPAddress`; `clientIPAddress` set to the
server's own IP or the proxy's; a booking blocked when Seon fails to load.

### B11. One payment method per `payment` (Errors: SINGLE_PAYMENT_TYPE)
The `payment` object must carry exactly one method (credit card, e-check,
net commission, and so on), especially when `amount` is sent.
**Flag:** `payment` objects with two methods, or none.

### B12. Payment methods the key and channel allow (Errors: PAYMENT_CHANNEL, Keys, Reference)
Channel keys can only use electronic methods (credit card and e-check).
Server keys can also use cash, check and other methods. A method the
Distribution Channel is not configured for (for example net commission on a
card-and-e-check channel) returns `PAYMENT_CHANNEL`. That setting is in
Track under Configuration > PMS Setup > Distribution Channels.
**Flag:** non-electronic methods sent with a channel key.

### B13. Server-only fields with a channel key (Reference, Keys)
The reference marks `typeId` as server/unit only, `channelId` as server
context only, and `discount` as not available in channel key context. With a
channel key, the reservation type is set by the channel.
**Flag:** these fields sent from a channel-key integration. Do not claim
what Track does with them; say the reference marks them server only.

### B14. Cancellation policy errors are configuration (Errors: INVALID_CANCEL_POLICY)
`INVALID_CANCEL_POLICY` means the policy sent is inactive or not available
to the unit, or the unit has no active cancellation policy. With a channel,
the cancellation policy is set automatically from the unit (Reference). The
fix is in Track (Configuration > PMS Setup > Cancellation Policies, or the
unit profile's Reservations tab).
**Flag:** hardcoded `cancellationPolicyId` values in a channel integration;
this error shown to guests as a generic failure.

### B15. Handle documented errors usefully (Errors, Guide, General practice)
Map `CONTACT_REQUIRED`, `SINGLE_PAYMENT_TYPE`, `PAYMENT_CHANNEL`,
`INVALID_CANCEL_POLICY` and payment validation errors to a clear message and
a log entry with the response body (minus card data). Configuration errors
(PAYMENT_CHANNEL, INVALID_CANCEL_POLICY, channel policy overrides) are for
the property manager to fix, not the guest.
**Flag:** all non-2xx responses collapsed into "Something went wrong".

### B16. Do not retry a create blindly (General practice)
A create that times out may still have succeeded. Before retrying, check
whether the booking exists. One way that uses documented features: send your
own booking reference in `altConf`, then search with
`GET /v2/pms/reservations?search={yourReference}` (the `search` parameter
matches alternate confirmation numbers). `search` is a broad prefix search
across many fields, so confirm the returned reservation's alternate
confirmation number equals your reference exactly. Say this is a suggested
pattern, not a Track requirement.
**Flag:** automatic retry of `POST /v2/pms/reservations` on timeout or 5xx
with no existence check.

### B17. Card data and PCI (Reference)
The reference states credit card data can only be provided by vendors who
are PCI compliant. Saved methods can be reused with `paymentMethodId`, which
must be attached to the reservation's contact.
**Flag:** raw card numbers logged, stored, or passed through systems the
developer has not confirmed are in PCI scope. Raise it; do not judge their
compliance.

### B18. Cancelling (Keys, Reference)
Guest cancellations go to `POST /pms/reservations/{reservationId}/cancel`,
which channel keys can call. `unitStatus` on cancel is server-key only, and
a channel cancel always cancels the reservation's work orders. Confirm,
check in, check out and their undos need a server key (channel keys get
`403`).
**Flag:** a channel-key integration calling confirm or check-in endpoints,
or sending `unitStatus` on cancel.

## Troubleshooting map (Guide)

| Symptom | Likely cause | Rule |
| --- | --- | --- |
| Every reservation comes back `Hold` | `Guarantee` policy, card sent as `payment` | B3 |
| `Hold` at one property manager, `Confirmed` at another, same payload | Different policy types | B2, B3 |
| Reservation's policy differs from the quote's | `guaranteePolicyId` not sent on create | B4 |
| Auto-cancelled about an hour after booking | `Deposit` policy with `holdCard`, nothing collected before `holdExpiresAt` | B3, B7 |
| `Deposit` booking stays `Hold`, nothing charged | No installment due today | B5 |
| Pinned `guaranteePolicyId` ignored | Channel key, channel has its own active policy | B4 |
| Still `Hold` with the right payment field | Request mode, or policy type `Hold` | Expected (B6) |
| `422 CONTACT_REQUIRED` | No contact, or both `contact` and `contactId` | B8 |
| Validation error on payment | Both `payment` and `holdCard` sent | B3 |

## Out of scope: do not claim

The published docs do not state these. Do not assert them. If the developer
asks, say the docs do not cover it and suggest they contact Track:

- Whether the create endpoint is idempotent, or accepts an idempotency key
- What Track does with server-only fields sent from a channel key
- Default or typical `holdExpiresAt` windows beyond "set per policy, as
  short as 60 minutes"
- How fraud scoring uses the session or IP, or whether it can block a booking
- V1 create (`POST /pms/reservations`) behavior beyond what the key guide says
