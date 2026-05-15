---
title: Quickstart
order: 1
---

The whole platform fits in one curl. No signup, no API key, no Docker.

```
curl -X POST https://api.instanode.dev/db/new \
  -H "Content-Type: application/json" \
  -d '{"name":"prod-db"}'
```

Every provisioning endpoint **requires** a `name` — a human-readable label
1–64 characters long, matching `^[A-Za-z0-9][A-Za-z0-9 _-]*$` (start with a
letter or digit; letters, digits, spaces, underscores and hyphens after).
Omitting it returns `400 {"error":"name_required"}`; an invalid value
returns `400 {"error":"invalid_name"}`.

The response includes a `connection_url` you can paste into any Postgres
client. The database is real, dedicated, and yours for 24 hours.

When you're ready to keep it, see the **Claim flow** section below.
