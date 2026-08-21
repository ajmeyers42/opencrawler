# Contributing

## Goals

- Keep `main` **sanitized and shareable** with customers and peers.
- Capture real engagement detail on `customer/<slug>` branches (these may be shared dirty).
- Grow [FAQ.md](FAQ.md) as the index; put depth in `artifacts/`.
- Stay scoped to **Open Crawler** and related asks that come up during crawler migrations (for example search-behavior analytics on 9.x).

## Branch naming

| Pattern | Purpose |
|---------|---------|
| `main` | Sanitized FAQ + artifacts only |
| `customer/<slug>` | One engagement; real URLs, YAML, notes |
| `faq/<id>-short-title` | Optional: work on a single FAQ before PR |

Example: `customer/customerA`.

## Adding a FAQ entry

1. Assign the next ID (`FAQ-00N`).
2. Add a row/section in [FAQ.md](FAQ.md) with: question, short resolution, artifact links, Elastic / Search Labs refs.
3. Prefer a dedicated folder under `artifacts/<area>/<slug>/` over dumping YAML into the FAQ.
4. Copy [templates/artifact-readme.md](templates/artifact-readme.md) when creating a new artifact.
5. On a customer branch, copy [templates/customer-branch-notes.md](templates/customer-branch-notes.md) and link back to the FAQ ID.

## Sanitize checklist (required before merge to `main`)

- [ ] No real customer or engagement names (use personas: `customerA`, `acme`, etc.)
- [ ] No real hostnames — use `example.com` / `support.example.com`
- [ ] No API keys, cloud IDs, passwords, or private endpoints
- [ ] Index / pipeline / space names are generic (`search-crawler-support`, not prod customer names)
- [ ] Asset CDN or third-party URLs generalized or removed
- [ ] Screenshots and logs redacted
- [ ] FAQ text describes the *pattern*, not the named customer

Customer branches may retain dirty detail until merge. Sharing a dirty branch is allowed; merging it dirty is not.

## Pull requests into `main`

Use the PR template. Reviewers should reject merges that fail the sanitize checklist.

## What belongs where

| Content | Location |
|---------|----------|
| Question + short answer + links | `FAQ.md` |
| Sanitized YAML, diagnostics, validation | `artifacts/...` |
| Real customer config and investigation | `customer/<slug>` branch only |
| General Open Crawler how-to | Link out (Search Labs, Elastic docs) — do not duplicate wholesale |

## Out of scope for this repo

Do not add personal agent notes, private learning logs, or unrelated Elastic topics. Keep new FAQs tied to Open Crawler ingest or a related ask from a crawler-migration conversation.
