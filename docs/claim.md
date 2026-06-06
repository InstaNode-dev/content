---
title: Claim flow (anonymous → paid)
order: 6
---

Anonymous resources expire in 24 hours. To keep them, claim them.

```
RESP=$(curl -X POST https://api.instanode.dev/db/new \
  -H "Content-Type: application/json" \
  -d '{"name":"prod-db"}')
JWT=$(echo $RESP | jq -r .upgrade_jwt)

# Optional preview — shows what would attach, no side effects
curl "https://api.instanode.dev/claim/preview?t=$JWT"

# Trigger the claim — sends a magic link to your email
curl -X POST https://api.instanode.dev/claim \
  -d "{\"jwt\":\"$JWT\", \"email\":\"you@example.com\"}"
```

Click the magic link to set a session cookie. Every resource attached to your
fingerprint transfers to your team atomically; the connection URLs don't
change so any already-running code keeps working.

Claimed resources move to your team's tier (hobby by default — $9/mo). There
is no separate trial period on paid tiers — **the 24-hour anonymous slice
is the trial**.
