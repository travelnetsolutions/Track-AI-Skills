# Readiness checks

This is a pre-flight built from Track's published guides. **It is not
Track's certification checklist**, which is not published. Each area names
the companion skill that checks it in depth; if that skill is installed and
the area applies, use its full checklist. Otherwise apply the short checks
here.

## Keys (track-api-getting-started, auth-model)

Source: [Which API Key Do I Need?](https://developer.trackhs.com/docs/which-api-key-do-i-need)

- Every call in the inventory is allowed for the key type it runs with.
  The Channel API section is the complete list for channel keys.
- **Fix:** a channel key calling confirm, check in, check out, their undos,
  folios, accounting, owners, housekeeping, CRM, guest communications or
  promo codes (these return `403`).
- **Review:** channel-key results that depend on channel settings the
  developer has not confirmed with the property manager (All Units
  Available, Enable Node API, All Reservations Available, Include Contact,
  Allow Custom Pricing).
- **Fix:** keys or secrets in code or committed config.

## Booking (track-api-booking)

Sources: [Creating Reservations With Reservations V2](https://developer.trackhs.com/docs/creating-reservations-with-reservations-v2), [Fraud Prevention](https://developer.trackhs.com/docs/enhancing-fraud-prevention-in-trackhs-api-requests)

Applies if the integration creates reservations.

- **Fix:** payment field not chosen from the quote's `guaranteePolicy.type`
  (`payment` for Deposit / FullDeposit, `holdCard` for Guarantee / Hold), or
  both sent.
- **Fix:** confirmation shown without reading `status` and `holdExpiresAt`;
  holds never followed up.
- **Fix:** quote's `guaranteePolicyId` not sent on create.
- **Review:** `payment.amount` set by hand; `session` and `clientIPAddress`
  missing; inline contacts without email or phone.

## Sync and caching (track-api-caching)

Sources: [Caching Data on a Local Server](https://developer.trackhs.com/docs/caching-data-on-local-server), [Change Detection and Incremental Sync](https://developer.trackhs.com/docs/change-detection-and-incremental-sync), [Rate Limits](https://developer.trackhs.com/docs/rate-limits)

Applies if the integration keeps a local copy of Track data.

- **Fix:** full reloads of units, rates, availability or reservations on a
  timer instead of delta / `updatedSince`.
- **Fix:** more than one `*UpdatedSince` parameter on a delta call.
- **Review:** timestamps without a UTC offset or with an unencoded `+`;
  cursor not saved durably or advanced on failure; pagination that stops
  at the first page; polling that could approach 10,000 requests per
  5 minutes per IP.

## Images (track-api-images)

Source: [The Image Serving Process](https://developer.trackhs.com/docs/the-image-serving-process)

Applies if the integration shows property photos.

- **Fix:** raw S3 image URLs loaded by browsers (hotlinking is unsupported
  and can be blocked).
- **Fix:** signed image URLs stored and shown after they could have
  expired (60 minutes).
- **Review:** more than 3 or 4 image sizes; no lazy loading on galleries.

## Errors and resilience (track-api-troubleshooting)

Sources: [401: Unauthorized](https://developer.trackhs.com/docs/401-unauthorized), [409: Invalid page provided](https://developer.trackhs.com/docs/status-code-409), [200: Empty Collection](https://developer.trackhs.com/docs/200-empty-collection), [Does Track have any webhooks?](https://developer.trackhs.com/docs/does-track-have-any-webhooks-that-push-data-out-to-channels)

- **Review:** `409` on paging treated as a fatal error rather than "stop
  paging"; an empty `200` treated as an error.
- **Review:** all non-2xx responses collapsed into one generic message,
  with no log of the response body.
- **Fix:** code waiting for a Track webhook (there are none; poll).

## Sandbox (Quick Start)

Source: [Quick Start Guide](https://developer.trackhs.com/docs/quick-start-guide)

- **Fix:** no way to point the integration at
  `https://api-integration-example.tracksandbox.io` without a code change.
- **Review:** a production tenant domain hardcoded anywhere.
