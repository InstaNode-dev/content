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
It's Sunday 2pm. I want to ship a paid waitlist for an indie SaaS called Inboxbird (AI triage for shared inboxes) before bed tonight. From inside Claude Code: provision Postgres + Redis + a container deploy on instanode.dev, scaffold a Fastify + Drizzle backend with a /signup endpoint that takes an email and creates a Stripe checkout session for the $29/mo plan, store the pending session_id in Redis with a 1-hour TTL, write the email + Stripe customer ID to Postgres on webhook confirmation, deploy the container, and print the live URL so I can paste it on Twitter. Stripe keys are in ~/.stripe-keys; pull them through.
```

## Steps to follow

We'll thread one app — **Inboxbird**, a waitlist that takes $29/mo via Stripe — through every step. By the last command you'll have a live URL.

- **Step 1: Provision the trio.** Three curls, three connection strings, one `.env` file. No console, no IAM, no project picker.

  ```bash
  mkdir inboxbird && cd inboxbird
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"one-afternoon-mvp-backend-db"}'      | jq -r .connection_url)
  RD=$(curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"one-afternoon-mvp-backend-cache"}'   | jq -r .connection_url)
  cat > .env <<EOF
  DATABASE_URL=$PG
  REDIS_URL=$RD
  STRIPE_SECRET=$(cat ~/.stripe-keys/secret)
  STRIPE_PRICE_ID=price_1Pinboxbird29
  EOF
  echo "provisioned in $SECONDS seconds"
  ```

- **Step 2: Scaffold + schema.** Fastify + Drizzle is a 60-second setup; the schema is two tables.

  ```bash
  pnpm create fastify-app@latest . --import-map
  pnpm add drizzle-orm postgres ioredis stripe zod
  pnpm add -D drizzle-kit tsx
  ```

  ```typescript
  // db/schema.ts
  import { pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
  export const signups = pgTable('signups', {
    id:           uuid('id').defaultRandom().primaryKey(),
    email:        text('email').notNull().unique(),
    stripe_cust:  text('stripe_cust'),
    plan:         text('plan').notNull().default('pending'),
    created_at:   timestamp('created_at').defaultNow(),
  })
  ```

  ```bash
  pnpm drizzle-kit push
  # ✓ Changes applied: CREATE TABLE "signups" ...
  ```

- **Step 3: The paid endpoint.** Email in, Stripe checkout out, pending session held in Redis for 1h.

  ```typescript
  // src/routes/signup.ts
  import Stripe from 'stripe'
  const stripe = new Stripe(process.env.STRIPE_SECRET!)

  app.post('/signup', async (req, reply) => {
    const { email } = z.object({ email: z.string().email() }).parse(req.body)
    const session = await stripe.checkout.sessions.create({
      mode: 'subscription',
      customer_email: email,
      line_items: [{ price: process.env.STRIPE_PRICE_ID!, quantity: 1 }],
      success_url: 'https://inboxbird.instanode.app/welcome?s={CHECKOUT_SESSION_ID}',
      cancel_url:  'https://inboxbird.instanode.app/',
    })
    await redis.setex(`pending:${session.id}`, 3600, email)
    return { url: session.url }
  })
  ```

- **Step 4: Stripe webhook confirms the row.** When Stripe POSTs the `checkout.session.completed` event, we hydrate the signup from Redis and write it to Postgres.

  ```typescript
  app.post('/stripe/webhook', async (req, reply) => {
    const evt = stripe.webhooks.constructEvent(req.rawBody, req.headers['stripe-signature']!, process.env.STRIPE_WEBHOOK_SECRET!)
    if (evt.type === 'checkout.session.completed') {
      const s = evt.data.object as Stripe.Checkout.Session
      const email = await redis.get(`pending:${s.id}`)
      if (email) {
        await db.insert(signups).values({ email, stripe_cust: s.customer as string, plan: 'paid' }).onConflictDoUpdate({ target: signups.email, set: { plan: 'paid', stripe_cust: s.customer as string } })
        await redis.del(`pending:${s.id}`)
      }
    }
    return { received: true }
  })
  ```

- **Step 5: Ship it.** Dockerfile is the standard Node 20 image; `/deploy/new` does the rest and returns a real DNS URL.

  ```bash
  curl -sX POST https://api.instanode.dev/deploy/new \
    -H "Authorization: Bearer $INSTANODE_TOKEN" \
    -F "name=inboxbird" \
    -F "dockerfile=Dockerfile" \
    -F "subdomain=inboxbird" \
    -F "env_vars=$(jq -Rs 'split("\n") | map(select(length>0) | split("=")) | map({(.[0]): (.[1:] | join("="))}) | add' < .env)" \
    | jq -r .url
  # https://inboxbird.instanode.app
  ```

- **Step 6: Verify it's alive.** Hit the health check, then the signup path end-to-end.

  ```bash
  curl -s https://inboxbird.instanode.app/healthz
  # {"ok":true,"db":"up","redis":"up"}

  curl -s -X POST https://inboxbird.instanode.app/signup \
    -H 'Content-Type: application/json' \
    -d '{"email":"first-customer@example.com"}'
  # {"url":"https://checkout.stripe.com/c/pay/cs_live_..."}
  ```

## Why this works on instanode.dev

The "Sunday-night ship" used to mean four signups — **Vercel** for the front, **Supabase** for the DB, **Upstash** for Redis, **Fly.io** or **Render** for the container — four dashboards, four billing relationships, four IAM models the founder has to reason about at midnight. Each one is good in isolation, but the seams between them are where the afternoon goes: Supabase's connection string format vs. Drizzle's expectations, Upstash's TLS flag, Fly's secret CLI, Vercel's build cache. instanode collapses the four into one claimed token: the same JWT that owns the DB owns the cache and the deploy URL, and `instanode.app` is real DNS — not a `*.fly.dev` you have to CNAME later. By the time Stripe sends its first webhook, the founder has spent zero minutes in a cloud console and the whole month's infra bills as one $9 hobby-tier line item until they outgrow it.

## Related cases

- [Full dev backend in one curl](/use-cases/full-dev-backend-in-one-curl) — the no-deploy precursor to this paid-by-evening ship flow
- [Side-project container deploy](/use-cases/side-project-container-deploy) — bare deploy primitive this MVP flow calls into
- [AI cofounder spawns a startup](/use-cases/ai-cofounder-spawns-a-startup) — agent-driven variant where the founder is an AI
