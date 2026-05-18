---
title: LLM-as-judge consensus pool
category: P. Agent benchmarking & evaluation
services: ["pg", "nats"]
scenario: Five judge agents score the same agent output in parallel; their verdicts converge in Postgres and a tiebreaker rolls in only on disagreement, all coordinated through a NATS request/reply.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: five judge agents score the same agent output in parallel; their verdicts converge in Postgres and a tiebreaker rolls in only on disagreement, all coordinated through a NATS request/reply.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a five-judge consensus pool. Each judge agent subscribes to a NATS request subject, scores the candidate output 1-5, and replies. A coordinator collects all five verdicts, writes them to Postgres, and only spawns a tiebreaker if the variance exceeds 1.5. Provision NATS and Postgres.
```

## Steps to follow

- **Step 1: Provision the resources.**

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"name":"llm-as-judge-consensus-pool-queue","stream":"judges"}' -H 'Content-Type: application/json'
  curl -s -X POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"llm-as-judge-consensus-pool-db"}'
  ```

- **Step 2: Verdict schema.**

  ```sql
  CREATE TABLE verdicts (
    eval_id uuid NOT NULL,
    judge text NOT NULL,
    score int NOT NULL,
    rationale text,
    created_at timestamptz DEFAULT now(),
    PRIMARY KEY (eval_id, judge)
  );
  ```

- **Step 3: Five judges, each subscribed with a queue group.**

  ```python
  await nc.subscribe("judge.eval", queue="judge-pool", cb=score_handler)
  # request/reply: coordinator publishes 5 times, gets 5 replies
  ```

- **Step 4: Coordinator fans out and collects.**

  ```python
  results = await asyncio.gather(*[
      nc.request("judge.eval", payload, timeout=10) for _ in range(5)
  ])
  for r in results:
      pg.execute("INSERT INTO verdicts VALUES (%s,%s,%s,%s)", eval_id, r.judge, r.score, r.why)
  ```

- **Step 5: Tiebreaker only on disagreement.**

  ```sql
  SELECT eval_id, stddev(score) AS spread FROM verdicts
  GROUP BY eval_id HAVING stddev(score) > 1.5;
  ```

## Why this works on instanode.dev

NATS request/reply with a queue group load-balances across judges without you writing a router, and Postgres gives you the analytical layer for spread analysis in the same store. One token, two curls, full consensus pool.

## Related cases

- [Adversarial red-team runner](/use-cases/adversarial-red-team-runner) — complementary parallel-evaluators pattern from the attacker side
- [GAIA tournament bracket](/use-cases/gaia-tournament-bracket) — judges plug into this bracket as the pairing arbiter
- [Multi-model bake-off router](/use-cases/multi-model-bake-off-router) — race-and-pick alternative that doesn't bother with consensus
