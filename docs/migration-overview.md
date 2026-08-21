# Migration overview: Enterprise Search crawler → Open Crawler (ECH)

Enterprise Search (App Search, Workplace Search, Elastic Web Crawler, managed/native connectors) is **not available in Elastic 9.0+**. Existing 8.x Enterprise Search customers can keep running 8.x until they migrate. This repo covers the **web crawler** path: App Search / Elastic Web Crawler → [Open Crawler](https://github.com/elastic/crawler), plus related 9.x search-app questions that appear during that move.

## Capability map (crawler-focused)

| 8.x Enterprise Search | 9.x / replacement | Notes |
|----------------------|-------------------|--------|
| Elastic / App Search Web Crawler | [Open Crawler](https://github.com/elastic/crawler) | Config-as-code YAML; run on customer infra |
| Crawler schedules / Kibana UI | External scheduler + Open Crawler CLI (schedules evolving) | Treat config as code / CI |
| Behavioral Analytics (collections, tracker, Kibana UI) | First-party events + ingest API + **ES\|QL queries/dashboards** (no BA app); optional EDOT latency | Removed from Kibana in 9.0; APIs deprecated — see [FAQ-002](../FAQ.md#faq-002) and [additional-analytics.md](../artifacts/search-analytics/behavioral-analytics-parity/additional-analytics.md). Open Crawler does **not** replace this. Analytics is **not** enabled via a Kibana UI. |

Connectors, Workplace Search sources, and App Search relevance/engines are **out of scope** here. Use the official 9.x migration guide for those.

## Suggested ECH approach

1. **Inventory** — crawler domains, seeds, sitemaps, extraction rules, schedules, and any Behavioral Analytics collections tied to the search UI.
2. **Dual-run** — keep 8.19 Enterprise Search serving production while standing up Open Crawler writing to new indices (side-by-side).
3. **Parity checks** — doc counts by locale/source, sample queries, extraction fields.
4. **Cutover** — point apps at new indices / Search Applications; decommission Enterprise Search only when verified.
5. **Upgrade cluster** — follow Upgrade Assistant after crawler (and related analytics) are off Enterprise Search.

## Official starting points

- [Migrating to 9.x from Enterprise Search 8.x](https://www.elastic.co/guide/en/enterprise-search/8.19/upgrading-to-9-x.html) — notebooks for App Search data and crawler YAML conversion
- [Open Crawler release (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-open-crawler-release)
- [Open Crawler config as code (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-open-crawler-config-as-code)
- [App Search data after Stack 9 (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-app-search-data-elasticsearch-9)

## This repo

Worked answers live in [FAQ.md](../FAQ.md) and `artifacts/`. Start with crawler locale parity ([FAQ-001](../FAQ.md#faq-001)), Behavioral Analytics parity on ECH 9.x ([FAQ-002](../FAQ.md#faq-002)), and the common migration Q&A pack ([FAQ-003](../FAQ.md#faq-003)).
