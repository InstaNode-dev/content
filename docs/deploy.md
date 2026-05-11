---
title: Deploying an app
order: 3
---

`POST /deploy/new` takes a multipart form with a gzipped tar archive
containing your Dockerfile + source.

```
curl -X POST https://api.instanode.dev/deploy/new \
  -H "Authorization: Bearer <JWT>" \
  -F "tarball=@app.tar.gz" \
  -F "name=my-app" \
  -F "port=8080" \
  -F 'env_vars={"DATABASE_URL":"postgres://..."}'
```

The build runs in-cluster on kaniko (~30–90s for typical Node/Python apps)
and the app rolls out behind a public HTTPS URL on
`*.deployment.instanode.dev` with a valid Let's Encrypt cert.

`env_vars` is optional — pass a JSON object and every key/value lands in
the app's environment on the first build. Saves you a follow-up PATCH+redeploy.

For multi-service apps see **Stacks** below.
