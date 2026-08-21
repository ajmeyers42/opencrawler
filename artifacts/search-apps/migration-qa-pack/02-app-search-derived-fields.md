# 02 — App Search derived fields (`delimiter`, `enum`, `joined`, `prefix`, `stem`)

## Customer ask

Nick said we don’t need the delimiter, enum, joined, prefix, and stem fields. Why not? What replaces that functionality?

## Why Nick is right

App Search engines historically **materialized** several derived subfields per text field so App Search’s query DSL could pick a strategy without exposing Elasticsearch analyzers. On native Elasticsearch / Search Applications those strategies are expressed with:

- field mappings (`text` + `keyword` multi-fields)
- custom analyzers
- query choice (`match`, `match_phrase_prefix`, `multi_match`, `search_as_you_type`, etc.)

Recreating every App Search subfield 1:1 inflates the index and is usually unnecessary after migration.

## Capability map

| App Search–style field | What it was for | Elasticsearch replacement |
|------------------------|-----------------|---------------------------|
| `*.enum` | Exact value / facets / filters | `keyword` multi-field (or dedicated `keyword` field) |
| `*.stem` | Stemmed full-text recall | `text` field with a stemming analyzer; optional exact field + [`quote_field_suffix`](https://www.elastic.co/docs/solutions/search/full-text/search-relevance/mixing-exact-search-with-stemming) |
| `*.prefix` | Prefix / typeahead | `search_as_you_type`, edge n-gram analyzer, or `match_phrase_prefix` / completion suggester |
| `*.delimiter` | Split on punctuation / path-like tokens | `char_group` / `word_delimiter_graph` token filter, or `path_hierarchy` for URLs |
| `*.joined` | Concatenated / alternate tokenization for matching | Often covered by `copy_to` into an `all_content` field, or a second analyzer via `fields` |

## What to do in practice

1. Inventory which App Search subfields queries actually use (App Search explain / query logs).
2. For each behavior still needed, add the **smallest** native equivalent (table above)—not all five on every field.
3. Put relevance in a Search Application template or application query builder rather than in dozens of stored subfields.
4. Re-test: exact filters, stemmed recall, and typeahead separately.

## Related

- [App Search data after Stack 9 (Search Labs)](https://www.elastic.co/search-labs/blog/elastic-app-search-data-elasticsearch-9)
- [Mixing exact search with stemming](https://www.elastic.co/docs/solutions/search/full-text/search-relevance/mixing-exact-search-with-stemming)
