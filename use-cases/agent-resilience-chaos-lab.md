---
title: Agent-resilience chaos lab
category: I. Hackathon & education
services: ["pg", "redis", "nats"]
scenario: A research hackathon tests how agents behave when their database, cache, and message bus randomly fail mid-task.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a research hackathon tests how agents behave when their database, cache, and message bus randomly fail mid-task.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis + NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're running a chaos lab for agent resilience research. Provision Postgres, Redis, and NATS from instanode.dev. Run a fleet of task-agents that depend on all three. A chaos controller randomly kills connections, drops messages, or freezes Redis for 30s windows. Measure how each agent design (retry-with-backoff vs. circuit-breaker vs. naive) degrades.
```

## Steps to follow

- **Step 1: Claim three resources.**

  ```bash
  curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"agent-resilience-chaos-lab-db"}'    > .pg.json
  curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"agent-resilience-chaos-lab-cache"}' > .redis.json
  curl -sX POST https://api.instanode.dev/queue/new -H 'Content-Type: application/json' -d '{"name":"agent-resilience-chaos-lab-queue"}' > .nats.json
  ```

- **Step 2: Define the agent's expected workflow.**

  ```python
  async def task_agent(job):
      job_record = pg.insert(jobs, job)            # Postgres
      cache_set(f"job:{job.id}", "running")        # Redis
      await nats.publish("agent.started", job.id)  # NATS
      result = do_work(job)
      pg.update(jobs, job.id, result=result)
      return result
  ```

- **Step 3: Chaos controller — drop one dependency at random.**

  ```python
  import random, asyncio
  async def chaos():
      while True:
          victim = random.choice(["pg","redis","nats"])
          firewall_block(victim, duration_s=30)
          await asyncio.sleep(60)
  ```

- **Step 4: Score agent strategies.**

  ```sql
  SELECT strategy, COUNT(*) FILTER (WHERE status='success') * 1.0 / COUNT(*) AS success_rate
  FROM jobs WHERE started_at > now() - interval '1 hour'
  GROUP BY strategy;
  ```

## Why this works on instanode.dev

Chaos testing needs real dependencies — mocked ones never fail the way prod fails. The anonymous tier gets you three real services in under three seconds; the 24-hour TTL means the lab self-cleans between research sessions. Compare retry strategies against the same backing stack, then throw it all away.

## Related cases

- [24-hour hackathon backend](/use-cases/24-hour-hackathon-backend) — another hackathon-day-shaped stack with a hard expiry
- [Per-agent dead-letter inspection queue](/use-cases/per-agent-dead-letter-inspection-queue) — captures the failures the chaos lab is designed to produce
- [Full dev backend in one curl](/use-cases/full-dev-backend-in-one-curl) — the no-chaos baseline of the same three-service bundle
