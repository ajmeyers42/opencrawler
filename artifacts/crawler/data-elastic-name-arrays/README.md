# Artifact: `data-elastic-name` multi-value array (Open Crawler)

| Field | Value |
|-------|--------|
| FAQ | [FAQ-005](../../../FAQ.md#faq-005) |
| Area | crawler |
| Audience | shareable (`main`) |

## Problem

Fields populated via `data-elastic-name` HTML attributes returned **arrays of strings** in the Enterprise Search crawler (8.x). The same pages in Open Crawler return only a **single string** — the last element's value — when multiple elements share the same attribute name.

## Diagnosis

**Root cause 1 — hash overwrite in `data_attributes_from_body`:**

Open Crawler v0.3.0+ collects `data-elastic-name` values into a Ruby hash with plain assignment. Each iteration overwrites the previous value, so only the last element survives:

```ruby
# elastic/crawler lib/crawler/data/crawl_result/html.rb (v1.0.0)
extractions = {}
parsed_content.body.select("[data-elastic-name]").each do |data|
  extractions[data.attr('data-elastic-name')] = truncated_content  # last-value-wins
end
```

**Root cause 2 — merge order in `document_mapper.rb`:**

Even if an extraction rule with `join_as: array` is added for the same field name, it is overwritten before indexing. `meta_tags_and_data_attributes` (which calls the built-in handler) merges **after** `extraction_rule_fields`:

```ruby
# elastic/crawler lib/crawler/document_mapper.rb (v1.0.0)
{}.merge(
  core_fields(crawl_result),
  html_fields(crawl_result),
  url_components(crawl_result.url),
  extraction_rule_fields(crawl_result),       # ← 4th
  meta_tags_and_data_attributes(crawl_result) # ← 5th — overwrites extraction rules
)
```

**Affected versions:** v0.3.0 – v1.0.0. v0.2.1 (ES 9.x minimum) predates `data-elastic-name` support entirely.

```text
data-elastic-name field is a single string?
├── Multiple elements share the same attribute name? → YES → this issue
├── Only one element?                               → not this issue
└── Field is missing entirely?                      → check v0.2.1 / field name validity rules
```

## Resolution pattern

See [open-crawler.yml](./open-crawler.yml) and [ingest-pipeline.json](./ingest-pipeline.json).

**Step 1 — Add an extraction rule with a staging field name:**

Use a CSS selector targeting `[data-elastic-name='your_field']` with `join_as: array`. Use a **different field name** (convention: suffix `_list`) to avoid the merge-order collision with the built-in handler.

```yaml
- action: extract
  field_name: my_field_list        # staging name — not the final field name
  selector: "[data-elastic-name='my_field']"
  join_as: array
  source: html
```

**Step 2 — Add an ingest pipeline `rename` processor** to promote the staging array over the single-string value from the built-in handler. Add one `rename` per field that needs array behaviour.

**No page changes are required.**

## Validation

Follow [diagnostics.md](./diagnostics.md): run a small crawl, confirm the staging field is an array, deploy the pipeline, confirm the final field is an array and the staging field is gone.

## References

- [elastic/crawler — `html.rb` `data_attributes_from_body` (v1.0.0)](https://github.com/elastic/crawler/blob/v1.0.0/lib/crawler/data/crawl_result/html.rb)
- [elastic/crawler — `document_mapper.rb` merge order (v1.0.0)](https://github.com/elastic/crawler/blob/v1.0.0/lib/crawler/document_mapper.rb)
- [Open Crawler extraction rules docs](https://github.com/elastic/crawler/blob/v1.0.0/docs/features/EXTRACTION_RULES.md)
- [Elasticsearch ingest pipelines — rename processor](https://www.elastic.co/docs/reference/enrich-processor/rename-processor)
- [Open Crawler version compatibility](https://github.com/elastic/crawler/blob/v1.0.0/README.md)
