# blog.tth.dev

Personal technical blog. Built with Jekyll and hosted on GitHub Pages.

**Live site:** [blog.tth.dev](https://blog.tth.dev)

## Local Development

```bash
bundle install
bundle exec jekyll serve
```

Visit http://localhost:4000

Alternatively, use the included DevContainer with VS Code or GitHub Codespaces.

## Writing Posts

Create a new file in `_posts/`:

```
YYYY-MM-DD-title.markdown
```

With front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM:SS +0800
categories: category
---
```

Push to `main` to deploy.

## License

Content is copyright the author. Code snippets are available under MIT unless otherwise noted.
