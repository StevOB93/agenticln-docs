---
title: Architecture
nav_order: 4
has_children: true
---

# Architecture

This page is a tour of how the system is put together. It is written for a reader who has finished an undergraduate computer-science degree but has not necessarily worked with the Bitcoin Lightning Network before. Terms that are specific to the Bitcoin or AI-agent domains are introduced in the [Glossary](#glossary) at the bottom of the page.

For the deep-dive on the data flow and file layout, see [`system-design.md`](system-design.md). For a by-tool reference of the 47 MCP tools, see [`../3.Components/TOOLS.md`](../3.Components/TOOLS.md).

---

## One-sentence summary

A browser-based prompt travels through a **four-stage AI pipeline** that converts it into a list of structured **MCP tool calls**, each of which is executed as an authorised `bitcoin-cli` or `lightning-cli` command against a **local regtest network**; the results are summarised back into a human answer and streamed to the browser.

---

## The five layers

The runtime is organised as a vertical stack of five layers. Each layer has one clear responsibility and talks to only the layer directly above and below it.

```
┌────────────────────────────────────────────────────────────────┐
│ 1. User interface                                              │
│    Browser dashboard (HTML · CSS · JS · D3) + HTTP/SSE server  │
└───────────────────────────────┬────────────────────────────────┘
                                │ JSONL inbox/outbox + SSE
┌───────────────────────────────▼────────────────────────────────┐
│ 2. AI pipeline                                                 │
│    Translator → Planner → Executor → Summarizer                │
│    (4 controllers coordinated by ai/pipeline.py)               │
└───────────────────────────────┬────────────────────────────────┘
                                │ JSON-RPC over stdio
┌───────────────────────────────▼────────────────────────────────┐
│ 3. Tool boundary (MCP server)                                  │
│    47 named tools, each wrapping exactly one CLI subprocess     │
│    (mcp/ln_mcp_server.py)                                      │
└───────────────────────────────┬────────────────────────────────┘
                                │ bitcoin-cli · lightning-cli
┌───────────────────────────────▼────────────────────────────────┐
│ 4. Payment layer                                               │
│    One or more Core Lightning nodes (lightningd processes)      │
└───────────────────────────────┬────────────────────────────────┘
                                │ P2P + RPC
┌───────────────────────────────▼────────────────────────────────┐
│ 5. Base layer                                                  │
│    Bitcoin Core (bitcoind) in regtest mode                     │
└────────────────────────────────────────────────────────────────┘
```

| # | Layer | Component | Why it exists |
|---|-------|-----------|---------------|
| 1 | User interface | `scripts/ui_server.py` + `web/*` | Lets the human submit a prompt and watch the pipeline work in real time. |
| 2 | AI pipeline | `ai/pipeline.py` + `ai/controllers/*` | Decomposes a natural-language request into a sequence of tool calls. |
| 3 | Tool boundary | `mcp/ln_mcp_server.py` | The *only* process allowed to execute shell commands against the infrastructure. |
| 4 | Payment layer | One or more `lightningd` processes under `runtime/lightning/` | Opens channels, issues invoices, routes and settles payments. |
| 5 | Base layer | `bitcoind` under `runtime/bitcoin/` | Provides the isolated regtest blockchain that anchors every channel. |

---

## Why four pipeline stages instead of one

A naive LLM agent would take the user's prompt, decide on a tool to call, call it, look at the result, decide on the next tool, and so on — all inside one unbroken conversation. That works, but it has three problems for a research harness:

1. **Opaque reasoning.** There is no structured intermediate artefact to inspect, replay, or assert against in a test.
2. **Expensive retries.** If the agent makes a wrong move on step 5, the only way to recover is to feed the whole conversation back to the LLM and hope it does better.
3. **No boundary between understanding and action.** You cannot audit what the system *thought the user wanted* separately from *what the system tried to do*.

The four-stage pipeline addresses all three by splitting the work into stages that each produce a typed, serialisable artefact:

| Stage | Input | Output | Uses LLM? |
|-------|-------|--------|-----------|
| **Translator** | Natural-language prompt | `IntentBlock` — goal, type, extracted context, success criteria | Yes |
| **Planner** | `IntentBlock` | `ExecutionPlan` — ordered list of `PlanStep`s with tool calls and dependencies | Yes |
| **Executor** | `ExecutionPlan` | `StepResult[]` — one result per step, including tool output and retries | **No** |
| **Summarizer** | `StepResult[]` + original prompt | Human-readable answer | Yes |

Key properties that follow from the split:

- **The Executor is deterministic.** It takes a plan and executes it with no LLM calls of its own. This means that once the Planner has produced a plan, the system's behaviour is reproducible given the same plan and the same infrastructure state. This is the property that makes the smoke-test battery (see [runbooks](../4.Runbooks/)) meaningful.
- **Every intermediate artefact is a frozen dataclass** defined in `ai/models.py`, which means they serialise cleanly to JSON, survive a process restart, and can be replayed from the outbox for debugging.
- **Goal verification** — after a state-changing step (`ln_pay`, `ln_openchannel`, …), the Executor automatically runs a read-only tool (`ln_listfunds`, `ln_listchannels`) to confirm the change actually happened. A successful tool call is not the same as a successful *goal*; the verification step catches the difference.

---

## Why a separate tool server

The AI pipeline does not call `bitcoin-cli` or `lightning-cli` directly. Every request to the infrastructure goes through the MCP server as a named tool call such as `ln_openchannel(from_node=1, peer_id=…, amount_sat=500000)`.

There are two reasons for this indirection:

1. **Security boundary.** The LLM-driven pipeline can propose *any* shell command. The MCP server exposes a fixed set of 47 tools, each with a validated argument schema. If a plan step names a tool or argument the server does not recognise, it is rejected before any subprocess runs. The passthrough tools (`ln_raw`, `btc_raw`) exist for completeness but enforce their own blocklists so that dangerous commands (`dumpprivkey`, `encryptwallet`, `stop`, …) are refused.
2. **Transport independence.** The pipeline and the MCP server communicate with [JSON-RPC](https://www.jsonrpc.org/specification) over standard input/output. That means the server could be swapped out for a different implementation (gRPC, HTTP, or even a mocked in-memory version for tests) without any changes to the pipeline. The [Model Context Protocol](https://modelcontextprotocol.io/) is the emerging open standard for this kind of tool interface.

---

## Why a local regtest network

Every payment, channel open, and block mined in this project happens on a **regtest** (regression-test) Bitcoin network. Regtest has three properties that make it ideal for a research harness:

1. **No real money.** Regtest coins have zero economic value, so experiments cannot accidentally spend mainnet Bitcoin.
2. **Deterministic blocks.** Blocks are only produced when the software calls `generatetoaddress(n)`, which lets tests control exactly when transactions confirm.
3. **Isolated from the public network.** A regtest node will not peer with mainnet or testnet nodes, so local experiments stay local.

The tradeoff is that nothing that happens on regtest is visible to the outside world; cross-machine experiments require each participant to run their own regtest and connect their Lightning nodes peer-to-peer via the `ln_connect` tool.

---

## Key design decisions, in one line each

- **The agent may only act via MCP tools.** No direct shell access, no direct file writes, no ability to run arbitrary commands. This is the central safety invariant of the system.
- **Every pipeline artefact is a dataclass.** `IntentBlock`, `PlanStep`, `ExecutionPlan`, `StepResult`, `PipelineResult` — all defined in `ai/models.py`, all JSON-serialisable, all frozen where they should be.
- **Prompts arrive as JSON lines, not HTTP requests.** The web UI writes to `runtime/agent/inbox.jsonl` and the pipeline reads from it, decoupling request submission from request processing and making the full history replayable.
- **Multi-turn context is carried by file, not by session.** The last N exchanges are persisted in `runtime/agent/history.jsonl`, which survives agent restarts and makes follow-up prompts like *"now pay that invoice"* work across a reboot.
- **After every state change, verify.** The Executor does not trust a successful tool call; it runs a read-only read-back to confirm the intended state was actually reached.
- **Per-stage LLM configuration.** Each pipeline stage can use a different LLM backend and model, configured by `TRANSLATOR_LLM_BACKEND`, `PLANNER_LLM_BACKEND`, `SUMMARIZER_LLM_BACKEND` env vars. This makes it cheap to experiment with, for example, a small local model for the Translator and a strong cloud model for the Planner.

---

## Glossary

**Bitcoin Core** — the reference implementation of the Bitcoin protocol, distributed as `bitcoind` (the daemon) and `bitcoin-cli` (the RPC client). Maintains the blockchain, wallet, and mempool.

**BOLT11 / BOLT12** — standard string formats for Lightning payment requests. BOLT11 is a one-shot invoice; BOLT12 is a reusable offer from which multiple invoices can be fetched. The prefixes `lnbcrt…` (BOLT11, regtest), `lno1…` (BOLT12 offer), and `lni1…` (BOLT12 invoice) identify which is which.

**Channel** — a two-party payment circuit anchored on-chain by a funding transaction. Once open, the two parties can exchange many off-chain payments instantly; closing the channel settles the final balances back on-chain.

**Core Lightning (CLN)** — Blockstream's implementation of the Lightning Network protocol, distributed as `lightningd` (the daemon) and `lightning-cli` (the RPC client). This project uses CLN exclusively.

**MCP (Model Context Protocol)** — an emerging open standard for how an AI agent calls external tools. The agent sends `{method: "tool_name", params: {…}}`; the server responds with `{ok: bool, payload: …}`. [modelcontextprotocol.io](https://modelcontextprotocol.io/).

**LLM backend** — a pluggable adapter that sends a prompt to a large-language-model service and returns the response. This project ships four: OpenAI, Anthropic Claude, Google Gemini, and local Ollama.

**Regtest** — a Bitcoin network mode that runs entirely on the local machine with manual block mining. See [Why a local regtest network](#why-a-local-regtest-network) above.

**Node** — in this codebase, "node" always means a single Core Lightning instance. Nodes are 1-indexed (node 1, node 2, …). Every node has its own data directory under `runtime/lightning/node-N/`.

**Pipeline** — the four-stage chain of controllers (Translator → Planner → Executor → Summarizer) that turns a prompt into an answer. Coordinated by `ai/pipeline.py`.

**SSE (Server-Sent Events)** — a simple unidirectional streaming protocol layered on HTTP. The web UI connects to `/api/stream` and receives `status`, `pipeline_result`, and `trace` events pushed by the server as they happen.

**x402** — a pattern that gates HTTP endpoints behind Lightning payments by returning an `HTTP 402 Payment Required` response with an embedded BOLT11 invoice; the client pays and retries with a preimage header.
