---
title: Components
nav_order: 5
has_children: true
---

# Components

The system is organised as five layers, each with a single clear responsibility:

| # | Layer | Component | Status |
|---|-------|-----------|--------|
| 1 | Bitcoin base | Bitcoin Core (`bitcoind`) | Running |
| 2 | Payment layer | Core Lightning (`lightningd` × N) | Running |
| 3 | Tool boundary | MCP server (`mcp/ln_mcp_server.py`) | Running |
| 4 | Decision layer | AI pipeline (`ai/pipeline.py`) | Running |
| 5 | Payment-gated APIs | x402 middleware + auto-pay executor | Running |

## Bitcoin Core

Provides the regtest blockchain. All Lightning nodes connect to the same `bitcoind` instance for block data and on-chain wallet operations. The MCP tools whose names start with `btc_*` wrap `bitcoin-cli` calls.

## Core Lightning

Each Lightning node is a separate `lightningd` process with its own data directory under `runtime/lightning/node-N/`. The MCP tools whose names start with `ln_*` wrap `lightning-cli` calls targeted at a specific node.

## MCP Tool Server

The execution boundary between the AI agent and the infrastructure. Exposes **47 tools** over JSON-RPC (stdio). See the [Tools reference](TOOLS.md) for the complete tool list and per-category reference.

## AI Pipeline

A four-stage processing loop:

```
[Translator] → IntentBlock → [Planner] → Plan → [Executor] → Results → [Summarizer] → Answer
```

All agent behaviour is constrained to MCP tool calls — the pipeline has no direct shell or filesystem access to the Bitcoin/Lightning processes.

## x402 — HTTP 402 Payment Required

Payment-gated HTTP endpoints. A server returns `402 Payment Required` with a BOLT11 Lightning invoice; the AI pipeline's executor pays it (auto-pay below a configurable approval threshold, or with an interactive approval prompt above the threshold) and retries with proof of payment in an `X-Payment-Preimage` header.


---

- [MCP Tools reference](TOOLS.md)
