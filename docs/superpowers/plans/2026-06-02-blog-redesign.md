# Blog Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the default minima Jekyll theme with a purpose-built custom theme — Bold & Branded direction (Entiva blue accent, clean sans-serif, white/light-gray layout) — plus navigation-based content categories and full SEO wiring.

**Architecture:** Custom layouts in `_layouts/`, Liquid partials in `_includes/`, a single Sass stylesheet at `_sass/main.scss`, category archive pages in `categories/`, and nav config in `_data/navigation.yml`. The `github-pages` gem pins all dependencies to match the GitHub Pages build environment exactly.

**Tech Stack:** Jekyll, GitHub Pages, Sass, Rouge (syntax highlighting), jekyll-seo-tag, jekyll-feed, jekyll-sitemap

---

## File Map

| Action | File | Purpose |
|---|---|---|
| Create | `Gemfile` | Pin to github-pages gem |
| Modify | `_config.yml` | Remove minima, add plugins, social links |
| Modify | `.gitignore` | Add `_site/`, `.jekyll-cache/`, `.superpowers/` |
| Create | `_sass/main.scss` | Full stylesheet (~250 lines) |
| Create | `assets/css/styles.scss` | Sass entrypoint (imports main) |
| Create | `_data/navigation.yml` | Nav items driving the header |
| Create | `_includes/header.html` | Sticky header with brand + nav |
| Create | `_includes/footer.html` | Footer with social links |
| Create | `_includes/post-card.html` | Reusable post card partial |
| Create | `_layouts/default.html` | HTML shell — head, body, header, footer |
| Create | `_layouts/home.html` | Hero + post list |
| Create | `_layouts/post.html` | Individual post page |
| Create | `_layouts/page.html` | Static pages (About) |
| Create | `_layouts/category.html` | Category archive |
| Create | `categories/business-central.md` | `/category/business-central/` archive |
| Create | `categories/ai-copilot.md` | `/category/ai-copilot/` archive |
| Create | `categories/architecture.md` | `/category/architecture/` archive |
| Modify | `index.md` | Trim to intro paragraph, update layout |
| Modify | `about.md` | Update layout to `page` |
| Modify | `_posts/2026-06-02-welcome.md` | Add category, tags, description |

---

## Task 1: Foundation — Gemfile, _config.yml, .gitignore

**Files:**
- Create: `Gemfile`
- Modify: `_config.yml`
- Modify: `.gitignore`

- [ ] **Step 1: Create Gemfile**

```ruby
source "https://rubygems.org"

gem "github-pages", group: :jekyll_plugins
gem "webrick"
```

- [ ] **Step 2: Replace _config.yml contents entirely**

```yaml
title: Maksims Klimanovičs
description: Business Central Architect, AI Builder, Consultant
url: "https://blog.entiva.io"
author: Maksims Klimanovičs
github_username: mrfloksis
linkedin_username: ""

show_excerpts: true

plugins:
  - jekyll-seo-tag
  - jekyll-feed
  - jekyll-sitemap

markdown: kramdown
highlighter: rouge

kramdown:
  syntax_highlighter: rouge
  syntax_highlighter_opts:
    block:
      line_numbers: false
```

Note: `theme: minima` is intentionally absent. The custom `_layouts/` directory takes over.

- [ ] **Step 3: Create .gitignore**

```
_site/
.jekyll-cache/
.bundle/
vendor/
.superpowers/
Gemfile.lock
```

- [ ] **Step 4: Install dependencies**

```bash
bundle install
```

Expected: Resolves `github-pages` and writes `Gemfile.lock`. If `bundle` is not installed, run `gem install bundler` first.

- [ ] **Step 5: Verify Jekyll starts**

```bash
bundle exec jekyll build 2>&1
```

Expected: Build completes (may warn about missing layouts — that's fine, we add them in Task 4).

- [ ] **Step 6: Commit**

```bash
git add Gemfile _config.yml .gitignore
git commit -m "feat: bootstrap custom theme — remove minima, add plugins"
```

---

## Task 2: Stylesheet

**Files:**
- Create: `assets/css/styles.scss`
- Create: `_sass/main.scss`

- [ ] **Step 1: Create assets/css/styles.scss**

The `---` front matter is required for Jekyll's Sass pipeline to process this file:

```scss
---
---
@import "main";
```

- [ ] **Step 2: Create _sass/main.scss**

```scss
// ── Variables ──────────────────────────────────────────────────────────────
$primary:        #2563eb;
$primary-light:  #eff6ff;
$bg:             #f8fafc;
$surface:        #ffffff;
$text-primary:   #0f172a;
$text-secondary: #475569;
$text-body:      #334155;
$text-muted:     #94a3b8;
$border:         #e2e8f0;
$code-bg:        #0f172a;
$font:           system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;

// ── Reset ───────────────────────────────────────────────────────────────────
*, *::before, *::after { box-sizing: border-box; }
body {
  margin: 0;
  font-family: $font;
  font-size: 15px;
  line-height: 1.8;
  background: $bg;
  color: $text-body;
}
a { color: $primary; text-decoration: none; }
a:hover { text-decoration: underline; }
img { max-width: 100%; }

// ── Header ──────────────────────────────────────────────────────────────────
.site-header {
  background: $surface;
  border-bottom: 1px solid $border;
  padding: 14px 32px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  position: sticky;
  top: 0;
  z-index: 10;
}
.header-brand a { text-decoration: none; }
.site-title {
  font-size: 15px;
  font-weight: 700;
  color: $text-primary;
  line-height: 1.1;
}
.site-tagline {
  font-size: 10px;
  color: $text-muted;
  margin-top: 2px;
}
.site-nav {
  display: flex;
  gap: 20px;
  a {
    font-size: 13px;
    color: $text-secondary;
    &:hover { color: $primary; text-decoration: none; }
    &.active { color: $primary; font-weight: 600; }
  }
}

// ── Hero ────────────────────────────────────────────────────────────────────
.hero {
  background: $surface;
  border-bottom: 1px solid $border;
  padding: 32px 32px 28px;
  h1 {
    font-size: 22px;
    font-weight: 700;
    color: $text-primary;
    line-height: 1.3;
    margin: 0 0 10px;
  }
  p {
    font-size: 14px;
    color: $text-secondary;
    line-height: 1.7;
    margin: 0 0 16px;
    max-width: 600px;
  }
}
.topic-pills {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

// ── Pills ───────────────────────────────────────────────────────────────────
.pill {
  background: $primary-light;
  color: $primary;
  font-size: 11px;
  padding: 3px 10px;
  border-radius: 12px;
  font-weight: 500;
  white-space: nowrap;
}
.pill-grey {
  background: #f1f5f9;
  color: $text-secondary;
  font-size: 11px;
  padding: 3px 10px;
  border-radius: 12px;
  white-space: nowrap;
}

// ── Post list ───────────────────────────────────────────────────────────────
.post-list-wrap {
  padding: 24px 32px;
  max-width: 752px;
}
.section-label {
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 1px;
  color: $text-muted;
  font-weight: 600;
  margin-bottom: 16px;
}
.post-card {
  background: $surface;
  border: 1px solid $border;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 12px;
  border-left: 3px solid $primary;
  transition: box-shadow 0.15s;
  &:hover { box-shadow: 0 2px 10px rgba(0, 0, 0, 0.07); }
  .post-meta {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 8px;
    font-size: 11px;
    color: $text-muted;
  }
  h2 {
    font-size: 16px;
    font-weight: 600;
    color: $text-primary;
    margin: 0 0 6px;
    a {
      color: $text-primary;
      &:hover { color: $primary; text-decoration: none; }
    }
  }
  p {
    font-size: 13px;
    color: $text-secondary;
    line-height: 1.6;
    margin: 0;
  }
}

// ── Post page ───────────────────────────────────────────────────────────────
.post-header {
  background: $surface;
  border-bottom: 1px solid $border;
  padding: 32px 32px 28px;
}
.post-breadcrumb {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 16px;
  font-size: 12px;
  .sep { color: $border; }
  a { color: $primary; }
}
.post-title {
  font-size: 26px;
  font-weight: 700;
  color: $text-primary;
  line-height: 1.25;
  margin: 0 0 12px;
}
.post-dateline {
  font-size: 12px;
  color: $text-muted;
}
.post-body {
  padding: 28px 32px;
  max-width: 720px;
  h2 {
    font-size: 18px;
    font-weight: 600;
    color: $text-primary;
    margin: 28px 0 12px;
    padding-left: 12px;
    border-left: 3px solid $primary;
  }
  h3 {
    font-size: 16px;
    font-weight: 600;
    color: $text-primary;
    margin: 22px 0 8px;
  }
  p { margin: 0 0 18px; }
  ul, ol { padding-left: 24px; margin: 0 0 18px; }
  li { margin-bottom: 4px; }
  blockquote {
    border-left: 3px solid $primary;
    margin: 18px 0;
    padding: 4px 0 4px 16px;
    color: $text-secondary;
    font-style: italic;
    p { margin: 0; }
  }
  // Inline code
  code {
    background: #f1f5f9;
    color: $text-primary;
    font-family: 'JetBrains Mono', 'Courier New', monospace;
    font-size: 13px;
    padding: 2px 5px;
    border-radius: 3px;
  }
  // Fenced code blocks (Rouge wraps output in .highlight)
  .highlight {
    background: $code-bg;
    border-radius: 6px;
    margin: 18px 0;
    overflow-x: auto;
    pre {
      margin: 0;
      padding: 16px;
      background: transparent;
    }
    code {
      background: transparent;
      color: #e2e8f0;
      font-size: 12px;
      padding: 0;
    }
  }
}
.post-tags {
  margin-top: 32px;
  padding-top: 20px;
  border-top: 1px solid $border;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.tags-row {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}
.post-nav {
  margin-top: 28px;
  padding-top: 20px;
  border-top: 1px solid $border;
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  .prev { color: $text-muted; }
  .next { color: $primary; font-weight: 500; }
}

// ── Static page (About) ─────────────────────────────────────────────────────
.page-wrap {
  padding: 32px;
  max-width: 720px;
  margin: 24px auto;
  background: $surface;
  border: 1px solid $border;
  border-radius: 8px;
  h1 { font-size: 24px; font-weight: 700; color: $text-primary; margin: 0 0 20px; }
  h2 {
    font-size: 17px;
    font-weight: 600;
    color: $text-primary;
    margin: 24px 0 10px;
    padding-left: 12px;
    border-left: 3px solid $primary;
  }
  ul { padding-left: 20px; margin: 0 0 16px; }
  li { margin-bottom: 4px; }
  p { margin: 0 0 16px; }
}

// ── Category page ───────────────────────────────────────────────────────────
.category-header {
  background: $surface;
  border-bottom: 1px solid $border;
  padding: 24px 32px;
  h1 { font-size: 22px; font-weight: 700; color: $text-primary; margin: 0; }
}

// ── Footer ──────────────────────────────────────────────────────────────────
.site-footer {
  border-top: 1px solid $border;
  padding: 16px 32px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: $surface;
  font-size: 11px;
  color: $text-muted;
  margin-top: 40px;
  .footer-links {
    display: flex;
    gap: 14px;
    a { color: $text-secondary; &:hover { color: $primary; text-decoration: none; } }
  }
}

// ── Rouge syntax highlighting (dark theme) ───────────────────────────────────
.highlight .k,  .highlight .kd, .highlight .kn,
.highlight .kp, .highlight .kr, .highlight .kt { color: #7dd3fc; }
.highlight .s,  .highlight .s2, .highlight .s1,
.highlight .sb, .highlight .sc, .highlight .sd,
.highlight .se, .highlight .si, .highlight .sx  { color: #86efac; }
.highlight .c,  .highlight .c1, .highlight .cm,
.highlight .cs, .highlight .cp                  { color: #64748b; font-style: italic; }
.highlight .nf, .highlight .fm                  { color: #a5f3fc; }
.highlight .nb, .highlight .bp                  { color: #fca5a5; }
.highlight .nc, .highlight .nn                  { color: #fde68a; }
.highlight .mi, .highlight .mf,
.highlight .mh, .highlight .mo                  { color: #fcd34d; }
.highlight .o,  .highlight .ow                  { color: #94a3b8; }
.highlight .na                                  { color: #c4b5fd; }
.highlight .n,  .highlight .no                  { color: #e2e8f0; }
.highlight .p                                   { color: #94a3b8; }
.highlight .nt                                  { color: #7dd3fc; }
.highlight .nv, .highlight .vc,
.highlight .vg, .highlight .vi                  { color: #fde68a; }
.highlight .err { color: #fca5a5; background: transparent; }
```

- [ ] **Step 3: Verify Sass compiles**

```bash
bundle exec jekyll build 2>&1
```

Expected: Build succeeds, `_site/assets/css/styles.css` exists.

```bash
ls _site/assets/css/
```

- [ ] **Step 4: Commit**

```bash
git add _sass/main.scss assets/css/styles.scss
git commit -m "feat: add custom stylesheet with Bold & Branded theme and Rouge dark highlighting"
```

---

## Task 3: Navigation data + Includes

**Files:**
- Create: `_data/navigation.yml`
- Create: `_includes/header.html`
- Create: `_includes/footer.html`
- Create: `_includes/post-card.html`

- [ ] **Step 1: Create _data/navigation.yml**

```yaml
- title: Business Central
  url: /category/business-central/
- title: AI & Copilot
  url: /category/ai-copilot/
- title: Architecture
  url: /category/architecture/
```

- [ ] **Step 2: Create _includes/header.html**

```liquid
<header class="site-header">
  <div class="header-brand">
    <a href="{{ '/' | relative_url }}">
      <div class="site-title">{{ site.author }}</div>
      <div class="site-tagline">Business Central · AI · Architecture</div>
    </a>
  </div>
  <nav class="site-nav">
    <a href="{{ '/' | relative_url }}"
       {% if page.url == "/" %}class="active"{% endif %}>Home</a>
    {% for item in site.data.navigation %}
      <a href="{{ item.url | relative_url }}"
         {% if page.url contains item.url %}class="active"{% endif %}>{{ item.title }}</a>
    {% endfor %}
    <a href="{{ '/about/' | relative_url }}"
       {% if page.url == "/about/" %}class="active"{% endif %}>About</a>
  </nav>
</header>
```

- [ ] **Step 3: Create _includes/footer.html**

```liquid
<footer class="site-footer">
  <span>© {{ 'now' | date: '%Y' }} {{ site.author }} · Entiva</span>
  <div class="footer-links">
    {% if site.github_username != "" %}
      <a href="https://github.com/{{ site.github_username }}"
         target="_blank" rel="noopener noreferrer">GitHub</a>
    {% endif %}
    {% if site.linkedin_username != "" %}
      <a href="https://www.linkedin.com/in/{{ site.linkedin_username }}"
         target="_blank" rel="noopener noreferrer">LinkedIn</a>
    {% endif %}
    <a href="{{ '/feed.xml' | relative_url }}">RSS</a>
  </div>
</footer>
```

- [ ] **Step 4: Create _includes/post-card.html**

```liquid
<article class="post-card">
  <div class="post-meta">
    {% if post.category != "" %}
      <span class="pill">{{ post.category | replace: "-", " " | capitalize }}</span>
    {% endif %}
    <span>{{ post.date | date: "%-d %b %Y" }}</span>
    <span>·</span>
    {% assign read_time = post.content | number_of_words | divided_by: 200 | at_least: 1 %}
    <span>{{ read_time }} min read</span>
  </div>
  <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
  {% if post.description %}
    <p>{{ post.description }}</p>
  {% elsif post.excerpt %}
    <p>{{ post.excerpt | strip_html | truncatewords: 25 }}</p>
  {% endif %}
</article>
```

- [ ] **Step 5: Commit**

```bash
git add _data/ _includes/
git commit -m "feat: add navigation data and header/footer/post-card includes"
```

---

## Task 4: Layouts

**Files:**
- Create: `_layouts/default.html`
- Create: `_layouts/home.html`
- Create: `_layouts/post.html`
- Create: `_layouts/page.html`
- Create: `_layouts/category.html`

- [ ] **Step 1: Create _layouts/default.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="{{ '/assets/css/styles.css' | relative_url }}">
  {% seo %}
</head>
<body>
  {% include header.html %}
  <main>{{ content }}</main>
  {% include footer.html %}
</body>
</html>
```

- [ ] **Step 2: Create _layouts/home.html**

```liquid
---
layout: default
---
<section class="hero">
  <h1>Hi, I'm Maksims.</h1>
  {{ content }}
  <div class="topic-pills">
    <span class="pill">Business Central</span>
    <span class="pill">AL Development</span>
    <span class="pill">AI &amp; Copilot</span>
    <span class="pill">Azure</span>
    <span class="pill">Architecture</span>
  </div>
</section>
<div class="post-list-wrap">
  <div class="section-label">Latest Posts</div>
  {% for post in site.posts %}
    {% include post-card.html %}
  {% endfor %}
  {% if site.posts.size == 0 %}
    <p style="color:#94a3b8;padding:12px 0;">No posts yet — check back soon.</p>
  {% endif %}
</div>
```

- [ ] **Step 3: Create _layouts/post.html**

```liquid
---
layout: default
---
<div class="post-header">
  <div class="post-breadcrumb">
    <a href="{{ '/' | relative_url }}">← All posts</a>
    {% if page.category != "" %}
      <span class="sep">|</span>
      <span class="pill">{{ page.category | replace: "-", " " | capitalize }}</span>
    {% endif %}
  </div>
  <h1 class="post-title">{{ page.title }}</h1>
  <div class="post-dateline">
    {{ page.date | date: "%-d %b %Y" }}
    {% assign read_time = content | number_of_words | divided_by: 200 | at_least: 1 %}
    · {{ read_time }} min read
  </div>
</div>
<article class="post-body">
  {{ content }}
  {% if page.tags.size > 0 %}
  <div class="post-tags">
    <div class="section-label">Tags</div>
    <div class="tags-row">
      {% for tag in page.tags %}
        <span class="pill-grey">#{{ tag }}</span>
      {% endfor %}
    </div>
  </div>
  {% endif %}
  <nav class="post-nav">
    {% if page.previous.url %}
      <a class="prev" href="{{ page.previous.url | relative_url }}">← {{ page.previous.title | truncatewords: 6 }}</a>
    {% else %}
      <span></span>
    {% endif %}
    {% if page.next.url %}
      <a class="next" href="{{ page.next.url | relative_url }}">{{ page.next.title | truncatewords: 6 }} →</a>
    {% endif %}
  </nav>
</article>
```

- [ ] **Step 4: Create _layouts/page.html**

```liquid
---
layout: default
---
<div class="page-wrap">
  <h1>{{ page.title }}</h1>
  {{ content }}
</div>
```

- [ ] **Step 5: Create _layouts/category.html**

```liquid
---
layout: default
---
<div class="category-header">
  <h1>{{ page.title }}</h1>
</div>
<div class="post-list-wrap">
  {% assign cat_posts = site.posts | where: "category", page.category %}
  {% for post in cat_posts %}
    {% include post-card.html %}
  {% endfor %}
  {% if cat_posts.size == 0 %}
    <p style="color:#94a3b8;padding:12px 0;">No posts in this category yet.</p>
  {% endif %}
</div>
```

- [ ] **Step 6: Build and check for errors**

```bash
bundle exec jekyll build 2>&1
```

Expected: Clean build, no layout-missing warnings.

- [ ] **Step 7: Commit**

```bash
git add _layouts/
git commit -m "feat: add custom layouts (default, home, post, page, category)"
```

---

## Task 5: Category archive pages

**Files:**
- Create: `categories/business-central.md`
- Create: `categories/ai-copilot.md`
- Create: `categories/architecture.md`

- [ ] **Step 1: Create categories/business-central.md**

```markdown
---
layout: category
title: Business Central
category: business-central
permalink: /category/business-central/
---
```

- [ ] **Step 2: Create categories/ai-copilot.md**

```markdown
---
layout: category
title: AI & Copilot
category: ai-copilot
permalink: /category/ai-copilot/
---
```

- [ ] **Step 3: Create categories/architecture.md**

```markdown
---
layout: category
title: Architecture
category: architecture
permalink: /category/architecture/
---
```

- [ ] **Step 4: Verify all three pages are generated**

```bash
bundle exec jekyll build 2>&1 && ls _site/category/
```

Expected output includes: `ai-copilot/  architecture/  business-central/`

- [ ] **Step 5: Commit**

```bash
git add categories/
git commit -m "feat: add category archive pages (BC, AI & Copilot, Architecture)"
```

---

## Task 6: Update existing content

**Files:**
- Modify: `index.md`
- Modify: `about.md`
- Modify: `_posts/2026-06-02-welcome.md`

- [ ] **Step 1: Replace index.md**

The `home.html` layout adds the H1 and topic pills. The markdown here is just the intro paragraph:

```markdown
---
layout: home
title: Home
description: "Personal blog by Maksims Klimanovičs on Business Central, AI, Azure, and ERP architecture."
---

I build Business Central solutions, AI agents, integrations, cloud platforms — and occasionally perform ERP archaeology on systems older than some junior developers.
```

- [ ] **Step 2: Replace about.md**

```markdown
---
layout: page
title: About
permalink: /about/
---

## About Me

Senior Business Central Developer, Solution Architect and Consultant.

## Projects

- Entiva
- Fiona AI
- Anatoly AI
- GEMS
- XNT System Utils

## Technologies

AL, C#, TypeScript, PowerShell, SQL, Azure, Docker, Microsoft 365, Entra ID.

## Things I Write About

- ERP architecture
- Business Central development
- AI adoption in business
- Integrations
- Lessons learned from real projects
```

- [ ] **Step 3: Replace _posts/2026-06-02-welcome.md**

```markdown
---
layout: post
title: "Welcome to the Blog"
date: 2026-06-02
category: architecture
tags: [ERP, Architecture, BusinessCentral]
description: "An introduction to this blog — covering Business Central, AI agents, Azure integrations, and lessons from the ERP trenches."
---

Welcome.

This blog contains technical articles, architecture discussions, Business Central deep dives, AI experiments, and stories from the ERP trenches.

If you've ever debugged OAuth at 2AM, you'll probably feel at home here.
```

- [ ] **Step 4: Build and verify**

```bash
bundle exec jekyll build 2>&1
```

Expected: Clean build, no warnings.

- [ ] **Step 5: Commit**

```bash
git add index.md about.md "_posts/2026-06-02-welcome.md"
git commit -m "feat: update existing content — new layouts, post metadata, trimmed index"
```

---

## Task 7: Final verification

- [ ] **Step 1: Start local server**

```bash
bundle exec jekyll serve --livereload
```

- [ ] **Step 2: Verify each page**

Open each URL and confirm the expected content:

| URL | Check |
|---|---|
| `http://localhost:4000/` | Sticky header, hero with intro + pills, one post card with blue left border |
| `http://localhost:4000/2026/06/02/welcome/` | Breadcrumb "← All posts \| Architecture", title, tags, prev/next nav |
| `http://localhost:4000/about/` | White card, "About" as h1, blue-bordered h2s |
| `http://localhost:4000/category/architecture/` | "Architecture" heading, welcome post card visible |
| `http://localhost:4000/category/business-central/` | "Business Central" heading, empty-state message |
| `http://localhost:4000/category/ai-copilot/` | "AI & Copilot" heading, empty-state message |
| `http://localhost:4000/feed.xml` | RSS feed renders |
| `http://localhost:4000/sitemap.xml` | Sitemap renders |

- [ ] **Step 3: Verify SEO tags in page source**

View source of the homepage. Confirm:
- `<title>Maksims Klimanovičs</title>` is present
- `<meta name="description" content="Business Central Architect...">` is present
- `<meta property="og:title" ...>` is present

- [ ] **Step 4: (Optional) Add LinkedIn username**

If you have a LinkedIn profile, edit `_config.yml` and set:

```yaml
linkedin_username: "your-linkedin-slug"
```

Then rebuild to see the LinkedIn link appear in the footer.

- [ ] **Step 5: Final commit**

```bash
git add -A
git commit -m "feat: complete blog redesign — Bold & Branded custom Jekyll theme"
```

---

## Spec Coverage

| Requirement | Covered by |
|---|---|
| Bold & Branded visual direction (`#2563eb`, sans-serif, light bg) | Task 2 |
| No logo — name + tagline only | Task 3 header.html |
| Nav: Home, BC, AI & Copilot, Architecture, About | Task 3 navigation.yml + header.html |
| Hero: intro paragraph + topic pills | Task 4 home.html |
| Post cards: blue left-border, category pill, date, excerpt | Task 3 post-card.html |
| Post page: breadcrumb, title, date/read-time, blue h2, dark code, tags, prev/next | Task 4 post.html |
| Category archive pages | Tasks 4 + 5 |
| About page | Task 4 page.html + Task 6 about.md |
| 1 category + tags per post | Task 6 welcome post |
| jekyll-seo-tag, jekyll-feed, jekyll-sitemap | Task 1 _config.yml |
| Social links from `_config.yml` | Task 3 footer.html |
| `.gitignore` updated | Task 1 |
| LinkedIn slot in `_config.yml` | Task 1 (`linkedin_username: ""`) |
