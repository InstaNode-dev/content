# instanode.dev — real infrastructure for AI agents

> Provision real Postgres, Redis, MongoDB, NATS, S3-compatible object storage, webhooks, and container deploys with single HTTP calls. No signup, no API key, no Docker, no cloud account. The first 24 hours are anonymous; claim a resource, then upgrade to a paid tier to keep it past 24 hours.

## The pitch in one curl

```
curl -X POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"home-db"}'
```

Sub-second response with a real `connection_url` to a dedicated Postgres database. No setup. No installer. No "verify your email." The platform shapes itself around what an AI agent or developer was going to type anyway.

## The seven services

- **Postgres** — `POST /db/new`. pgvector pre-installed.
- **Redis** — `POST /cache/new`. ACL'd per token.
- **MongoDB** — `POST /nosql/new`. Per-token user.
- **NATS JetStream** — `POST /queue/new`. Durable pub/sub + request/reply.
- **S3-compatible storage (DigitalOcean Spaces)** — `POST /storage/new`. Standard S3 SDKs work as-is.
- **Webhook receiver** — `POST /webhook/new`. Public URL captures any HTTP method.
- **Container deploy** — `POST /deploy/new`. Tarball in, HTTPS URL out. ~30–90 seconds.

## How it works

- **Anonymous-first.** The first 24 hours of every resource are free with no signup. Try the platform before deciding to pay.
- **Claim, then upgrade to keep.** Claiming the resource (magic-link email; no password) attaches it to an account. The Free tier mirrors the anonymous 24h TTL — to keep resources past 24 hours, upgrade to a paid tier (Hobby $9/mo or above) in the dashboard.
- **Pay from day one of upgrading.** Hobby is $9/mo, Pro is $49/mo, Team is $199/mo. No trial period on paid tiers — the anonymous slice IS the trial.

## Built for agents

The platform exists because AI agents (Claude Code, Cursor, MCP tools) need real backing services on demand. Single HTTP calls fit in muscle memory for LLM tool use. Predictable JSON responses. OpenAPI 3.1 spec at [openapi.json](https://api.instanode.dev/openapi.json). One `llms.txt` at [/llms.txt](/llms.txt) so any LLM can use the API directly.

## Pages

- [/pricing](/pricing) — full tier comparison
- [/use-cases](/use-cases) — 100+ scenarios with detail pages and paste-ready LLM prompts
- [/docs](/docs) — quickstart, services, deploy, claim flow
- [/for-agents](/for-agents) — specifically for AI agent operators
- [/blog](/blog) — build notes and retrospectives
- [/status](/status) — current platform health

## Try it now

```
curl -X POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"home-db"}'
```

That's the entire onboarding. The response includes everything you need.
