# Artifact: Behavioral Analytics parity on ECH 9.x

| Field | Value |
|-------|--------|
| FAQ | [FAQ-002](../../../FAQ.md#faq-002) |
| Area | search-apps / analytics |
| Audience | shareable (`main`) |

## Problem

Customers migrating the App Search / Elastic web crawler to [Open Crawler](https://github.com/elastic/crawler) on **Elastic Cloud Hosted (ECH) 9.x** lose **Behavioral Analytics** (removed from Kibana in 9.0; `_application/analytics*` APIs deprecated). Open Crawler replaces **content ingest only** — it does not collect search, click, or page-view behavior. Teams need collection and reporting **at least on par** with Enterprise Search 8.19 Behavioral Analytics.

## Diagnosis

1. **Crawler ≠ analytics** — Open Crawler logs and crawl metrics are not a substitute for user-behavior collections.
2. **No drop-in 9.x BA product** — Search UI’s `@elastic/search-ui-analytics-plugin` targets the discontinued Behavioral Analytics product.
3. **OTel browser RUM is not the production BA replacement** — Elastic documents OpenTelemetry RUM as technical preview, not for production.
4. **Parity is a build** — instrument the search UI, store events on an Elasticsearch data stream, report with ES|QL dashboards.

## Reporting model (explicit)

On 9.x there is **no Behavioral Analytics Kibana UI** and no collections app to “enable.” Analytics is:

1. **Instrument** the search experience (and optionally the search API with EDOT).
2. **Store** events (and optional traces) in Elasticsearch.
3. **Report** with **ES|QL queries and Kibana dashboards** (plus optional alerts).

There is nothing under a dedicated Analytics menu to turn on.

## Resolution pattern

| Layer | What to do | Artifact |
|-------|------------|----------|
| ECH setup | Kibana space, scoped API keys, hot-only data stream, ingest API (no browser ES keys) | [ech-prerequisites.md](./ech-prerequisites.md) |
| Collection | `page_view` / `search` / `search_click` contract + Search UI / custom hooks | [event-contract.md](./event-contract.md) |
| Reporting (parity) | ES\|QL **dashboards** matching 8.19 BA — not a product UI | [esql-dashboard.md](./esql-dashboard.md) |
| Additional | Latency via EDOT/mOTLP; impressions, experiments, funnels, alerts | [additional-analytics.md](./additional-analytics.md) |
| Cutover | Dual-run 8.19 BA → staging events → parity week → retire BA → 9.x | [cutover.md](./cutover.md) |

```text
Website / Search UI
        │
        ▼
 Analytics ingest API (same origin or reverse proxy)
        │
        ▼
 logs-search_analytics.events-*  (ECH data stream)
        │
        ▼
 Kibana ES|QL queries + dashboards (CTR, top queries, no-results, …)
        │  optional: EDOT search spans → mOTLP → traces-* (latency)
        └─ still queries/dashboards — no BA Kibana app
```

Open Crawler remains a **separate** path: customer infra → content indices → Search Application / Search UI connector.

## Validation

1. Staging UI emits events; Discover shows `event.action` in {`page_view`,`search`,`search_click`}.
2. ES|QL overview metrics within expected range vs a parallel week of 8.19 BA charts (same queries/clicks).
3. No Elasticsearch write API key in browser JavaScript or public bundles.

## References

- [Migrating to 9.x from Enterprise Search 8.x](https://www.elastic.co/guide/en/enterprise-search/8.19/upgrading-to-9-x.html)
- [Kibana 9.0 breaking changes](https://www.elastic.co/docs/release-notes/kibana/breaking-changes) — Behavioral Analytics removed from UI
- [Elasticsearch deprecations](https://www.elastic.co/docs/release-notes/elasticsearch/deprecations) — Behavioral Analytics CRUD APIs
- [Search UI Analytics Plugin](https://www.elastic.co/docs/reference/search-ui/api-core-plugins-analytics-plugin) — discontinued in 9.0
- [docs/migration-overview.md](../../../docs/migration-overview.md)
