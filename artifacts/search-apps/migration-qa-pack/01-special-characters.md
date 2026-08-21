# 01 — Special characters in index templates

## Customer ask

What’s the best way to handle special characters? Currently using a `pattern_replace` filter that only keeps `.` `/` `-`.

## Summary answer

That pattern is a strong *denylist* (everything else is dropped). It fits path-like or SKU-like tokens; it is a poor default for product/support free text because punctuation such as `&`, `'`, `+`, and Unicode letters often matter for recall.

## Recommended approach

1. **Decide by field role**
   - Free-text (`title`, `body`, `description`): tokenize and normalize; do not strip to a tiny allowlist.
   - Identifiers / URLs / SKUs: keep a `keyword` (or `keyword` + custom normalizer) multi-field; optionally a path hierarchy tokenizer for URL paths.
2. **Prefer targeted char filters** over “keep only these three characters”:
   - `mapping` char_filter to map smart quotes → ASCII, or strip a known bad set.
   - `pattern_replace` only for a **known** noise pattern (e.g. strip HTML entities already handled at crawl time).
3. **Validate** with the [Analyze API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-analyze) on sample titles from the crawl index before promoting the analyzer.

### Example: conservative text analyzer (illustrative)

```json
{
  "settings": {
    "analysis": {
      "char_filter": {
        "quotes_to_ascii": {
          "type": "mapping",
          "mappings": ["‘ => '", "’ => '", "“ => \"", "” => \""]
        }
      },
      "analyzer": {
        "content_text": {
          "type": "custom",
          "char_filter": ["quotes_to_ascii", "html_strip"],
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  }
}
```

### Example: when the current allowlist *is* appropriate

Use a dedicated field (e.g. `part_number`) with your `pattern_replace` keep-`. /-` logic; do **not** apply that filter to `body` / `title`.

## Open Crawler note

Prefer cleaning in extraction / ingest pipelines when the noise is HTML-specific; keep index analyzers for linguistic normalization.
