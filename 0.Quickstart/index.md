---
title: Quickstart
nav_order: 2
---

# Quickstart

Get the Lightning Network AI agent running on your machine in a few minutes, then exercise the full end-to-end Lightning payment flow.

> **Core invariant:** the AI pipeline acts only via MCP tools. The MCP server is the execution boundary between the LLM and the infrastructure. Nothing in the AI layer ever runs `bitcoin-cli` or `lightning-cli` directly.

## What you will end up with

- One **Bitcoin Core** node running in `regtest` mode (an isolated local blockchain with no real money).
- Two **Core Lightning** nodes connected to that regtest, each with its own on-chain wallet.
- An **AI pipeline** that turns your natural-language prompts into structured tool calls and runs them against the nodes.
- A **web dashboard** at `http://127.0.0.1:8008` where you submit prompts and watch the pipeline work.

All of this runs on your local machine. Nothing in this project ever touches real Bitcoin.

## Background concepts

A reader coming from a general CS background may not have worked with the Bitcoin Lightning Network before. Three concepts come up repeatedly:

- **Regtest.** A Bitcoin network mode that runs entirely on the local machine. Blocks are only produced when the software explicitly mines them. No real money is ever involved.
- **Lightning channel.** A two-party payment circuit anchored on-chain by a funding transaction. Once open, the two parties can exchange many off-chain payments instantly. Closing the channel settles the final balances back on-chain.
- **BOLT11 invoice.** A standard string format starting with `lnbcrt…` on regtest (or `lnbc…` on mainnet) that encodes a payment request: the amount, the destination, and a cryptographic payment hash. To pay one, you call `ln_pay(bolt11=…)`; the network finds a route and settles it.

You do not need to be an expert in any of these to follow the walkthrough. Understanding that the project is an *AI harness* over this stack (not a new routing protocol, not a wallet, not a mainnet system) sets expectations correctly.

## Prerequisites

- **Linux or WSL2** (Ubuntu 22.04+ recommended). The full infrastructure setup uses `apt`, bash scripts, and POSIX features (Unix domain sockets) that do not exist on native Windows. A Python-only path exists on Windows for running the test suite and the agent in mock mode.
- **Python 3.10+**
- **`git`**
- An **LLM backend** — either:
  - An API key for [OpenAI](https://platform.openai.com), [Anthropic Claude](https://console.anthropic.com), or [Google Gemini](https://ai.google.dev); or
  - A local [Ollama](https://ollama.com) install with a tool-calling model pulled (`qwen2.5:7b` or `qwen2.5:14b` are the recommended defaults).

## Three steps

### 1 — Install (once)

```bash
./install.sh
```

This downloads Bitcoin Core and Core Lightning binaries, creates a Python virtual environment, and installs every Python dependency. The installer is idempotent — safe to re-run if it fails partway through.

### 2 — Configure your LLM

```bash
cp ln-ai-network/.env.example ln-ai-network/.env
```

Edit `ln-ai-network/.env` and set **`ALLOW_LLM=1`** plus credentials for your chosen backend:

| Backend | Settings |
|---------|----------|
| OpenAI (default) | `OPENAI_API_KEY=sk-…` |
| Anthropic Claude | `LLM_BACKEND=claude` and `ANTHROPIC_API_KEY=…` |
| Google Gemini | `LLM_BACKEND=gemini` and `GEMINI_API_KEY=…` |
| Ollama (local, free) | `LLM_BACKEND=ollama` and `OLLAMA_MODEL=qwen2.5:14b` |

For deterministic tool-calling behaviour during experiments, also set `LLM_TEMPERATURE=0`.

### 3 — Start

```bash
./start.sh          # default: 2 Lightning nodes
./start.sh 3        # or 3 nodes
```

The boot sequence takes 30–60 seconds on a first run and starts every component in order:

1. **Bitcoin Core** in regtest mode (`bitcoind`).
2. **Core Lightning** nodes — one `lightningd` process per node. Each on-chain wallet is funded and channels are opened between them.
3. **MCP tool server**.
4. **AI pipeline**.
5. **Web UI** — auto-opens the browser at `http://127.0.0.1:8008`.

Most of the first-run startup time is spent mining the 101-block coinbase-maturity window plus the 6 blocks needed to confirm channel funding transactions.

Stop the full stack with:

```bash
./stop.sh
```

Shutdown runs in reverse order: agent → MCP server → Lightning nodes → Bitcoin Core → UI server.

## First prompt (smoke test)

Type a prompt into the **Agent** tab and press **Enter**:

```
Check the network health and report the status of all nodes.
```

Watch the **Pipeline** tab — you will see, in order:

- **Translator card** — the parsed `IntentBlock` (goal, intent type, extracted context, success criteria).
- **Planner card** — the `ExecutionPlan`: which tools the planner picked, in what order, with what arguments.
- **Executor card** — step-by-step `StepResult`s: tool called, arguments sent, output received, retries used.
- **Summary card** — the Summarizer's plain-English answer plus a ✓/✗ success indicator.

Stage cards are colour-coded: green = succeeded, red = failed, grey = not yet run.

What happens under the hood:

1. The browser writes your prompt to a file-based inbox queue.
2. The AI pipeline picks it up and runs it through the four stages above.
3. The web UI streams every intermediate artefact back to you over Server-Sent Events.

## End-to-end payment test

Try the full golden path:

```
Have node 2 create an invoice for 10,000 msat, then pay it from node 1. Verify the payment succeeded.
```

The agent decomposes this into roughly:

1. `network_health` — confirm the infrastructure is up.
2. `ln_node_status` + `ln_node_start` — make sure node 2 is running.
3. `ln_getinfo(node=2)` — read node 2's pubkey and binding address.
4. `ln_connect` — peer node 1 to node 2 (no-op if already connected).
5. `btc_wallet_ensure` + `ln_newaddr` × 2 + `btc_sendtoaddress` × 2 + mine 101 blocks — fund both nodes.
6. `ln_openchannel` + mine 6 blocks — open and confirm the channel (no-op if one exists).
7. `ln_invoice(node=2, amount_msat=10000, …)` — create a fresh invoice on node 2.
8. `ln_pay(from_node=1, bolt11=…)` — pay it from node 1.
9. `ln_listfunds(node=2)` — verify node 2's channel balance actually increased. This is the **goal verification** step the Executor runs automatically after a `pay_invoice` intent.

The Summary card shows the final answer and a ✓/✗ verdict.

More things to try:

```
Open a 500,000 sat channel from node 1 to node 2.
```

```
What is the balance of node 1?
```

## Restart just the agent

If you change pipeline code and want the new version without rebooting the infrastructure:

```bash
cd ln-ai-network
./scripts/restart_agent.sh          # keep inbox/outbox state
./scripts/restart_agent.sh fresh    # clear queue state
```

This stops the running pipeline process, starts a new one, and (optionally) wipes the inbox/outbox cursors so the next prompt is processed against an empty queue.

## Debugging a prompt

Three diagnostic files cover almost every failure mode:

```bash
# The most recent pipeline result
tail -n 1 ln-ai-network/runtime/agent/outbox.jsonl

# The full trace of the most recent prompt (resets on every new prompt)
tail -n 120 ln-ai-network/runtime/agent/trace.log

# The agent process startup log
tail -n 50 ln-ai-network/logs/system/0.3.agent_boot.log
```

The **Logs** tab of the web UI exposes the same files and adds a one-click **Copy Crash Kit** button that formats all three into a pasteable report.

For common failure modes (missing API key, wallet not loaded, channel not confirmed, port conflicts) see [Troubleshooting](../1.Setup/TROUBLESHOOTING.md).

## Next

- [Setup guide](../1.Setup/) — manual install, environment variables, troubleshooting
- [Architecture overview](../2.Architecture/) — how the pieces fit together
- [Tools reference](../3.Components/TOOLS.md) — the 47 MCP tools the agent can call
