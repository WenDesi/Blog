---
layout: post
title: "Getting Started with Jekyll"
date: 2026-03-15
categories: [tech, tutorial]
tags: [jekyll, blogging, static-sites]
---

Jekyll is a fantastic tool for creating blogs and static websites. In this post, I'll walk through the basics of how this blog is set up.

## What is Jekyll?

Jekyll is a static site generator written in Ruby. Unlike WordPress or other CMS platforms, Jekyll doesn't need a database. It takes your content (written in Markdown), applies layouts, and generates a complete static website.

**Key advantages:**

- **Fast** — Static HTML files load incredibly quickly
- **Secure** — No database means fewer attack vectors
- **Free hosting** — GitHub Pages hosts Jekyll sites for free
- **Version controlled** — Your entire blog lives in a Git repository

## Project Structure

Here's how a typical Jekyll blog is organized:

```
Blog/
├── _config.yml        # Site configuration
├── _posts/            # Your blog posts (Markdown)
├── _layouts/          # HTML templates
├── _includes/         # Reusable HTML components
├── _sass/             # Stylesheets
├── assets/            # CSS, images, etc.
├── index.html         # Home page
├── about.md           # About page
└── archive.md         # Archive page
```

## Writing a New Post

To create a new post, simply add a Markdown file to the `_posts` directory with the naming convention:

```
YYYY-MM-DD-title-of-your-post.md
```

Each post starts with **front matter** — a YAML block that tells Jekyll how to process the file:

```yaml
---
layout: post
title: "Your Post Title"
date: 2026-03-15
categories: [category1, category2]
tags: [tag1, tag2]
---
```

## Running Locally

To preview your blog locally:

```bash
# Install dependencies
bundle install

# Start the development server
bundle exec jekyll serve
```

Then visit `http://localhost:4000/Blog/` in your browser.

## Deploying to GitHub Pages

The easiest way to deploy is with GitHub Pages:

1. Push your code to a GitHub repository
2. Go to **Settings > Pages**
3. Select the branch to deploy from
4. Your blog will be live at `https://username.github.io/Blog/`

Happy blogging!
