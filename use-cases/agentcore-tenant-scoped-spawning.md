---
title: AgentCore tenant-scoped spawning
category: L. Agent-factory / spawning patterns
services: ["pg", "redis"]
scenario: An AWS-AgentCore-style multi-tenant SaaS spins up one isolated agent runtime per customer org on demand, each pulling its own connection_url from the platform's vault.
---

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
