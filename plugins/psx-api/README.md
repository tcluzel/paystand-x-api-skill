# psx-api plugin (BETA)

> ⚠️ **Beta.** This plugin packaging is new and may change. The skill content itself is stable; if the plugin install misbehaves, use the manual skill install from the [repo README](../../README.md).

Claude Code plugin that bundles the **Paystand X (PSX) API** skill — deep, accurate knowledge of the Paystand X public REST API for building ERP / middleware / iPaaS integrations.

## Install

```
/plugin marketplace add tcluzel/paystand-x-api-skill
/plugin install psx-api@paystand
```

Then it loads automatically when you work on a Paystand X integration, or invoke it directly with **`/psx-api`**.

## What's inside

The plugin ships the skill at `skills/psx-api/` — `SKILL.md` plus reference files (endpoints, webhooks, quickstart, connector pattern, gotchas, implementation lessons, sandbox testing) and a runnable Postman collection. See the [main README](../../README.md) for the full contents list and the authoritative docs link.
