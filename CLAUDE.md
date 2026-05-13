# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hugo-based blog (`blog.norin.me`) with content in Chinese about autonomous driving control systems and other technical topics. The site uses the PaperMod theme as a Git submodule.

## Project Structure

- **`content/`** - Hugo content organized by series/categories
  - `auto-control-beginner/` - Series on autonomous driving control (trajectory tracking, P controllers, Pure Pursuit algorithm)
  - `posts/` - Individual blog posts
- **`layouts/`** - Custom Hugo templates (overwrites theme defaults)
  - `index.html` - Homepage layout
  - `_default/` - Default content layouts
  - `partials/` - Reusable template components
- **`static/`** - Static assets (CSS, images) served as-is
- **`themes/PaperMod/`** - Hugo theme (Git submodule)
- **`config.toml`** - Hugo configuration
  - Base URL: `https://blog.norin.me/`
  - Locale: Chinese (zh-CN)
  - Table of contents starts at H2, ends at H3

## Common Commands

### Development
- **Serve locally:** `hugo server -D` (includes draft posts)
- **Build:** `hugo` (generates static site to `public/`)
- **Build with drafts:** `hugo -D`

### Content
- **Create new post:** `hugo new content/posts/filename.md` or `hugo new content/series-name/post.md`
- **Draft structure:** Posts use frontmatter with `title`, `description`, `date`, and `draft: true` for unpublished posts

## Content Guidelines

- Posts use Markdown with YAML frontmatter
- Table of contents is auto-generated (H2-H3 headings)
- Chinese locale is configured; write content accordingly
- The auto-control-beginner series uses code blocks and diagrams to explain control system concepts

## Deployment

The `public/` directory contains the generated static site. After running `hugo build`, this is what gets deployed to the live site.
