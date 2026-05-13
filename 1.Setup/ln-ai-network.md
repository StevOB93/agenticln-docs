---
title: Local Lightning Network Harness
parent: Setup
nav_order: 2
---

# Local Lightning Network Harness (ln-ai-network)

The `ln-ai-network/` directory contains the full runnable harness: Bitcoin + Lightning infrastructure, the AI agent pipeline, the MCP tool server, and the web UI. This page covers configuration, starting/stopping, and the directory layout you will see on disk.

For the layer-by-layer tour of how the pieces fit together, see [Architecture](../2.Architecture/). For the by-tool reference, see [Components → Tools](../3.Components/TOOLS.md).

---

## Installation

### Linux / WSL (full setup)

From the repo root, run the one-time installer:

```bash
./install.sh
```

This installs Bitcoin Core, Core Lightning, and sets up the Python virtual environment at `ln-ai-network/.venv/`. The install log is saved to `ln-ai-network/logs/install.log`.

### Windows (Python-only)

Native Windows cannot run the full stack (Core Lightning is POSIX-only), but the Python components — tests, pipeline in mock mode, agent — run on Windows:

```powershell
cd ln-ai-network
.\windows.ps1 -Install
```

To run the full infrastructure from Windows, install [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) and use the Linux instructions above inside a WSL shell.

### Force-reinstall Python dependencies

```bash
REINSTALL_PY_DEPS=1 ./start.sh
```

The starter re-runs `pip install -r requirements.txt` into `.venv/` before booting.

---

## Configuration

Copy the template and edit:

```bash
cp ln-ai-network/.env.example ln-ai-network/.env
```

Key settings (the full list is in `.env.example`):

| Variable | Default | What it controls |
|----------|---------|------------------|
| `ALLOW_LLM` | `0` | `1` to enable LLM calls (the pipeline refuses them otherwise). |
| `LLM_BACKEND` | `openai` | Global LLM backend: `openai`, `claude`, `ollama`, or `gemini`. |
| `OPENAI_API_KEY` | — | Required when `LLM_BACKEND=openai`. |
| `OPENAI_MODEL` | `gpt-4o-mini` | OpenAI model name. |
| `ANTHROPIC_API_KEY` | — | Required when `LLM_BACKEND=claude`. |
| `CLAUDE_MODEL` | latest Claude family | Anthropic Claude model name. |
| `GEMINI_API_KEY` | — | Required when `LLM_BACKEND=gemini`. |
| `OLLAMA_BASE_URL` | `http://127.0.0.1:11434` | Ollama HTTP endpoint. |
| `OLLAMA_MODEL` | `qwen2.5:14b` | Ollama model name. `qwen2.5:7b` on GPU is a fast local baseline. |
| `LLM_TEMPERATURE` | `0` | `0` for deterministic tool-calling, the recommended default. |
| `MCP_CALL_TIMEOUT_S` | `30` | Timeout per MCP tool call (sec). |
| `LIGHTNING_BASE_PORT` | `9735` | Node N's peer port is `9735 + N`. |
| `UI_HOST` | `127.0.0.1` | Web UI bind address. |
| `UI_PORT` | `8008` | Web UI port. |

Per-stage LLM overrides let you route each pipeline stage to its own backend/model:

```bash
TRANSLATOR_LLM_BACKEND=ollama    # Cheap, local model for NL parsing
PLANNER_LLM_BACKEND=openai       # Strong model for the decision-heavy stage
SUMMARIZER_LLM_BACKEND=openai    # Same model as Planner for consistent voice
```

---

## Starting and stopping

```bash
./start.sh          # Start with 2 Lightning nodes (default)
./start.sh 3        # Start with 3 nodes
```

The web UI opens automatically at `http://127.0.0.1:8008`. On WSL2 this uses the Windows-side browser.

```bash
./stop.sh           # Stop everything cleanly
```

The `NODE_COUNT` you started with is persisted to `runtime/node_count` so `./stop.sh` knows how many nodes to shut down without requiring the argument again.

---

## Restart just the agent

If you change pipeline code, restart only the AI agent without touching the Bitcoin/Lightning infrastructure:

```bash
cd ln-ai-network
./scripts/restart_agent.sh          # keep inbox/outbox state
./scripts/restart_agent.sh fresh    # clear queue state (empty inbox.jsonl, reset cursor)
```

This stops the `ai.pipeline` process, starts a new one, and (with `fresh`) wipes `inbox.jsonl` + `inbox.offset` + `msg.counter` so the next prompt is processed against an empty queue. The `lightningd` and `bitcoind` processes keep running throughout.

---

## Directory layout

```
ln-ai-network/
├── ai/                      # AI pipeline
│   ├── pipeline.py          #   Main pipeline entry point and coordinator
│   ├── tools.py             #   MCP tool registry, normalization, schema generation
│   ├── models.py            #   Frozen dataclasses: IntentBlock, ExecutionPlan, StepResult, PipelineResult
│   ├── mcp_client.py        #   JSON-RPC client for talking to the MCP server
│   ├── intent_validate.py   #   Safety gate: rejects malicious-looking intents
│   ├── controllers/         #   translator.py, planner.py, executor.py, summarizer.py, shared.py
│   ├── llm/                 #   LLMBackend protocol + 4 adapters + GuardedBackend
│   ├── core/                #   config, scheduler, registry, rate_limiter, backoff, token_estimation, concurrency
│   └── tests/               #   791 pytest tests, offline by default
├── mcp/                     # MCP tool server
│   ├── ln_mcp_server.py     #   The 47-tool server: only process that runs bitcoin-cli / lightning-cli
│   ├── client/              #   FastMCPClient used by the pipeline
│   └── README.md            #   Wire-format + per-tool developer reference
├── scripts/                 # Boot and operations scripts
│   ├── 0.install.sh         #   One-time install (called by ./install.sh)
│   ├── 1.start.sh           #   Full system launcher (called by ./start.sh)
│   ├── shutdown.sh          #   Graceful shutdown (called by ./stop.sh)
│   ├── restart_agent.sh     #   Agent-only restart
│   ├── ui_server.py         #   HTTP + SSE web server
│   ├── startup/             #   Per-layer boot sub-scripts (infra, control plane, agent, UI)
│   ├── shutdown/            #   Per-layer shutdown sub-scripts (reverse order)
│   └── tools/               #   Developer utilities: run_battery, audit_dispatch_discipline, mine_blocks, pull_ollama_model
├── web/                     # Frontend (HTML, JS, CSS)
│   ├── index.html           #   Single-page app shell
│   ├── app.js               #   UI logic: SSE client, D3 graph, tab router
│   └── styles.css           #   Dark-theme design tokens
├── config/                  # Static templates consumed by boot scripts
│   ├── network.defaults.yml #   Network-scale defaults (managed_nodes, path roots)
│   └── cln/lightning.conf.tpl  # Core Lightning config template rendered per node
├── 402x/                    # x402 payment-gated HTTP middleware
├── docs/                    # Subsystem-level developer docs (API, SECURITY, TESTING, TROUBLESHOOTING, …)
├── .env.example             # Template — copy to .env
└── runtime/                 # Created at runtime (gitignored)
    ├── agent/               #   inbox, outbox, history, trace, lock files
    ├── bitcoin/shared/      #   bitcoind data directory
    └── lightning/node-N/    #   lightningd data (one per node)
```

