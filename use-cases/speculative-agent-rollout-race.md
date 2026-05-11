---
title: Speculative agent rollout race
category: M. Parallel tool execution
services: ["nats", "pg"]
scenario: A speculative-decoding-style orchestrator runs the same task at three temperatures in parallel; a verifier agent picks the best output and discards the rest, all coordinated via NATS request/reply.
---
