---
title: x402 micropayment per tool call
category: O. Cross-agent commerce & payments
services: ["redis", "pg", "webhook"]
scenario: An agent calling another agent's premium MCP tool receives HTTP 402, settles a 0.3-cent USDC payment, and retries; the receiver tracks paid calls in Redis with a Postgres ledger.
---
