# Track AI Skills

Agent skills for developers building on the [Track Hospitality APIs](https://developer.trackhs.com).

## Install (Claude Code)

```
/plugin marketplace add travelnetsolutions/Track-AI-Skills
/plugin install track-api-integration@track-ai-skills
```

## Other agents

Each skill is a plain folder with a `SKILL.md`. Copy the folder under
`plugins/<plugin>/skills/` into wherever your agent loads skills from.

## Skills

| Plugin | Skill | What it does |
| --- | --- | --- |
| track-api-integration | [track-api-caching](plugins/track-api-integration/skills/track-api-caching/SKILL.md) | Reviews your integration's caching and sync code against the [Caching Data on a Local Server](https://developer.trackhs.com/docs/caching-data-on-local-server) guide and fixes what it finds. |

## Adding a skill

1. Pick the plugin it belongs to, or add a new one under `plugins/` with its own `.claude-plugin/plugin.json` and an entry in `.claude-plugin/marketplace.json`.
2. Add `skills/<skill-name>/SKILL.md`.
3. Only state behavior that is published on developer.trackhs.com.
4. Bump the plugin `version` in both `plugin.json` and `marketplace.json`.
