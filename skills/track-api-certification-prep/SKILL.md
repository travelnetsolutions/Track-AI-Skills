---
name: track-api-certification-prep
description: Get a Track Hospitality API integration (developer.trackhs.com) ready for Track's certification review. Scans the codebase for every Track API call, checks the integration has been tested against the Track sandbox, runs a readiness pass using the companion Track skills, plans the fresh set of test calls Track asks for, and writes the certification package Track requests (integration overview, API call list, sandbox confirmation, test run log) plus a cover message. Use when a developer says they are done building on Track and asks "how do we get certified", "prepare for Track certification", "are we ready to go live with Track", "what does Track need from us", "write our integration overview", "list our API calls for Track", or "pre-certification checklist".
---

# Track API certification prep

Help a developer arrive at Track's certification review with everything
Track asks for, and with the avoidable problems already fixed.

## Ground rules

- **What Track asks for is published; how Track judges it is not.** The
  requirements in [references/certification-requirements.md](references/certification-requirements.md)
  come from the Quick Start Guide. Track's review criteria are not
  published. The readiness pass in step 3 is this skill's own pre-flight
  based on Track's published guides, **not** Track's certification
  checklist. Say so every time you show it, and never tell the developer
  they "will pass" or "are certified".
- **Do not tell the developer about Track behavior that is not on
  developer.trackhs.com** or in the reference files of this skill and its
  companions.
- **Never make live API calls yourself.** The test run is the developer's
  integration exercising the sandbox, on their schedule. You plan it and
  record it.
- **Never put API keys, secrets, card data or guest personal data** in the
  package, the log, the cover message, or your reply.

## Workflow

### 1. Inventory the integration

Find every Track API call in the codebase (hosts `*.trackhs.com`,
`*.tracksandbox.io`, a configured tenant domain ending in `/api`; paths
under `/pms/`, `/v2/pms/`, `/crm/`, `/channel-management/`,
`/custom-fields`). For each, record:

- Method and path (normalize ids: `/pms/units/{unitId}`)
- Purpose in plain language (from the surrounding code)
- Key type it runs with (server, channel, Sales Link), from how the client
  is configured
- What triggers it (guest action, schedule, startup) and how often
- File and line

Also find how the base URL is configured per environment.

### 2. Check the sandbox setup

Track asks the developer to confirm testing ran against
`https://api-integration-example.tracksandbox.io`. Check:

- Is there a test or staging configuration that points at that domain?
- Is a production tenant domain hardcoded anywhere, or is the domain
  configurable?
- Do test credentials come from environment or secret storage, not code?

Report what you find. If the integration has been tested against a
different sandbox, say that the Quick Start names this specific domain and
the developer should confirm with their Track technical contact.

### 3. Readiness pass (this skill's pre-flight, not Track's criteria)

Run the checks in [references/readiness-checks.md](references/readiness-checks.md)
against the inventory. If a companion skill is installed and the area
applies, tell the developer you are using it and apply its full checklist;
otherwise apply the short checks listed for that area.

Report it like this:

1. **Summary**: two or three lines, starting with: "This is a pre-flight
   based on Track's published guides. Track's own certification criteria
   are not published."
2. **Readiness table**, most important first:

   | # | Area | Status | Where | Action |
   | --- | --- | --- | --- | --- |
   | 1 | Keys | Fix | `src/kiosk.js:12` | Check-in needs a server key; channel keys get `403` |
   | 2 | Booking | OK | `src/book.js` | |

   Status is one of **Fix** (likely to cause errors or bad data), **Review**
   (worth a look before submitting), **OK**, or **N/A**.

3. For each **Fix**, one short block: where, which guide it comes from,
   why it matters, the fix. Point to the companion skill for depth.

Ask whether the developer wants to fix anything before moving on. Offer to
make the fixes; ask before editing.

### 4. Plan the fresh test run

Track asks for "a new/fresh set of test calls" and the time and date they
ran. Build a test plan from the inventory: one scenario per key user flow
(for example "search and book a Deposit-policy unit", "guest cancels",
"overnight sync"), each listing the API calls it should produce, in order,
and what to check in the response. Include a negative case where the code
handles one (for example a hold that is not confirmed).

Give the developer a run log to fill in as they run it, with date, time
**and timezone**, scenario, calls made, and resulting ids (reservation id,
unit id). Track reviews activity in its API logs, so exact times matter.

### 5. Write the package

Ask before writing any file. Then write `track-certification-package.md` in
the repo root (or where the developer prefers), using
[references/package-template.md](references/package-template.md):

- Integration overview (from steps 1 and 2, in the developer's terms)
- API call list (from step 1)
- Sandbox confirmation (from step 2, with blanks the developer must
  confirm; never assert it on their behalf)
- Test run log (from step 4, left for the developer to fill in)
- Open items from step 3 they chose not to fix, so Track sees them up front

Then draft the cover message to their Track technical contact. Leave
placeholders for anything you do not know. Do not send it.

### 6. Close

Tell the developer what Track does next, from the Quick Start: Track's
technical team reviews the activity in the API logs; if they cannot proceed
they give guidance and ask for re-testing; if it is satisfactory they issue
a certificate of integration with resources for going live with mutual
customers.
