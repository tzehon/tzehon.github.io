# blog.tth.dev

[![GitHub Pages](https://github.com/tzehon/tzehon.github.io/actions/workflows/pages/pages-build-deployment/badge.svg)](https://github.com/tzehon/tzehon.github.io/actions/workflows/pages/pages-build-deployment)

Personal technical blog. Built with Jekyll and hosted on GitHub Pages.

**Live site:** [blog.tth.dev](https://blog.tth.dev)

## Wardrobe public pages

The [Wardrobe support page](https://blog.tth.dev/wardrobe/),
[privacy policy](https://blog.tth.dev/wardrobe/privacy/) and
[Terms](https://blog.tth.dev/wardrobe/terms/) are maintained in `wardrobe/`.
The 12 September 2026 privacy revision explains optional AI provider retention and the
separate aggregate service budget ledger. It does not enable AI or change the iOS app.

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
