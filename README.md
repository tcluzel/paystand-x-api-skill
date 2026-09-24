# Paystand X (PSX) API — Agent Skill

A ready-to-install **Agent Skill** that gives an AI coding assistant (Claude Code, or any tool that supports the [Agent Skills open standard](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)) deep, accurate knowledge of the **Paystand X public REST API** — so it can help you build an ERP / middleware / iPaaS integration without re-reading the docs every time.

It covers authentication, the full endpoint catalog, webhooks, ERP sync patterns, sandbox testing, reconciliation (including the journal-entry model), a step-by-step first-integration quick start, a runnable Postman collection, and the non-obvious gotchas that bite real integrations.

> Built from Paystand's official developer docs plus real partner-integration experience. Contains **no secrets** — only public sandbox test credentials that are already published in Paystand's docs. Safe to share.

---

## What's inside

```
psx-api/
├── SKILL.md                        # entry point: overview, auth, base URLs, sync order, critical issues
├── references/
│   ├── quickstart.md               # ordered first integration: token → customer → invoice → PDF → pay → webhook
│   ├── endpoints.md                # full endpoint catalog + incremental-sync query pattern
│   ├── webhooks.md                 # setup, event structure, retry logic, event types
│   ├── connector-pattern.md        # iPaaS/ERP connector: ownership matrix + happy-path journal entries
│   ├── gotchas.md                  # real-world troubleshooting (field renames, fee split, dedupe, etc.)
│   ├── implementation-lessons.md   # onboarding, credential handoff, data migration, architecture constraints
│   └── sandbox-testing.md          # public sandbox test cards, ACH, bank-login, MFA credentials
└── assets/
    └── paystand-x-quickstart.postman_collection.json   # runnable — add your own client_id/secret/customer_id
```

---

## Install

### Claude Code — plugin (BETA, one-command)

> ⚠️ **Beta.** Plugin packaging is new; if it misbehaves, use the manual skill install below (stable).

```
/plugin marketplace add tcluzel/paystand-x-api-skill
/plugin install psx-api@paystand
```

The skill activates immediately. It loads automatically when you work on a Paystand X integration, or invoke it with **`/psx-api`**.

### Claude Code — manual skill install (stable)

**Personal (available in all your projects):**
```bash
mkdir -p ~/.claude/skills
cp -R psx-api ~/.claude/skills/
```

**Project-scoped (commit it so your whole team gets it):**
```bash
mkdir -p .claude/skills
cp -R psx-api .claude/skills/
git add .claude/skills/psx-api && git commit -m "Add Paystand X API skill"
```

Then start (or restart) Claude Code. It loads the skill automatically when you ask about Paystand X, or you can invoke it directly with **`/psx-api`**.

### Claude API / claude.ai

Upload the `psx-api/` folder (or the zip) as a custom Skill via the Skills API, or add it in claude.ai settings → Skills. See Anthropic's [Agent Skills docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview).

### Any other Agent-Skills-compatible tool (OpenAI Codex, OpenCode, VS Code, ...)

Most clients also discover skills in the cross-client `.agents/skills/` directory, so one copy serves every tool that follows the convention:

```bash
mkdir -p ~/.agents/skills            # personal, all projects
cp -R psx-api ~/.agents/skills/
# or, per project:
mkdir -p .agents/skills && cp -R psx-api .agents/skills/
```

If your tool uses its own directory instead, point it at `psx-api/`. The format is the open standard: a `SKILL.md` with YAML frontmatter (`name`, `description`) plus supporting files loaded on demand. The `/psx-api` slash command used in this README is Claude Code's way of invoking a skill; other tools load it from its description or their own command syntax.

---

## Verify it installed

In Claude Code, run `/psx-api` — you should see the skill load. Or ask: *"What are the Paystand X base URLs and how do I authenticate?"* and confirm it answers with the OAuth `client_credentials` flow, the two required headers (`Authorization` + `X-CUSTOMER-ID`), and the sandbox `.co` / production `.com` split.

---

## Authoritative reference

This skill mirrors and annotates the official docs at
**https://paystand-developers.netlify.app/docs/paystand-x-api** — keep that open alongside for field-by-field schemas. Where this skill and the live docs ever disagree, the skill flags it and explains why.

---

## Contributing / maintainers

This skill is public and shareable with external developers. When editing, keep specific customer names, support-ticket numbers, and internal-only URLs OUT of the skill files — describe behavior generically so any integrator can use it.

**Repository layout — the skill exists in two places (keep them in sync):**
- `psx-api/` — the standalone skill (used by the manual install and by other Agent-Skills tools).
- `plugins/psx-api/skills/psx-api/` — the same skill, bundled inside the Claude Code plugin (BETA).

When you change the skill, update **both** copies (they are byte-identical) before committing.

---

*Version 0.9.0-beta*
