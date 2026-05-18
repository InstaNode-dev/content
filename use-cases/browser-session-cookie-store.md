---
title: Browser-session cookie store
category: E. Browser & automation agents
services: ["redis"]
scenario: A browser agent persists login cookies and session state per target site so it doesn't re-auth on every run.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a browser agent persists login cookies and session state per target site so it doesn't re-auth on every run.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're a browser agent that targets 8 different sites. Each site needs its own logged-in session. Persist cookies per site in Redis as a hash (key: `session:{site}`, fields: cookie name to value). Before each run, hydrate the browser context from Redis. After login or refresh, write the new cookie set back.
```

## Steps to follow

- **Step 1: Provision Redis.**

  ```bash
  REDIS_URL=$(curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"browser-session-cookie-store-cache"}' \
    -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Hydrate the Playwright context from Redis.**

  ```python
  import redis, json
  from playwright.sync_api import sync_playwright

  r = redis.from_url(REDIS_URL)

  def load_cookies(site):
      raw = r.get(f"session:{site}")
      return json.loads(raw) if raw else []

  ctx = browser.new_context()
  ctx.add_cookies(load_cookies("linkedin"))
  ```

- **Step 3: Persist after every successful action.**

  ```python
  def save_cookies(site, ctx):
      cookies = ctx.cookies()
      r.set(f"session:{site}", json.dumps(cookies), ex=86400*7)
  ```

- **Step 4: Refresh-on-expiry workflow.**

  ```python
  page = ctx.new_page()
  page.goto("https://linkedin.com/feed")
  if "login" in page.url:
      run_login_flow(page)
      save_cookies("linkedin", ctx)
  ```

## Why this works on instanode.dev

Cookie state is small (~10KB per site), hot (read at every run start), and ephemeral (7-day TTL is a feature, not a bug). Redis is the right shape, but spinning up ElastiCache for this is overkill. One curl, real Redis, EXPIRE built-in. Across 8 sites you avoid 8 re-auth flows per agent run — that's minutes saved per session.

## Related cases

- [Accessibility-tree selector cache](/use-cases/accessibility-tree-selector-cache.md) — complementary Redis cache that speeds up the same browser agent
- [Form-fill state machine](/use-cases/form-fill-state-machine.md) — auth state these cookies provide is what lets form-fill resume
- [Per-agent rate-limited API key vault](/use-cases/per-agent-rate-limited-api-key-vault.md) — another secrets-in-Redis pattern with scoped access
