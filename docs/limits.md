---
title: Tiers and limits
order: 7
---

| Tier      | Postgres   | Redis     | MongoDB    | TTL  | Price       |
| --------- | ---------- | --------- | ---------- | ---- | ----------- |
| Anonymous | 10MB / 2c  | 5MB       | 5MB / 2c   | 24h  | free        |
| Hobby     | 1GB / 8c   | 50MB      | 1GB / 8c   | none | $9 / mo     |
| Pro       | 5GB / 20c  | 256MB     | 5GB / 20c  | none | $49 / mo    |
| Team      | unlimited  | unlimited | unlimited  | none | $199 / mo — coming soon |

"c" = simultaneous connections. The full table is at `/pricing`.

**Team tier status:** the tier is defined in `plans.yaml` with the limits
above, but customer-initiated checkout for it is blocked at the API level
(`POST /api/v1/billing/checkout` returns `400 tier_unavailable` for
`plan=team`). Email support@instanode.dev for early access. Pro is the
ceiling for self-serve upgrades today.

Limits are enforced at the Postgres user level (`CONNECTION LIMIT` on the
role) and via per-bucket storage quotas. Exceeding a limit returns a 402 with
an upgrade URL — your app keeps running, the next provision just fails.
