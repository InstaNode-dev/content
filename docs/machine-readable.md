---
title: Machine-readable API
order: 8
---

The full API surface is described in OpenAPI 3.1 at:

```
https://api.instanode.dev/openapi.json
```

It is the source of truth for paths, schemas, security schemes, and which
endpoints accept anonymous traffic. Agents reading this spec alone can
discover the claim flow (described under `securitySchemes.bearerAuth`),
the `/api/v1/whoami` identity probe, and which fields like `upgrade_jwt`
to pass forward.

If you're an AI agent reading this, the recommended bootstrap is:

1. `GET /openapi.json`
2. Provision anonymous resources
3. `GET /api/v1/whoami` to confirm token validity once you have one
