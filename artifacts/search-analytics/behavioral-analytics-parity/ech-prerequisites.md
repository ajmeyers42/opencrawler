# ECH 9.x prerequisites (space, keys, data stream, ingest API)

Sanitized pattern for standing up search-behavior analytics on Elastic Cloud Hosted **9.x** without Behavioral Analytics.

## 1. Cluster and Kibana space

| Item | Guidance |
|------|----------|
| Stack | ECH **9.x** (latest unless pinned). Managed OTLP (optional latency path) needs 9.0+. |
| Solutions | Elasticsearch + Kibana. Observability/APM optional for BA parity. |
| Space | Create a dedicated Kibana space, e.g. `search-analytics` or engagement slug. |
| Naming | Tag assets with an `index_prefix` (no trailing hyphen) for cleanup. |

## 2. API keys (never put write keys in the browser)

Create **two** scoped API keys (or roles):

### Open Crawler (content)

- Privileges: `create_index`, `write`, `manage` (or narrower) on content indices / aliases only, e.g. `search-crawler-*`.
- Used by Open Crawler on **customer infrastructure**, not by the browser.

### Analytics ingest service

- Privileges: `create_doc` / `write` (and `create_index` if the data stream is auto-created via template) on `logs-search_analytics.events-*` only.
- Used by the **ingest API** process. The browser talks only to that API.

Example role descriptor shape (adjust names for the customer):

```json
{
  "name": "search-analytics-ingest",
  "role_descriptors": {
    "search_analytics_writer": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["logs-search_analytics.events-*"],
          "privileges": ["create_doc", "auto_configure"]
        }
      ]
    }
  }
}
```

## 3. Hot-only data stream

Prefer a **logs-*** data stream so Discover and ES|QL work with familiar conventions. Default name in this artifact:

`logs-search_analytics.events-default`

### ILM (hot-only)

Unless the customer asks for frozen/cold tiers, keep analytics on **hot** with rollover + delete:

```json
PUT _ilm/policy/search-analytics-hot
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "7d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

Tune `max_age` / `delete.min_age` to retention and volume. See [data-stream-template.json](./data-stream-template.json) for index template + component mapping.

### Apply template

```http
PUT _index_template/logs-search_analytics.events
```

Body: [data-stream-template.json](./data-stream-template.json)

Then create the data stream (or let first document create it if `data_stream` is configured):

```http
PUT _data_stream/logs-search_analytics.events-default
```

## 4. Ingest API (required front door)

**Do not** send analytics from the browser straight to Elasticsearch. 8.19 Behavioral Analytics used a public DSN in front of ES; on 9.x, put a small service (or reverse proxy) in front.

### Responsibilities

1. Accept `POST /api/search-analytics/events` (batch of 1–N events).
2. Validate `event.action` ∈ {`page_view`,`search`,`search_click`} and required fields (see [event-contract.md](./event-contract.md)).
3. Stamp `@timestamp` (server time) and `event.dataset: search_analytics`.
4. Bulk-index to `logs-search_analytics.events-default`.
5. Rate-limit and optionally require a same-origin cookie / CSRF token.
6. Gate on consent (drop or no-op when consent denied).

### Sketch (Node / Express-style)

See [ingest-api.sketch.md](./ingest-api.sketch.md) for a minimal handler outline. Implement in the customer’s stack (Node, FastAPI, nginx + sidecar, etc.).

### Security checklist

- [ ] No Elasticsearch API key in frontend bundles or `localStorage`
- [ ] CORS allowlist = search app origin only
- [ ] Payload size limit (e.g. 64 KB / request)
- [ ] Anonymous IDs only (`session.id`, optional `user.anonymous_id`)
- [ ] Redact query text from **application** logs if search queries may contain PII

## 5. Optional ingest pipeline

Attach a pipeline for `event.ingested`, user-agent parsing, or geo — only if needed:

```json
PUT _ingest/pipeline/search-analytics-default
{
  "description": "Stamp ingest time for search analytics events",
  "processors": [
    {
      "set": {
        "field": "event.ingested",
        "value": "{{_ingest.timestamp}}"
      }
    }
  ]
}
```

Reference it from the index template `default_pipeline` (already stubbed in [data-stream-template.json](./data-stream-template.json); create the pipeline before first write or remove `default_pipeline`).

## What not to do

- Put a cluster-admin or content-index write key in the browser.
- Reuse `_application/analytics` as the long-term 9.x design (deprecated; Kibana UI gone).
- Rely on Open Crawler `crawler_event.log` for user-behavior KPIs.
