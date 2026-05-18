---
title: Cloudflare sub-agent factory per user
category: L. Agent-factory / spawning patterns
services: ["pg", "deploy"]
scenario: A Durable-Object-style root agent mints a fresh sub-agent for each end-user, each with its own SQL-backed memory and a deploy slot for that user's tools.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Durable-Object-style root agent mints a fresh sub-agent for each end-user, each with its own SQL-backed memory and a deploy slot for that user's tools.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You operate a Cloudflare-Durable-Object-style sub-agent factory. When an end-user first interacts, mint them a fresh sub-agent: a row in agents Postgres, an isolated database for that agent's memory, and a deploy slot hosting that user's tool-runner. Resolve subsequent requests by user_id to the same sub-agent.
```

## Steps to follow

- **Step 1: Provision the control-plane Postgres.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"cloudflare-sub-agent-factory-per-u-db"}' -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Agent registry schema.**

  ```sql
  CREATE TABLE sub_agents (
    user_id text PRIMARY KEY,
    agent_db_url text NOT NULL,
    deploy_url text NOT NULL,
    spawned_at timestamptz DEFAULT now(),
    last_active_at timestamptz
  );
  ```

- **Step 3: On first request, mint memory DB + tool-runner deploy.**

  ```python
  def get_or_spawn(user_id):
      row = pg.fetchone("SELECT agent_db_url, deploy_url FROM sub_agents WHERE user_id=%s",
                        (user_id,))
      if row: return row

      mem = requests.post("https://api.instanode.dev/db/new",
                          headers={"Authorization": f"Bearer {T}"}).json()
      dep = requests.post("https://api.instanode.dev/deploy/new",
                          headers={"Authorization": f"Bearer {T}"},
                          json={"image": "ghcr.io/me/tool-runner:latest",
                                "env": {"DATABASE_URL": mem["connection_url"],
                                        "USER_ID": user_id}}).json()
      pg.execute("INSERT INTO sub_agents VALUES (%s,%s,%s,now(),now())",
                 (user_id, mem["connection_url"], dep["app_url"]))
      return mem["connection_url"], dep["app_url"]
  ```

- **Step 4: Route the user's request to their sub-agent.**

  ```python
  db_url, deploy_url = get_or_spawn(user_id)
  requests.post(f"{deploy_url}/handle", json=request_payload)
  ```

## Why this works on instanode.dev

Per-user isolation done right needs per-user resources, not per-user schemas inside one giant DB. `POST /db/new` + `POST /deploy/new` per first-time user gives the same isolation guarantee Durable Objects provide, without writing a Workers-runtime port. The control-plane Postgres maps user_id to (db, deploy) in one row; subsequent requests skip provisioning entirely. Cold-start spawn is two curls, ~2 seconds.

## Related cases

- [AgentCore tenant-scoped spawning](/use-cases/agentcore-tenant-scoped-spawning.md) — tenant-org-scoped variant of the same factory pattern
- [Per-tenant chatbot factory at signup](/use-cases/per-tenant-chatbot-factory-at-signup.md) — B2B SaaS sibling that adds Mongo+Redis per tenant
- [Smithery one-MCP-per-skill mint](/use-cases/smithery-one-mcp-per-skill-mint.md) — factory variant that mints one runtime per skill, not per user
