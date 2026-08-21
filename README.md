# Open Crawler: Enterprise Search migration FAQ

Shareable playbooks for migrating from Elastic Enterprise Search (8.x App Search / Elastic web crawler) to **[Open Crawler](https://github.com/elastic/crawler)** on Elastic Cloud Hosted (ECH), including related 9.x search-app questions that come up during that move.

This repository is meant to be shared with customers and peers. It contains sanitized FAQ answers and worked configs only.

## How to use this repo

1. Start at **[FAQ.md](FAQ.md)** — customer questions, short resolutions, and links to artifacts.
2. Open the linked folder under `artifacts/` for YAML, diagnostics, and validation steps.
3. Read **[docs/migration-overview.md](docs/migration-overview.md)** for the 8.19 → 9.x crawler map.

Official Elastic migration notebooks and Search Labs posts are linked from the FAQ and each artifact; this repo does not replace them.

## Branch model

| Branch | Contents |
|--------|----------|
| `main` | Sanitized, shareable playbooks and examples only |
| `customer/<slug>` | Customer-specific notes, real hostnames, and configs |

Customer branches **may be shared before sanitization**. Before merging into `main`, content must pass the sanitize checklist in [CONTRIBUTING.md](CONTRIBUTING.md).

Do **not** add a `customers/` tree on `main`.

## Layout

```text
FAQ.md                              # Question index
docs/migration-overview.md          # ECH 8.19 → 9.x crawler narrative
artifacts/crawler/                  # Open Crawler patterns
artifacts/search-analytics/         # Behavioral Analytics parity (FAQ-002)
templates/                          # Artifact and customer-branch templates
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for branch naming, sanitization rules, and how to add a new FAQ entry.
