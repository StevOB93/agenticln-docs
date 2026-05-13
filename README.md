# agenticln-docs

Documentation site for **Agentic Payments with Bitcoin on Lightning Network**, a CSCI 499 senior capstone project at East Texas A&M University, Spring 2026.

The project pairs an AI agent pipeline with a local Bitcoin Core + Core Lightning regtest network, mediated by a Model Context Protocol (MCP) tool boundary. This repository contains only the public documentation; the project's source code lives in a separate, private repository.

## Live site

The published site is served by GitHub Pages at:

**https://stevob93.github.io/agenticln-docs/**

## Building locally (optional)

The site is built by GitHub Pages automatically. To preview locally with Jekyll:

```bash
bundle exec jekyll serve
```

You'll need Ruby + Bundler with the `jekyll` and `jekyll-remote-theme` gems installed. The theme is fetched from the [`just-the-docs`](https://github.com/just-the-docs/just-the-docs) project at build time.

## Contents

- `index.md` — landing page
- `0.Quickstart/` — quickstart guide
- `1.Setup/` — environment setup, troubleshooting
- `2.Architecture/` — system overview, data flow
- `3.Components/` — MCP tools reference
- `4.Runbooks/` — operating guides, prompt battery, LLM evaluation
- `5.Roadmap/` — implemented features and planned work
- `agent/` — AI pipeline internals
- `research/` — research note
- `team/` — team and professor
- `assets/` — figures and images
- `_config.yml` — Jekyll configuration
