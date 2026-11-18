# Scheduled writes from your own runner

For data Zeitline has a source for, attach the source and let the owner turn on Keep updated instead (SKILL.md section 4a); no runner or key is needed. Use a runner for private data, internal APIs and anything without a source.

Schedule the collector in the environment that can reach its sources and `https://zeitline.com`. Use the same command for a first manual run and recurring runs. Put the saved document ID in configuration and inject `ZEITLINE_API_KEY` from the runner's secret store. For connector-derived lanes, the scheduled write is `write_series` with `refresh_data_source` using the saved IDs. For direct imports, the collector produces keyed events or point batches on each run.

## GitHub Actions

This workflow tracks a personal `my-app` Zeitline. Daily runs chart commits on the default branch by UTC day over the last seven completed days. Release runs add a dated event with a release milestone. The window lets a later run fill in missed days within that week. The workflow uses the hosted API and requires Node.js 22 or later on the runner.

Before enabling it, create a private Zeitline and set the repository Actions variable `ZEITLINE_ID` to its ID. Set the Actions secret `ZEITLINE_API_KEY` to a general key with `zeitline:read` and `zeitline:write`. If the target is in a workspace, confirm authority and carry the same resolved workspace `target` in the MCP calls that need it. Keep this file on the default branch for the schedule. Use **Run workflow** once and verify the resulting Chart points and digest.

```yaml
name: Update Zeitline

on:
  schedule:
    - cron: '17 6 * * *'
  workflow_dispatch:
  release:
    types: [published]

permissions:
  contents: read

concurrency:
  group: my-app-zeitline
  cancel-in-progress: false

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false
      - name: Write and verify the tracker
        env:
          ZEITLINE_ID: ${{ vars.ZEITLINE_ID }}
          ZEITLINE_API_KEY: ${{ secrets.ZEITLINE_API_KEY }}
        run: |
          node --input-type=module <<'JS'
          import { createHash } from 'node:crypto';
          import { execFileSync } from 'node:child_process';
          import { readFileSync } from 'node:fs';

          const { ZEITLINE_ID: zeitlineId, ZEITLINE_API_KEY: apiKey } = process.env;
          if (!zeitlineId || !apiKey) throw new Error('Set the Zeitline ID and API-key secret.');

          async function tool(name, args) {
            const response = await fetch('https://zeitline.com/api/v1/mcp', {
              method: 'POST',
              redirect: 'error',
              signal: AbortSignal.timeout(45_000),
              headers: {
                'Accept': 'application/json, text/event-stream',
                'Authorization': 'Bearer ' + apiKey,
                'Content-Type': 'application/json',
              },
              body: JSON.stringify({
                jsonrpc: '2.0',
                id: 1,
                method: 'tools/call',
                params: { name, arguments: args },
              }),
            });
            if (!response.ok) throw new Error(name + ': HTTP ' + response.status);
            const body = await response.text();
            const isSse = response.headers.get('content-type')?.includes('text/event-stream');
            const payload = isSse
              ? body.split(/\r?\n/).filter(line => line.startsWith('data:')).at(-1)?.slice(5)
              : body;
            if (!payload) throw new Error(name + ': missing JSON-RPC response');
            const { error, result } = JSON.parse(payload);
            const data = result?.structuredContent;
            if (error || result?.isError || data?.ok !== true) {
              const code = data?.code;
              const printable = typeof code === 'string' &&
                /^[a-z0-9_]{1,64}$/.test(code) && !code.includes(apiKey);
              throw new Error(name + ': ' + (printable ? code : 'request failed'));
            }
            return data;
          }

          async function write(name, change) {
            const args = { zeitlineId, changes: [change] };
            const digest = createHash('sha256').update(JSON.stringify(args)).digest('hex');
            const idempotencyKey = 'my-app-' + process.env.GITHUB_RUN_ID + '-' + digest;
            await tool(name, { ...args, idempotencyKey });
          }

          if (process.env.GITHUB_EVENT_NAME === 'release') {
            const { release } = JSON.parse(readFileSync(process.env.GITHUB_EVENT_PATH, 'utf8'));
            if (release?.id == null || !release.published_at || !release.tag_name) {
              throw new Error('The release event is missing its ID, publish time or tag.');
            }
            await write('write_events', {
              kind: 'upsert_events',
              keyScope: 'my-app-releases',
              swimlaneKey: 'releases',
              swimlaneName: 'Releases',
              seriesKey: 'release-history',
              seriesName: 'Release history',
              missingEventAction: 'keep',
              fieldSchema: {
                version: 2,
                revision: 'my-app-release-fields-v1',
                fingerprint: 'my-app-release-fields-v1',
                fields: [],
                dateFields: [],
                logicalFields: [{
                  id: 'released-at', key: 'released_at', label: 'Released',
                  type: 'date', ownership: 'local', semanticRole: 'event.milestone',
                }],
              },
              events: [{
                sourceKey: 'release/' + release.id,
                date: release.published_at,
                title: 'my-app ' + release.tag_name,
                milestones: { 'released-at': release.published_at },
              }],
            });
          } else {
            const dayMs = 86_400_000;
            const end = Math.floor(Date.now() / dayMs) * dayMs;
            const start = end - 7 * dayMs;
            const cutoff = new Date(start).toISOString();
            const counts = Array(7).fill(0);
            const lines = execFileSync(
              'git', ['log', '--since-as-filter=' + cutoff, '--format=%ct', 'HEAD'],
              { encoding: 'utf8' },
            ).trim();
            for (const line of lines ? lines.split('\n') : []) {
              const at = Number(line) * 1000;
              if (at >= start && at < end) counts[Math.floor((at - start) / dayMs)] += 1;
            }
            await write('write_series', {
              kind: 'upsert_series_points',
              keyScope: 'my-app-metrics',
              swimlaneKey: 'build-metrics',
              swimlaneName: 'Build metrics',
              seriesKey: 'commits-daily',
              seriesName: 'Commits per UTC day',
              points: counts.map((value, index) => ({
                date: new Date(start + index * dayMs).toISOString(),
                value,
              })),
            });
          }

          await tool('get_zeitline_digest', { zeitlineId, textOnly: true });
          console.log('Zeitline accepted the write and returned a digest.');
          JS
```

For a manual run, select the default branch to match the scheduled metric. For a longer outage, backfill the affected UTC days before resuming the normal window. The workflow records a confirmed API write and digest read. To verify contents, open the tracker or inspect the digest for the expected release and points. On an uncertain dispatch, reconcile the original write before rerunning. A run ID plus the request hash is stable for that run's exact payload, while the event's `sourceKey` and each metric's timestamp identify the data across runs.

## CLI queue and flush

Use this path only if the `zeitline` CLI is installed (npm distribution is unverified) and exposes `track batch` and `track flush`. The track presets default to the Agent ops lane layout; confirm a point lands in a generic tracker with a dry run and a read-back before relying on it. Use the hosted API example above on a runner without that build. Set `ZEITLINE_BASE_URL=https://zeitline.com`, `ZEITLINE_AGENT_OPS_ZEITLINE_ID` to the destination, and `ZEITLINE_API_KEY` through the runner's secrets. A persistent runner can set `ZEITLINE_AGENT_OPS_HOME` to a private spool directory.

Prepare NDJSON with one JSON argv array per line. This example shows count points. Keep `--scope`, `--lane`, `--series` and the observation timestamp stable across runs.

```jsonl
["point","--scope","my-app-metrics","--lane","build-metrics","--series","commits-daily","--value","5","--at","2026-11-01T00:00:00Z","--format","count"]
["point","--scope","my-app-metrics","--lane","build-metrics","--series","commits-daily","--value","7","--at","2026-11-02T00:00:00Z","--format","count"]
```

```sh
zeitline track batch --file updates.ndjson --dry-run
zeitline track batch --file updates.ndjson
zeitline track flush --json
zeitline track status --json
```

If `flush` reports `flusher_running`, the batch command's background flusher may hold the lease. Check `track status` and let it finish, then flush again if records remain. Keep the spool through recoverable failures, inspect queued, held and rejected records, and verify the target before discarding the spool. Use one writer per feed. Switching from direct events to the CLI event writer requires checking the series schema and stored keys first.

## Cron and agent schedulers

- **Cron**. Run the same checked-in writer with the intended working directory, runtime and secret injection. Set the cadence in the runner's timezone and confirm an actual delivery.
- **Claude Code**. Use `/schedule` where cloud scheduled tasks are enabled. Select the repo and an environment with access to the source and `zeitline.com`. For an Anthropic-hosted environment use a destination-scoped network secret that adds the bearer header outside the sandbox. Use the direct HTTP call shape above with the runtime supplying that header. Choose GitHub Actions or cron if the environment cannot inject the credential or reach the source. [Claude Code scheduled tasks](https://code.claude.com/docs/en/routines), [Cloud environment network secrets](https://code.claude.com/docs/en/cloud-environments)
- **Codex**. In a host with automation support, schedule the same writer against the intended project. Check that the scheduled runtime can access the repo, reach the API and receive `ZEITLINE_API_KEY` from a secret provider. Local scheduled tasks need the host running. Use GitHub Actions or cron when those conditions are unavailable. [Scheduled tasks](https://learn.chatgpt.com/docs/automations?surface=app)
