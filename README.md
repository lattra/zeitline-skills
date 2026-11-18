# Zeitline agent skill

Build and keep a living tracker on [zeitline.com](https://zeitline.com) from Claude Code, Codex or any agent that reads skills: releases, milestones and metrics from GitHub, CSV, JSON or an API, updated on a schedule, with a public link or a website embed.

## Install

```sh
npx skills add lattra/zeitline-skills
```

Then connect the hosted MCP server, `https://zeitline.com/api/v1/mcp`.

**Claude Code**

```sh
claude mcp add-json --scope user zeitline '{"type":"http","url":"https://zeitline.com/api/v1/mcp","oauth":{"scopes":"zeitline:read zeitline:write zeitline:search zeitline:datasource workspace:read workspace:write"}}'
```

Start a new session, run `/mcp`, pick `zeitline` and sign in.

**Codex**

```sh
codex mcp add zeitline --url https://zeitline.com/api/v1/mcp
codex mcp login zeitline --scopes zeitline:read,zeitline:write,zeitline:search,zeitline:datasource,workspace:read,workspace:write
```

Sign-in is for private work. To keep a tracker current, attach a live source (GitHub, FRED and more) and turn on **Keep updated** in the app: Zeitline then refreshes it on its own. For private data, your own scheduled runner writes with an API key that you create at [Settings → API Keys](https://zeitline.com/settings/api-keys) and keep in your runner's secret store. Never paste a key into a chat.

## Try it

Paste one of these into your agent:

- *"Use the Zeitline skill to create a live tracker for this repo, kept updated by Zeitline."*
- *"Use the Zeitline skill to turn this CSV into a public interactive timeline I can share on X."*

The full prompts are in [skills/zeitline/references/starter-prompts.md](skills/zeitline/references/starter-prompts.md).

## Contents

- [`skills/zeitline/SKILL.md`](skills/zeitline/SKILL.md): connect, create, populate, schedule, read back, publish and embed.
- [`references/writes.md`](skills/zeitline/references/writes.md): payloads for events, milestones and chart points.
- [`references/scheduling.md`](skills/zeitline/references/scheduling.md): your own runner for private data: a GitHub Actions workflow, cron and agent schedulers.
- [`references/publication.md`](skills/zeitline/references/publication.md): owner review, publishing and unpublishing.
