# Desi Blog

A personal blog built with [Jekyll](https://jekyllrb.com/) and hosted on [GitHub Pages](https://pages.github.com/).

## Prerequisites

- [Ruby](https://www.ruby-lang.org/en/downloads/) (version 3.0 or higher)
- [Bundler](https://bundler.io/) (`gem install bundler`)

## Local Development

```bash
# Install dependencies
bundle install

# Start the local development server
bundle exec jekyll serve

# Visit http://localhost:4000/Blog/ in your browser
```

The site will auto-reload when you make changes to files.

## Writing a New Post

1. Create a new file in the `_posts/` directory with the format:
   ```
   YYYY-MM-DD-title-of-your-post.md
   ```

2. Add front matter at the top of the file:
   ```yaml
   ---
   layout: post
   title: "Your Post Title"
   date: YYYY-MM-DD
   categories: [category]
   tags: [tag1, tag2]
   ---
   ```

3. Write your content in Markdown below the front matter.

## Deploying

This blog automatically deploys to GitHub Pages when you push to the `main` branch via the included GitHub Actions workflow.

## Project Structure

```
Blog/
├── _config.yml          # Site configuration
├── _posts/              # Blog posts (Markdown)
├── _layouts/            # HTML templates
│   ├── default.html     # Base layout
│   ├── post.html        # Single post layout
│   └── page.html        # Static page layout
├── _includes/           # Reusable components
│   └── post-card.html   # Post preview card
├── _sass/               # SCSS stylesheets
│   ├── _variables.scss  # Design tokens
│   ├── _base.scss       # Base styles & typography
│   └── _layout.scss     # Layout & component styles
├── assets/
│   ├── css/style.scss   # Main stylesheet entry
│   └── images/          # Image assets
├── index.html           # Home page
├── about.md             # About page
├── archive.md           # Post archive page
└── .github/workflows/   # CI/CD pipeline
```
