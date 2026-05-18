---
title: Daytona warm-pool data workspace
category: K. Ephemeral agent runtimes
services: ["redis", "deploy"]
scenario: A Daytona warm pool of 50 Python sandboxes each pre-attaches to a per-sandbox Redis namespace for caching pip-resolver results, so cold start to first agent action is under 100ms.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Daytona warm pool of 50 Python sandboxes each pre-attaches to a per-sandbox Redis namespace for caching pip-resolver results, so cold start to first agent action is under 100ms.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Set up a 50-sandbox Daytona warm pool. For each sandbox, pre-claim a Redis namespace on instanode.dev and an instanode deploy slot. Cache pip-resolver output in Redis so the first `pip install` in a fresh sandbox hits warm cache instead of cold PyPI. Target: <100ms from sandbox up to first agent action.
```

## Steps to follow

- **Step 1: Provision a warm pool of Redis caches.** Loop the API at pool init.

  ```bash
  for i in $(seq 1 50); do
    curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"daytona-warm-pool-data-workspace-cache"}' | jq -r .connection_url > caches/$i.url
  done
  ```

- **Step 2: Pre-pull a base image with the resolver wrapper baked in.** Each Daytona sandbox boots from this.

  ```bash
  curl -X POST https://api.instanode.dev/deploy/new \
    -H "Authorization: Bearer $INSTANODE_TOKEN" \
    -F "name=daytona-agent-base" \
    -F "image=ghcr.io/me/daytona-agent-base:latest" \
    -F "cpu=1000" \
    -F "mem_mb=2048"
  ```

- **Step 3: Resolver hits Redis before PyPI.** Cache key = (python_version, requirements_hash).

  ```python
  key = f"resolve:{py}:{hashlib.sha256(reqs.encode()).hexdigest()}"
  cached = r.get(key)
  if cached:
      open("requirements.lock","w").write(cached)
  else:
      lock = subprocess.check_output(["pip-compile","-"], input=reqs)
      r.setex(key, 86400, lock)
  ```

- **Step 4: Reap on sandbox destroy.** Anonymous caches auto-expire at the 24h TTL, so a destroyed sandbox's Redis namespace reaps itself — no teardown call needed. If you claimed a cache (holding a session token) and want it gone sooner, delete it by resource id:

  ```bash
  # claimed-flow only — anonymous caches just expire at 24h.
  curl -X DELETE "https://api.instanode.dev/api/v1/resources/$RESOURCE_ID" \
    -H "Authorization: Bearer $INSTANODE_TOKEN"
  ```

## Why this works on instanode.dev

Per-sandbox Redis namespaces mean noisy neighbors can't poison each other's resolver cache. Provisioning is fast enough (~600ms) that pool refill at scale stays under 30s for 50 sandboxes. The deploy slot for the base image lives at a `*.instanode.dev` URL, so Daytona pulls it without authenticating to a private registry. No infrastructure to maintain beyond the agent code itself.

## Related cases

- [E2B microVM sandbox per agent turn](/use-cases/e2b-microvm-sandbox-per-agent-turn.md) — per-turn cold-start alternative to a kept-warm pool
- [Sandboxed test runner per task](/use-cases/sandboxed-test-runner-per-task.md) — the test-runner equivalent of the same pool idea
- [Pyodide notebook cell agent farm](/use-cases/pyodide-notebook-cell-agent-farm.md) — another pool of ready-to-run Python workers
