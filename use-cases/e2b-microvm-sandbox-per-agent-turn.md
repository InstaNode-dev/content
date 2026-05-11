---
title: E2B microVM sandbox per agent turn
category: K. Ephemeral agent runtimes
services: ["pg", "deploy"]
scenario: Each turn of a code-writing agent spins up a fresh E2B-style microVM whose scratch Postgres connection_url lives only for the lifetime of the sandbox, then is reaped.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: each turn of a code-writing agent spins up a fresh E2B-style microVM whose scratch Postgres connection_url lives only for the lifetime of the sandbox, then is reaped.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

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

## Related cases

- [Daytona warm-pool data workspace](/use-cases/daytona-warm-pool-data-workspace.md) — kept-warm alternative to spinning microVMs from cold
- [Sandboxed test runner per task](/use-cases/sandboxed-test-runner-per-task.md) — the test-runner-specific variant of per-turn sandboxing
- [Replit-Agent preview backend](/use-cases/replit-agent-preview-backend.md) — preview-deployment sibling with a 24h reaper
