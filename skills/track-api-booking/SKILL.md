---
name: track-api-booking
description: Review and fix code that quotes and creates reservations through the Track Hospitality API (developer.trackhs.com), following Track's "Creating Reservations With Reservations V2" guide, the Fraud Prevention guide and the Create Reservation V2 reference. Use when a developer's Track bookings come back on Hold, auto-cancel after an hour, charge the wrong amount or nothing, fail with CONTACT_REQUIRED, SINGLE_PAYMENT_TYPE, PAYMENT_CHANNEL or INVALID_CANCEL_POLICY, create duplicate guests, or ignore a pinned guarantee policy; or when they ask "review my Track booking flow", "payment or holdCard", "why is my reservation on Hold", "how do I confirm a Track booking", "POST /v2/pms/reservations", "quote then book", or "send Seon session and client IP". Works in any language.
---

# Track API booking review

Audit the code path that turns a guest's checkout into a Track reservation
and bring it in line with Track's published guidance. The core lesson of the
guide: **a successful create is not a confirmed booking.** Quote first, pick
the payment field from the guarantee policy `type`, pin the policy, then read
the outcome and watch anything left on Hold.

Source of truth is the public docs. Read [references/track-booking-rules.md](references/track-booking-rules.md)
before reviewing. It lists every rule this skill checks, each tied to a
published page. **Do not tell the developer about Track behavior that is not
in that file or on developer.trackhs.com.** If you are unsure how the API
behaves, say so and point them to the docs or to Track support
(support@trackhs.com).

## Workflow

### 1. Trace the booking path

Find every call in the checkout flow and follow it from the guest's click to
the confirmation screen. Typical signals:

- Paths: `/v2/pms/quotes`, `/pms/quote`, `/v2/pms/reservations`, `/pms/reservations`, `/pms/reservations/{id}/cancel`
- Fields: `guaranteePolicyId`, `guaranteePolicy`, `payment`, `holdCard`, `paymentMethodId`, `creditCard`, `bankAccount`, `contact`, `contactId`, `holdExpiresAt`, `status`, `session`, `clientIPAddress`, `altConf`
- Error codes: `CONTACT_REQUIRED`, `SINGLE_PAYMENT_TYPE`, `PAYMENT_CHANNEL`, `INVALID_CANCEL_POLICY`
- Frontend: the Seon SDK (`@seontechnologies/seon-javascript-sdk`, `seon.init`, `seon.getSession`)

Build an inventory: the quote call and what is kept from its response, how
the create request is assembled, which payment field is chosen and on what
condition, what the code does with the response, how holds and cancellations
are followed up, which key type the integration uses, and where the guest's
IP and Seon session come from.

If the project uses the V1 create (`POST /pms/reservations`), say that the
guide this skill checks against covers V2, apply only the rules that name V1
or both, and do not claim V1 behavior beyond that.

If you find no Track booking code, say so and stop.

### 2. Check the path against the rules

Go through the checklist in [references/track-booking-rules.md](references/track-booking-rules.md).
For the path, answer:

1. Is there a quote before every create, and is its guarantee policy kept?
2. Is the payment field chosen from `guaranteePolicy.type`?
3. Is the quote's `guaranteePolicyId` sent on the create?
4. Is `payment.amount` left out?
5. Does the code read `status` and `holdExpiresAt` before telling the guest they are booked?
6. Are holds polled until they confirm or cancel?
7. Is the contact sent correctly, in a way that matches existing guests?
8. Are the Seon session and the real client IP sent?
9. Are the documented error codes handled with a useful message?
10. Is a failed or timed-out create retried safely?

### 3. Report

Use this order:

1. **Summary**: two or three lines (does a booking reliably end up
   Confirmed or correctly on Hold, and the biggest risk).
2. **Summary table**: one row per finding, most impactful first. Keep the
   fix to one short line; the detail goes in the blocks below.

   | # | Severity | Where | Rule | Fix |
   | --- | --- | --- | --- | --- |
   | 1 | High | `src/checkout/book.ts:88` | B3 | Send `holdCard` when `guaranteePolicy.type` is `Guarantee` or `Hold` |
   | 2 | High | `src/checkout/book.ts:120` | B6 | Read `status`; show "request received" for `Hold` |

   If there are no findings, say so instead of printing an empty table.

3. **Findings**: one block per row, in the same order and numbering:

```
1. [High] Card always sent as `payment`
     Where: src/checkout/book.ts:88
     Rule: B3 (payment vs holdCard by policy type)
     Why it matters: under Guarantee and Hold policies a `payment` object is
       not processed, so the card is neither charged nor stored and the
       booking stays on Hold, where it can auto-cancel.
     Fix: branch on the quote's guaranteePolicy.type: `payment` for Deposit
       and FullDeposit, `holdCard` for Guarantee and Hold.
```

Severity guide:

- **High**: payment field not chosen by policy `type`; confirmation shown
  without reading `status`; holds never followed up; policy not pinned;
  both `payment` and `holdCard` sent; no quote before create.
- **Medium**: `payment.amount` set by hand; fraud data missing or the IP is
  the proxy's; contact sent without email or phone; documented error codes
  surfaced as a generic failure; create retried blindly on timeout.
- **Low**: server-only fields sent with a channel key; Request-mode copy
  that says "confirmed"; missing logging of the applied `guaranteePolicy`.

4. **Already good**: what the code does right, so the developer keeps it.

### 4. Offer the fix

Ask before editing. When asked, implement fixes in the project's own
language and HTTP client. The reference flow in
[references/booking-flow.md](references/booking-flow.md) shows the shape;
adapt it, do not paste it.

Never put API keys, card numbers or bank details in code, logs, test
fixtures or your report. If you see any committed, flag it as a separate
High finding. If the code handles raw card data, remind the developer that
the Create Reservation V2 reference says credit card data can only be
provided by PCI-compliant vendors.
