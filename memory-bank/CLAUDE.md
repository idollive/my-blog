# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

My-blog — a personal technical blog. Core workflow: **write MDX articles → publish → readers visit**. Prioritizes writing experience and reading performance over feature complexity.

## Tech Stack

| Layer | Choice |
|-------|--------|
| Framework | Astro 5 (static HTML, zero JS by default) |
| Styling | Tailwind CSS 4 |
| Content format | MDX (Markdown + components) |
| Database | MySQL 8 + Prisma ORM |
| Hosting | PlanetScale (serverless MySQL) |
| Deployment | Vercel (git-push deploy, auto SSL, global CDN) |
| Auth | Username/password (single user, no OAuth needed) |
| Comments | Giscus (GitHub Discussions-based, zero self-host) |

Astro was chosen over Next.js because a blog is a content site — it doesn't need SSR, React Server Components, Streaming, or Suspense boundaries. Astro outputs zero-JS HTML by default; interactive islands (React/Vue/Svelte) are added only where needed.

## Key Architecture Decisions

1. **Articles as files** — MDX lives in `src/content/blog/`, read via Astro content collections API. Article body is NEVER stored in the database. Files are the single source of truth for content.
2. **Database for metadata only** — structured fields (title, slug, excerpt, status, category, tags) go in MySQL for admin CRUD. Post table has NO `content` column — article body stays in files.
3. **Local admin + static deploy** — `/admin` backend runs locally (`astro dev`) for writing/editing articles. `git push` → Vercel `astro build` → pure static HTML. Vercel does NOT need database access.
4. **Static-first** — all public-facing pages are built to HTML at build time using Astro content collections. Zero server-side computation at runtime.
5. **JS on demand** — only interactive elements (search, comments) load JavaScript. Static pages stay JS-free.

## Page Structure

```
/                 Post list, paginated, filterable by category/tag
/posts/[slug]     Post detail — MDX rendering + code highlighting + table of contents
/archive          Grouped by year/month
/about            Personal intro
/search           Full-text search (MySQL FULLTEXT)
/admin/posts      Post CRUD, draft/published toggle
/admin/posts/new  New post (MDX editor)
/admin/posts/[id]/edit  Edit post
/admin/settings   Site title, description, logo
```

## Directory Structure

```
src/
├── content/blog/          *.mdx articles (Astro content collection)
├── pages/
│   ├── index.astro        Home
│   ├── posts/[slug].astro Post detail
│   ├── archive.astro      Archive
│   ├── about.astro        About
│   └── admin/             Admin pages
├── components/            Reusable components (React Islands as needed)
├── lib/
│   ├── prisma.ts          Database client
│   └── utils.ts
└── styles/globals.css
prisma/schema.prisma
```

## Data Model

```
Post: id, title, slug(unique), content(MDX), excerpt, coverImage,
      status(draft|published), viewCount, createdAt, updatedAt
      categoryId → Category
      tags → PostTag → Tag

Category: id, name, slug
Tag: id, name, slug
```

## Core Dependencies

Only 8 core packages:
- `astro` ^5.0.0
- `@astrojs/mdx` ^5.0.0
- `@astrojs/tailwind` ^5.0.0
- `@prisma/client` ^6.0.0
- `tailwindcss` ^4.0.0
- `@tailwindcss/typography` ^4.0.0
- `pinyin-pro` ^3.0.0 (Chinese title → pinyin slug)
- `prisma` ^6.0.0 (dev)

## Explicit Non-Goals

- No multi-user system
- No self-built comments (use Giscus)
- No self-built analytics (use Umami or Simple Analytics)
- No theme switching (single color scheme for v1)
- No i18n
- No complex animations
