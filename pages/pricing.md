# Pricing — instanode.dev

> Anonymous is the funnel; Hobby pays for the side project; Pro lets one product run in production + staging + development. Hobby and Pro are self-serve checkout. Team is launching soon — contact sales for onboarding.

## Tiers

### Anonymous — free, 24-hour TTL

- 10 MB Postgres / 2 connections
- 5 MB Redis
- 5 MB MongoDB / 2 connections
- NATS: 64 MB JetStream storage
- 10 MB object storage (S3-compatible, DigitalOcean Spaces)
- Deploy: not available (requires Hobby or above)
- Webhook: last 100 received payloads
- **Lifetime: 24 hours, then the resource is reaped unless claimed.**

### Hobby — $9 / month

- 1 GB Postgres / 8 connections
- 50 MB Redis
- 100 MB MongoDB / 5 connections
- 512 MB object storage
- 1 container deploy (*.deployment.instanode.dev subdomain, no custom domain)
- No TTL on resources
- Self-serve checkout; no sales call

### Pro — $49 / month

- 10 GB Postgres / 20 connections
- 512 MB Redis
- 5 GB MongoDB / 20 connections
- 50 GB S3-compatible storage
- 10 medium container deploys
- Multi-environment workflow (dev / staging / prod)
- Vault for secrets
- Priority queue access on the build cluster

### Team — $199 / month (launching soon — contact sales)

Team is **not yet self-serve**: it cannot be purchased or claimed today. Email contact@instanode.dev for onboarding. When it ships, Team is planned with high finite limits (not unlimited):

- 50 GB Postgres / 100 connections
- 1.5 GB Redis
- 40 GB MongoDB / 50 connections
- 40 GB queues
- 300 GB object storage
- 30 GB vector
- 100 deployment apps
- 1000 vault entries
- 100k stored webhooks
- 50 custom domains
- 90-day backups with self-serve restore
- RBAC + audit log
- SSO / SAML and a 99.9% SLA are coming soon (not yet available)

Need more than these caps — or dedicated/isolated infra, multi-region, or compliance (SOC2 / BAA / SSO / SLA / DPA)? That's Enterprise — contact sales@instanode.dev.

## How billing works

- Claim a resource with `POST /claim` using the `upgrade_jwt` from any provisioning response.
- A magic link arrives by email — clicking it sets a session cookie.
- Newly claimed teams start on the Free tier (24h TTL, same limits as anonymous) — claiming gives you an account, not durability. Resources keep expiring at 24h until you upgrade to Hobby ($9/mo) or above in the dashboard.
- All payments via Razorpay (cards, UPI, net banking).
- No invoicing gate, no contracts, no minimums.

## The anonymous-tier philosophy

Most platforms run a 14-day trial. We run a 24-hour anonymous tier. The decision to pay isn't "is this product good enough to justify a trial" — it's "is keeping this specific running thing worth $9/mo." See the blog post [Why anonymous is the trial](/blog/why-anonymous-is-the-trial) for the reasoning.

## Related

- [/llms.txt](/llms.txt) — API manifest for LLMs
- [/docs.md](/docs.md) — quickstart and service reference
- [/use-cases.md](/use-cases.md) — 100+ scenarios across tiers
