# For AI agents — instanode.dev

> The platform exists because AI agents need real backing services on demand. Every design choice — anonymous-first, single-HTTP-call endpoints, predictable JSON responses, OpenAPI spec, llms.txt — exists so a coding agent or autonomous loop can use the platform without any setup ceremony.

## The bootstrap an agent should follow

1. Read [/llms.txt](/llms.txt) for the platform's API contract.
2. (Optional) Read [openapi.json](https://api.instanode.dev/openapi.json) for the formal schema.
3. Provision the services the task needs. Each is one HTTP call:

```bash
H='Content-Type: application/json'
curl -X POST https://api.instanode.dev/db/new      -H "$H" -d '{"name":"prod-db"}'        # Postgres
curl -X POST https://api.instanode.dev/cache/new   -H "$H" -d '{"name":"sessions-cache"}' # Redis
curl -X POST https://api.instanode.dev/nosql/new   -H "$H" -d '{"name":"events-store"}'   # MongoDB
curl -X POST https://api.instanode.dev/queue/new   -H "$H" -d '{"name":"jobs-queue"}'     # NATS
curl -X POST https://api.instanode.dev/storage/new -H "$H" -d '{"name":"uploads-bucket"}' # S3-compatible
curl -X POST https://api.instanode.dev/webhook/new -H "$H" -d '{"name":"github-webhook"}' # Public receive URL
curl -X POST https://api.instanode.dev/vector/new  -H "$H" -d '{"name":"embeddings-db"}'  # pgvector Postgres
```

Each response includes `connection_url` and `upgrade_jwt`. The agent uses the URL immediately; the JWT is kept in case the user wants to claim the resource later.

## Conventions an LLM should follow

- **`name` is required on every provision.** All nine provisioning endpoints (`/db/new`, `/vector/new`, `/cache/new`, `/nosql/new`, `/queue/new`, `/storage/new`, `/webhook/new`, `/deploy/new`, `/stacks/new`) reject a request with no `name`. Pass a JSON string field — or a multipart form field on `/deploy/new` and `/stacks/new`. Rules: 1–64 characters, `^[A-Za-z0-9][A-Za-z0-9 _-]*$`. A missing `name` returns `400 {"error":"name_required"}`; a malformed one returns `400 {"error":"invalid_name"}`. Choose a descriptive name per resource so the user can tell them apart.
- **One curl per service.** No batch endpoints — fan out in parallel from bash with `&` or async in code.
- **Parse `connection_url` directly** — every response has `.connection_url` (or `.receive_url`, `.endpoint`); never construct URLs by hand.
- **Save `upgrade_jwt`** if the user wants to claim the resource later. It's also required as a Bearer token for `POST /deploy/new`.
- **No retry-on-201.** Provisioning is synchronous. If the call returns 200/201, the resource is ready to accept connections.
- **Internal vs public URL.** When the calling code itself runs inside a `/deploy/new` container, prefer `internal_url` (cluster-internal hostname). For everything else, use `connection_url` (public).
- **Fingerprint dedup.** Anonymous callers are rate-limited to 5 provisions per day per IP /24 subnet. The 6th call returns the existing token. Don't expect 6 different DBs from 6 calls on the same IP.

## Why the platform is friendly to agents

- No signup means no email-verification redirect in the middle of a tool call.
- No API key means no secrets to ferry between the operator and the agent.
- The 24-hour TTL means a wrong experiment costs nothing — let it expire.
- The claim flow is JWT-based, so an agent can produce the JWT and hand it back to the human user without exposing credentials.
- Endpoint paths fit in muscle memory: `/db/new`, `/cache/new`, `/deploy/new`. Predictable for both humans and LLM completions.

## What an agent can build in one session

See [/use-cases.md](/use-cases.md) for 100+ scenarios across 18 archetypes, including paste-ready prompts for vanilla LLMs (no MCP, no install). Highlights:

- [Ephemeral test database for a risky migration](/use-cases/ephemeral-test-database-for-a-risky-migration)
- [Coding-agent cross-session memory](/use-cases/coding-agent-cross-session-memory)
- [LangGraph fan-out research agents](/use-cases/langgraph-fan-out-research-agents)
- [Devin-style PR-bot fleet](/use-cases/devin-style-pr-bot-fleet)
- [One-afternoon MVP backend](/use-cases/one-afternoon-mvp-backend)

## MCP server

A read-only MCP server is available at [github.com/InstaNode-dev/mcp](https://github.com/InstaNode-dev/mcp) for Claude Desktop / Cursor / agentic IDEs that want first-class tool integration. Not required — the HTTP API alone is sufficient.
