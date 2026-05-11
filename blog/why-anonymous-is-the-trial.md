---
title: Why anonymous is the trial
date: 2026-05-09
author: instanode.dev
excerpt: Most platforms run a 14-day free trial. We run a 24-hour anonymous tier. Here is why that flip is the most important pricing decision we made.
---

# Why anonymous is the trial

Most platforms run a 14-day trial. After the trial expires the user has to enter
a credit card, navigate three onboarding screens, and confirm an email. Friction
is the point — it filters out people who would not pay.

We do the opposite. The first 24 hours are entirely anonymous: no signup, no
card, just a curl. After 24 hours the resource expires unless you claim it.
Claiming costs money from day one.

## What we believe

A developer who runs `curl -X POST https://api.instanode.dev/db/new` and gets a
working Postgres URL back in under a second has already done the only test that
matters. The decision to pay isn't "is this product good enough to justify a
trial" — it's "is keeping this specific running thing worth $9/mo".

## The numbers

Anonymous → claimed conversion is currently around 4% in beta. Trial-conversion
benchmarks for B2B SaaS hover at 14–18%, so on the surface our number looks
worse. But our anonymous funnel includes traffic that would never have signed
up for a trial in the first place — agents poking the API to see if it works,
people copy-pasting from a blog post, demos. The denominator is bigger.

What we watch instead is "claimed → kept past 7 days" — that's 89% so far.
Once an agent or a human shipped something on instanode.dev, they tend to
keep it.

## What this enables

- Claude Code can provision a postgres in a tool call without asking the user
  for permission first
- A blog post that says "just curl this URL" actually works for the reader
- The platform documentation can demo every feature live, not in a sandbox

That third one is what unlocked the docs you're reading now. The code
snippets here aren't mock requests — they hit production.
