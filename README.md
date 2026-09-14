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

### Claude Code

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

### Any other Agent-Skills-compatible tool

Point the tool at the `psx-api/` directory. The format is the open standard: a `SKILL.md` with YAML frontmatter (`name`, `description`) plus supporting files loaded on demand.

---

## Verify it installed

In Claude Code, run `/psx-api` — you should see the skill load. Or ask: *"What are the Paystand X base URLs and how do I authenticate?"* and confirm it answers with the OAuth `client_credentials` flow, the two required headers (`Authorization` + `X-CUSTOMER-ID`), and the sandbox `.co` / production `.com` split.

---

## Authoritative reference

This skill mirrors and annotates the official docs at
**https://paystand-developers.netlify.app/docs/paystand-x-api** — keep that open alongside for field-by-field schemas. Where this skill and the live docs ever disagree, the skill flags it and explains why.

---

*Version 1.0.0*
