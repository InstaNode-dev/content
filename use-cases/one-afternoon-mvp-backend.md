---
title: One-afternoon MVP backend
category: H. Indie & SaaS founders
services: ["pg", "redis", "deploy"]
scenario: A solo founder spins up Postgres + Redis from a curl call in their Claude Code session and ships a paid product by evening.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a solo founder spins up Postgres + Redis from a curl call in their Claude Code session and ships a paid product by evening.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
I want to ship a paid SaaS by tonight. From inside Claude Code, provision Postgres, Redis, and a container deploy on instanode. Generate the Fastify+Drizzle scaffold, wire the Stripe key into env, run migrations, deploy, and print the live URL. I'll handle the marketing site separately.
```

## Steps to follow

- **Step 1: Provision the trio.**

  ```bash
  PG=$(curl -s -X POST https://api.instanode.dev/db/new | jq -r .connection_url)
  RD=$(curl -s -X POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  printf "DATABASE_URL=%s\nREDIS_URL=%s\n" "$PG" "$RD" > .env
  ```

- **Step 2: Scaffold and migrate.**

  ```bash
  pnpm create fastify-app server && cd server
  pnpm add drizzle-orm postgres ioredis stripe
  pnpm drizzle-kit push
  ```

- **Step 3: Minimal paid endpoint.**

  ```javascript
  app.post('/checkout', async (req, reply) => {
    const session = await stripe.checkout.sessions.create({
      mode: 'subscription',
      line_items: [{ price: 'price_xxx', quantity: 1 }],
      success_url: 'https://app.example.com/ok'
    })
    await redis.setex(`pending:${session.id}`, 3600, req.body.email)
    return { url: session.url }
  })
  ```

- **Step 4: Deploy from the current directory.**

  ```bash
  curl -s -X POST https://api.instanode.dev/deploy/new \
    -H 'Content-Type: application/json' \
    -d "$(jq -n --arg env "$(cat .env)" '{dockerfile:"Dockerfile", env:$env}')"
  ```

- **Step 5: Hit the live URL.**

  ```bash
  curl https://your-app.instanode.dev/healthz
  ```

## Why this works on instanode.dev

Three curls replace three cloud signups, three IAM dashboards, and the entire Docker-image-push dance. By dinnertime you have a real domain, a working DB, a cache, and a paid checkout flow — and the same claimed token bills as one $9 line item.

## Related cases

- [Full dev backend in one curl](/use-cases/full-dev-backend-in-one-curl.md) — the no-deploy precursor to this paid-by-evening ship flow
- [Side-project container deploy](/use-cases/side-project-container-deploy.md) — bare deploy primitive this MVP flow calls into
- [AI cofounder spawns a startup](/use-cases/ai-cofounder-spawns-a-startup.md) — agent-driven variant where the founder is an AI
