## Summary

- FAQ ID(s):
- Artifact path(s):

## Sanitize checklist (required for merge to `main`)

- [ ] No real customer / engagement names
- [ ] No real hostnames (use `example.com` personas)
- [ ] No secrets (API keys, cloud IDs, passwords)
- [ ] Generic index / pipeline / space names
- [ ] Logs and screenshots redacted
- [ ] FAQ describes the pattern, not the named customer

## Customer branch

- [ ] Dirty detail remains only on `customer/<slug>` (not copied into `main` paths)

## Test plan

- [ ] Linked artifact README diagnosis matches the FAQ resolution
- [ ] YAML / examples render and use documented Open Crawler fields
- [ ] External Elastic / Search Labs links resolve
