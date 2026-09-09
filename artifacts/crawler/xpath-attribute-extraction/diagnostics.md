# Diagnostics: XPath attribute extraction

## 1. Confirm the selector is attribute-axis

If a selector ends with `/@attribute-name`, it is an attribute-axis expression and will always return empty in Open Crawler v0.3.0+. Confirm:

```bash
# Does the selector contain /@?
echo "//*[@class='product-variant_list']/@data-attribute-type" | grep -o '/@'
# Should print: /@
```

## 2. Verify the element and attribute exist on the page

```bash
# Fetch the page and grep for the class and attribute
curl -sL -A "Elastic-Crawler (Open Crawler)" \
  "https://support.example.com/us/en/products/example-product" \
  | grep -i 'product-variant_list'
```

Confirm `data-attribute-type` appears in the output. If not, the selector is wrong regardless of XPath vs pipeline approach.

## 3. Confirm full_html is populated (file sink)

Switch to a file/console sink temporarily:

```yaml
output_sink: file
output_dir: /config/results/xpath-attr
max_crawl_depth: 1
full_html_extraction_enabled: true
```

```bash
# After a crawl, check for full_html in a result document
cat /config/results/xpath-attr/*.json | python3 -c "
import json, sys
for line in sys.stdin:
    doc = json.loads(line)
    fh = doc.get('full_html', '')
    if 'product-variant_list' in fh:
        print('FOUND — full_html contains target element')
        break
else:
    print('NOT FOUND — check element name or full_html_extraction_enabled')
"
```

## 4. Test the ingest pipeline in Dev Tools

Deploy the pipeline from `ingest-pipeline.json`, then simulate against a document with `full_html`:

```json
POST _ingest/pipeline/extract-html-attributes/_simulate
{
  "docs": [
    {
      "_source": {
        "full_html": "<ul class='product-variant_list' data-attribute-type='color'>...</ul><ul class='product-variant_list' data-attribute-type='finish'>...</ul>"
      }
    }
  ]
}
```

Expected response includes `pdp_variant_label: ["color", "finish"]` and no `full_html` field.

## 5. Index checks (ES|QL)

After a full crawl with the pipeline enabled:

```esql
FROM search-crawler-example
| WHERE pdp_variant_label IS NOT NULL
| KEEP url, pdp_variant_label
| LIMIT 20
```

```esql
FROM search-crawler-example
| WHERE full_html IS NOT NULL
| STATS leftover = COUNT(*)
```

The second query should return `0` if the pipeline's `remove` processor ran correctly.
