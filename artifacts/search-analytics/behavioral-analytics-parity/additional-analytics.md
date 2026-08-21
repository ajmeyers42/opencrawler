# Additional analytics (beyond 8.19 BA parity)

Extend the [FAQ-002](../../../FAQ.md#faq-002) events + ES|QL baseline with **search latency (OpenTelemetry / EDOT)** and other common product asks. This is additive — it does not replace the events data stream.

## Reporting model (read this first)

On Elastic 9.x there is **no Behavioral Analytics Kibana application** and no “Collections” UI to turn analytics on.

| What customers had in 8.19 | What they get on 9.x |
|----------------------------|----------------------|
| Behavioral Analytics Kibana UI | **Gone** |
| `_application/analytics*` collections API | **Deprecated** — do not design on it |
| Built-in BA charts | **You build** ES\|QL queries and Kibana **dashboards** (Discover ES\|QL, Lens/ES\|QL panels, alerts) |

**Analytics is enabled by instrumentation + queries/dashboards**, not by flipping a Kibana feature flag or opening a dedicated analytics app.

- **Collection** — search UI → ingest API → `logs-search_analytics.events-*` ([event-contract.md](./event-contract.md)); optional EDOT spans → Managed OTLP → `traces-*` (this doc).
- **Reporting** — ES\|QL (and standard visualizations) against those indices/data streams ([esql-dashboard.md](./esql-dashboard.md), latency queries below).
- **Not** — Behavioral Analytics menu, Search UI Analytics plugin, or browser OTel RUM as the production BA product (RUM is tech preview).

```text
Events path (BA parity)          Latency path (additional)
Search UI → ingest API           Search API → EDOT → mOTLP
        → events DS                      → traces-*
                \                      /
                 ES|QL dashboards / alerts
              (no dedicated BA Kibana UI)
```

---

## 1. Search latency via OpenTelemetry (EDOT)

### When to add this

After BA-parity events are flowing. Latency answers “was search slow?” — BA events alone do not.

### Production path (GA)

1. Instrument the **search backend** with [EDOT](https://www.elastic.co/docs/reference/opentelemetry) (Python / Java / .NET as applicable).
2. Export OTLP to the [Elastic Cloud Managed OTLP Endpoint](https://www.elastic.co/docs/solutions/observability/get-started/quickstart-elastic-cloud-otel-endpoint) (ECH **9.0+**).
3. Use an API key with `apm` `event:write` (server-side only — never in the browser).
4. Stamp the **same** correlation fields you use on events:

| Attribute on span | Purpose |
|-------------------|---------|
| `session.id` | Join to events DS |
| `search.query` | Slow-query tables |
| `search.result_count` | Context |
| `search.zero_results` | Correlate slow + empty |
| `search.search_application` | Multi-app deployments |
| optional `search.trace_id` on the **event** | Carry APM/OTel trace id onto the `search` event for one-hop joins |

### Example ES|QL — p95 latency

```esql
FROM traces-*
| WHERE span.name == "search" OR transaction.name == "search"
| WHERE attributes.session.id IS NOT NULL OR labels.session_id IS NOT NULL
| STATS
    searches = COUNT(*),
    p50_ms = ROUND(PERCENTILE(event.duration / 1000000.0, 50), 2),
    p95_ms = ROUND(PERCENTILE(event.duration / 1000000.0, 95), 2),
    p99_ms = ROUND(PERCENTILE(event.duration / 1000000.0, 99), 2)
```

Field names for duration and custom attributes vary slightly by ingest path (classic APM vs mOTLP). Confirm with one sample document in Discover, then lock panel queries.

### Example ES|QL — slow queries

```esql
FROM traces-*
| WHERE attributes.search.query IS NOT NULL OR labels.search_query IS NOT NULL
| EVAL query = COALESCE(attributes.search.query, labels.search_query)
| EVAL duration_ms = event.duration / 1000000.0
| STATS
    n = COUNT(*),
    p95_ms = ROUND(PERCENTILE(duration_ms, 95), 2),
    avg_ms = ROUND(AVG(duration_ms), 2)
  BY query
| WHERE n >= 5
| SORT p95_ms DESC
| LIMIT 50
```

### Dashboard placement

Add panels to the **same** search-analytics Kibana space as the BA-parity dashboard — still **custom dashboards**, not a product UI. Alert on p95 above a threshold if ops needs it.

### Browser RUM

[OpenTelemetry RUM](https://www.elastic.co/docs/solutions/observability/applications/otel-rum) can add client-side timing later. Treat as **optional / tech preview**; if used, put a reverse proxy in front of mOTLP so the API key is not in the page. Do **not** present RUM as the Behavioral Analytics replacement.

---

## 2. Other common expansions

All of these are still **events and/or spans + ES|QL dashboards** — no dedicated Kibana analytics app.

| Feature | Collection | Reporting |
|---------|------------|-----------|
| **MRR / mean click position** | `search.click_position` on `search_click` (parity contract) | [esql-dashboard.md](./esql-dashboard.md) |
| **Result impressions** | New `search_impression` (doc ids + positions shown) | Impression-weighted CTR |
| **Filter / facet usage** | `search.filters` on `search` | Top filters, zero-results by filter |
| **Experiments / A/B** | `labels.experiment`, `labels.variant` on events + spans | CTR / MRR / p95 by variant |
| **Dwell / abandon** | Time between `search` and next event, or `search_abandon` | Soft quality signal |
| **Conversion / funnel** | Extra actions (`contact`, `add_to_cart`, `purchase`) + originating `search.query` / `search.trace_id` | Search-attributed outcomes |
| **Personalization context** | Consent-gated anonymous attributes | Segmented CTR / latency |
| **Closed loop** | No-result + low-CTR ES\|QL tables | Synonyms / query rules backlog |
| **Ops alerts** | Same data streams | Zero-results %, CTR floor, p95 latency |

### Suggested event additions (optional)

```text
search_impression  — after results render (ids + ranks)
search_abandon     — user left without click (timeout or navigation)
contact | purchase — business outcome (carry search.query / session.id)
```

Keep mapping updates in [data-stream-template.json](./data-stream-template.json) when you add actions.

---

## 3. Rollout order

1. **Parity** — events + ES\|QL dashboard ([README](./README.md)).
2. **Latency** — EDOT on search API + p50/p95/slow-query panels (this doc §1).
3. **Impressions + experiment labels** — fairer CTR and ranking tests.
4. **Funnel / conversion** — only with a clear business event.
5. **Browser RUM** — only if preview risk is accepted.

---

## 4. What not to tell customers

- “Turn on Behavioral Analytics in Kibana” — **there is no such UI on 9.x**.
- “Open Crawler gives you analytics” — crawler is content ingest only.
- “Use the Search UI Analytics plugin” — targets discontinued BA.
- “Ship the mOTLP API key in the frontend” — never.

## References

- [FAQ-002](../../../FAQ.md#faq-002) — BA parity baseline
- [esql-dashboard.md](./esql-dashboard.md) — behavior panels
- [Managed OTLP Endpoint](https://www.elastic.co/docs/solutions/observability/get-started/quickstart-elastic-cloud-otel-endpoint)
- [Elastic OpenTelemetry](https://www.elastic.co/docs/reference/opentelemetry)
- [OTel RUM (tech preview)](https://www.elastic.co/docs/solutions/observability/applications/otel-rum)
