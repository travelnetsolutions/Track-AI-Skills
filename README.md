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
| track-api-integration | [track-api-caching](skills/track-api-caching/SKILL.md) | Reviews your integration's caching and sync code against the [Caching Data on a Local Server](https://developer.trackhs.com/docs/caching-data-on-local-server) guide and fixes what it finds. |

## Adding a skill

1. Add `skills/<skill-name>/SKILL.md`.
2. List it under the right plugin's `skills` array in `.claude-plugin/marketplace.json`, or add a new plugin entry there.
3. Only state behavior that is published on developer.trackhs.com.
4. Bump that plugin's `version` in `marketplace.json`.
