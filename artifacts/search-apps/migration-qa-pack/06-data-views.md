# 06 — Typical Kibana Data View use cases

## Customer ask

What are the typical use cases for using Data Views?

## Summary answer

A [Data View](https://www.elastic.co/docs/explore-analyze/find-and-organize/data-views) is a Kibana saved object that points at one or more indices, aliases, or data streams and defines the time field (and optional runtime fields / field formatters). It is how **classic** Discover, Lens, most dashboard data sources, and many Observability/Security apps know what to query.

## Typical use cases

| Use case | Why a Data View |
|----------|-----------------|
| Explore crawled content in Discover (KQL / Lucene) | Select `search-crawler-*` (or similar) and `@timestamp` / crawl time field |
| Lens / classic dashboard panels | Data View is the data source for non-ES\|QL visualizations |
| Space defaults | One default Data View per Kibana space for the engagement |
| Runtime fields | Compute display fields without reindexing |
| Cross-index patterns | One view over `logs-*` or multiple crawl indices |

## What Data Views are *not*

- **Not required for ES\|QL** Discover sessions or ES\|QL dashboard panels (you name the index pattern in the query).
- **Not** a substitute for Elasticsearch index templates / mappings.
- **Not** Behavioral Analytics collections (see [FAQ-002](../../../FAQ.md#faq-002)).

## Migration tip

Create a Data View for Open Crawler content indices for ad-hoc exploration, and use **ES\|QL** (no Data View required) for the search-analytics dashboards in [behavioral-analytics-parity](../../search-analytics/behavioral-analytics-parity/esql-dashboard.md).
