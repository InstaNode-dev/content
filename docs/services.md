---
title: The six services
order: 2
---

Every endpoint returns a `connection_url` (or `endpoint` / `receive_url`)
plus an `upgrade_jwt` you can hand to /claim.

- `POST /db/new` — Postgres (pgvector pre-installed)
- `POST /cache/new` — Redis (ACL'd, per-token key prefix)
- `POST /nosql/new` — MongoDB
- `POST /queue/new` — NATS JetStream
- `POST /storage/new` — S3-compatible (DigitalOcean Spaces, `nyc3`)
- `POST /webhook/new` — public URL that receives any HTTP method

Every response has the same shape: `{ ok, token, connection_url, internal_url,
tier, limits, note, upgrade_jwt }`. `internal_url` is the address to use
when the caller itself runs inside our cluster (i.e. via /deploy/new) —
public hostnames don't hairpin reliably from inside.
