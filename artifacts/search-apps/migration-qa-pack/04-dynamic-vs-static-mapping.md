# 04 — Dynamic mapping vs static crawler index templates

## Customer ask

In the crawler index template from Elastic, why isn’t dynamic mapping used? We use dynamic mapping and dynamic templates today. Is there a specific use case?

## Summary answer

Static / `dynamic: false` (or carefully constrained dynamic templates) is the common **search-content** default so applications, ingest pipelines, and relevance configs see a stable schema. Dynamic mapping remains a valid choice when you intentionally want new extracted fields to appear without a mapping change.

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Static / `dynamic: false` | Predictable types; safer for Search Applications; fewer mapping explosions | New Open Crawler extraction fields need an explicit mapping update |
| Fully dynamic | Fast iteration; new HTML fields show up automatically | Wrong types (`text` vs `keyword`), mapping bloat, hard-to-fix queries |
| Dynamic templates (selective) | Allow known patterns (`extracted_*`, `meta.*`) without opening everything | Still needs discipline on naming and types |

## When Elastic / Open Crawler templates prefer static

- Crawled documents feed a **product search** or support search UI that expects known fields (`title`, `body`, `url`, locale, …).
- Semantic / `copy_to` / ingest pipelines assume fixed field names.
- Multiple crawls and locales must stay comparable for parity checks.

## When to keep dynamic (or dynamic templates)

- Early extraction experimentation (console/file sink first, then lock the mapping).
- Many site-specific meta fields with a shared prefix → dynamic template to `keyword` or `text` as appropriate.

## Practical recommendation for migration

1. Start from the Elastic / Open Crawler template (stable core).
2. Add **explicit** mappings for fields you already rely on from App Search.
3. Optionally add **dynamic templates** only for agreed prefixes from extraction rules.
4. Avoid unbounded full dynamic on production content indices.

Related: Open Crawler extraction patterns in [artifacts/crawler/multi-locale-support](../../crawler/multi-locale-support/).
