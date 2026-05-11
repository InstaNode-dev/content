---
title: Pre-launch waitlist store
category: H. Indie & SaaS founders
services: ["webhook", "pg"]
scenario: A landing page captures emails via a webhook receiver and stores them with UTM context for nurture campaigns.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a landing page captures emails via a webhook receiver and stores them with UTM context for nurture campaigns.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
For our pre-launch landing page, set up a waitlist capture. Provision a webhook the landing form POSTs to, and a Postgres table that the webhook drainer fills with email + UTM source/medium/campaign + timestamp. Output the receiver URL so I can wire the form.
```

## Steps to follow

- **Step 1: Provision the receiver and DB.**

  ```bash
  WH=$(curl -s -X POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  curl -s -X POST https://api.instanode.dev/db/new
  echo "Form action: $WH"
  ```

- **Step 2: Waitlist schema.**

  ```sql
  CREATE EXTENSION IF NOT EXISTS citext;
  CREATE TABLE waitlist (
    id bigserial PRIMARY KEY,
    email citext UNIQUE NOT NULL,
    utm_source text, utm_medium text, utm_campaign text,
    referrer text, ip inet,
    created_at timestamptz DEFAULT now()
  );
  ```

- **Step 3: Landing page form.**

  ```html
  <form method="POST" action="https://hooks.instanode.dev/wh_xxx">
    <input name="email" type="email" required>
    <input type="hidden" name="utm_source" value="">
    <input type="hidden" name="utm_campaign" value="">
    <button>Join waitlist</button>
  </form>
  <script>
    const p = new URLSearchParams(location.search)
    document.querySelector('[name=utm_source]').value = p.get('utm_source') || ''
    document.querySelector('[name=utm_campaign]').value = p.get('utm_campaign') || ''
  </script>
  ```

- **Step 4: Drainer inserts into Postgres.**

  ```python
  for r in fetch_webhook_batch(WH):
      pg.execute("""INSERT INTO waitlist(email, utm_source, utm_medium, utm_campaign, referrer, ip)
                    VALUES (%s,%s,%s,%s,%s,%s) ON CONFLICT (email) DO NOTHING""",
                 r["email"], r.get("utm_source"), r.get("utm_medium"),
                 r.get("utm_campaign"), r.get("referrer"), r["_ip"])
  ```

- **Step 5: Top channels report.**

  ```sql
  SELECT utm_source, count(*) FROM waitlist GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
  ```

## Why this works on instanode.dev

Two curls replace a Mailchimp signup + a Zap + a Google Sheet, and your data lives in real Postgres so the nurture campaign queries are SQL instead of CSV exports. The webhook tolerates traffic spikes from HN frontpage without a single 503.

## Related cases

- [Stripe-event entitlements](/use-cases/stripe-event-entitlements.md) — the post-launch payments side of the same indie-founder funnel
- [AI cofounder spawns a startup](/use-cases/ai-cofounder-spawns-a-startup.md) — agent-driven generator of the landing page this waitlist sits on
- [Solo-founder analytics warehouse](/use-cases/solo-founder-analytics-warehouse.md) — downstream Postgres where waitlist UTMs end up for analysis
