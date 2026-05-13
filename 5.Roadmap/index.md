---
title: Roadmap
nav_order: 7
---

# Roadmap

## Implemented

The core AI agent loop and the payment-gateway layer are both shipped:

- **Four-stage pipeline** — Translator → Planner → Executor → Summarizer
- **47 MCP tools** — full Bitcoin Core + Core Lightning control surface
- **Multi-turn history** — follow-up prompts resolve without restating context
- **Goal verification** — automatic read-only confirmation after state-changing actions
- **Web UI** — real-time dashboard with Pipeline tab, Network graph, Logs, Settings
- **SSE streaming** — live pipeline events and LLM token stream
- **Cross-machine Lightning connectivity** — `sys_netinfo` + `ln_node_start` with `bind_host`/`announce_host`
- **Four LLM backends** — OpenAI, Anthropic Claude, Google Gemini, and local Ollama
- **x402 payment-gated endpoints** — HTTP middleware issues BOLT11 invoices on 402 responses; the pipeline executor automatically pays (with a human-approval threshold) and retries.
- **Crash Kit** — one-click debug snapshot from the web UI
- **Persisted smoke-test battery** — 25 scripted prompts covering 42 of 47 tools.

## In flight

- **Battery Phase 3** — LLM-planned multi-step payment flows
- **AI-coverage audit** — tooling that verifies when the deterministic Planner dispatches can be demoted to fallback on stronger LLMs

## Research directions

- **Metrics** — per-stage latency, payment success rate, tool-failure rate, LLM token counts
- **Security review** — authentication, replay protection, and abuse prevention for payment-gated APIs beyond the current session-auth + CSRF + rate-limit stack
- **Multi-machine harness** — scripted setup for running nodes on separate machines with cross-machine peer connectivity
