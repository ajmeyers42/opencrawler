# Diagnostics: `data-elastic-name` multi-value array

## 1. Confirm multiple elements share the same attribute name

```bash
curl -sL -A "Elastic-Crawler (Open Crawler)" \
  "https://support.example.com/us/en/products/example" \
  | grep -o 'data-elastic-name="[^"]*"' | sort | uniq -c | sort -rn
```

Any attribute name with a count > 1 is a candidate for this issue.

## 2. Confirm the built-in handler produces a single string (file sink)

Run a small crawl with `output_sink: file`, no pipeline, and `max_crawl_depth: 1`:

```bash
cat /config/results/den-arrays/*.json | python3 -c "
import json, sys
for line in sys.stdin:
    doc = json.loads(line)
    val = doc.get('pdp_variant_label')  # replace with your field name
    if val is not None:
        print(type(val).__name__, repr(val))
        break
"
```

Expected (broken): `str 'color'` — confirms last-value-wins.

## 3. Confirm the staging field is an array (extraction rule, no pipeline)

Add the extraction rule from `open-crawler.yml` and crawl again to a file sink **without** the pipeline. Check the staging field:

```bash
cat /config/results/den-arrays/*.json | python3 -c "
import json, sys
for line in sys.stdin:
    doc = json.loads(line)
    val = doc.get('pdp_variant_label_list')  # staging field
    if val is not None:
        print(type(val).__name__, repr(val))
        break
"
```

Expected (correct): `list ['functionality', 'type', 'color']`.

Note: you will also see the single-string `pdp_variant_label` field alongside it in the file output. The pipeline's `rename` will overwrite it at index time.

## 4. Test the ingest pipeline in Dev Tools

Deploy the pipeline from `ingest-pipeline.json`, then simulate:

```json
POST _ingest/pipeline/fix-data-elastic-name-arrays/_simulate
{
  "docs": [
    {
      "_source": {
        "pdp_variant_label": "color",
        "pdp_variant_label_list": ["functionality", "type", "color"]
      }
    }
  ]
}
```

Expected response: `pdp_variant_label: ["functionality", "type", "color"]`, no `pdp_variant_label_list` field.

## 5. Index checks (ES|QL)

After a full crawl with the pipeline enabled:

```esql
FROM search-crawler-example
| WHERE pdp_variant_label IS NOT NULL
| EVAL is_array = MV_COUNT(pdp_variant_label) > 1
| STATS single = COUNT(is_array == false), multi = COUNT(is_array == true)
```

```esql
FROM search-crawler-example
| WHERE pdp_variant_label_list IS NOT NULL
| STATS leftover = COUNT(*)
```

The second query should return `0` — staging fields should be absent after the pipeline rename.
