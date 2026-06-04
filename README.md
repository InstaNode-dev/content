# instanode.dev — public content

Source of truth for the public content of **[instanode.dev](https://instanode.dev)** —
zero-friction developer infrastructure for AI agents. Provision a real Postgres, pgvector,
Redis, MongoDB, NATS queue, or S3 bucket (and deploy your app) with a single HTTP call.
No account, no Docker, no setup. Free anonymous tier.

This repo holds the marketing copy, blog posts, use cases, and docs that ship to
https://instanode.dev — plus **[`llms.txt`](./llms.txt)**, the LLM-friendly index that lets
AI coding agents (Claude, Cursor, Copilot, Windsurf) discover and use the platform. The live
copy is served at https://instanode.dev/llms.txt, with a `/.md` mirror of every page and the
full text at https://instanode.dev/llms-full.txt.

The web app repo (`InstaNode-dev/instanode-web`) clones this repo into its `.content/`
directory at build time and inlines the markdown into the static HTML via Vite's
`import.meta.glob` raw imports. The same flow runs in `npm run dev` so authors get hot
reload on edits in a local clone.

## Layout

```
.
├── llms.txt               LLM-friendly platform index (served at /llms.txt)
├── blog/
│   └── <slug>.md          one post per file; filename = URL slug
├── use-cases/
│   └── <slug>.md          one use case per file; filename = URL slug
├── docs/
│   └── <section>.md       quickstart, services, deploy, stacks, limits, auth, …
└── pages/
    └── <name>.md          top-level marketing pages (home, pricing, for-agents, status)
```

## Blog post format

```markdown
---
title: Maya shipped on Sunday night
date: 2026-05-12
author: instanode.dev
excerpt: One-line summary used in the /blog index card and meta tags.
---

# Maya shipped on Sunday night

Body in CommonMark. The dashboard renderer supports a minimal subset —
H1–H3, paragraphs, fenced code blocks, unordered lists, inline code,
bold. No HTML passthrough (XSS safety).
```

Frontmatter rules:
- `title`, `date` (ISO `YYYY-MM-DD`), `author`, `excerpt` — required
- All values are single-line strings — no YAML block scalars
- Slug = filename without `.md`

## Editing flow

1. `git clone git@github.com:InstaNode-dev/content.git`
2. Edit a `.md` file. Single line edits ship the same way structural ones do.
3. Open a PR. Reviewers see only the diff to the prose — no React, no CSS, no build config.
4. Merge. The next dashboard build picks up the change.

## What NOT to put here

This repo is **public**. Everything in it ships to the public marketing
domain. Do not commit:

- Internal cluster hostnames (`*.svc.cluster.local`)
- Secret variable names (`AES_KEY`, `JWT_SECRET`, `RAZORPAY_*`)
- Customer team IDs, real user emails, real production identifiers
- Stack traces from production
- Roadmap items that haven't been publicly announced

If a piece of content is engineering-internal, it belongs in the
private `InstaNode-dev/docs` repo instead.
