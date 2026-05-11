---
title: AgentCore tenant-scoped spawning
category: L. Agent-factory / spawning patterns
services: ["pg", "redis"]
scenario: An AWS-AgentCore-style multi-tenant SaaS spins up one isolated agent runtime per customer org on demand, each pulling its own connection_url from the platform's vault.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an AWS-AgentCore-style multi-tenant SaaS spins up one isolated agent runtime per customer org on demand, each pulling its own connection_url from the platform's vault.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're operating an AWS-AgentCore-style multi-tenant runtime. When a new customer org signs up, mint a fresh agent runtime for them: a row in the tenants Postgres table, a tenant-scoped Redis namespace for ephemeral state, and a connection_url tuple stored in the platform vault. The runtime reads its credentials at boot from the vault row.
```

## Steps to follow

- **Step 1: Provision the tenant control-plane.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .connection_url)
  ```

- **Step 2: Tenant vault schema.**

  ```sql
  CREATE TABLE tenants (
    org_id uuid PRIMARY KEY,
    name text NOT NULL,
    db_url text NOT NULL,
    redis_namespace text NOT NULL,
    created_at timestamptz DEFAULT now(),
    status text DEFAULT 'active'
  );
  ```

- **Step 3: On signup, provision the tenant's own Postgres and persist the URL.**

  ```python
  def onboard(org_id, name):
      r = requests.post("https://api.instanode.dev/db/new",
                        headers={"Authorization": f"Bearer {INSTANT_TOKEN}"}).json()
      pg.execute("""INSERT INTO tenants (org_id, name, db_url, redis_namespace)
                    VALUES (%s,%s,%s,%s)""",
                 (org_id, name, r["connection_url"], f"tenant:{org_id}"))
      return r["connection_url"]
  ```

- **Step 4: Runtime boots and pulls its credentials.**

  ```python
  row = pg.fetchone("SELECT db_url, redis_namespace FROM tenants WHERE org_id=%s", (org_id,))
  tenant_db = psycopg2.connect(row["db_url"])
  tenant_cache_prefix = row["redis_namespace"]
  ```

## Why this works on instanode.dev

Multi-tenant agent runtimes need real per-tenant isolation — shared schemas with row-level security only get you so far when a noisy neighbor blows your connection pool. `POST /db/new` returns a fresh database per call (not a schema, an actual database), so each tenant is bulkheaded. Signup latency is one round-trip; no Terraform reconcile, no AWS quota request.
