---
title: Agent-resilience chaos lab
category: I. Hackathon & education
services: ["pg", "redis", "nats"]
scenario: A research hackathon tests how agents behave when their database, cache, and message bus randomly fail mid-task.
---

## Sample agent prompt

```
You're running a chaos lab for agent resilience research. Provision Postgres, Redis, and NATS from instanode.dev. Run a fleet of task-agents that depend on all three. A chaos controller randomly kills connections, drops messages, or freezes Redis for 30s windows. Measure how each agent design (retry-with-backoff vs. circuit-breaker vs. naive) degrades.
```

## Steps to follow

- **Step 1: Claim three resources.**

  ```bash
  curl -sX POST https://api.instanode.dev/db/new    > .pg.json
  curl -sX POST https://api.instanode.dev/cache/new > .redis.json
  curl -sX POST https://api.instanode.dev/queue/new > .nats.json
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
