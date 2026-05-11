---
title: LLM-as-judge consensus pool
category: P. Agent benchmarking & evaluation
services: ["pg", "nats"]
scenario: Five judge agents score the same agent output in parallel; their verdicts converge in Postgres and a tiebreaker rolls in only on disagreement, all coordinated through a NATS request/reply.
---
