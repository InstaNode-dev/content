---
title: Live agent status broadcast
category: B. Multi-agent systems
services: ["nats", "redis"]
scenario: Worker agents broadcast heartbeats on a status subject so a dashboard shows which agent is stuck mid-tool-call.
---

## Sample agent prompt

```
Build a heartbeat fabric for our worker swarm. Each agent publishes a status heartbeat to NATS every 2s with current tool call and last token timestamp. A dashboard subscriber updates Redis keys per agent with a 10s TTL so the UI shows red on any agent missing two beats. Provision NATS and Redis and wire both sides.
```

## Steps to follow

- **Step 1: Provision NATS and Redis.**

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"stream":"status"}' -H 'Content-Type: application/json'
  curl -s -X POST https://api.instanode.dev/cache/new
  ```

- **Step 2: Heartbeat publisher inside each agent.**

  ```python
  async def heartbeat(agent_id, current_tool):
      await nc.publish(f"agents.heartbeat.{agent_id}",
          json.dumps({"tool": current_tool, "ts": time.time()}).encode())
  ```

- **Step 3: Subscriber writes to Redis with a 10s expiry.**

  ```python
  async def handler(msg):
      agent = msg.subject.split(".")[-1]
      await redis.set(f"agent:{agent}:status", msg.data, ex=10)
  await nc.subscribe("agents.heartbeat.*", cb=handler)
  ```

- **Step 4: Dashboard query for live status.**

  ```python
  keys = await redis.keys("agent:*:status")
  live = {k.decode(): json.loads(await redis.get(k)) for k in keys}
  ```

- **Step 5: Stale detector loop.**

  ```bash
  redis-cli --scan --pattern 'agent:*:status' | while read k; do
    redis-cli ttl "$k" | awk -v k="$k" '$1 < 5 {print k " stale"}'
  done
  ```

## Why this works on instanode.dev

NATS fanout is sub-millisecond, so the dashboard reflects the swarm in real time even at thousands of agents. Redis TTLs do the "is it alive" check for free, and both resources sit behind one token so the heartbeat fabric is provisioned in two curls.
