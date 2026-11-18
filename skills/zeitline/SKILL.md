---
name: zeitline
description: Build and maintain a living project tracker on zeitline.com. Use when the user says "track this in Zeitline", "create a live tracker", "publish this as an interactive timeline", "use zeitline.com", or wants a timeline of releases, milestones or metrics from GitHub, JSON, CSV, an API or public data, updated on a schedule, with a public share link or website embed.
---

# Zeitline

Hosted at https://zeitline.com. Install: `npx skills add lattra/zeitline-skills`. Hosted MCP: `https://zeitline.com/api/v1/mcp`.

**No connection yet?** Without the MCP connection or an API key you can still draw a one-off picture: `POST https://zeitline.com/api/v1/visualize` (anonymous, no account) returns `data.visualizeUrl`. The preview lives 1 hour and nothing is saved. Fetch `GET /api/v1/visualize/schema` first and `POST /api/v1/visualize/validate` to check the payload. To keep, update or publish a tracker, install this skill and connect (section 1).

## 1. Connect

Pick one path.

- **Interactive (OAuth, private work only).** Codex: `codex mcp add zeitline --url https://zeitline.com/api/v1/mcp`, then `codex mcp login zeitline --scopes zeitline:read,zeitline:write,zeitline:search,zeitline:datasource,workspace:read,workspace:write`. Claude Code: add the server with `oauth.scopes` set to the same scopes space-separated (exact command in [references/starter-prompts.md](references/starter-prompts.md)), then `/mcp` to sign in. Refreshing a token never adds scopes; re-login after changing them. Start with `get_capabilities`.
- **Headless and publishing (API key).** The owner mints a key at https://zeitline.com/settings/api-keys, selecting scopes explicitly. A key cannot mint keys. Supply it as `ZEITLINE_API_KEY` from the shell or a secret manager. Never ask the user to paste a key into chat; never put it in files, arguments or logs.

**OAuth cannot publish.** `validate_publication` and `publish_zeitline` are API-key only. The only other route is the owner approving the review in the browser Share dialog (section 6).

| Scope | Use |
| --- | --- |
| `zeitline:read` + `zeitline:write` | Create, read, write events and points, version updates, `validate_publication`, `publish_zeitline`. The tracker baseline. |
| `zeitline:search` + `zeitline:datasource` | `search_data_sources`, `preview_data_source`; attach and refresh also need write. A new key defaults to read, write, generate and search, so select `zeitline:datasource` explicitly. |
| `workspace:read`, `workspace:write` | Workspace-scoped list and create. Select each; there is no `workspace:*`. |

A key bound to one tracker is for ingest only: `get_zeitline`, `update_zeitline`, `write_events`, `write_series`. Use a general key for digest, versions and publication.

## 2. Create and seed

1. Resolve the tracker: `list_zeitlines`, or ask for its link. Store the ID in non-secret job config.
2. `create_zeitline` with `title`, `from: {kind: "blank"}`, `storage: {mode: "hosted", visibility: "private"}`, a caller-generated `idempotencyKey`, `preview: true`; add `workspaceId` if the user chose one. If the outcome is unknown, `get_mutation_status` with `{toolName, idempotencyKey}` (or `operationId`) before any second create.
3. `write_events` with `changes: [{kind: "upsert_events", keyScope, swimlaneKey, swimlaneName, seriesKey, seriesName, missingEventAction: "keep", events: [{sourceKey, date, title}]}]`. `sourceKey` is a durable ID (release ID, issue ID). At most 100 events per change.
4. `write_series` with `kind: "upsert_series_points"`, `points: [{date, value}]`, addressed by `seriesId` or `keyScope` + `seriesKey`. One series per metric and unit. The same timestamp in the same series overwrites.
5. Never pass `preview` to `write_events` or `write_series`; their schemas refuse it. Pass `expectedRevision` when fencing matters.

Milestone fields and payload examples: [references/writes.md](references/writes.md).

## 3. Populate from sources

- **Catalog metrics** (GitHub repo metrics, FRED, World Bank): `search_data_sources`, then `preview_data_source` with the returned `dataSourceId`. Ask before any disclosed charge. Then `write_series` with `kind: "attach_data_source"` on a Chart lane, `refresh: true` (30 s initial-fetch budget). For a slow source use `refresh: false`, then `kind: "refresh_data_source"` with `swimlaneId` and `seriesId`.
- **Private GitHub or custom API:** the owner connects GitHub in Settings → Connections or stores a secret reference. Never put raw credentials in a source configuration; they are refused.
- **Releases, issues, CSV, JSON:** parse in the runner and write keyed events and points. `attach_data_source` cannot create a new GitHub Events lane; it only binds Chart outputs or re-binds an existing Events series. Map IDs, dates, timezones and units explicitly; skip and report invalid rows; leave absent rows alone on partial feeds (`missingEventAction: "keep"`).

## 4. Keep it updated

Run the same collector from the owner's cron, GitHub Actions, Claude Code `/schedule` or a Codex automation. Run once manually, read back, then enable the cadence. Details and a GitHub Actions workflow: [references/scheduling.md](references/scheduling.md).

- **No CLI needed:** POST JSON-RPC `tools/call` to `https://zeitline.com/api/v1/mcp` with `Authorization: Bearer $ZEITLINE_API_KEY`.
- **CLI:** `zeitline track batch --file <ndjson>`, then `zeitline track flush --json` and `zeitline track status --json`. `track` verbs only spool; `flush` delivers.
- **Dated updates (the Read memo):** `manage_zeitline_version` with `action: "publish_update"`, `zeitlineId`, `date` (`YYYY-MM-DD`), and exactly one of `text` or `explanation`; optional `effectiveAt`, `description`, `milestone`, `expectedRevision`. CLI: `zeitline update <id> --date YYYY-MM-DD --file <path>`.
  - **Publish oldest-first.** A date before the newest update is refused with 409 (`update_date_before_latest_update`); an `effectiveAt` before the newest update's is refused with 409 `effective_at_before_latest_update`; `effectiveAt` more than 5 minutes ahead is refused. Re-sending the same memo for a date changes nothing. Overwriting a date's memo is not supported yet, so backfill in date order and never re-publish an older date.
- Timeout: `get_mutation_status`, then read the document before retrying. Revision conflict: re-read and rebuild. Advance source cursors only after confirmed writes.

## 5. Query back

`get_zeitline_digest` with `zeitlineId` (`textOnly: true` for compact text). Keep the returned `cursor` and pass it as `since` with the same `lanes` selection. `sinceLastUpdate: true` compares with the latest dated update. Exact data: `get_zeitline`. CLI: `zeitline glance <id> --json [--since <cursor>]`.

## 6. Publish and embed

Publish only when the owner asks for a public link. Review outward text, links, people, metrics and media with them first.

1. **API-key path:** `validate_publication {zeitlineId}`, then `publish_zeitline {zeitlineId, validationReceipt, allowComments: false, showInGallery: false}` with the passing receipt (`pass` or `pass_with_exceptions`). Omit `slug`. Add `timeZone` (IANA) for relative periods. REST equivalents: `POST /api/v1/zeitlines/<id>/publication-validation`, `/publish`, `/unpublish`. Details: [references/publication.md](references/publication.md).
2. **First publish needs owner review.** Validation without a stored review returns `blocked_report` and no receipt, and no tool records the review. Tell the owner to open the tracker, **Share → Public content review**, check the confirmation, then **Publish to web** (later **Update public snapshot**). That saves the review, validates and publishes. Later API-key runs can republish while the review still covers the content; new people, links or prose need another browser review.
3. **Private writes do not change the public snapshot.** Validate and republish after confirmed updates. If validation blocks, leave the last snapshot live and report the blocker.
4. Verify `publicUrl` signed out. Share the `/view/<slug>` link on X. For a website:

```html
<iframe src="https://zeitline.com/view/<slug>?embed=1"
  title="Project tracker" loading="lazy"
  sandbox="allow-scripts allow-same-origin"></iframe>
```

The viewer is anonymous and read-only, and defaults to `noindex`. `embed=1` only hides chrome; it grants no access. A private, unpublished or wrong slug returns 404.

## Failure handling

- **Auth or scope refused:** re-authenticate with the scopes above, check workspace role, key expiry, revocation and document binding. Ask the owner for a replacement key; never ask for the value in chat.
- **Credits, cap or subscription refused:** surface the operation and reason. Where enforcement is on, paid operations need an active or trialing Heading subscription; metered sources also need credits. Offer a deterministic direct import instead.
- **Validation blocked:** send the owner the browser review step or the specific blockers. Permitted exceptions are approved only in a browser session. Revalidate after content changes or an expired receipt.
