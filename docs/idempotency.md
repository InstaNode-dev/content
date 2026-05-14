# Idempotency

Every `POST` endpoint that creates a resource on `https://api.instanode.dev` is idempotent. Two layered guards cover the full retry matrix so an accidental double-create never costs you a duplicate database, a duplicate Razorpay subscription, or a duplicate team-invite email.

## Two layers, one contract

### 1. Explicit `Idempotency-Key` header (Stripe-shape, 24h TTL)

The standard mechanism. Generate a UUID per *logical* attempt and pass it on every retry:

```bash
KEY=$(uuidgen)
curl -X POST https://api.instanode.dev/db/new \
  -H "Idempotency-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-database"}'
```

- The first response is cached for **24 hours** in the per-tenant key space (`team_id` when authenticated, network fingerprint when anonymous).
- Every subsequent call carrying the same key replays the cached response verbatim with `X-Idempotent-Replay: true`.
- Reusing the same key with a *different* request body returns `409 idempotency_key_conflict` — that's almost always a client bug (the key should change when the work changes).
- Replays still consume rate-limit budget (anti-abuse) but do **not** consume quota budget (the original call already did).

Use the explicit header when you need exactly-once across a long window — e.g. a deploy job that may not retry for several minutes, a billing checkout the user might re-click after lunch.

### 2. Body-fingerprint fallback (120s TTL)

When the caller omits `Idempotency-Key`, the server still protects against double-creates by synthesising a fallback key from `sha256(scope, route, canonical-body)`:

- Scope = `team_id` when authenticated, network fingerprint (`/24` subnet + ASN) when anonymous.
- Route = the registered route pattern (e.g. `/db/new`), not the full URL.
- Canonical body = JSON keys sorted recursively for `application/json`; SHA-256 of the file content + sorted form fields for `multipart/form-data` (used by `/deploy/new`).

Two POSTs from the same caller, to the same route, with the same body, within 120 seconds → the second replays the first. Mobile double-taps, browser back-button resubmits, agent retries on transient 5xx, and reverse-proxy network-blip retries are all absorbed.

The 120s window is deliberately short. If you need true exactly-once across a longer window, pass an explicit `Idempotency-Key` (which gets the full 24h cache).

### Worked example: a double-click produces one resource

The simplest demonstration is two back-to-back `/db/new` calls from the same caller with the same body. Without `Idempotency-Key`, the fingerprint fallback collapses them:

```bash
# Two POSTs ~50ms apart — same caller, same body, no Idempotency-Key.
curl -sS -D /tmp/h1 -X POST https://api.instanode.dev/db/new \
  -H 'Content-Type: application/json' -d '{"name":"my-db"}' > /tmp/r1.json &
curl -sS -D /tmp/h2 -X POST https://api.instanode.dev/db/new \
  -H 'Content-Type: application/json' -d '{"name":"my-db"}' > /tmp/r2.json &
wait

grep -i "X-Idempotency-Source\|X-Idempotent-Replay" /tmp/h1 /tmp/h2
#   /tmp/h1:X-Idempotency-Source: miss
#   /tmp/h2:X-Idempotency-Source: fingerprint
#   /tmp/h2:X-Idempotent-Replay: true

jq -r .token /tmp/r1.json /tmp/r2.json
#   tok_3jX...               (same token on both — one resource was created)
#   tok_3jX...
```

The second call replays the first response verbatim. Only one Postgres database was provisioned. The same shape holds for `/deploy/new` (multipart, canonicalised by hashing the tarball + sorted form fields) and for `/api/v1/billing/checkout` (where the dashboard's client-side debounce, this fingerprint fallback, and the per-team `checkout_in_flight` SETNX guard layer to make sure a double-click never charges twice).

## Response headers

Every response from a create endpoint carries:

| Header | Values | Meaning |
| --- | --- | --- |
| `X-Idempotency-Source` | `explicit` | Caller passed an `Idempotency-Key`; the explicit cache served or stored the response. |
| `X-Idempotency-Source` | `fingerprint` | The body-fingerprint cache served the response (it had been seen in the last 120s). |
| `X-Idempotency-Source` | `miss` | Handler ran fresh. The fingerprint cache will store the response for the next 120s. |
| `X-Idempotent-Replay` | `true` | The response body was served from the cache (either path). Absent on fresh handler runs. |

Agents can branch on `X-Idempotency-Source` to distinguish "I just created this" from "I created this earlier and the server replayed".

## Covered endpoints

Idempotency is automatic on every POST that creates a resource:

- Provisioning: `/db/new`, `/cache/new`, `/nosql/new`, `/queue/new`, `/storage/new`, `/webhook/new`, `/vector/new`.
- Compute: `/deploy/new`, `/stacks/new`, `/stacks/{slug}/redeploy`, `/api/v1/resources/{id}/backup`, `/api/v1/resources/{id}/restore`, `/api/v1/resources/{id}/provision-twin`, `/api/v1/families/bulk-twin`, `/api/v1/stacks/{slug}/promote`.
- Billing + team: `/api/v1/billing/checkout`, `/api/v1/team/members/invite`, `/api/v1/teams/{team_id}/invitations`, `/api/v1/auth/api-keys`.
- Vault: `/api/v1/vault/{env}/{key}/rotate`. Each call inserts a new versioned secret row; dedup prevents double-click duplicate versions. `PUT /api/v1/vault/{env}/{key}` is state-replacement by contract (caller supplies the value) and `DELETE` is idempotent by construction, so neither needs the middleware.

`DELETE`, `PATCH`, and read-only `GET` routes are not covered — they're either intrinsically idempotent (state-replacement) or token-bound single-use (e.g. `/claim`).

## Fail-open posture

If Redis (the cache backing the idempotency state) is unavailable, the middleware logs a warning and falls through to the handler. Idempotency degrades to "no dedup", never to "cannot create resource". Defense in depth — handler-level dedup mechanisms (per-fingerprint daily caps on provisioning, unique-index constraints on connections) still apply.

## Why this matters for AI agents

An autonomous agent that retries on transient 5xx without idempotency creates a duplicate resource every time the upstream wobbles. With the fingerprint fallback on by default, those retries are now safe even when the agent doesn't manage retry keys. With the explicit header, you get the same guarantee across arbitrarily long retry windows. Either way: one logical attempt produces exactly one resource.
