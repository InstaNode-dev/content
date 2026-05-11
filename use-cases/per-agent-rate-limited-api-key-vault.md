---
title: Per-agent rate-limited API key vault
category: O. Cross-agent commerce & payments
services: ["redis", "pg"]
scenario: A vault service mints scoped, rate-limited API keys for child agents on demand; Redis tracks per-key usage and the spend rolls up to a Postgres billing event stream.
---
