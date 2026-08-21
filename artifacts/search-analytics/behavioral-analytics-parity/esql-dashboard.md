# ES|QL reporting (8.19 Behavioral Analytics parity)

Build a Kibana dashboard against `logs-search_analytics.events-*` using **ES|QL** and standard visualizations (bar, pie, line — not Vega unless needed for network/sankey). Prefer Lens / Discover ES|QL mode. On Kibana 9.4+, dashboards can also be created via the declarative Dashboard API.

Data stream: `logs-search_analytics.events-*` (see [ech-prerequisites.md](./ech-prerequisites.md)).

## Parity panel map

| 8.19 BA report | Dashboard panel | Query |
|----------------|-----------------|-------|
| Total events / overview | Metric: event counts | [Overview metrics](#overview-metrics) |
| Unique sessions | Metric: unique `session.id` | [Overview metrics](#overview-metrics) |
| Searches / clicks / CTR | Metrics + gauge | [Overview metrics](#overview-metrics) |
| Top queries | Table / bar | [Top queries by volume](#top-queries-by-volume) |
| No-result queries | Table | [No-result queries](#no-result-queries) |
| Top clicked documents | Table / bar | [Top clicked documents](#top-clicked-documents) |
| Top pages | Table / bar | [Top pages](#top-pages) |
| *(beyond 8.19)* Low CTR / MRR | Tables | [Low-CTR queries](#high-volume-low-ctr-queries) · [MRR](#mean-reciprocal-rank-mrr) |

Use a shared time picker (e.g. last 7 days / 30 days) on all panels.

---

## Overview metrics

```esql
FROM logs-search_analytics.events-*
| WHERE event.dataset == "search_analytics"
| STATS
    total_events = COUNT(*),
    unique_sessions = COUNT_DISTINCT(session.id),
    searches = COUNT(CASE(event.action == "search", 1)),
    clicks = COUNT(CASE(event.action == "search_click", 1)),
    page_views = COUNT(CASE(event.action == "page_view", 1))
| EVAL ctr = CASE(searches > 0, ROUND(clicks * 100.0 / searches, 2), 0)
```

Zero-results rate (overall):

```esql
FROM logs-search_analytics.events-*
| WHERE event.action == "search"
| STATS
    total_searches = COUNT(*),
    zero_result_searches = COUNT(CASE(search.zero_results == true, 1))
| EVAL zero_results_rate = CASE(
    total_searches > 0,
    ROUND(zero_result_searches * 100.0 / total_searches, 2),
    0
  )
```

---

## Top queries by volume

```esql
FROM logs-search_analytics.events-*
| WHERE event.action == "search" AND search.query.keyword IS NOT NULL
| STATS
    search_count = COUNT(*),
    avg_result_count = ROUND(AVG(search.result_count), 0)
  BY search.query.keyword
| RENAME search.query.keyword AS user_query
| SORT search_count DESC
| LIMIT 50
```

If the `.keyword` multi-field is not present in an older index, use `search.query` with `STATS … BY MV_CONCAT(…)` or ensure the template from [data-stream-template.json](./data-stream-template.json) is applied.

---

## No-result queries

```esql
FROM logs-search_analytics.events-*
| WHERE event.action == "search" AND search.zero_results == true
| STATS occurrences = COUNT(*) BY search.query.keyword
| RENAME search.query.keyword AS user_query
| SORT occurrences DESC
| LIMIT 100
```

Feed this list into synonyms / query rules (closed loop).

---

## Top clicked documents

```esql
FROM logs-search_analytics.events-*
| WHERE event.action == "search_click" AND document.id IS NOT NULL
| STATS
    clicks = COUNT(*),
    avg_position = ROUND(AVG(search.click_position), 2)
  BY document.id, document.title.keyword
| SORT clicks DESC
| LIMIT 50
```

---

## Top pages

```esql
FROM logs-search_analytics.events-*
| WHERE event.action == "page_view" AND page.url IS NOT NULL
| STATS views = COUNT(*) BY page.url
| SORT views DESC
| LIMIT 50
```

---

## CTR by query

```esql
FROM logs-search_analytics.events-*
| WHERE event.action IN ("search", "search_click")
| STATS
    searches = COUNT(CASE(event.action == "search", 1)),
    clicks = COUNT(CASE(event.action == "search_click", 1))
  BY search.query.keyword
| EVAL ctr = CASE(searches > 0, ROUND(clicks * 100.0 / searches, 2), 0)
| RENAME search.query.keyword AS user_query
| SORT searches DESC
| LIMIT 50
```

---

## High-volume, low-CTR queries

Relevance backlog (tune thresholds to traffic):

```esql
FROM logs-search_analytics.events-*
| WHERE event.action IN ("search", "search_click")
| STATS
    searches = COUNT(CASE(event.action == "search", 1)),
    clicks = COUNT(CASE(event.action == "search_click", 1))
  BY search.query.keyword
| EVAL ctr = CASE(searches > 0, ROUND(clicks * 100.0 / searches, 2), 0)
| WHERE searches >= 10 AND ctr < 20
| RENAME search.query.keyword AS user_query
| SORT searches DESC
| LIMIT 50
```

---

## Mean reciprocal rank (MRR)

```esql
FROM logs-search_analytics.events-*
| WHERE event.action == "search_click" AND search.click_position IS NOT NULL
| EVAL reciprocal = 1.0 / search.click_position
| STATS
    total_clicks = COUNT(*),
    mrr = ROUND(AVG(reciprocal), 4),
    avg_click_position = ROUND(AVG(search.click_position), 2)
```

Click position distribution:

```esql
FROM logs-search_analytics.events-*
| WHERE event.action == "search_click" AND search.click_position IS NOT NULL
| STATS clicks = COUNT(*) BY search.click_position
| SORT search.click_position ASC
| LIMIT 20
```

---

## Trend over time (line chart)

Searches and clicks by day:

```esql
FROM logs-search_analytics.events-*
| WHERE event.action IN ("search", "search_click")
| EVAL day = DATE_TRUNC(1 day, @timestamp)
| STATS
    searches = COUNT(CASE(event.action == "search", 1)),
    clicks = COUNT(CASE(event.action == "search_click", 1))
  BY day
| SORT day ASC
```

---

## Optional alerts

If the customer used BA mainly for ops, add threshold rules (Kibana Alerting) on scheduled ES|QL or Elasticsearch query rules:

| Alert | Suggested threshold (tune) |
|-------|----------------------------|
| Zero-results rate | > 15% over 1h / 24h |
| Overall CTR | < 25% over 24h (after warm-up) |
| MRR | < 0.4 over 24h |

Keep alert actions in the same Kibana space as the dashboard.

## Dashboard construction tips

1. Create the dashboard in the dedicated space from [ech-prerequisites.md](./ech-prerequisites.md).
2. One panel per query above; use metric for overview, bar for top-N, line for trends, table for query lists.
3. Tag saved objects with the engagement slug for cleanup.
4. Validate with [cutover.md](./cutover.md) parity week against 8.19 BA charts before retiring Enterprise Search.

## What this exceeds vs 8.19 BA

Same event types and core reports, plus **CTR-by-query**, **MRR / mean click position**, and **low-CTR backlog** tables that the old UI only partially surfaced.
