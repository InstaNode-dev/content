# Pricing — instanode.dev

> Self-serve at every tier. No talk-to-sales gate. Anonymous is the funnel; Hobby pays for the side project; Pro lets one product run in production + staging + development; Team (coming soon) is for the company that ships every day.

## Tiers

### Anonymous — free, 24-hour TTL

- 10 MB Postgres / 2 connections
- 5 MB Redis
- 5 MB MongoDB / 2 connections
- NATS: 1 stream, 100 messages
- 100 MB MinIO bucket
- 1 small container deploy
- Webhook: last 100 received payloads
- **Lifetime: 24 hours, then the resource is reaped unless claimed.**

### Hobby — $9 / month

- 1 GB Postgres / 8 connections
- 50 MB Redis
- 1 GB MongoDB / 8 connections
- 5 GB MinIO storage
- 1 container deploy with custom domain
- No TTL on resources
- Self-serve checkout; no sales call

### Pro — $49 / month

- 5 GB Postgres / 20 connections
- 256 MB Redis
- 5 GB MongoDB / 20 connections
- 50 GB MinIO storage
- 5 medium container deploys
- Multi-environment workflow (dev / staging / prod)
- Vault for secrets
- Priority queue access on the build cluster

### Team — $199 / month (coming soon)

Dedicated infrastructure: your own Postgres instances, your own Redis cluster, isolated network. The cluster is provisioned in your name; we operate it. No sales call required.

## How billing works

- Claim a resource with `POST /claim` using the `upgrade_jwt` from any provisioning response.
- A magic link arrives by email — clicking it sets a session cookie.
- The Hobby tier is the default for newly claimed resources. Upgrade in the dashboard.
- All payments via Razorpay (cards, UPI, net banking).
- No invoicing gate, no contracts, no minimums.

## The anonymous-tier philosophy

Most platforms run a 14-day trial. We run a 24-hour anonymous tier. The decision to pay isn't "is this product good enough to justify a trial" — it's "is keeping this specific running thing worth $9/mo." See the blog post [Why anonymous is the trial](/blog/why-anonymous-is-the-trial.md) for the reasoning.

## Related

- [/llms.txt](/llms.txt) — API manifest for LLMs
- [/docs.md](/docs.md) — quickstart and service reference
- [/use-cases.md](/use-cases.md) — 100+ scenarios across tiers
