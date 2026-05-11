---
title: Maya shipped on Sunday night
date: 2026-05-12
author: instanode.dev
excerpt: A solo founder, six hours of laptop battery, a friend who asked for "the thing" by standup. Three curls and she went to bed. Plus four others who showed up the same week with different problems and the same shape of session.
---

# Maya shipped on Sunday night

It's 11:47 PM on a Sunday. Maya has six hours of laptop battery, a FastAPI app
she finished an hour ago, and a friend who asked to see "the thing" before
standup. Tabs open: Heroku pricing, Fly.io machine docs, a half-read Render
tutorial. None of them are loaded.

She types `curl -X POST https://api.instanode.dev/db/new` and her laptop,
somehow, replies with a working Postgres URL in 945 milliseconds.

This is the third tab Maya didn't have to read. The other four people in this
post had different problems but the same Sunday-night shape: a problem at the
front of their head, low tolerance for ceremony, and the next twenty minutes
to make the problem disappear.

## Maya's full night

Maya is shipping a tool called Bookbase. People upload a CSV of book titles
and get back tagged, summarized entries with embedding vectors attached so
the next "find me books like this one" query is fast. The work was the
modeling, the prompt design, the rate-limiting. The work was not supposed to
be the deploy.

Maya does not run Kubernetes. Maya does not want to.

At 11:50 she has the Postgres URL. She runs `curl -X POST .../cache/new`
for Redis — that's where she'll cache the tag dictionary so the LLM doesn't
have to re-derive labels on every request. Two URLs in her clipboard.

```
# 1. Postgres for books + embeddings, Redis for tag cache
curl -X POST https://api.instanode.dev/db/new
curl -X POST https://api.instanode.dev/cache/new

# 2. Tar the app, ship it
tar -czf app.tar.gz .
curl -X POST https://api.instanode.dev/deploy/new \
  -H "Authorization: Bearer $JWT" \
  -F "tarball=@app.tar.gz" \
  -F 'env_vars={"DATABASE_URL":"...","REDIS_URL":"..."}'
```

`env_vars` is a JSON map; whatever's in there lands in the deployed pod's
environment on the first build. Maya pastes the two connection URLs in. The
multipart upload completes in three seconds.

90 seconds of build later — there's a kaniko Job grinding away in the platform
cluster, but Maya doesn't have to know that — and the response includes:

```
https://bookbase-7a3f.deployment.instanode.dev
```

A working HTTPS URL on the deployment subdomain, with a valid Let's Encrypt
cert that cert-manager handled in the background.

Maya curls her own URL: `{"ok":true,"books_indexed":0}`. She uploads a sample
CSV. She refreshes. Twelve rows of tagged, summarized books come back.

She sends the URL to her friend at 12:14 AM and goes to bed.

## What that 90 seconds usually costs

Every alternative Maya didn't read assumes she's awake. Heroku still wants
a `Procfile`, a `runtime.txt`, and a credit card before her first push —
fifteen minutes of yak-shaving on a laptop at midnight. Fly's
`flyctl launch` is fast, but its first run drops her in an interactive
prompt about regions she didn't know she had to pick. Render and Railway
want her to `git push`, which means a new repo, a new remote, and reading
their auto-detected build settings before anything happens. None of these
are bad products. All of them assume she'll wait until tomorrow.

The instanode endpoints fit in muscle memory: `/db/new`, `/cache/new`,
`/deploy/new`. Maya read zero documentation this session. The platform
shapes itself around what she was going to type anyway.

When she wakes up tomorrow her resources will be marked anonymous and
expire at midnight Monday. If Bookbase has a user by then she'll click the
claim link in the response and pay $9/mo. If it doesn't, the URL goes away
and nothing was wasted.

## Four others, same Sunday-night shape

Maya is not unusual. Four other people showed up this week with different
problems and the same low-ceremony arrival.

### Cleo — a long-running coding agent that needed persistent memory

Cleo runs inside someone's terminal across days, handed tasks like "ship the
auth refactor by Friday." Her old memory was a flat `memory.md` file: broke
on corrupted writes, out of sync across terminal tabs, couldn't be queried.
She made one tool call to `/db/new` and a Postgres URL came back. Cleo's
schema is a `memories` table with a `vector(1536)` embedding column —
pgvector (Postgres's vector-similarity extension) ships pre-installed, so
similarity search is a single `SELECT ... ORDER BY embedding <-> $query`.
When the user asked about a decision two days later, Cleo answered correctly
and cited the original turn.

### Anders — RAG over a stack of legal PDFs

Anders is building Lawclerk, a tiny SaaS that answers questions from a
corpus of legal PDFs (RAG = retrieval-augmented generation: feed the LLM
relevant snippets from your own corpus instead of relying on its training
data). Pinecone rate-limited him at 11 PM, Weaviate was awkward to deploy,
Qdrant's auth setup beat him. Same `/db/new`, same default pgvector. He
created an HNSW index (the standard graph-based nearest-neighbor index;
`CREATE INDEX ON docs USING hnsw`), fed in 47,000 chunks, and watched the
99th-percentile latency stay under 80 ms. The whole vector layer was
Postgres, and he already knew Postgres.

### Priya — debugging a Stripe webhook

Priya at an established company. Their Stripe handler drops events
intermittently. `stripe trigger` won't fire the specific event she needs;
ngrok wants a paid plan for her account size. Two minutes of searching, then
`curl -X POST .../webhook/new` got her a public `receive_url`. She pasted
it into Stripe's test endpoint config. The next 14 payloads landed in the
platform's request log; she found the malformed field in the second one. The
fix shipped before standup.

### Reza and Tamika — hackathon, 24 hours, one demo

They met three hours into a hackathon and decided to build "Daily Standup
Bot." They had never collaborated on a deploy. Three curls in their group
chat at 3 AM — `/db/new`, `/cache/new`, `/webhook/new` — and within 10
minutes both had identical working backing services. They wrote the bot in
Python, shipped it with `/deploy/new`, demoed at 9 AM, won second place.
They didn't claim; the resources expired at noon the next day. They wrote
the names down to come back to it.

## What ties them together

These five people don't have much in common. A 24/7 coding agent and a
sleep-deprived hackathon team are not the same customer.

What they share is the moment they show up: **eyes glued to the problem
they want to disappear**. Anything between them and "the thing is alive
on the internet" is friction. Anything that survives that gap is a story
they tell their friends.

Five different starting points, five different problems, one shape of
solution: curl, build, ship, optionally claim.

If you're somewhere in this list — or in a sixth shape we haven't
documented yet — the curl works right now. No signup.
