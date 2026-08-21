# Cutover sequence: 8.19 Behavioral Analytics → ECH 9.x events + Open Crawler

Coordinate **content** (Open Crawler) and **behavior** (new analytics) so Enterprise Search and Behavioral Analytics are retired only after parity is proven.

```text
8.19 production
  ├── App Search / Elastic crawler → engines
  └── Behavioral Analytics collections + tracker
                │
                │  dual-run
                ▼
Staging / side-by-side
  ├── Open Crawler → new ECH content indices
  └── Ingest API + tracker → logs-search_analytics.events-*
                │
                │  parity week
                ▼
Cutover
  ├── Search UI → Open Crawler indices / Search Application
  ├── Tracker → new events only (stop BA plugin / DSN)
  └── Retire Enterprise Search + BA
                │
                ▼
ECH 9.x upgrade (Upgrade Assistant; connectors self-managed)
```

## Prerequisites before dual-run

- [ ] Inventory: crawlers, App Search engines, BA collections, Search UI / custom tracker usage.
- [ ] ECH space, API keys, data stream template applied ([ech-prerequisites.md](./ech-prerequisites.md)).
- [ ] Open Crawler YAML converted (Elastic App Search / Elastic Crawler → Open Crawler notebooks).
- [ ] Event contract agreed ([event-contract.md](./event-contract.md)).

## Phase 1 — Dual-run content

1. Run Open Crawler on customer infra writing to **new** indices (do not overwrite production App Search engines yet).
2. Keep 8.19 Enterprise Search serving production search.
3. Parity checks: doc counts (by locale/source if applicable), sample queries, extraction fields.
4. Point a **staging** Search UI / Search Application at the new indices.

## Phase 2 — Dual-run analytics (keep 8.19 BA live)

1. Deploy ingest API + tracker to **staging** search UI only.
2. Confirm events in Discover: `event.action` in `page_view`, `search`, `search_click`.
3. Stand up ES|QL dashboard panels from [esql-dashboard.md](./esql-dashboard.md) in the dedicated space.
4. Keep production on Behavioral Analytics (DSN / Search UI Analytics plugin) until Phase 4.

## Phase 3 — Parity week

Compare the same calendar window on both systems (staging traffic or shadowed production if available):

| Check | 8.19 BA | New events |
|-------|---------|------------|
| Search volume trend | BA overview | Overview ES\|QL |
| Top queries (top 20) | BA report | Top queries ES\|QL |
| No-result queries | BA report | No-result ES\|QL |
| Click / CTR ballpark | BA overview | CTR metrics |

Acceptable: directionally similar lists and rates (exact counts differ if traffic split or sampling differs). Investigate large gaps (missing click hooks, total-hits vs page size, consent gating).

## Phase 4 — Production cutover

1. Cut search app to Open Crawler indices / Search Application.
2. Enable tracker → ingest API in production; **remove** `@elastic/search-ui-analytics-plugin` and BA DSN script.
3. Monitor Discover + dashboard for 24–48h.
4. Freeze / archive 8.19 BA collection data if needed for historical reporting (export before decommission).
5. Decommission Enterprise Search only when crawler, search app, analytics, and connectors are verified.

## Phase 5 — Upgrade to ECH 9.x

1. Convert managed connectors to self-managed (Upgrade Assistant) — ECH blocks 9.x until done.
2. Resolve Upgrade Assistant warnings.
3. Upgrade deployment to 9.x.
4. Confirm analytics data stream and dashboard still work post-upgrade (BA Kibana UI will be gone — expected).

## Rollback

| Failure | Action |
|---------|--------|
| Content / relevance regression | Point Search UI back to App Search engines; keep Open Crawler dual-run |
| Analytics gap | Re-enable BA tracker on 8.19 only if still on 8.x; fix ingest hooks on staging before retry |
| Already on 9.x | Cannot restore BA UI — fix event pipeline; keep historical BA indices read-only if migrated |

## What not to do

- Upgrade to 9.x while production still depends on Behavioral Analytics UI or Enterprise Search crawler.
- Cut analytics and content in the same hour without a staging parity check.
- Leave an Elasticsearch write key in the production frontend during cutover.

## Related

- Content migration overview: [docs/migration-overview.md](../../../docs/migration-overview.md)
- FAQ: [FAQ-002](../../../FAQ.md#faq-002)
