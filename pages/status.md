# Status — instanode.dev

> Live operational status for the platform. The same status page is served as JSON at [/api/v1/status](https://api.instanode.dev/api/v1/status) for machine consumption.

## Current status

All systems operational. (For the live indicator and per-service uptime metrics, see the HTML page at [/status](/status).)

## Services tracked

- **/db/new** — Postgres provisioning
- **/cache/new** — Redis provisioning
- **/nosql/new** — MongoDB provisioning
- **/queue/new** — NATS provisioning
- **/storage/new** — S3-compatible object storage provisioning (DigitalOcean Spaces)
- **/vector/new** — pgvector-enabled Postgres provisioning
- **/webhook/new** — Webhook receiver
- **/deploy/new** — Container deploy build pipeline
- **/claim** — Anonymous → paid claim flow

## SLOs

- **Provisioning latency** (anonymous tier): p99 < 2 seconds for /db/new, /cache/new, /nosql/new.
- **Build latency** (deploy): p99 < 120 seconds for typical Node / Python apps.
- **Availability**: 99.9% target for the agent-facing API and the customer data plane (databases, caches).

## Incident history

For a chronological list of incidents, see the HTML page. Recent fixes and platform speed-ups are also documented in build-notes posts on [/blog.md](/blog.md), including:

- [How /db/new dropped from 17s to under a second](/blog/why-pool-makes-curl-instant)
- [Strict-discipline shipping — change → live test → PR → merge](/blog/shipping-with-strict-discipline)

## Machine-readable

- Status JSON: `https://api.instanode.dev/api/v1/status`
- Per-service metrics (Prometheus format): `https://api.instanode.dev/metrics` (requires a bearer token — not public; returns `401` without it)
- OpenAPI spec: `https://api.instanode.dev/openapi.json`
