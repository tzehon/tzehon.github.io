# blog.tth.dev

Personal technical blog covering cloud infrastructure, DevOps, and modern development practices. Built with Jekyll and hosted on GitHub Pages.

## Topics

- Google Cloud Platform (GKE, Cloud Run, Cloud Functions)
- Service mesh (Istio)
- Infrastructure automation
- Development environments (DevContainers)
- Databases and AI/ML (MongoDB, Vector Search, LLMs)

## Local Development

### Prerequisites

- Ruby 2.7.6+
- Bundler

### Setup

```bash
bundle install
bundle exec jekyll serve
```

Visit http://localhost:4000

### Using DevContainers

Open the repository in VS Code with the Dev Containers extension, or use GitHub Codespaces/Gitpod for browser-based development.

## Writing Posts

Create a new file in `_posts/` with the naming convention:

```
YYYY-MM-DD-title.markdown
```

Include front matter at the top:

```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM:SS +0800
categories: category1 category2
---
```

Push to `main` to deploy automatically via GitHub Pages.

## License

Content is copyright the author. Code snippets in blog posts are available under MIT unless otherwise noted.
