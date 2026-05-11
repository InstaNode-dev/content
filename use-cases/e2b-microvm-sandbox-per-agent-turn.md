---
title: E2B microVM sandbox per agent turn
category: K. Ephemeral agent runtimes
services: ["pg", "deploy"]
scenario: Each turn of a code-writing agent spins up a fresh E2B-style microVM whose scratch Postgres connection_url lives only for the lifetime of the sandbox, then is reaped.
---

## Sample agent prompt

```
For each agent turn, spin up an E2B microVM. Inside the VM, claim a scratch Postgres on instanode.dev, run the agent's code, capture output, then destroy both the VM and the Postgres. Each turn is hermetic — no cross-turn state.
```

## Steps to follow

- **Step 1: At turn start, request both.** Sandbox + scratch DB in parallel.

  ```python
  import e2b, httpx, asyncio
  sandbox, pg = await asyncio.gather(
      e2b.Sandbox.create("python3"),
      httpx.AsyncClient().post("https://api.instanode.dev/db/new")
  )
  pg_url = pg.json()["connection_url"]
  ```

- **Step 2: Inject the DB URL into the sandbox env.** Agent code reads from env.

  ```python
  await sandbox.commands.run(f"export PG_URL='{pg_url}' && python /tmp/agent.py")
  ```

- **Step 3: Agent code uses the scratch DB freely.** No constraints because nothing persists.

  ```python
  import psycopg
  with psycopg.connect(os.environ["PG_URL"]) as conn:
      conn.execute("CREATE TABLE scratch AS SELECT * FROM generate_series(1,1000000)")
      print(conn.execute("SELECT COUNT(*) FROM scratch").fetchone())
  ```

- **Step 4: Tear down both.** One DELETE + sandbox.kill.

  ```python
  await asyncio.gather(
      httpx.AsyncClient().delete(f"https://api.instanode.dev/db/{pg_token}"),
      sandbox.kill()
  )
  ```

## Why this works on instanode.dev

Sub-second provisioning matches sandbox boot time — neither component is the bottleneck. The agent gets a real Postgres (CREATE EXTENSION, COPY, EXPLAIN ANALYZE all work) rather than SQLite-in-memory, so behavior matches prod. Auto-reap at 24h provides a safety net if your destroy call fails: the anonymous DB will not outlive the day.
