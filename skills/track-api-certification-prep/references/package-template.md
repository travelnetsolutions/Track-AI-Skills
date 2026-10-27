# Certification package template

Write the package with this structure. Replace everything in `<angle
brackets>`. Leave a visible `TODO (developer):` wherever only the
developer can supply or confirm the answer. Never include keys, secrets,
card data or guest personal data.

````markdown
# Track integration certification package: <Product name>

Prepared <YYYY-MM-DD> by <company>. Contact: <name, email>.

## 1. Integration overview

<Two to five sentences: who uses the product, what it does with Track, and
the key functionality it covers. For example: "Acme Booking is a direct
booking website for property managers. It reads units, availability and
pricing from Track, lets guests quote and book stays with a card, and lets
guests cancel. It keeps a local cache of listing content and syncs changes
every 15 minutes.">

**Key types used:** <channel key | server key | both, and what each is used for>

**Main flows:**
1. <flow, e.g. Listing sync>
2. <flow, e.g. Search and book>
3. <flow, e.g. Guest cancellation>

## 2. API call list

| # | Method | Path | Key | Purpose | Trigger and frequency |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `/pms/units` | Channel | Initial load of listings | Nightly full reconciliation |
| 2 | GET | `/pms/units/delta?contentUpdatedSince=` | Channel | Detect content changes | Every 15 min |

## 3. Sandbox confirmation

Testing has been run against `https://api-integration-example.tracksandbox.io`.

TODO (developer): confirm this is true, or name the domain you tested on.

## 4. Fresh test run

Track asks for a new set of test calls and the time and date they ran.

| # | Scenario | Date | Start time | End time | Timezone | Calls made | Resulting ids | Outcome |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | <Search and book a Deposit-policy unit> | | | | | <quote, create> | <reservation id> | <Confirmed / Hold> |
| 2 | <Guest cancels> | | | | | <cancel> | <reservation id> | <Cancelled> |

TODO (developer): fill in after running each scenario against the sandbox.

## 5. Known open items

<Anything from the readiness pass you chose not to fix before submitting,
with the reason. "None" if there are none.>
````

## Cover message

Draft this for the developer to send to their Track technical contact. Do
not send it.

```
Subject: Certification request: <Product name> Track integration

Hi <Track contact>,

We have finished development and testing of our Track integration and would
like to start certification. Attached is our package with the integration
overview, the API calls we use, our sandbox confirmation, and the times of
our fresh test run on <date> between <start> and <end> <timezone>.

Let us know if you need anything else.

<Name>
<Company>
```
