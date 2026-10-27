---
name: track-api-troubleshooting
description: Diagnose a failing or surprising Track Hospitality API call (developer.trackhs.com) from an error, a response body or a symptom, and say what it means, the likely cause, how to confirm it, the fix, and who has to make it (the developer, the property manager in Track, or Track support). Use when a developer pastes a Track API error or response, or reports "401 Unauthorized", "403 Forbidden", "409 Invalid page provided", "422", "CONTACT_REQUIRED", "SINGLE_PAYMENT_TYPE", "PAYMENT_CHANNEL", "INVALID_CANCEL_POLICY", "Rate type is invalid or not active", an empty collection, missing units or reservations, missing description fields, a delta that misses changes, bookings stuck on Hold or auto-cancelled, a pinned policy being ignored, duplicate guests, images that stop loading, rate limiting, or "why does Track return X". Works without access to their code; uses it when available.
---

# Track API troubleshooting

Turn "this Track call is failing" into a short, specific diagnosis the
developer can act on, and make clear who has to act. Many Track problems are
not code bugs at all: they are the key type, or a setting the property
manager controls in Track.

Source of truth is the public docs. Read [references/symptom-catalog.md](references/symptom-catalog.md)
to match the symptom. It lists every diagnosis this skill gives, each tied to
a published page. **Do not tell the developer about Track behavior that is
not in that file or on developer.trackhs.com.** If nothing matches, say so,
suggest how to narrow it down, and point them to Track support.

## Workflow

### 1. Get the facts

From what the developer pasted, pull out:

- Method and path (for example `POST /v2/pms/reservations`)
- HTTP status, and the response body. Track errors follow RFC 7807:
  `type`, `title`, `status`, `detail`, and sometimes `code` and
  `validation_messages`. The `type` or `code` often names the error
  (`CONTACT_REQUIRED`).
- Key type in use: server, channel or Sales Link. If it is not clear, ask.
  It changes most diagnoses.
- For "missing data" problems: what they expected, what came back, and the
  query string.

Ask only for what is missing and needed. One question at a time.

If they pasted credentials, card numbers or guest personal data, do not
repeat them. Tell them to rotate any exposed key with the property manager.

If you are inside their codebase, find the call that produced the error and
read how the request is built. That usually confirms the cause.

### 2. Match it

Look up the status code, error code or symptom in
[references/symptom-catalog.md](references/symptom-catalog.md). If more than
one entry fits, rank them by how well the facts match and say what would
tell them apart.

### 3. Answer

Keep it short. Use this shape:

```
**What it means:** one or two sentences.

**Most likely cause:** the top match, tied to their facts.
(Other possibilities, if any, as a short list.)

**How to confirm:** one concrete check (a field to read, a call to make
with a server key, a setting to ask about).

**Fix:** what to change.

**Who fixes it:** Developer | Property manager (in Track: <menu path>) | Track support

**Docs:** link(s) to the published page(s).
```

If the fix is on the property manager's side, give the developer a one or
two sentence request they can send them, naming the Track setting.

If the companion skills would help them go further, mention them:
`track-api-booking` (booking flow review), `track-api-caching` (sync and
delta), `track-api-images` (photos), `track-api-getting-started` (which
endpoints and keys).

### 4. When to send them to Track support

If nothing in the catalog matches, or the facts contradict the docs, say so
plainly and suggest they email support@trackhs.com (source:
[Getting Help](https://developer.trackhs.com/docs/getting-help)). Help them
write the ticket, which should include:

- Tenant domain (for example `{customer}.trackhs.com`)
- Method, path and query string
- Request body with card data, keys and personal data removed
- Response status and body
- Time of the request with timezone
- Key type (never the key itself)
- What they expected instead
