---
title: Accessibility-tree selector cache
category: E. Browser & automation agents
services: ["redis"]
scenario: A scraping agent caches a11y-tree snapshots in Redis so repeat selectors resolve in single-digit milliseconds.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a scraping agent caches a11y-tree snapshots in Redis so repeat selectors resolve in single-digit milliseconds.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You are a scraping agent that walks a11y trees. Each page costs ~400ms to re-snapshot, but selectors repeat across runs. Provision a Redis from instanode.dev, key snapshots by sha256(url + viewport), and short-circuit to the cached selector when the hash matches. Target: single-digit ms cache hits.
```

## Steps to follow

- **Step 1: Claim a Redis.** Anonymous works for prototyping; claim a token for persistence.

  ```bash
  REDIS_URL=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: Compute the cache key from URL + viewport.**

  ```python
  import hashlib, redis
  r = redis.from_url(os.environ["REDIS_URL"])

  def key(url: str, viewport: tuple[int,int]) -> str:
      h = hashlib.sha256(f"{url}|{viewport[0]}x{viewport[1]}".encode()).hexdigest()
      return f"a11y:{h}"
  ```

- **Step 3: Try the cache before re-snapshotting.**

  ```python
  def get_selector(url, viewport, target_role):
      cached = r.hget(key(url, viewport), target_role)
      if cached:
          return cached.decode()  # ~2ms round trip
      tree = page.accessibility.snapshot()  # ~400ms
      sel = resolve(tree, target_role)
      r.hset(key(url, viewport), target_role, sel)
      r.expire(key(url, viewport), 3600)
      return sel
  ```

- **Step 4: Warm cache for known flows.** Pre-populate before a parallel scrape.

  ```python
  for url in seed_urls:
      get_selector(url, (1280, 800), "button[name=Submit]")
  ```

## Why this works on instanode.dev

You don't want to spin up ElastiCache for a 25MB working set. The free anonymous Redis gives you a real instance — pipelining, EXPIRE, HSET, all of it — over the public internet with p99 well under 10ms from most regions. Snapshot cache hits go from 400ms (re-parse the DOM) to ~2ms (one HGET), which compounds fast across a 1000-URL crawl.
