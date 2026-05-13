---
title: Troubleshooting
parent: Setup
nav_order: 3
---

# Troubleshooting

If something goes wrong, start by collecting the three diagnostic files below. The Summary card in the web UI will usually also show a specific error message pointing at the right section of this page.

```bash
tail -n 1   ln-ai-network/runtime/agent/outbox.jsonl
tail -n 120 ln-ai-network/runtime/agent/trace.log
tail -n 120 ln-ai-network/logs/system/0.3.agent_boot.log
```

The **Copy Crash Kit** button on the Logs tab of the web UI packages all three plus non-sensitive config into a clipboard-ready report.

---

## Common failures

### 1. `OPENAI_API_KEY not set` / LLM credentials error

The pipeline validates LLM credentials at startup and exits cleanly if they are missing or still set to the placeholder value in `.env.example`.

**Fix:**

```bash
cp ln-ai-network/.env.example ln-ai-network/.env
# Edit .env: set a real API key + ALLOW_LLM=1
./start.sh
```

If you changed `.env` without restarting, note that most pipeline env vars are only read on startup. Either restart the stack or send a `SIGHUP` to the agent to trigger a live config reload.

---

### 2. Web UI doesn't open automatically

Navigate to `http://127.0.0.1:8008` manually. On WSL2, Windows' `localhost` routes to the WSL instance automatically; if that's not working, check `scripts/0.4.ui_server.sh`'s log:

```bash
tail -n 50 ln-ai-network/logs/system/0.4.ui_server.log
```

Suppress the auto-open behaviour entirely with `LN_AI_NO_BROWSER=1`.

---

### 3. Agent not responding to prompts

The agent process may not be running, or its single-instance lock file may be stale.

**Check:**

```bash
cat  ln-ai-network/runtime/agent/pipeline.lock    # should show pid=NNNN
ps aux | grep "python -m ai.pipeline" | grep -v grep
tail -n 50 ln-ai-network/logs/system/0.3.agent_boot.log
```

**Fix:**

```bash
cd ln-ai-network
./scripts/restart_agent.sh fresh
```

`fresh` also wipes the inbox queue state so the next prompt is processed against an empty queue.

---

### 4. "exceeded max steps" — agent loops without completing

The LLM is producing tool-call loops without making progress (often oscillating between the same two read-only tools).

**Check `trace.log` for:**
- Repeated `planner_llm_dispatch` events with the same tool choice.
- The Executor's oscillation-detector emitting `tool_call_blocked_oscillation`.

**Fix:**
- Set `LLM_TEMPERATURE=0` in `.env` to eliminate sampling variance.
- Try a stronger backend for the Planner stage: `PLANNER_LLM_BACKEND=openai`.
- Restart the agent: `./scripts/restart_agent.sh`.

---

### 5. Tool error: `Missing required param: 'node'`

The LLM called a tool with a malformed argument shape — usually wrapping the real args under a `"args": {…}` key.

The pipeline normalises arguments before dispatch (it unwraps nested args, coerces `"1"` → `1`, and validates `node` against `runtime/node_count`), so most malformed plans are fixed silently. If the trace shows `tool_args_invalid`, the normaliser could not recover, and you should expect the step to retry or abort according to its `on_error` policy.

---

### 6. Bitcoin Core error `-19 Wallet file not specified`

```
error code: -19
error message: Wallet file not specified
```

The `shared-wallet` Bitcoin wallet isn't loaded. The MCP tool `btc_wallet_ensure` handles this automatically:

> "Make sure the shared-wallet wallet exists."

Or load it manually:

```bash
bitcoin-cli -regtest -rpcport=18443 -rpcuser=lnrpc -rpcpassword=lnrpcpass loadwallet shared-wallet
```

---

### 7. `Connection refused` from `lightning-cli`

A Lightning node is not running or its RPC socket is not yet ready.

**Fix path:**

1. Ask the agent "what is the status of node 2?" — it will call `ln_node_status(node=2)` and `ln_node_start(node=2)` if needed.
2. Or check directly:

   ```bash
   lightning-cli --lightning-dir=ln-ai-network/runtime/lightning/node-2 \
                 --network=regtest getinfo
   ```

Node start polls the RPC socket for up to 30 s by default (`MCP_NODE_START_TIMEOUT_S`).

---

### 8. No peers / no route / payment fails

Nodes are not connected, or no confirmed channel exists between them.

**Step through manually** (or just ask the agent to do it):

1. `ln_getinfo(node=2)` → read pubkey + binding address.
2. `ln_connect(from_node=1, peer_id=…, host=…, port=…)` — peer them.
3. `ln_listpeers(node=1)` — verify the peer connected.
4. Fund both nodes on-chain and mine 101 blocks (`btc_generatetoaddress blocks=101 …`).
5. `ln_openchannel(from_node=1, peer_id=…, amount_sat=500000)` — open, then mine 6 blocks to confirm.
6. Verify: `ln_listchannels(node=1)` → look for `"state": "CHANNELD_NORMAL"`.

---

### 9. Channel pending / not active

The channel funding transaction has not confirmed yet. CLN needs at least `minimum_depth` confirmations (6 by default on regtest).

**Fix:**

Ask the agent to mine blocks:

> "Mine 6 blocks."

Or call the tool directly:

```
btc_getnewaddress → btc_generatetoaddress(blocks=6, address=$step1.payload)
```

Then re-check status with `ln_listchannels(node=1)` — look for state `CHANNELD_NORMAL`.

---

### 10. `bitcoind` or `lightningd` not found

```bash
./install.sh
```

Runs the one-time installer to download and install both binaries. See [Bitcoin Core + Core Lightning](core-lightning.md) for what this does under the hood and the manual-install fallbacks.

---

### 11. Port conflicts

If another process is already bound to a port the harness wants to use, override the default in `ln-ai-network/.env`:

```bash
BITCOIN_RPC_PORT=18443
BITCOIN_P2P_PORT=18444
LIGHTNING_BASE_PORT=9735
UI_PORT=8008
```

Restart the stack: `./stop.sh && ./start.sh`.

---

## Saving a trace for debugging

Trace logs reset on every new prompt. To preserve a run:

```bash
mkdir -p ln-ai-network/runtime/agent/archive
cp ln-ai-network/runtime/agent/trace.log \
   ln-ai-network/runtime/agent/archive/trace.$(date +%Y%m%d_%H%M%S).log
```

The Logs tab in the web UI also has an **Archive** panel showing past pipeline runs. Every completed run is automatically archived under `logs/pipeline/` by the agent shutdown script (opt out with `SHUTDOWN_ARCHIVE=0`).

---

## Collecting a bug report

Paste the output of:

```bash
tail -n 1   ln-ai-network/runtime/agent/outbox.jsonl
tail -n 160 ln-ai-network/runtime/agent/trace.log
tail -n 120 ln-ai-network/logs/system/0.3.agent_boot.log
```

Or use the **Copy Crash Kit** button in the web UI's Logs tab — it formats the same files plus non-sensitive configuration into a ready-to-paste report.
