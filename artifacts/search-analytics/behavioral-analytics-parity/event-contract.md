# Event contract and Search UI wiring

Replace 8.19 Behavioral Analytics tracker / `@elastic/search-ui-analytics-plugin` with first-party events that map 1:1 to the old mental model.

## Event types (parity with 8.19)

| `event.action` | When to emit | Required fields |
|----------------|--------------|-----------------|
| `page_view` | Route change / initial load of a tracked page | `session.id`, `page.url` |
| `search` | User submits a search (or Search UI issues a query) | `session.id`, `search.query`, `search.result_count`, `search.zero_results` |
| `search_click` | User clicks a result | `session.id`, `search.query`, `document.id`, `search.click_position` (1-based) |

Optional on all events: `user.anonymous_id`, `labels.environment`, `labels.engagement_slug`, `search.search_application`.

### Field notes

- **`search.zero_results`**: `true` when `result_count === 0`.
- **`search.click_position`**: 1-based rank in the result list (needed for MRR / mean click position).
- **`search.filters`**: optional object of active facets; mapped as `flattened` in the template.
- **`document.title` or `page.url`**: include at least one for “top clicked docs / pages” reporting.

## Example payloads

### page_view

```json
{
  "event": { "action": "page_view" },
  "session": { "id": "sess-…" },
  "user": { "anonymous_id": "anon-…" },
  "page": {
    "url": "https://www.example.com/search",
    "title": "Search",
    "referrer": "https://www.example.com/"
  }
}
```

### search

```json
{
  "event": { "action": "search" },
  "session": { "id": "sess-…" },
  "search": {
    "query": "return policy",
    "filters": { "locale": "en-us" },
    "result_count": 12,
    "zero_results": false,
    "search_application": "support-search"
  }
}
```

### search_click

```json
{
  "event": { "action": "search_click" },
  "session": { "id": "sess-…" },
  "search": {
    "query": "return policy",
    "click_position": 2
  },
  "document": {
    "id": "doc-abc123",
    "title": "How to return an item"
  },
  "page": {
    "url": "https://www.example.com/help/returns"
  }
}
```

## Search UI: drop the Analytics plugin

Do **not** use `@elastic/search-ui-analytics-plugin` against Elastic 9.x — it targets discontinued Behavioral Analytics.

### Connector

Point the Elasticsearch / Search Application connector at Open Crawler indices (or the Search Application that wraps them), not App Search.

### Hooks

Emit via the ingest helper from [ingest-api.sketch.md](./ingest-api.sketch.md):

1. **Search** — after results return (so `result_count` is known):

```javascript
// Pseudocode inside Search UI / app search handler
async function onResults({ query, results, filters }) {
  const count = results?.length ?? 0; // or total from response
  await trackSearchAnalytics({
    event: { action: "search" },
    session: { id: getSessionId() },
    user: { anonymous_id: getAnonymousUserId() },
    search: {
      query: query || "",
      filters: filters || {},
      result_count: count,
      zero_results: count === 0,
      search_application: "site-search",
    },
  });
}
```

Prefer the **total hits** from the Elasticsearch response when available (pagination may return fewer hits than total).

2. **Result click** — wrap result links / `onClick`:

```javascript
function onResultClick({ query, result, position }) {
  trackSearchAnalytics({
    event: { action: "search_click" },
    session: { id: getSessionId() },
    search: {
      query: query || "",
      click_position: position, // 1-based
    },
    document: {
      id: result.id,
      title: result.title,
    },
    page: { url: result.url },
  });
}
```

3. **Page view** — on app mount and client-side route changes:

```javascript
function onPageView() {
  trackSearchAnalytics({
    event: { action: "page_view" },
    session: { id: getSessionId() },
    page: {
      url: window.location.href,
      title: document.title,
      referrer: document.referrer || undefined,
    },
  });
}
```

### Debouncing

- Emit one `search` event per committed query (Enter / Search button / facet change), not per keystroke.
- Deduplicate rapid identical `page_view`s if the router fires twice.

## Custom UI (no Search UI)

Same three hooks: query submit → `search`; result click → `search_click`; navigation → `page_view`. Keep field names identical so ES|QL dashboards stay reusable.

## Privacy

- Anonymous IDs only; no email / account id in analytics docs unless the customer’s privacy review allows it.
- Gate `trackSearchAnalytics` on consent.
- If site search may contain PII in the query box, document retention and access controls on `logs-search_analytics.events-*`.

## Mapping to 8.19 BA

| 8.19 Behavioral Analytics | This contract |
|---------------------------|---------------|
| Collection | Data stream `logs-search_analytics.events-*` |
| JS tracker / Search UI plugin | `trackSearchAnalytics` + ingest API |
| `page_view` / `search` / `search_click` | Same `event.action` values |
| Session | `session.id` |
| DSN → ES | Ingest API → bulk index |
