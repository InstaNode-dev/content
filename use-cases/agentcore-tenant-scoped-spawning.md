---
title: AgentCore tenant-scoped spawning
category: L. Agent-factory / spawning patterns
services: ["pg", "redis"]
scenario: An AWS-AgentCore-style multi-tenant SaaS spins up one isolated agent runtime per customer org on demand, each pulling its own connection_url from the platform's vault.
---
