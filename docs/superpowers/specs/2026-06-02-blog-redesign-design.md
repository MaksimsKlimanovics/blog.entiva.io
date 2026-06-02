# Blog Redesign & Content Structure — Design Spec

**Date:** 2026-06-02  
**Project:** blog.entiva.io  
**Status:** Approved

---

## Goals

The blog serves three equally weighted goals:

1. **Personal brand** — showcase Maksims Klimanovičs as a senior BC developer, architect, and consultant
2. **Entiva company presence** — build credibility for the Entiva brand with potential clients
3. **Developer community** — share deep technical content with BC/AI/Azure developers

---

## Visual Direction: Bold & Branded

Clean, confident, and distinctly Entiva without relying on a logo. The look bridges professional credibility (for consultancy clients) and developer-friendliness (for technical readers).

### Color Palette

| Token | Value | Use |
|---|---|---|
| Primary | `#2563eb` | Accents, links, category pills, left-border on cards/headings |
| Primary light | `#eff6ff` | Category pill backgrounds |
| Background | `#f8fafc` | Page background |
| Surface | `#ffffff` | Cards, header, footer |
| Text primary | `#0f172a` | Headings, post titles |
| Text secondary | `#475569` | Body copy |
| Text muted | `#94a3b8` | Dates, meta, labels |
| Border | `#e2e8f0` | Dividers, card borders |
| Code background | `#0f172a` | Syntax-highlighted code blocks |

### Typography

- **All text:** `system-ui, -apple-system, sans-serif` — no web font load, fast and clean
- **Headings:** `font-weight: 700`, `color: #0f172a`
- **Body:** `font-size: 15px`, `line-height: 1.8`, `color: #334155`
- **Code:** `font-family: 'Courier New', monospace` (inline); dark block for fenced code
- **Nav/meta labels:** `11px`, `text-transform: uppercase`, `letter-spacing: 1px`

---

## Approach: Minimal Custom Jekyll Theme

Replace the default `minima` gem theme with a purpose-built set of Jekyll layouts and a single CSS file. The blog is brand new (one post), so the migration risk is zero.

### File structure added

```
_layouts/
  default.html     # html shell, head, includes header + footer
  home.html        # homepage: hero + post list
  post.html        # individual post
  page.html        # static pages (About)
  category.html    # category archive page

_includes/
  header.html      # nav bar
  footer.html      # footer with social links
  post-card.html   # reusable post card partial

_sass/
  main.scss        # single stylesheet, ~200 lines

assets/css/
  styles.scss      # imports _sass/main.scss

_data/
  navigation.yml   # nav items (name + category slug)

_config.yml        # updated with plugins + category config
```

### Jekyll plugins (GitHub Pages safe)

- `jekyll-seo-tag` — `<title>`, Open Graph, Twitter cards, canonical URLs
- `jekyll-feed` — `/feed.xml` RSS
- `jekyll-sitemap` — `/sitemap.xml`
- Rouge — syntax highlighting (already bundled with Jekyll)

---

## Page Designs

### Header (all pages)

- Left: **Maksims Klimanovičs** (bold, 15px) + subtitle "Business Central · AI · Architecture" (muted, 10px)
- Right: nav links — Home | Business Central | AI & Copilot | Architecture | About
- Active link: `#2563eb`, bold
- Background: `#ffffff`, bottom border `#e2e8f0`
- No logo

### Homepage (`/`)

1. **Hero section** — white background, bottom border
   - H1: "Hi, I'm Maksims."
   - Intro paragraph (kept from current `index.md`, the ERP archaeology line)
   - Topic pills (blue on light-blue): Business Central, AL Development, AI & Copilot, Azure, Architecture

2. **Post list** — `f8fafc` background, max-width 720px
   - Section label: "LATEST POSTS" (muted uppercase)
   - Post cards: white, `1px solid #e2e8f0`, `border-radius: 8px`, `border-left: 3px solid #2563eb`
   - Card contents: category pill + date/read-time row, post title (bold 16px), excerpt (muted 13px)

3. **Footer** — white, top border
   - Left: © year · Maksims Klimanovičs · Entiva
   - Right: GitHub · LinkedIn · RSS
   - Social links are driven by `github_username` and `linkedin_username` in `_config.yml`; RSS links to `/feed.xml`

### Post page (`/YYYY/MM/DD/slug/`)

- **Breadcrumb:** `← All posts` (blue link) + `|` + category pill
- **Title:** bold 26px, `#0f172a`
- **Meta:** date + estimated read time, muted 12px
- **Body:** max-width 720px, 15px/1.8 line-height
- **H2 headings in body:** `border-left: 3px solid #2563eb`, `padding-left: 12px`
- **Code blocks:** dark (`#0f172a` bg), rounded, language label in muted text above
- **Tags section:** "TAGS" label + grey pills (`#f1f5f9` bg)
- **Prev/Next:** bottom, ← Previous | Next post →

### Category pages (`/category/business-central/` etc.)

- Same header/footer
- Page title: category name (e.g., "Business Central")
- Post cards identical to homepage list
- Implemented as individual markdown files (`_pages/business-central.md` etc.) with `layout: category` and `category: business-central` in front matter; Jekyll's `site.categories` groups the posts automatically

### About page (`/about/`)

- Same header/footer
- Page layout with standard body typography
- Content from existing `about.md` (kept as-is)

---

## Content Organization

### Navigation categories (3, in header nav)

| Category | Slug | Covers |
|---|---|---|
| Business Central | `business-central` | AL, extensions, integrations, upgrades, BC dev |
| AI & Copilot | `ai-copilot` | AI agents, Copilot, LLMs, Fiona AI, Anatoly AI |
| Architecture | `architecture` | System design, Azure, cloud patterns, ERP architecture |

Each post gets **exactly one category** (set via front matter `category:`).

### Tags (per-post metadata, not in nav)

Applied as `tags: [AL, Azure]` in front matter. 2–4 tags per post. Initial tag set:

`#AL` `#Azure` `#PowerShell` `#Migrations` `#PowerBI` `#Docker` `#EntraID` `#OAuth` `#CSharp` `#TypeScript` `#ERP` `#Integrations` `#M365` `#SQL`

Tag pages are not implemented in v1 (low value until there are many posts). Tags are stored in front matter and visible on the post page only.

---

## SEO & Discoverability

- `jekyll-seo-tag` handles title, description, Open Graph, and Twitter cards automatically
- Each post's `description:` front matter field feeds the meta description
- `jekyll-sitemap` generates `/sitemap.xml`
- `jekyll-feed` generates `/feed.xml`
- Canonical URLs via SEO tag plugin

---

## `.gitignore` additions

```
.superpowers/
_site/
.jekyll-cache/
```

---

## Out of Scope (v1)

- Search (not enough posts yet)
- Comments
- Newsletter
- Tag archive pages
- Dark mode toggle
- Reading progress bar
