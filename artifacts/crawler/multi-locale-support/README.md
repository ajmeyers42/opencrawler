# Artifact: Multi-locale support site (Open Crawler)

| Field | Value |
|-------|--------|
| FAQ | [FAQ-001](../../../FAQ.md#faq-001) |
| Area | crawler |
| Audience | shareable (`main`) |

## Problem

After migrating an Elastic / App Search web crawler config to Open Crawler, **one locale path on a shared host works** (for example `/us/en`) while **sibling locales** (for example `/uk/en`, `/mx/en`) do not behave the same — missing docs, wrong country filters, or empty extraction.

## Diagnosis

Work top-down; most regressions are config, not “Open Crawler cannot do locales.”

1. **Hardcoded locale metadata** — catch-all `set` rules that always write `crawled_country: us` (or similar) make every locale look like the primary market in search facets.
2. **Weak multi-locale discovery** — a single root seed plus hope that sitemaps cover every locale. Prefer explicit seeds per locale path and keep per-locale sitemaps.
3. **Locale-unaware URL filters** — `begins` patterns like `/product/...` miss real paths shaped `/{country}/{lang}/product/...`. Use `contains`, locale-aware `regex`, or filters that include the locale prefix.
4. **Bot protection / WAF** — Imperva, Cloudflare, and similar can challenge crawlers. Confirm with crawl logs (403 or challenge HTML). Allowlist crawler egress; try sitemap-first. Secondary unless logs show locale-specific blocks.

```text
Locale A works, B/C do not?
├── Do B/C URLs appear in the crawl / index at all?
│   ├── No → seeds, sitemaps, crawl_rules, or WAF
│   └── Yes → check crawled_country / language fields
├── Are country/language hardcoded in extraction?
│   └── Yes → derive from URL (see open-crawler.yml)
└── Do content-type rulesets use begins without locale prefix?
    └── Yes → switch to contains / regex with /{cc}/{lang}/
```

## Resolution pattern

See [open-crawler.yml](./open-crawler.yml):

- Seed each locale home (`/us/en`, `/uk/en`, `/mx/en`, …).
- Keep per-locale `sitemap_urls`.
- **Derive** `crawled_country` and `crawled_language` from the URL with `source: url` — never hardcode a single country on the catch-all ruleset.
- Align extraction `url_filters` with `/{country}/{lang}/product/...` (regex or `contains`).
- Optionally split into one config (or `domains` entry) per locale if schedules or quotas differ.

## Validation

Follow [diagnostics.md](./diagnostics.md): small console/file crawl per locale, then ES|QL counts by `crawled_country`.

## References

- [Migrating to 9.x from Enterprise Search 8.x](https://www.elastic.co/guide/en/enterprise-search/8.19/upgrading-to-9-x.html)
- [Open Crawler release (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-open-crawler-release)
- [Open Crawler config as code (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-open-crawler-config-as-code)
- [App Search data after Stack 9 (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-app-search-data-elasticsearch-9)
