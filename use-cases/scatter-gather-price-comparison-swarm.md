---
title: Scatter-gather price comparison swarm
category: J. Agent swarms & fan-out
services: ["nats", "redis"]
scenario: A consumer agent broadcasts a product query to 40 retailer-specific shopper agents over a NATS subject; the first 5 cheapest replies win and the rest are cancelled mid-flight.
---

## Sample agent prompt

```
Provision a NATS JetStream and Redis via instanode.dev. Publish "shop.query" with a product description; 40 retailer-shopper agents respond on "shop.reply.<id>" with their best price. Accept the first 5 cheapest within 3 seconds, cache the winning prices in Redis under the product hash, and send "shop.cancel" to abort the slow ones.
```

## Steps to follow

- **Step 1: Provision both services.**

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: Broadcast the query.**

  ```python
  import nats, asyncio
  nc = await nats.connect(NATS)
  msg_id = "q-001"
  await nc.publish("shop.query", json.dumps({"id": msg_id, "q": "Bose QC45"}).encode())
  ```

- **Step 3: Race-collect 5 replies, then cancel.**

  ```python
  sub = await nc.subscribe(f"shop.reply.{msg_id}")
  winners = []
  async with asyncio.timeout(3):
      async for m in sub.messages:
          winners.append(json.loads(m.data))
          if len(winners) >= 5: break
  await nc.publish(f"shop.cancel.{msg_id}", b"")
  ```

- **Step 4: Cache the winners.**

  ```python
  r = redis.from_url(REDIS)
  r.setex(f"shop:{sha(q)}", 900, json.dumps(sorted(winners, key=lambda x: x["price"])))
  ```

- **Step 5: Return the top result** to the user agent.

## Why this works on instanode.dev

JetStream gives you wildcard subscribe (`shop.reply.*`) and at-least-once delivery so a slow shopper agent doesn't lose its message just because the consumer was already done. Redis handles the 15-minute price cache so repeated queries don't re-fan out across 40 agents.
