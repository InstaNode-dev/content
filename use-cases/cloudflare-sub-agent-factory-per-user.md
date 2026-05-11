---
title: Cloudflare sub-agent factory per user
category: L. Agent-factory / spawning patterns
services: ["pg", "deploy"]
scenario: A Durable-Object-style root agent mints a fresh sub-agent for each end-user, each with its own SQL-backed memory and a deploy slot for that user's tools.
---

## Sample agent prompt

```
You operate a Cloudflare-Durable-Object-style sub-agent factory. When an end-user first interacts, mint them a fresh sub-agent: a row in agents Postgres, an isolated database for that agent's memory, and a deploy slot hosting that user's tool-runner. Resolve subsequent requests by user_id to the same sub-agent.
```

## Steps to follow

- **Step 1: Provision the control-plane Postgres.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "Authorization: Bearer $T" | jq -r .connection_url)
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
