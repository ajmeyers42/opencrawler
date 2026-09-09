# FAQ: App Search / Elastic crawler → Open Crawler

Index of recurring customer questions about Open Crawler migrations (and related 9.x search-app asks that come up during those migrations). Each entry links to in-repo artifacts and Elastic / Search Labs references.

How to add entries: [CONTRIBUTING.md](CONTRIBUTING.md). Overview: [docs/migration-overview.md](docs/migration-overview.md).

---

## Index

| ID | Topic | Artifact |
|----|--------|----------|
| [FAQ-001](#faq-001) | Multi-locale site: one locale works in Open Crawler, siblings do not | [artifacts/crawler/multi-locale-support](artifacts/crawler/multi-locale-support/) |
| [FAQ-002](#faq-002) | Behavioral Analytics parity when moving App Search crawler → Open Crawler on ECH 9.x | [artifacts/search-analytics/behavioral-analytics-parity](artifacts/search-analytics/behavioral-analytics-parity/) |
| [FAQ-003](#faq-003) | Common App Search → Open Crawler / 9.x Q&A (analyzers, schema fields, ILM, dynamic mapping, Data Views) | [artifacts/search-apps/migration-qa-pack](artifacts/search-apps/migration-qa-pack/) |
| [FAQ-004](#faq-004) | XPath attribute-axis selectors return empty array in Open Crawler | [artifacts/crawler/xpath-attribute-extraction](artifacts/crawler/xpath-attribute-extraction/) |
| [FAQ-005](#faq-005) | `data-elastic-name` multi-value fields return single string instead of array | [artifacts/crawler/data-elastic-name-arrays](artifacts/crawler/data-elastic-name-arrays/) |

---

## FAQ-001

**Question:** After migrating an Elastic / App Search web crawler to Open Crawler, some locales on the same host work (for example `/us/en`) and others do not (for example `/uk/en`, `/mx/en`). Why, and what can Open Crawler do to overcome it?

**Resolution:** Open Crawler can crawl multi-locale sites on one host. The usual gap is migrated config, not a product limitation:

1. Catch-all extraction often **hardcodes** `crawled_country` / `crawled_language` to the primary locale — sibling locales that *are* crawled still look wrong in filters and apps.
2. Discovery often uses a **single root seed**; add **explicit seeds per locale** and keep per-locale sitemaps.
3. Extraction `url_filters` that `begins` with `/product/...` miss paths shaped `/{country}/{lang}/product/...` — use locale-aware `regex` or `contains`.
4. If crawl logs show challenge HTML or 403, treat **WAF allowlisting** next (Imperva/Cloudflare, etc.).

**Artifacts**

- Sanitized pattern: [artifacts/crawler/multi-locale-support](artifacts/crawler/multi-locale-support/) — [README](artifacts/crawler/multi-locale-support/README.md), [open-crawler.yml](artifacts/crawler/multi-locale-support/open-crawler.yml), [diagnostics](artifacts/crawler/multi-locale-support/diagnostics.md)

**References**

- [Migrating to 9.x from Enterprise Search 8.x](https://www.elastic.co/guide/en/enterprise-search/8.19/upgrading-to-9-x.html) — App Search / Elastic Crawler → Open Crawler notebooks
- [Open Crawler release (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-open-crawler-release)
- [Open Crawler: config as code (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-open-crawler-config-as-code)
- [App Search data after Elastic Stack 9 (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-app-search-data-elasticsearch-9)
- [docs/migration-overview.md](docs/migration-overview.md)

---

## FAQ-002

**Question:** When migrating from the App Search / Elastic web crawler to Open Crawler on an ECH 9.x deployment, how do we collect and report behavioral analytics at least on par with Enterprise Search 8.19 Behavioral Analytics?

**Resolution:** Open Crawler replaces **content ingest only**. Behavioral Analytics was **removed from Kibana in 9.0** (APIs deprecated). There is **no drop-in BA product UI** on 9.x — analytics is **not** turned on in a Kibana Analytics app. You enable it by instrumenting the search experience, writing events (and optional traces) to Elasticsearch, and building **ES|QL queries / dashboards** (and alerts).

Rebuild parity as:

1. **ECH prerequisites** — dedicated Kibana space, scoped API keys, hot-only `logs-search_analytics.events-*` data stream, and a same-origin **ingest API** (never put an ES write key in the browser).
2. **Event contract** — emit `page_view`, `search`, and `search_click` from the search UI (drop `@elastic/search-ui-analytics-plugin`); map fields 1:1 to the 8.19 mental model.
3. **Reporting** — **ES|QL dashboards** (not a BA UI) for sessions, CTR, top queries, no-results, top clicks, top pages (plus low-CTR / MRR beyond 8.19).
4. **Additional (optional)** — search **latency via EDOT → Managed OTLP**, plus impressions, experiments, funnels, and ops alerts — still queries/dashboards; see [additional-analytics.md](artifacts/search-analytics/behavioral-analytics-parity/additional-analytics.md).
5. **Cutover** — dual-run Open Crawler + keep 8.19 BA live → staging events → parity week → cut search app and tracker → retire Enterprise Search → then 9.x upgrade.

Do **not** use OpenTelemetry browser RUM as the production BA replacement (tech preview). Do **not** treat crawler logs as user-behavior analytics.

**Artifacts**

- Sanitized pattern: [artifacts/search-analytics/behavioral-analytics-parity](artifacts/search-analytics/behavioral-analytics-parity/) — [README](artifacts/search-analytics/behavioral-analytics-parity/README.md), [ECH prerequisites](artifacts/search-analytics/behavioral-analytics-parity/ech-prerequisites.md), [event contract](artifacts/search-analytics/behavioral-analytics-parity/event-contract.md), [ES\|QL dashboard](artifacts/search-analytics/behavioral-analytics-parity/esql-dashboard.md), [additional analytics (latency / expansions)](artifacts/search-analytics/behavioral-analytics-parity/additional-analytics.md), [cutover](artifacts/search-analytics/behavioral-analytics-parity/cutover.md)

**References**

- [Migrating to 9.x from Enterprise Search 8.x](https://www.elastic.co/guide/en/enterprise-search/8.19/upgrading-to-9-x.html)
- [Kibana 9.0 breaking changes](https://www.elastic.co/docs/release-notes/kibana/breaking-changes) — Behavioral Analytics removed from UI
- [Elasticsearch deprecations](https://www.elastic.co/docs/release-notes/elasticsearch/deprecations) — Behavioral Analytics CRUD APIs
- [Search UI Analytics Plugin](https://www.elastic.co/docs/reference/search-ui/api-core-plugins-analytics-plugin) — discontinued in 9.0
- [docs/migration-overview.md](docs/migration-overview.md)

---

## FAQ-003

**Question:** During an App Search / Elastic crawler → Open Crawler move on Elastic 9.x, what changes for index templates (special characters, App Search derived fields), ILM tiers, dynamic mapping, Behavioral Analytics, and Kibana Data Views?

**Resolution:** These come up as a pack on most migrations:

1. **Special characters** — Avoid a global `pattern_replace` that only keeps `.` `/` `-` on free-text fields; use targeted char filters / per-field analyzers and validate with the Analyze API.
2. **App Search `delimiter` / `enum` / `joined` / `prefix` / `stem`** — Do not recreate 1:1; replace behaviors with `keyword`, stemming analyzers, `search_as_you_type` / edge n-grams, and delimiter token filters as needed.
3. **ILM** — Policy “enabled” ≠ tier moves; confirm attach, timing, and that warm/cold/frozen nodes exist (many ECH deployments are hot-only).
4. **Dynamic mapping** — Crawler templates often prefer static schemas for stable search apps; dynamic or selective dynamic templates remain optional.
5. **Behavioral Analytics on 9.x** — No BA Kibana UI; enable via instrumentation + **ES|QL queries/dashboards** (optional EDOT latency). See [FAQ-002](#faq-002).
6. **Data Views** — For classic Discover/Lens over crawl indices; not required for ES\|QL panels.

**Artifacts**

- Sanitized pattern: [artifacts/search-apps/migration-qa-pack](artifacts/search-apps/migration-qa-pack/) — [customer-responses.md](artifacts/search-apps/migration-qa-pack/customer-responses.md) (paste-ready table), plus per-topic detail files

**References**

- [Migrating to 9.x from Enterprise Search 8.x](https://www.elastic.co/guide/en/enterprise-search/8.19/upgrading-to-9-x.html)
- [Mixing exact search with stemming](https://www.elastic.co/docs/solutions/search/full-text/search-relevance/mixing-exact-search-with-stemming)
- [Kibana data views](https://www.elastic.co/docs/explore-analyze/find-and-organize/data-views)
- [docs/migration-overview.md](docs/migration-overview.md)

## FAQ-004

**Question:** An XPath extraction rule that selects an HTML element **attribute** (e.g. `//*[@class='product-variant_list']/@data-attribute-type`) worked in the Enterprise Search crawler but returns an empty array in Open Crawler. Why?

**Resolution:** Open Crawler v0.3.0+ switched from Nokogiri to Jsoup for HTML parsing. Its `extract_by_xpath_selector` method hardcodes `TextNode.java_class` as the target node type, so attribute-axis selectors silently return nothing — the crawler only yields text nodes, never attribute nodes. The XPath itself is valid; it just resolves to the wrong node type at the JVM layer.

Workaround — two steps:

1. Set `full_html_extraction_enabled: true` in the crawler config. This stores raw page HTML in the `full_html` field of every indexed document.
2. Add an Elasticsearch ingest pipeline with a Painless script that regex-extracts the attribute values from `full_html`, writes them to the target field, then removes `full_html` to avoid index bloat.

No page changes required. Confirmed on Open Crawler v0.3.0 – v1.0.0 (all ES 9.x-compatible releases). v0.2.1 (the ES 9.x minimum) used Nokogiri and was not affected, but also predates several other features.

**Artifacts**

- Sanitized pattern: [artifacts/crawler/xpath-attribute-extraction](artifacts/crawler/xpath-attribute-extraction/) — [README](artifacts/crawler/xpath-attribute-extraction/README.md), [open-crawler.yml](artifacts/crawler/xpath-attribute-extraction/open-crawler.yml), [ingest-pipeline.json](artifacts/crawler/xpath-attribute-extraction/ingest-pipeline.json), [diagnostics](artifacts/crawler/xpath-attribute-extraction/diagnostics.md)

**References**

- [elastic/crawler — `html.rb` `extract_by_xpath_selector` (v1.0.0)](https://github.com/elastic/crawler/blob/v1.0.0/lib/crawler/data/crawl_result/html.rb)
- [Jsoup `selectXpath` docs](https://jsoup.org/apidocs/org/jsoup/nodes/Element.html#selectXpath(java.lang.String,java.lang.Class))
- [Open Crawler config reference — `full_html_extraction_enabled`](https://github.com/elastic/crawler/blob/v1.0.0/docs/CONFIG.md)
- [Elasticsearch ingest pipelines — script processor](https://www.elastic.co/docs/reference/enrich-processor/script-processor)
- [docs/migration-overview.md](docs/migration-overview.md)

---

## FAQ-005

**Question:** Fields extracted via `data-elastic-name` HTML attributes returned arrays of strings in the Enterprise Search crawler. Open Crawler returns only a single string. Why, and how do we restore array behaviour?

**Resolution:** Open Crawler v0.3.0+ collects `data-elastic-name` values into a Ruby hash with plain assignment (`extractions[name] = value`). When multiple elements share the same attribute name, each iteration overwrites the previous, leaving only the last element's text.

There is a secondary complication: in `document_mapper.rb`, `meta_tags_and_data_attributes` (which calls the built-in handler) merges **after** `extraction_rule_fields`. An extraction rule with the same field name is overwritten by the single-string value before the document is indexed.

Workaround — two steps:

1. Add an extraction rule using a CSS selector (`[data-elastic-name='field_name']`) with `join_as: array` and a **staging field name** (e.g. `my_field_list`) to avoid the merge-order collision.
2. Add an Elasticsearch ingest pipeline `rename` processor to promote the staging array over the single-string value from the built-in handler.

Confirmed on Open Crawler v0.3.0 – v1.0.0. v0.2.1 predates `data-elastic-name` support entirely.

**Artifacts**

- Sanitized pattern: [artifacts/crawler/data-elastic-name-arrays](artifacts/crawler/data-elastic-name-arrays/) — [README](artifacts/crawler/data-elastic-name-arrays/README.md), [open-crawler.yml](artifacts/crawler/data-elastic-name-arrays/open-crawler.yml), [ingest-pipeline.json](artifacts/crawler/data-elastic-name-arrays/ingest-pipeline.json), [diagnostics](artifacts/crawler/data-elastic-name-arrays/diagnostics.md)

**References**

- [elastic/crawler — `html.rb` `data_attributes_from_body` (v1.0.0)](https://github.com/elastic/crawler/blob/v1.0.0/lib/crawler/data/crawl_result/html.rb)
- [elastic/crawler — `document_mapper.rb` merge order (v1.0.0)](https://github.com/elastic/crawler/blob/v1.0.0/lib/crawler/document_mapper.rb)
- [Open Crawler extraction rules docs](https://github.com/elastic/crawler/blob/v1.0.0/docs/features/EXTRACTION_RULES.md)
- [Elasticsearch ingest pipelines — rename processor](https://www.elastic.co/docs/reference/enrich-processor/rename-processor)
- [docs/migration-overview.md](docs/migration-overview.md)

---
