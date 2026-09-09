# Artifact: XPath attribute extraction (Open Crawler)

| Field | Value |
|-------|--------|
| FAQ | [FAQ-004](../../../FAQ.md#faq-004) |
| Area | crawler |
| Audience | shareable (`main`) |

## Problem

An XPath extraction rule that selects an HTML element **attribute value** — for example `//*[@class='product-variant_list']/@data-attribute-type` — returned the correct values in the Enterprise Search crawler (8.x) but returns an **empty array** in Open Crawler.

## Diagnosis

Open Crawler v0.3.0+ migrated its HTML parser from Nokogiri to Jsoup. The `extract_by_xpath_selector` method in [`html.rb`](https://github.com/elastic/crawler/blob/v1.0.0/lib/crawler/data/crawl_result/html.rb) calls Jsoup's `selectXpath` with `TextNode.java_class` hardcoded as the target node type:

```ruby
parsed_content.selectXpath(selector, TextNode.java_class).map { |node| ... }
```

XPath attribute-axis expressions (e.g. `/@data-attribute-type`) resolve to **Attribute nodes**, not TextNodes. Jsoup filters them out silently — the XPath is syntactically valid but always yields zero results at the JVM layer.

**Affected versions:** v0.3.0 – v1.0.0 (all ES 9.x-compatible releases).  
v0.2.1 (the ES 9.x minimum) used Nokogiri's `search()`, which handles attribute nodes correctly, but v0.2.1 also predates several other features in common use.

```text
XPath selector returns empty array?
├── Does selector use /@attribute-name axis?  → YES → this issue
├── Does selector target element text?        → probably a different config problem
└── v0.2.1 only?                              → Nokogiri; attribute XPath works there
```

## Resolution pattern

See [open-crawler.yml](./open-crawler.yml) and [ingest-pipeline.json](./ingest-pipeline.json).

**Step 1 — Enable raw HTML storage in the crawler config:**

```yaml
full_html_extraction_enabled: true
```

This writes the full page HTML into the `full_html` field of every indexed document.

**Step 2 — Add an ingest pipeline with a Painless script** that regex-extracts the attribute values from `full_html`, writes them to the target field as an array, then removes `full_html` to avoid index bloat. The pipeline is referenced in the crawler config via `elasticsearch.pipeline`.

The regex approach handles both attribute-ordering variants in the HTML tag. Target field names and class/attribute names are placeholders — substitute your own.

**No page changes are required.**

## Validation

Follow [diagnostics.md](./diagnostics.md): enable `full_html_extraction_enabled`, run a small console/file crawl, confirm `full_html` is populated, then deploy the pipeline and confirm the target field is an array and `full_html` is absent.

## References

- [elastic/crawler — `html.rb` `extract_by_xpath_selector` (v1.0.0)](https://github.com/elastic/crawler/blob/v1.0.0/lib/crawler/data/crawl_result/html.rb)
- [Jsoup `selectXpath` docs](https://jsoup.org/apidocs/org/jsoup/nodes/Element.html#selectXpath(java.lang.String,java.lang.Class))
- [Open Crawler config reference — `full_html_extraction_enabled`](https://github.com/elastic/crawler/blob/v1.0.0/docs/CONFIG.md)
- [Elasticsearch ingest pipelines — script processor](https://www.elastic.co/docs/reference/enrich-processor/script-processor)
- [Open Crawler version compatibility](https://github.com/elastic/crawler/blob/v1.0.0/README.md)
