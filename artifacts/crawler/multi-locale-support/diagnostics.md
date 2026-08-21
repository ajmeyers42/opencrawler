# Diagnostics: multi-locale Open Crawler

Run these from an environment that can reach the target site (customer egress / allowlisted IP). Shared CI or laptops are often blocked by WAF challenge pages.

## 1. Confirm HTML vs bot challenge

```bash
LOCALE_PATH="/uk/en"   # also try /us/en and /mx/en
curl -sL -A "Elastic-Crawler (Open Crawler)" \
  "https://support.example.com${LOCALE_PATH}" \
  | head -c 2000
```

- Real support markup → Open Crawler can fetch content (subject to JS-rendered sections).
- “Pardon Our Interruption”, Incapsula, Cloudflare challenge, or empty body → WAF allowlisting or alternate ingest path required before locale tuning matters.

## 2. Sitemap reachability

```bash
for s in sitemap_us.xml sitemap_uk.xml sitemap_mx.xml; do
  echo "=== $s ==="
  curl -sI -A "Elastic-Crawler (Open Crawler)" \
    "https://support.example.com/$s" | head -5
done
```

Expect `200` and `application/xml` (or similar). `403` on sitemaps blocks locale discovery even when a home page loads.

## 3. Small parity crawl (no Elasticsearch)

Point Open Crawler at a **file** or **console** sink with a tiny seed set (one URL per locale) and `max_crawl_depth: 1` or `2`. Confirm:

- Each locale URL produces a document.
- `crawled_country` / `crawled_language` match the path (`uk`/`en`, `mx`/`en`, `us`/`en`) — not all `us`.
- Article/product rulesets populate `crawled_content_body` where expected.

## 4. Index checks (ES|QL)

After indexing to `search-crawler-support` (or your index):

```esql
FROM search-crawler-support
| STATS docs = COUNT(*) BY crawled_country, crawled_language
| SORT crawled_country
```

```esql
FROM search-crawler-support
| WHERE crawled_country == "uk"
| KEEP url, crawled_country, crawled_language, crawled_header
| LIMIT 20
```

```esql
FROM search-crawler-support
| WHERE url LIKE "*://support.example.com/uk/*"
  AND (crawled_country != "uk" OR crawled_country IS NULL)
| KEEP url, crawled_country
| LIMIT 50
```

The last query should return **zero** rows after the URL-derived locale fix.

## 5. Filter regression check

Pick one known article URL per locale. If extraction is empty only when `url_filters` use `begins: /product/...`, rewrite filters to include the locale prefix or use `contains` / `regex` as in [open-crawler.yml](./open-crawler.yml).
