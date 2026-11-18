# Write payloads

Use these JSON objects as MCP tool arguments. The dates and values below are sample data. Replace them with observed records.

## Create a private tracker

`create_zeitline`

```json
{
  "title": "my-app tracker",
  "idempotencyKey": "my-app-tracker-create-v1",
  "from": {"kind": "blank"},
  "storage": {"mode": "hosted", "visibility": "private"},
  "preview": true
}
```

`preview: true` is accepted on `create_zeitline` only; `write_events` and `write_series` refuse it. Use the returned Zeitline ID for all future calls. A preview link does not grant anonymous access. If you lose a create response, check `get_mutation_status` with the original key and recover the original document before creating another.

## Write a release milestone

`write_events`

```json
{
  "zeitlineId": "<saved-id>",
  "idempotencyKey": "my-app-release-42-initial",
  "changes": [{
    "kind": "upsert_events",
    "keyScope": "my-app-releases",
    "swimlaneKey": "releases",
    "swimlaneName": "Releases",
    "seriesKey": "release-history",
    "seriesName": "Release history",
    "missingEventAction": "keep",
    "fieldSchema": {
      "version": 2,
      "revision": "my-app-release-fields-v1",
      "fingerprint": "my-app-release-fields-v1",
      "fields": [],
      "dateFields": [],
      "logicalFields": [{
        "id": "released-at",
        "key": "released_at",
        "label": "Released",
        "type": "date",
        "ownership": "local",
        "semanticRole": "event.milestone"
      }]
    },
    "events": [{
      "sourceKey": "release/42",
      "date": "2026-11-01T14:00:00Z",
      "title": "my-app v1.2.0",
      "milestones": {"released-at": "2026-11-01T14:00:00Z"}
    }]
  }]
}
```

On later writes keep the same lane, series and row identities and use the saved field schema. Providing `fieldSchema` on an upsert creates the schema on a new series. For an existing series, inspect its schema before writing milestone values. Each `upsert_events` change accepts up to 100 events. Keep `missingEventAction: "keep"` for incremental feeds.

## Merge measured values

`write_series`

```json
{
  "zeitlineId": "<saved-id>",
  "idempotencyKey": "my-app-metrics-backfill-1",
  "changes": [{
    "kind": "upsert_series_points",
    "keyScope": "my-app-metrics",
    "swimlaneKey": "build-metrics",
    "swimlaneName": "Build metrics",
    "seriesKey": "successful-builds-daily",
    "seriesName": "Successful builds per day",
    "points": [
      {"date": "2026-11-01T00:00:00Z", "value": 3},
      {"date": "2026-11-02T00:00:00Z", "value": 5}
    ]
  }]
}
```

Keep source datetimes for observations and use one consistent timezone and bucketing rule for aggregates. Missing observations stay missing. For a Percentage series, supply `percentageValueUnit: "fractional_ratio"` for values like `0.42`, or `"percent_points"` for values like `42`. Choose and disclose retention before using `keepLatest`, which drops older points.

## Attach and refresh a connector

Call `search_data_sources` with a query and inspect the matching definition. For a repo metric, `github-repo` is the catalog ID. Its configuration uses `repoUrl`, `resource`, `outputType` and, for a time series, `timeBucket`. Choose a supported combination exposed by search and confirmed by `preview_data_source`, using `outputType: "timeseries"` for a Chart lane. Use its `liveSeriesGuidance` when provided and the preview's `outputKey` values for any presentation overrides.

In `write_series` send `{zeitlineId, expectedRevision, idempotencyKey, changes: [{kind: "attach_data_source", target: {swimlaneName: "Repository activity"}, dataSourceId, configuration, refresh: true}]}`. Set `refresh: false` if the source cannot finish the initial fetch. Once the binding exists, send `{zeitlineId, expectedRevision, idempotencyKey, changes: [{kind: "refresh_data_source", swimlaneId, seriesId}]}` with the saved IDs and fresh revision. Read coverage and diagnostics before reporting that data is ready.

For CSV or JSON, build a repeatable mapping from the source's row IDs to `sourceKey` and from its measured timestamps to point dates. For a custom API, either use a supported connector mapping or fetch it in the runner, supplying source credentials from the runner's secret manager.
