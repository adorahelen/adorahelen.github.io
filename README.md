# adorahelen.github.io

Personal site — security and AI engineering. Live at **https://adorahelen.github.io**.

Built with [Hugo](https://gohugo.io) and the [Congo](https://github.com/jpanther/congo) theme (vendored under `themes/congo`, MIT, © James Panther). GitHub Actions builds and deploys on every push to `main`.

## Why Hugo

Hugo ships as a single static binary, so it installs **without root** and the site can be previewed locally before anything is published:

```bash
curl -sSL -o hugo.tar.gz https://github.com/gohugoio/hugo/releases/download/v0.165.0/hugo_extended_0.165.0_linux-amd64.tar.gz
tar -xzf hugo.tar.gz hugo && install -m 0755 hugo ~/.local/bin/hugo
hugo server            # http://localhost:1313, live reload
```

That matters more than it sounds: the previous stack needed Node, which isn't installed here, so pages were being pushed without ever being looked at. Don't publish a layout change you haven't seen.

## Structure

| Path | Contents |
| --- | --- |
| `content/_index.md` | Homepage body — positioning, selected work, publication |
| `content/work.md` | Long-form project write-ups (the page a hiring reader lands on) |
| `content/about.md` | Background, contact |
| `content/posts/` | Blog posts. `<slug>.md` is English, `<slug>.ko.md` is the Korean version |
| `config/_default/` | `hugo.toml`, per-language `languages.*.toml` and `menus.*.toml`, `params.toml` |
| `assets/css/custom.css` | Overrides on top of Congo — single-column alignment, spacing, heading style |
| `.github/workflows/deploy.yml` | Hugo build → Pages deploy |

## Adding a post

`content/posts/<slug>.md`:

```yaml
---
title: "Post title"
date: 2026-08-14T09:00:00+09:00
summary: "One or two sentences — shown in listings and used for SEO."
tags: ["ai-security", "llm"]
---
```

For a Korean version, add `content/posts/<slug>.ko.md` with the same slug. Hugo keeps the languages separate — English listings show only English posts — and the footer language switcher moves between them. No cross-linking by hand.

## Cross-posting

This site is the **canonical source**. When republishing to dev.to, Zenn or similar, set that platform's canonical URL to the post's URL here so the copy doesn't compete with the original in search.

## License

Theme: MIT, © James Panther — see `themes/congo/LICENSE`.
Site content and configuration: © 2026 Kangmin Kim.
