---
title: Tiers and limits
order: 7
---

| Tier      | Postgres   | Redis     | MongoDB    | TTL  | Price       |
| --------- | ---------- | --------- | ---------- | ---- | ----------- |
| Anonymous | 10MB / 2c  | 5MB       | 5MB / 2c   | 24h  | free        |
| Hobby     | 1GB / 8c   | 50MB      | 1GB / 8c   | none | $9 / mo     |
| Pro       | 5GB / 20c  | 256MB     | 5GB / 20c  | none | $49 / mo    |
| Team      | coming soon | — | — | — | $199 / mo |

"c" = simultaneous connections. The full table is at `/pricing`.

Limits are enforced at the Postgres user level (`CONNECTION LIMIT` on the
role) and via per-bucket storage quotas. Exceeding a limit returns a 402 with
an upgrade URL — your app keeps running, the next provision just fails.
