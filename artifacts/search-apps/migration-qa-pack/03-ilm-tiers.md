# 03 — ILM policies and data tiers

## Customer ask

How do we ensure ILM policies are working? Elastic Cloud logs don’t seem to move to other tiers even though ILM is enabled.

## Summary answer

Enabling a policy is necessary but not sufficient. Tier moves require: the index is managed by that policy, phase timing conditions have elapsed, and the cluster **has nodes that can host** the target tier (warm/cold/frozen). Many ECH deployments are **hot-only**; a policy that says `cold` will not move shards until cold capacity exists.

## Checklist (quick call or self-serve)

1. **Which policy is attached?**  
   `GET my-index/_ilm/explain` (or the data stream’s backing index). Confirm `phase` / `action` / `step` and any `step_info` errors.
2. **Have conditions been met?**  
   Rollover (`max_age`, `max_primary_shard_size`) and `min_age` in later phases. New indices stay in `hot` until rollover succeeds.
3. **Do node roles match the policy?**  
   In Elastic Cloud, confirm the deployment actually includes warm/cold/frozen tiers if the policy references them. Hot-only → data stays hot (or ILM waits / errors on allocate).
4. **Is this a managed logging stream?**  
   `elastic-cloud-logs-*` / system logging may use Elastic-managed templates. Custom policies can conflict with managed settings—prefer adjusting the deployment’s logging retention / tier settings where Elastic manages them, or use a custom data stream you fully control.
5. **Allocation filters**  
   `_tier_preference` and cluster routing must allow the move; explain API often surfaces `rejected` reasons.

## Search content vs logs

For **crawled search indices**, default this repo’s guidance is **hot-only** unless the customer asks for other tiers (smaller indices, frequent updates). Apply multi-tier ILM when volume and retention justify cold/frozen nodes.

## Offer

Walk policy JSON + deployment topology on a short call: compare phase actions to provisioned tiers, then fix either the policy or the deployment sizing.
