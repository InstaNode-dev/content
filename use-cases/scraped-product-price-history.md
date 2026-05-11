---
title: Scraped product-price history
category: E. Browser & automation agents
services: ["pg"]
scenario: A shopping-watcher agent scrapes prices hourly and stores time-series rows for "alert me when it drops 20%".
---

## Sample agent prompt

```
Build a price-watcher. Provision Postgres via instanode.dev, scrape the listed product URLs hourly, and append (url, price, observed_at) rows. On every insert, check if the new price is at least 20% below the trailing-30-day median and emit an alert. Use Playwright for scraping.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Time-series schema.**

  ```sql
  CREATE TABLE prices (
    url TEXT, price_cents INT, observed_at TIMESTAMPTZ DEFAULT now()
  );
  CREATE INDEX ON prices (url, observed_at DESC);
  ```

- **Step 3: Hourly scraper writes one row per product.**

  ```python
  from playwright.sync_api import sync_playwright
  with sync_playwright() as p, psycopg.connect(DATABASE_URL) as conn:
      browser = p.chromium.launch()
      for url in WATCHED:
          page = browser.new_page(); page.goto(url)
          cents = int(float(page.inner_text(".price").strip("$")) * 100)
          conn.execute("INSERT INTO prices VALUES (%s,%s)", (url, cents))
  ```

- **Step 4: Drop-detection query.**

  ```sql
  WITH med AS (
    SELECT url, percentile_cont(0.5) WITHIN GROUP (ORDER BY price_cents) m
    FROM prices WHERE observed_at > now() - interval '30 days' GROUP BY url)
  SELECT p.url, p.price_cents, m.m FROM prices p JOIN med m USING (url)
  WHERE p.observed_at > now() - interval '1 hour' AND p.price_cents <= m.m * 0.8;
  ```

- **Step 5: On a hit, push to the user** via their notification channel.

## Why this works on instanode.dev

Postgres percentile functions handle time-series median cleanly without needing TimescaleDB. The 500MB hobby tier holds years of hourly samples across hundreds of products, and the URL-keyed index keeps drop detection sub-millisecond.
