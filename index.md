---
title: Home
nav_order: 1
---

# Lightning Network AI Agents

A runnable research harness for AI agents that can understand natural language, call structured tools (MCP), and move value on the Bitcoin Lightning Network — all on a local regtest so no real funds are ever at risk.

## What this project demonstrates

- **Bitcoin base layer** — Bitcoin Core running in `regtest` mode
- **Lightning payments** — one or more Core Lightning nodes with real payment channels between them
- **Tool-boundary execution** — every Bitcoin/Lightning operation is a named MCP tool, not a free-form shell command
- **A four-stage AI pipeline** — natural-language prompt → structured intent → execution plan → tool calls → human answer
- **Payment-gated HTTP APIs** — HTTP 402 (x402) endpoints that issue Lightning invoices and verify payment before serving a response

## Start here

- **Quickstart:** [Run the demo in minutes](0.Quickstart/)
- **Setup:** [Install and configure the environment](1.Setup/)
- **Architecture:** [How the system is put together](2.Architecture/)
- **Components:** [MCP tools, x402, the pipeline stages](3.Components/)
- **Runbooks:** [Start/stop, debugging, smoke-test battery](4.Runbooks/)
- **Roadmap:** [What's shipped and what's next](5.Roadmap/)
- **Team:** [Contributors](team/)

## Repository layout (top level)

- `docs/` — this documentation site
- `ln-ai-network/` — the runnable harness (Python code, boot scripts, web UI)
- `PLAN.md` — a living planning journal kept at the repo root so it's the first thing a new contributor sees
