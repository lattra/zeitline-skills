# Zeitline starter prompts

## Install once

```sh
npx skills add lattra/zeitline-skills
```

Then connect the hosted MCP.

### Claude Code

```sh
claude mcp add-json --scope user zeitline '{"type":"http","url":"https://zeitline.com/api/v1/mcp","oauth":{"scopes":"zeitline:read zeitline:write zeitline:search zeitline:datasource workspace:read workspace:write"}}'
```

Start a fresh session, run `/mcp`, choose `zeitline` and sign in. Basic read and write only: `claude mcp add --transport http --scope user zeitline https://zeitline.com/api/v1/mcp`. [Claude Code MCP setup](https://code.claude.com/docs/en/mcp#restrict-oauth-scopes)

### Codex

```sh
codex mcp add zeitline --url https://zeitline.com/api/v1/mcp
codex mcp login zeitline --scopes zeitline:read,zeitline:write,zeitline:search,zeitline:datasource,workspace:read,workspace:write
```

Complete the browser sign-in and start a fresh session.

OAuth is private-only. Publishing and unattended runs need an API key: the owner creates it at [Settings → API Keys](https://zeitline.com/settings/api-keys) and the runner reads `ZEITLINE_API_KEY` from its secret manager. Paste only a prompt, never a key.

## A. Create a live tracker for this repo, updated daily

```text
Use the Zeitline skill against https://zeitline.com. If the skill or hosted MCP is missing, help me install it and finish browser sign-in. Create or reuse a private tracker for this repo. Backfill its releases and meaningful dated project milestones, and chart a metric you can measure from repo data. Preserve real source IDs, dates, units and links. Set up a daily refresh at 06:17 UTC using this repo's GitHub Actions if available, or the scheduler I already use. Tell me exactly where to add the API-key secret. Run one update when credentials are available, verify the writes with a digest, and give me the tracker link, first-run result and schedule status. Keep the tracker private.
```

## B. Turn this CSV or API into a public interactive timeline I can share on X

```text
Use the Zeitline skill. Turn the CSV or API I supplied into a hosted Zeitline I can share on X. Map stable row IDs and dates to events or milestones, and chart measured numeric columns with their units. Show me the fields and links that would become public, and leave private fields out. If it needs owner review, guide me through Share → Public content review in the browser. Validate and publish through the API-key path when a current review already exists. Verify the public viewer signed out and give me its URL and a website iframe. If I give you a cadence, set up private refreshes and republish approved updates while the review remains valid.
```

## C. Add Zeitline tracking to my app's release process

```text
Use the Zeitline skill to add tracking to this repo's release workflow. Reuse an existing tracker when possible, otherwise create a private one. After each successful release, upsert a release event keyed by the provider's immutable release ID and record the actual publish time as a milestone. Add a release metric only if the workflow measures it. Use a runner-held ZEITLINE_API_KEY and persist the tracker ID as non-secret configuration. Keep re-runs safe, verify a permitted replay or test delivery with a digest, and show me the workflow changes and how to query the tracker. Keep publication off unless I request it.
```
