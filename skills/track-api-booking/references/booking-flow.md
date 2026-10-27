# Reference booking flow

Language-neutral shape of a safe quote-and-book flow. Adapt it to the
project's HTTP client, framework and storage. Rule ids refer to
[track-booking-rules.md](track-booking-rules.md). The guide's
[Quote, Check the Guarantee Policy, and Book](https://developer.trackhs.com/recipes/quote-check-guarantee-policy-and-book)
recipe has working PHP and Node.js.

## Frontend

```
seon.init({ behavioralDataCollection: { targets: 'input, textarea, select',
                                        formFilloutDurationTargetId: 'paymentForm' } })
session = await withTimeout(seon.getSession(), 3s) or null    # B10: still book if missing
send { stay, guest, card, session } to backend
```

## Backend

```
# 1. Quote (B1)
quote = POST /v2/pms/quotes { unitId, arrivalDate, departureDate, occupants }
policyType = quote.guaranteePolicy.type                         # B2: type, never name
policyId   = quote.guaranteePolicyId

# 2. Pick the payment field (B3, B11, B12)
if policyType in ["Deposit", "FullDeposit"]:
    paymentField = { "payment":  { creditCard: card } }        # no amount (B5)
else:                                                          # Guarantee, Hold
    paymentField = { "holdCard": { creditCard: card } }

# 3. Create (B4, B8, B9, B10, B16)
ref = new_booking_reference()
body = {
    unitId, arrivalDate, departureDate, occupants,
    guaranteePolicyId: policyId,                               # flat, from the quote
    contact: { firstName, lastName, primaryEmail, cellPhone }, # or contactId, never both
    session: session,                                          # may be null
    clientIPAddress: leftmost(X-Forwarded-For) or remote_addr,
    altConf: [ref],
    ...paymentField
}
try:
    res = POST /v2/pms/reservations body
except Timeout:
    res = find_by_alt_conf(ref)                                # B16: check before retrying
    if not res: retry once

# 4. Read the outcome (B6)
if res.status == "Confirmed":
    show "Booking confirmed"
elif res.status == "Hold":
    show "Request received"                                    # not "confirmed"
    watch(res.id, res.holdExpiresAt)                           # B7
log(res.id, res.guaranteePolicy.type, res.guestBreakdown.netPayments)

# 5. Errors (B15)
on 422 CONTACT_REQUIRED      -> fix request (exactly one contact field)
on SINGLE_PAYMENT_TYPE       -> fix request (one payment method)
on PAYMENT_CHANNEL           -> config: property manager's channel payment settings
on INVALID_CANCEL_POLICY     -> config: property manager's cancellation policies
```

## Watching holds (B7)

```
every few minutes:
    for r in paginate(GET /v2/pms/reservations?updatedSince={cursor}):   # see track-api-caching
        if r.id in watched:
            if r.status == "Confirmed": notify guest, unwatch
            if r.status == "Cancelled": notify guest and ops, unwatch
```

Poll faster than the shortest `holdExpiresAt` window you see, so a hold that
expires is noticed before the guest finds out another way.
