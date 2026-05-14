---
title: GAIA tournament bracket
category: P. Agent benchmarking & evaluation
services: ["mongo", "minio", "redis"]
scenario: A tournament service runs 16 agent variants head-to-head on GAIA tasks; pairings live in Mongo, intermediate transcripts in S3-compatible storage, win counts in Redis sorted sets for the live leaderboard.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a tournament service runs 16 agent variants head-to-head on GAIA tasks; pairings live in Mongo, intermediate transcripts in S3-compatible storage, win counts in Redis sorted sets for the live leaderboard.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB + S3-compatible storage + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Run a 16-agent GAIA tournament. Claim Mongo + S3-compatible storage + Redis on instanode.dev. Pairings + per-task outcomes in Mongo. Full per-task transcripts in S3-compatible storage. Live leaderboard win counts in Redis sorted set. After every match, update all three.
```

## Steps to follow

- **Step 1: Provision all three stores.** Tournament infra in 3 curls.

  ```bash
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new | jq -r .connection_url)
  S3=$(curl -sX POST https://api.instanode.dev/storage/new)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: Define bracket.** 16 agents → 8 matches round 1.

  ```python
  pairings = [{"round":1,"match":i,"a":agents[2*i],"b":agents[2*i+1]} for i in range(8)]
  mongo.pairings.insert_many(pairings)
  ```

- **Step 3: Run a match.** Both agents answer the same GAIA task; transcripts to S3, score to Mongo + Redis.

  ```python
  for p in pairings:
      a_ans, a_trace = run_agent(p["a"], task)
      b_ans, b_trace = run_agent(p["b"], task)
      winner = judge(task, a_ans, b_ans)
      s3.put_object(Bucket=bucket, Key=f"r{p['round']}/m{p['match']}/{p['a']}.json", Body=json.dumps(a_trace))
      s3.put_object(Bucket=bucket, Key=f"r{p['round']}/m{p['match']}/{p['b']}.json", Body=json.dumps(b_trace))
      mongo.pairings.update_one({"_id":p["_id"]}, {"$set":{"winner":winner}})
      r.zincrby("leaderboard", 1, winner)
  ```

- **Step 4: Live leaderboard.** Sub-millisecond reads.

  ```bash
  redis-cli -u $REDIS_URL ZREVRANGE leaderboard 0 -1 WITHSCORES
  ```

## Why this works on instanode.dev

Each service maps to exactly its strength: Mongo's flexible schema for bracket structure, S3-compatible storage for fat transcripts (avg ~2MB), Redis sorted sets for the live leaderboard. Provisioning all three with the same anonymous token means the tournament runner has no IAM glue to write. If a result is contested, the object in S3-compatible storage is the source of truth — replay it deterministically.

## Related cases

- [SWE-bench parallel rollout harness](/use-cases/swe-bench-parallel-rollout-harness.md) — code-eval cousin that scales to 500 isolated tasks
- [LLM-as-judge consensus pool](/use-cases/llm-as-judge-consensus-pool.md) — judge layer that picks winners between bracket pairs
- [Adversarial red-team runner](/use-cases/adversarial-red-team-runner.md) — parallel-attacker variant of the same eval-fleet pattern
