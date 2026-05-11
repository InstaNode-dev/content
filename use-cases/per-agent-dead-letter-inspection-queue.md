---
title: Per-agent dead-letter inspection queue
category: N. Multi-agent observability
services: ["nats", "mongo"]
scenario: Every failed tool call from a swarm is republished to a NATS DLQ; an investigator agent pulls them in batches, classifies the failure mode, and persists clusters in Mongo for the operator UI.
---
