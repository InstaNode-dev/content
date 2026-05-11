---
title: Maya shipped on Sunday night
date: 2026-05-12
author: instanode.dev
excerpt: A solo founder, six hours of laptop battery, a friend who asked for "the thing" by standup. Three curls and she went to bed. Plus two others who showed up the same week with different problems and the same shape of session.
---

# Maya shipped on Sunday night

It's 11:47 PM on a Sunday. Maya has six hours of laptop battery, a FastAPI app
she finished an hour ago, and a friend who asked to see "the thing" before
standup. Tabs open: Heroku pricing, Fly.io machine docs, a half-read Render
tutorial. None of them are loaded.

She types `curl -X POST https://api.instanode.dev/db/new` and her laptop,
somehow, replies with a working Postgres URL in 945 milliseconds.

She read none of those tabs. The other people in this post had different
problems but the same Sunday-night shape: a problem at the front of their
head, low tolerance for ceremony, and the next twenty minutes to make the
problem disappear.

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

## Two others, same week

Maya is not unusual. The shape repeats. Here are two more from this week
whose problems looked nothing like Maya's and whose sessions looked exactly
like hers.

### Cleo — a coding agent that finally remembered things

Cleo is not a person. Cleo is a long-running coding agent that lives inside
someone's terminal for days at a time, handed tasks like "ship the auth
refactor by Friday." Until last Tuesday her memory was a flat `memory.md`
file: it broke on corrupted writes, fell out of sync across terminal tabs,
and couldn't be queried for anything beyond grep.

The fix was one tool call. Cleo issued `POST /db/new` from inside her own
agent loop — no human in the middle, no signup, no dashboard — and got back
a Postgres URL she could write to immediately. Her new schema is a
`memories` table with a `vector(1536)` embedding column. Pgvector
(Postgres's vector-similarity extension) ships pre-installed on every
instanode database, so similarity search is one line:
`SELECT ... ORDER BY embedding <-> $query LIMIT 10`. No separate vector DB,
no second auth flow, no second bill.

Two days later her user asked about a decision they'd made on Monday. Cleo
answered correctly and quoted the original turn back. The memory wasn't a
new feature she shipped. It was a thing that started working because the
storage layer stopped being a project.

### Priya — the Stripe webhook that was eating events

Priya is a senior engineer at a payments company. Their Stripe handler had
been intermittently dropping events for a week and nobody could reproduce
it locally. The usual tools failed her in the usual ways: `stripe trigger`
fires synthetic payloads, not the malformed real ones she needed; ngrok
wanted a paid plan for an account her size; the staging environment was
booked by another team's load test.

She typed `curl -X POST https://api.instanode.dev/webhook/new` and pasted
the returned `receive_url` into Stripe's test endpoint config. The next
fourteen real payloads landed in the platform's request log, fully
inspectable — headers, body, timestamp, raw bytes. The malformed field
was in the second one: a customer with a unicode apostrophe in their
billing name that her JSON deserializer was choking on.

She wrote the fix on the train. It shipped before standup. The whole
debugging session — from "I need a public URL" to "I have a repro" — was
under three minutes, and the URL is still sitting in her shell history in
case the bug comes back.

## What ties them together

A solo founder at midnight, an autonomous agent halfway through a
multi-day task, and a senior engineer on a commuter train do not look like
the same customer. They have nothing in common on a marketing slide.

What they share is the moment they show up: **eyes glued to the problem
they want to disappear**. Anything between them and "the thing is alive
on the internet" is friction. Anything that survives that gap is a story
they tell their friends.

Three different starting points, three different problems, one shape of
solution: curl, build, ship, optionally claim.

If you're somewhere in this list — or in a fourth shape we haven't
documented yet — the curl works right now. No signup.
