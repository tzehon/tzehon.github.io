---
layout: post
title:  "Automating README Summaries with LLM CLI and GitHub Actions"
date:   2025-12-27 12:00:00 +0800
categories: llm automation github-actions
---

Maintaining documentation for a monorepo with multiple projects is tedious. Each time you add a project, you need to update the root README, write a summary, and keep everything in sync. I discovered a pattern from [Simon Willison's research repository](https://github.com/simonw/research) that automates this entirely using his [LLM CLI tool](https://github.com/simonw/llm) and [cogapp](https://github.com/nedbat/cog).

## The Problem

My [research repository](https://github.com/tzehon/research) contains multiple independent projects - MongoDB tooling, automation scripts, and various experiments. Every time I add a new project:

1. Write the project's README
2. Update the root README with a summary
3. Keep the summary in sync when the project evolves
4. Manually sort projects by date

This is exactly the kind of repetitive work that should be automated.

## The Solution: Cogapp + LLM

Cogapp is a code generation tool that runs Python snippets embedded in your files. Combined with Simon Willison's LLM CLI, you can generate content on the fly during CI.

The README.md contains embedded Python between `[[[cog` and `]]]` markers:

```python
<!--[[[cog
import subprocess
from pathlib import Path

MODEL = "claude-sonnet-4.5"

def get_summary(folder):
    summary_file = Path(folder) / "_summary.md"
    if summary_file.exists():
        return summary_file.read_text().strip()

    readme_file = Path(folder) / "README.md"
    result = subprocess.run(
        ["llm", "-m", MODEL],
        input=f"Summarize this project concisely...\n\n{readme_file.read_text()}",
        capture_output=True, text=True
    )
    summary = result.stdout.strip()
    if summary:
        summary_file.write_text(summary)  # Cache for next run
    return summary

# ... generate project list sorted by git commit date
]]]-->
```

When cogapp runs, it:
1. Scans for project directories
2. Reads each project's README
3. Calls Claude via the LLM CLI to generate a summary
4. Caches summaries in `_summary.md` files to avoid redundant API calls
5. Outputs a formatted project list sorted by date

## The GitHub Action

The workflow runs on every push to main:

```yaml
name: Update README with cogapp

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  update-readme:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for git dates

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'

      - run: pip install -r requirements.txt

      - name: Run cogapp to update README
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: cog -r -P README.md

      - name: Commit and push if changed
        run: |
          git config --local user.email "github-actions[bot]@users.noreply.github.com"
          git config --local user.name "github-actions[bot]"
          git add README.md */_summary.md
          git diff --staged --quiet || git commit -m "Auto-update README [skip ci]" && git push
```

Key points:
- `fetch-depth: 0` ensures full git history for accurate commit dates
- The `[skip ci]` in the commit message prevents infinite loops
- Summaries are cached, so API calls only happen for new/changed projects

## Why This Works Well

**Self-Updating Documentation**: Push a new project, and the README updates automatically with a proper summary. No manual intervention needed.

**Consistent Formatting**: The LLM generates summaries in a consistent style. I include instructions in the prompt to vary the opening and include relevant links.

**Cost Efficient**: Caching summaries in `_summary.md` files means you only pay for API calls when content changes.

**Chronological Organization**: Projects are automatically sorted by their first commit date, giving readers a timeline of work.

## Setup Requirements

```bash
pip install cogapp llm llm-anthropic
```

Store your API key:
```bash
llm keys set anthropic
# or use ANTHROPIC_API_KEY environment variable
```

Add to your repository secrets for GitHub Actions:
- `ANTHROPIC_API_KEY`: Your Anthropic API key

## Adapting for Your Own Use

The pattern works for any documentation that benefits from LLM-generated content:

- Project summaries (as shown here)
- Changelog entries from git commits
- API documentation from code comments
- Dependency summaries from package files

The key insight is using cogapp as the orchestrator and caching generated content to avoid unnecessary API calls.

Check out [Simon Willison's research repo](https://github.com/simonw/research) for the original implementation, or my [research repo](https://github.com/tzehon/research) for a working example.
