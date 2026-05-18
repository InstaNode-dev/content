---
title: The seven services
order: 2
---

Every endpoint returns a `connection_url` (or `endpoint` / `receive_url` /
application URL) plus an `upgrade_jwt` you can hand to /claim.

- `POST /db/new` — Postgres (pgvector pre-installed)
- `POST /cache/new` — Redis (ACL'd, per-token key prefix)
- `POST /nosql/new` — MongoDB
- `POST /queue/new` — NATS JetStream
- `POST /storage/new` — S3-compatible (DigitalOcean Spaces, `nyc3`)
- `POST /webhook/new` — public URL that receives any HTTP method
- `POST /deploy/new` — container deploy (tarball in, HTTPS URL out)

## The required `name` field

Every provisioning endpoint above — plus `/stacks/new` —
**requires** a `name`. It is the human-readable label shown in the dashboard
and in `GET /api/v1/resources`.

- Send `name` as a JSON string field on `/db/new`, `/cache/new`, `/nosql/new`,
  `/queue/new`, `/storage/new`, `/webhook/new`, and `/stacks/new`.
- `/deploy/new` and `/stacks/new` are multipart — pass `name` as a form field.
- **Validation:** 1–64 characters, must match `^[A-Za-z0-9][A-Za-z0-9 _-]*$`
  (start alphanumeric; letters, digits, spaces, underscores, hyphens after).
- Omitting `name` → `400 {"error":"name_required"}`.
- An invalid value → `400 {"error":"invalid_name"}`.

```
curl -X POST https://api.instanode.dev/db/new \
  -H "Content-Type: application/json" \
  -d '{"name":"prod-db"}'

curl -X POST https://api.instanode.dev/cache/new \
  -H "Content-Type: application/json" \
  -d '{"name":"sessions-cache"}'

curl -X POST https://api.instanode.dev/nosql/new \
  -H "Content-Type: application/json" \
  -d '{"name":"events-store"}'

curl -X POST https://api.instanode.dev/queue/new \
  -H "Content-Type: application/json" \
  -d '{"name":"jobs-queue"}'

curl -X POST https://api.instanode.dev/storage/new \
  -H "Content-Type: application/json" \
  -d '{"name":"uploads-bucket"}'

curl -X POST https://api.instanode.dev/webhook/new \
  -H "Content-Type: application/json" \
  -d '{"name":"github-webhook"}'
```

Every response has the same shape: `{ ok, token, connection_url, internal_url,
tier, limits, note, upgrade_jwt }`. `internal_url` is the address to use
when the caller itself runs inside our cluster (i.e. via /deploy/new) —
public hostnames don't hairpin reliably from inside.
