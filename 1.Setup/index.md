---
title: Setup
nav_order: 3
has_children: true
---

# Setup

Everything needed to get the project running from scratch, along with the reasoning for each step.

Start at the [Quickstart](../0.Quickstart/) for the shortest path. This section goes deeper into what each step is doing and why.

## Order of operations

The system bootstraps in three layers, from infrastructure up to the browser. You set them up in the same order:

1. **Install prerequisites** — OS packages, Bitcoin Core, Core Lightning, Python.
2. **Configure credentials** — LLM API key or a local Ollama install, copied into `.env`.
3. **Start the harness** — one command brings up Bitcoin, Lightning, the AI pipeline, and the web UI in sequence.

The top-level `./install.sh` handles step 1 on WSL2 / Linux (Ubuntu 22.04+) automatically. Step 2 is manual (a copy-and-edit of `.env.example`). Step 3 is `./start.sh`.

## Pages

- [Bitcoin Core + Core Lightning](core-lightning.md) — what each daemon does in this project, how the installer provisions them, manual install steps.
- [Local Harness (ln-ai-network)](ln-ai-network.md) — how to configure `.env`, start the Python environment, and restart just the agent after code changes.
- [Troubleshooting](TROUBLESHOOTING.md) — common failures with copy-pasteable fixes.

## Minimum requirements

| Requirement | Version |
|-------------|---------|
| OS | Linux or WSL2 (Ubuntu 22.04+ recommended) |
| Python | 3.10+ |
| Bitcoin Core | 25.0+ |
| Core Lightning | 23.11+ |
| RAM | 2 GB free (regtest is lightweight) |
| Disk | ~1 GB (binaries + runtime data) |

## Why these specific versions

- **Python 3.10+** — the codebase uses PEP 604 union syntax (`int | str`) and PEP 622 structural pattern matching in a few places. Older Python will fail at import time.
- **Bitcoin Core 25.0+** — required for descriptor-wallets-by-default (`btc_wallet_ensure` uses the descriptor wallet API, which became the default in 25.0).
- **Core Lightning 23.11+** — the project relies on `listpeerchannels` for channel state and BOLT12 `offer`/`fetchinvoice` for reusable payment requests. Both were stable by 23.11.

## Environment variables

All runtime configuration lives in `ln-ai-network/.env`. Copy the template to start:

```bash
cp ln-ai-network/.env.example ln-ai-network/.env
```

### Minimum required settings

```bash
# Enable LLM calls (the pipeline refuses to call any LLM when this is 0)
ALLOW_LLM=1

# Pick a backend
LLM_BACKEND=openai

# Credentials for the chosen backend
OPENAI_API_KEY=sk-...
```

Supported backends and their credential keys:

| `LLM_BACKEND` | Credential env var | Notes |
|---------------|--------------------|-------|
| `openai` | `OPENAI_API_KEY` | Default. `OPENAI_MODEL` defaults to `gpt-4o-mini`. |
| `claude` | `ANTHROPIC_API_KEY` | `CLAUDE_MODEL` defaults to the latest Claude model family. |
| `gemini` | `GEMINI_API_KEY` | `GEMINI_MODEL` defaults to `gemini-2.5-flash`. |
| `ollama` | — (local) | Requires Ollama running and a tool-calling model pulled (`OLLAMA_MODEL=qwen2.5:14b` recommended). |

### Recommended tuning

```bash
# Deterministic, reproducible tool calls
LLM_TEMPERATURE=0

# Each LLM role can override the global backend
# (useful for running Translator on a cheap model and Planner on a strong one)
# TRANSLATOR_LLM_BACKEND=ollama
# PLANNER_LLM_BACKEND=openai
# SUMMARIZER_LLM_BACKEND=openai
```

### Infrastructure ports (only change if you have a conflict)

```bash
BITCOIN_RPC_PORT=18443
BITCOIN_P2P_PORT=18444
LIGHTNING_BASE_PORT=9735   # node N binds to 9735 + N
UI_HOST=127.0.0.1
UI_PORT=8008
```

See `.env.example` for the complete annotated list. Every variable has a sensible default so the minimum-viable `.env` is only a few lines.
