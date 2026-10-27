---
name: track-api-getting-started
description: Orient a developer who is new to the Track Hospitality API (developer.trackhs.com) so they do not have to read 35+ specs cold. Recommends the minimal set of endpoints, the API key type each needs, and the call sequence for what they are building (booking engine or booking website, single-unit microsite, channel manager or OTA, revenue management, guest messaging, reviews on a website, damage or insurance program, owner portal, guest portal, back-office tools); explains server keys vs channel keys vs Sales Link keys and why a call returns 401 or 403; explains how Track entities relate (node, unit, unit type, reservation, contact, folio, guest communication thread, owner); and maps a feature ("show a nightly rate calendar", "take a booking", "detect cancellations") to the endpoints that deliver it. Use for "where do I start with the Track API", "which Track endpoints do I need for X", "which API key do I need", "why am I getting 403 from Track", "how do units and nodes relate", "how do I build a booking engine on Track", or "what endpoint shows Y".
---

# Track API getting started

Help a developer go from "I want to build X on Track" to a short, correct
plan: which endpoints, which key, in what order, and what to watch out for.

## Ground rules

- **Source of truth is developer.trackhs.com.** Everything in the reference
  files below comes from published guides and API reference pages, and each
  file names its sources. Do not tell the developer about Track behavior that
  is not in these files or on developer.trackhs.com. If you do not know, say
  so and point them to the docs or to Track support (support@trackhs.com).
- **Check the live spec when you can.** If the Track docs MCP server is
  available (tools like `search-endpoints`, `get-endpoint`, `list-specs`),
  use it to confirm paths, parameters and fields before you quote them. The
  reference files are a map, not a copy of the spec, and the spec wins if
  they disagree.
- **Label anything assembled, not published.** Some build paths have a
  published guide; others are assembled from the API reference. The build
  paths file marks which is which. When you give an assembled path, say so
  and suggest the developer confirm it with Track before building on it.
- **Never ask for or handle real API keys.** Keys come from the property
  manager, not from Track support and not from you.

## Reference files

| File | Use it for |
| --- | --- |
| [references/auth-model.md](references/auth-model.md) | Key types, what each can call, how results differ by key, 401 vs 403 |
| [references/build-paths.md](references/build-paths.md) | Recommended endpoint sets and call sequences per app type |
| [references/data-model.md](references/data-model.md) | How Track entities link to each other and which IDs connect them |
| [references/capability-map.md](references/capability-map.md) | Feature in plain language to endpoint |

Read only the files the question needs.

## Workflow

### 1. Work out what they are building

If the developer has not said, ask one question: what are they building, and
for whom (a property manager's own tool, or a distribution channel)? That
answer decides the key type, and the key type decides most of the plan.

If they are asking a narrow question (one key, one feature, one entity),
skip straight to the matching reference file.

### 2. Answer with the right shape

**Build path request** ("I'm building a booking engine"):

1. One-line summary: the key type(s) they need and why.
2. Endpoint table, in call order:

   | Step | Endpoint | Key | Purpose |
   | --- | --- | --- | --- |
   | 1 | `GET /pms/units` | Channel | Load the units enabled for the channel |

3. The call sequence as a short numbered flow (initial load, then the
   recurring sync, then the transactional calls).
4. Pitfalls for this path, from the reference files.
5. Links to the published guides behind it.
6. If the path is assembled rather than published, say so here.

**Key question** ("which key do I need", "why 403"): answer from
[references/auth-model.md](references/auth-model.md), name the key type, and
say who issues it (the property manager).

**Data model question**: give the relevant slice of the entity graph from
[references/data-model.md](references/data-model.md), with the ID fields that
connect each pair, and the endpoint that reads each entity.

**Feature question**: give the endpoint(s) from
[references/capability-map.md](references/capability-map.md), the key, and
the one gotcha that most often bites.

### 3. Point to the deeper skills

When the plan involves these, mention the companion skills in this repo:

- Keeping a local copy of units, rates, availability or reservations:
  `track-api-caching`
- Showing property photos: `track-api-images`

### 4. If you are inside their codebase

If the developer runs this in a repo that already calls Track, you can also
check their existing calls against [references/auth-model.md](references/auth-model.md):
flag any endpoint their key type cannot call (likely `403`), and any
channel-key call whose result depends on a channel setting they may not have
asked the property manager about. Report file and line, the endpoint, the
key type in use, and what to change. Ask before editing.
