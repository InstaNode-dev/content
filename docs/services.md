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

### NATS queue credentials (2026-05-20+)

The `/queue/new` response also includes a `credentials` object with per-tenant
NATS account credentials (MR-P0-5):

```json
{
  "ok": true,
  "connection_url": "nats://nats.instanode.dev:4222",
  "subject_prefix": "tenant_a1b2c3d4....",
  "auth_mode": "isolated",
  "credentials": {
    "auth_mode": "isolated",
    "nats_jwt":  "<base64 user JWT>",
    "nats_nkey": "SUAA…",
    "creds_file": "-----BEGIN NATS USER JWT-----\n…",
    "key_id":   "ABBA…"
  }
}
```

Pass `(nats_jwt, nats_nkey)` to `nats.UserJWTAndSeed()` in the NATS Go client,
or write `creds_file` to disk and pass the path to `nats.UserCredentials()`.
The tenant's JWT only permits pub/sub on the `subject_prefix.*` namespace —
publish to other tenants' subjects is denied at the server.

Resources provisioned before the operator-mode cutover (2026-05-20) carry
`auth_mode: "legacy_open"` and have no `credentials` field; they keep working
on the unauthenticated path until they recycle.

### Storage isolation mode (2026-05-20+)

The `/storage/new` response also includes a `mode` field that names the
isolation level the tenant landed on:

| mode | Meaning |
|---|---|
| `shared-master-key` | DO Spaces today. Every tenant holds the master key; isolation is by `prefix` convention. |
| `prefix-scoped` | Backend IAM enforces `s3:prefix` against `<prefix>/*` (R2, S3, MinIO). |
| `prefix-scoped-temporary` | Same as prefix-scoped but credentials are STS — they expire. |
| `broker` | No long-lived credential is issued. Use `POST /storage/:token/presign` for short-lived signed URLs (max 1h TTL). |

The mode is decided at boot time by the `OBJECT_STORE_BACKEND` env var and
the backend's `Capabilities()`. Agents should branch on `mode` if they
need to behave differently — e.g. when `broker`, never try to write
directly with `(access_key_id, secret_access_key)` since the response
won't carry them.

### Broker-mode access: `POST /storage/{token}/presign`

When the `/storage/new` response carries `mode: "broker"`, no long-lived
credential was issued. Use this endpoint to mint a short-lived signed S3
URL (≤1h TTL) constrained to the resource's own `prefix/*`:

```
curl -X POST https://api.instanode.dev/storage/$TOKEN/presign \
  -H "Content-Type: application/json" \
  -d '{"operation":"PUT","key":"uploads/photo.jpg","expires_in":600}'
# => {"ok":true,"url":"https://s3.instanode.dev/...?X-Amz-Signature=...","expires_at":"..."}
```

- `operation` — `"PUT"` (upload) or `"GET"` (download)
- `key` — object key, will be prefixed with the resource's `<prefix>/`
  internally; the path is scoped so a signed URL cannot escape the
  prefix even if leaked
- `expires_in` — TTL in seconds, clamped to `[1, 3600]`; values ≤ 0 are
  rejected with `400 invalid_expires_in`

The URL is signed by the platform master key but enforces the
tenant's prefix at sign time, so leaked URLs cannot read or write other
tenants' objects. Rate-limited per token.

## Operational endpoints

- `GET /healthz` — shallow liveness probe. Returns 200 with `{ok, commit_id, build_time, version}` if the binary is up and can ping its primary platform DB. Use this to verify a deploy SHA matches what you pushed.
- `GET /readyz` — deep readiness probe (added 2026-05-20). Multi-component upstream-reachability matrix returning per-check status + latency + last_checked timestamp. Per-check criticality decides 200 vs 503. See [Deploying an app](/docs#deploy) for the full envelope shape.
- `POST /webhooks/brevo/:secret` — Brevo delivery webhook receiver (internal). Authenticated by URL token. Overwrites `forwarder_sent.classification` with the real outcome (`delivered`, `bounced_hard`, `bounced_soft`, `rejected`, `complaint`, `deferred`, `unsubscribed`, `error`) and stamps `delivered_at` on `delivered`. The truth surface for "did the user receive the email" — the worker's 201 from the Brevo API only means the relay accepted the POST; the ledger row's classification (set by this webhook) is the real outcome.
