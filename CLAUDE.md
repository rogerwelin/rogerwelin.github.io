# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Site Overview

This is a Hugo static site (personal tech blog) deployed to GitHub Pages. Theme is **PaperMod**, included as a git submodule under `themes/PaperMod`.

## Commands

```bash
# Local development server (with drafts)
hugo server -D

# Production build
hugo --gc --minify

# Create a new blog post
hugo new posts/my-post-title.md
```

## Deployment

Pushes to `master` trigger the GitHub Actions workflow (`.github/workflows/hugo.yml`) which builds with Hugo v0.155.0 (extended) and deploys to GitHub Pages.

## Content & Structure

- **Blog posts** go in `content/posts/` — older posts use `.markdown` extension with YAML front matter (`---`), newer posts use `.md` with TOML front matter (`+++`)
- **Static images** go in `static/assets/images/` and are referenced in posts as `/assets/images/filename`
- Permalink pattern: `/:year/:month/:day/:slug/`

## Customizations (overrides on PaperMod theme)

- `layouts/partials/comments.html` — Giscus comment integration
- `layouts/shortcodes/callout.html` — custom callout box shortcode, usage: `{{< callout icon="💡" title="Title" >}}content{{< /callout >}}`
- `assets/css/extended/custom.css` — callout styling, terminal-style code blocks (iTerm2 look with traffic light dots), light/dark theme adjustments
- Hugo config allows raw HTML in markdown (`markup.goldmark.renderer.unsafe = true`)
