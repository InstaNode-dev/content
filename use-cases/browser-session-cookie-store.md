---
title: Browser-session cookie store
category: E. Browser & automation agents
services: ["redis"]
scenario: A browser agent persists login cookies and session state per target site so it doesn't re-auth on every run.
---

## Sample agent prompt

```
You're a browser agent that targets 8 different sites. Each site needs its own logged-in session. Persist cookies per site in Redis as a hash (key: `session:{site}`, fields: cookie name to value). Before each run, hydrate the browser context from Redis. After login or refresh, write the new cookie set back.
```

## Steps to follow

- **Step 1: Provision Redis.**

  ```bash
  REDIS_URL=$(curl -sX POST https://api.instanode.dev/cache/new \
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
