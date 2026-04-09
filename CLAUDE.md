# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal website and blog for David Beck built with Jekyll 3.7.4, hosted on GitHub Pages at davidbeck.co.

## Build Commands

```bash
# Install dependencies (requires Ruby 2.6.1 via RVM)
bundle install

# Run local development server at http://localhost:4000
bundle exec jekyll serve

# Build static files to _site/
bundle exec jekyll build
```

## Architecture

- **Jekyll static site** with GitHub Pages deployment (auto-deploys on push to master)
- **Blog posts**: `_posts/YYYY-MM-DD-slug.md` with YAML front matter
- **Layouts**: `_layouts/` (default.html wraps all pages, post.html for blog posts)
- **Includes**: `_includes/` for reusable components like SVG icons
- **Styling**: `assets/main.scss` - SCSS with Google Fonts (Josefin Sans, Lobster)
- **Portfolio**: `apps/` directory for app showcase pages (e.g., ThinkSocial)

## Content Format

Blog posts use this front matter structure:
```yaml
---
layout: post
title: "Post Title"
date: 2023-02-05
tags: [tag1, tag2]
---
```

Permalink pattern: `/posts/YYYY-MM-DD-slug/`

## Key Configuration

- Markdown processor: kramdown
- Syntax highlighter: rouge
- Plugins: jekyll-feed (RSS), jekyll-redirect-from
