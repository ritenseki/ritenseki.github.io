# CLAUDE.md

This file provides guidance to coding agents working in this repository. `AGENTS.md` imports it, so keep project guidance here rather than duplicating it.

## Project

Personal Hugo blog deployed at https://ritenseki.github.io/ via GitHub Pages.

## Build & deploy

- `hugo` — build site to `docs/`
- `hugo server` — local preview at http://localhost:1313/
- `hugo server -D` — preview including drafts
- `hugo new essays/<slug>.md` — scaffold a new post (defaults to `draft = true`)

Deploy flow: edit source → run `hugo` to rebuild → commit both source and `docs/` → push to `main`. GitHub Pages is configured to serve from the `main` branch `/docs` directory.

## Architecture

Single custom theme `themes/mytheme`. Non-obvious points:

- **publishDir is `docs/`** (set in `hugo.toml`), not the default `public/`. This is a GitHub Pages user-page requirement: Pages can only publish from `/` or `/docs`.
- **Section index pages** (`content/{essays,projects,gallery}/_index.md`) put a Radiohead lyric in their body. `themes/mytheme/layouts/_default/list.html` renders it above the article list via `{{ with .Content }}<div class="section-desc">{{ . }}</div>{{ end }}`.
- **Homepage** is `content/_index.md` rendered by `themes/mytheme/layouts/index.html`. It carries two co-worker notes built from the same shortcode pattern: `claude-note` (`themes/mytheme/layouts/shortcodes/claude-note.html`) — orange (`#E07B54`) text + starburst SVG mark + "written from Claude" signature; and `ds-note` (`themes/mytheme/layouts/shortcodes/ds-note.html`) — DeepSeek blue (`#4D6BFE`) text + whale mark + "written from DeepSeek" signature, with an optional `date` parameter appended to the signature.

## Conventions

- Posts go in `content/essays/<slug>.md`. Set `draft = false` to publish.
- Theme CSS lives in `themes/mytheme/static/css/main.css`. iA Writer Mono throughout, max-width 680px, warm off-white background `#faf8f5`.
- `.gitignore` excludes `public/` (legacy build dir), `.hugo_build.lock`, and `.claude/`. The `docs/` build output IS tracked — required for Pages.
