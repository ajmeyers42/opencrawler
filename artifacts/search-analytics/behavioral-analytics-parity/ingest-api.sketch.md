# Ingest API sketch

Minimal pattern for a same-origin (or reverse-proxied) analytics endpoint. Not production-ready code — adapt to the customer stack.

## Endpoint

`POST /api/search-analytics/events`

Content-Type: `application/json`

```json
{
  "events": [
    {
      "event": { "action": "search" },
      "session": { "id": "uuid-tab-session" },
      "user": { "anonymous_id": "uuid-device" },
      "search": {
        "query": "wireless headphones",
        "result_count": 42,
        "zero_results": false,
        "search_application": "site-search"
      },
      "labels": { "environment": "staging" }
    }
  ]
}
```

## Handler outline (pseudocode)

```text
on POST /api/search-analytics/events:
  if consent cookie/header missing → 204 No Content (or drop silently)
  body = parse JSON; reject if > 64KB or events.length > 50
  docs = []
  for each event in body.events:
    validate action in {page_view, search, search_click}
    validate required fields for that action (see event-contract.md)
    docs.append({
      "@timestamp": now_utc(),
      "event": { "action": ..., "dataset": "search_analytics" },
      ...normalized fields...
    })
  bulk index to logs-search_analytics.events-default with ingest API key
  return 202 { "accepted": docs.length }
```

## Environment (server only)

```bash
ES_URL=https://example.es.region.gcp.elastic.cloud
ES_API_KEY=<search-analytics-ingest-key>
ANALYTICS_DATA_STREAM=logs-search_analytics.events-default
```

## Reverse proxy alternative

If the search app is static:

1. Browser → `https://search.example.com/api/search-analytics/events`
2. Edge/nginx terminates TLS, injects `Authorization: ApiKey …` toward an internal indexer, or forwards to the app’s BFF.
3. Never expose the Elasticsearch URL or API key to the client.

## Client helper (browser)

```javascript
async function trackSearchAnalytics(events) {
  if (!window.__analyticsConsent) return;
  await fetch("/api/search-analytics/events", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ events: Array.isArray(events) ? events : [events] }),
    keepalive: true,
  });
}
```

Session IDs: `sessionStorage` for tab `session.id`; optional `localStorage` for `user.anonymous_id` (anonymous only).
