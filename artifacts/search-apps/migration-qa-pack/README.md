# Artifact: Open Crawler / App Search → Elastic 9.x Q&A pack

| Field | Value |
|-------|--------|
| FAQ | [FAQ-003](../../../FAQ.md#faq-003) |
| Area | search-apps / crawler / settings |
| Audience | shareable (`main`) |

## Problem

During App Search / Elastic crawler → Open Crawler migrations on ECH 9.x, customers typically ask the same set of questions: index-template analyzers and App Search schema fields, ILM tier movement, dynamic vs static crawler mappings, Behavioral Analytics on 9.x, and Kibana Data Views.

## Resolution pattern

| # | Topic | Priority | Detail |
|---|--------|----------|--------|
| 1 | Special characters in analyzers | HIGH | [01-special-characters.md](./01-special-characters.md) |
| 2 | App Search `delimiter` / `enum` / `joined` / `prefix` / `stem` fields | HIGH | [02-app-search-derived-fields.md](./02-app-search-derived-fields.md) |
| 3 | ILM not moving data across tiers | MEDIUM | [03-ilm-tiers.md](./03-ilm-tiers.md) |
| 4 | Dynamic mapping vs static crawler templates | MEDIUM | [04-dynamic-vs-static-mapping.md](./04-dynamic-vs-static-mapping.md) |
| 5 | Behavioral Analytics on 9.x | LOW | [FAQ-002](../../../FAQ.md#faq-002) · [behavioral-analytics-parity](../../search-analytics/behavioral-analytics-parity/) |
| 6 | Typical Data View use cases | LOW | [06-data-views.md](./06-data-views.md) |

## Customer-facing one-liners

Paste-ready summaries live in [customer-responses.md](./customer-responses.md). Link each row to the detail file above (or FAQ-002 for #5).

## References

- [Migrating to 9.x from Enterprise Search 8.x](https://www.elastic.co/guide/en/enterprise-search/8.19/upgrading-to-9-x.html)
- [docs/migration-overview.md](../../../docs/migration-overview.md)
- [Mixing exact search with stemming](https://www.elastic.co/docs/solutions/search/full-text/search-relevance/mixing-exact-search-with-stemming)
- [Kibana data views](https://www.elastic.co/docs/explore-analyze/find-and-organize/data-views)
