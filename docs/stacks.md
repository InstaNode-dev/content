---
title: Stacks (multi-service deploy)
order: 4
---

`POST /stacks/new` takes an `instant.yaml` manifest plus one tarball per
service. Services can reference each other with `service://<name>` env
values — those resolve to cluster-internal `http://<name>:<port>` URLs at
deploy time.

```
services:
  api:
    build: ./api
    port: 3000
  web:
    build: ./web
    port: 8080
    expose: true
    env:
      API_URL: service://api
```

Only services with `expose: true` get a public URL — the rest are
in-cluster only. The whole stack rolls out together; partial failure is
reported per-service in `GET /stacks/{slug}`.
