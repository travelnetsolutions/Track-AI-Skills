# Track AI Skills

Agent skills for developers building on the [Track Hospitality APIs](https://developer.trackhs.com).

## Install (Claude Code)

```
/plugin marketplace add travelnetsolutions/Track-AI-Skills
/plugin install track-api-integration@track-ai-skills
```

## Other agents

Each skill is a plain folder with a `SKILL.md` under `skills/`. Copy the
folder into wherever your agent loads skills from.

## Skills

| Plugin | Skill | What it does |
| --- | --- | --- |
| track-api-integration | [track-api-getting-started](skills/track-api-getting-started/SKILL.md) | Start here. Recommends the endpoints, key type and call order for what you are building, explains server vs channel keys and 401 vs 403, maps how Track entities connect, and turns a feature into the endpoints that deliver it. |
| track-api-integration | [track-api-booking](skills/track-api-booking/SKILL.md) | Reviews your quote-and-book flow against [Creating Reservations With Reservations V2](https://developer.trackhs.com/docs/creating-reservations-with-reservations-v2): payment vs holdCard by policy type, pinned policy, Hold follow-up, contacts, fraud data and documented errors. |
| track-api-integration | [track-api-caching](skills/track-api-caching/SKILL.md) | Reviews your integration's caching and sync code against the [Caching Data on a Local Server](https://developer.trackhs.com/docs/caching-data-on-local-server) guide and fixes what it finds. |
| track-api-integration | [track-api-certification-prep](skills/track-api-certification-prep/SKILL.md) | Gets you ready for Track certification: inventories your API calls, checks your sandbox setup, runs a readiness pre-flight, plans the fresh test run, and writes the package Track asks for (overview, call list, sandbox confirmation, test log). |
| track-api-integration | [track-api-images](skills/track-api-images/SKILL.md) | Reviews how your site stores and serves Track listing images against [The Image Serving Process](https://developer.trackhs.com/docs/the-image-serving-process) guide: proxy use, signed URL expiry, sizes and lazy loading. |
| track-api-integration | [track-api-troubleshooting](skills/track-api-troubleshooting/SKILL.md) | Paste a Track API error, response or symptom and get what it means, the likely cause, how to confirm it, the fix, and who has to make it (you, the property manager, or Track support). |

## Adding a skill

1. Add `skills/<skill-name>/SKILL.md`.
2. List it under the right plugin's `skills` array in `.claude-plugin/marketplace.json`, or add a new plugin entry there.
3. Only state behavior that is published on developer.trackhs.com.
4. Bump that plugin's `version` in `marketplace.json`.
