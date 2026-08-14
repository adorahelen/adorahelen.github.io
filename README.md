# adorahelen.github.io

Personal blog — security and AI engineering write-ups. Live at **https://adorahelen.github.io**.

Built with [AstroPaper](https://github.com/satnaing/astro-paper) by Sat Naing (MIT). Deployed by GitHub Actions to GitHub Pages on every push to `main`; there is no local build step in the publishing flow.

## Structure

| Path | Contents |
| --- | --- |
| `src/content/posts/` | Blog posts (`.md` / `.mdx`). Files starting with `_` are ignored by the loader. |
| `src/content/pages/` | Standalone pages (`about.md`) |
| `astro-paper.config.ts` | Site title, author, socials, feature toggles |
| `.github/workflows/deploy.yml` | Build (pnpm + Node 24) → upload artifact → deploy to Pages |

## Adding a post

Create `src/content/posts/<slug>.md` with this frontmatter:

```yaml
---
title: "Post title"
pubDatetime: 2026-08-14T09:00:00+09:00
tags: [ai-security, llm]
description: "One or two sentences — used for SEO and the post list."
featured: false # optional
draft: false # optional
canonicalURL: "" # optional — set when the post is republished elsewhere first
---
```

`description` and `pubDatetime` are required; the build fails without them. Push to `main` and the workflow publishes.

Insert `## Table of contents` where you want the generated TOC.

## Cross-posting

This site is the **canonical source**. When republishing to dev.to, Zenn, or similar, set that platform's canonical URL to the post's URL here so the duplicate doesn't compete in search.

## Local development

Requires Node ≥ 22.12 and pnpm.

```bash
pnpm install
pnpm dev      # http://localhost:4321
pnpm build    # astro check + build + pagefind index
```

## License

Theme code: MIT, © 2023 Sat Naing — see [LICENSE](LICENSE).
Post content: © 2026 Kangmin Kim.
