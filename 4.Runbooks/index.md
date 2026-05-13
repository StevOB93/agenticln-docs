---
title: Runbooks
nav_order: 6
has_children: true
---

# Runbooks

Practical guides for operating and debugging the system.

## Start / stop

```bash
# Start with 2 Lightning nodes (default)
./start.sh

# Start with 3 nodes
./start.sh 3

# Stop everything cleanly
./stop.sh
```

`NODE_COUNT` is persisted to `ln-ai-network/runtime/node_count` so `stop.sh` always shuts down the right number of nodes without needing the argument again.

## Restart the agent only

Use this when you change AI pipeline code without touching the Bitcoin/Lightning infrastructure:

```bash
cd ln-ai-network

# Keep inbox/outbox state
./scripts/restart_agent.sh

# Clear queue state (fresh start)
./scripts/restart_agent.sh fresh
```

## Web UI restart / shutdown

The status bar in the web UI has two buttons:

- **↺ Restart** — stops everything and starts it again (runs `stop.sh && start.sh` in the background). The page briefly disconnects and reconnects.
- **⏻ Shutdown** — stops everything and exits. The UI goes offline.

Both buttons require confirmation.

## Check system health

Use the **Health** button in the web UI (Agent tab), or send a prompt:

> "Check the network health and tell me the status of all nodes."

The agent calls `network_health()` and reports the result.

## Run an end-to-end payment test

Ask the agent:

> "Have node 2 create an invoice for 10,000 msat and pay it from node 1. Verify the payment succeeded."

The agent will call `ln_getinfo`, `ln_connect`, fund both nodes, `ln_openchannel`, `ln_invoice`, `ln_pay`, and `ln_listfunds` — the full golden path.

## Make a node reachable from another machine

Ask the agent:

> "Make node 1 reachable from another machine on the LAN."

The agent will:
1. Call `sys_netinfo()` to detect the machine's LAN IP
2. Stop node 1: `ln_node_stop(node=1)`
3. Restart with routable binding: `ln_node_start(node=1, bind_host="0.0.0.0", announce_host=<detected_ip>)`
4. Return the node's pubkey and address for the remote operator to connect to

## Logs

| File | Contents |
|------|---------|
| `ln-ai-network/runtime/agent/trace.log` | Per-prompt trace (resets each request) |
| `ln-ai-network/runtime/agent/outbox.jsonl` | All pipeline results |
| `ln-ai-network/logs/system/0.3.agent_boot.log` | Pipeline process startup log |
| `ln-ai-network/logs/system/0.4.ui_server.log` | Web UI server log |
| `ln-ai-network/logs/system/0.1.infra_boot.log` | Infrastructure boot log |
| `ln-ai-network/logs/system/shutdown.log` | Shutdown log |

The Logs tab in the web UI shows a live trace stream and an archive panel for past pipeline runs.

## Save a trace before it resets

Trace logs reset on every new prompt. To preserve a run:

```bash
mkdir -p ln-ai-network/runtime/agent/archive
cp ln-ai-network/runtime/agent/trace.log \
   ln-ai-network/runtime/agent/archive/trace.$(date +%Y%m%d_%H%M%S).log
```

## Collect a debug snapshot (Crash Kit)

In the web UI, go to the **Logs tab** and click **Copy Crash Kit**. This copies a formatted plain-text report to your clipboard containing:

- System info and configuration (non-sensitive)
- Runtime status (lock file, last request ID, queue counts)
- Last pipeline result
- Recent trace events
- Metrics

Paste this into a bug report or debugging session.

## Smoke-test prompts

A reference set of prompts you can copy-paste into the web UI (or drive via `/api/ask`) to exercise each part of the pipeline. They're grouped by what they exercise. You don't need to run all of them; pick whichever cover the code paths you're touching.

For the issue/solution history of past runs, see [Prompt Battery Archive](./prompt-battery.md).

### Infrastructure & health

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 1 | `Check the network health and report the status of all nodes.` | `network_health` — multi-field diagnostic (chain + per-node status) |
| 2 | `What is the current Bitcoin block height?` | `btc_getblockchaininfo` — single-field bitcoin read |
| 3 | `Show me the info for node 1 (its pubkey and version).` | `ln_getinfo` — per-node CLN metadata |

### Read-only Lightning queries

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 4 | `List all funds (on-chain and channels) for node 1.` | `ln_listfunds` — on-chain UTXOs + channel balances |
| 5 | `How many channels does node 1 currently have?` | `ln_listchannels` / `ln_listfunds` — channel enumeration |
| 6 | `List all peers connected to node 1.` | `ln_listpeers` — peer list + connection state |
| 7 | `Show me node 1's recent payments.` | `ln_listpays` — payment history |
| 8 | `What invoices has node 2 issued?` | `ln_listinvoices` — invoice history |

### State-changing Lightning ops

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 9 | `Start node 2.` | `ln_node_start` (+ `ln_node_status` pre-check) |
| 10 | `Have node 2 create an invoice for 5000 msat with the label 'smoke_test_<N>'.` | `ln_invoice` — change the `<N>` each run to avoid "Duplicate label" errors |
| 11 | `Open a channel from node 1 to node 2 with 100000 sats.` | `ln_getinfo` → `ln_connect` → `ln_openchannel` → `btc_getnewaddress` → `btc_generatetoaddress` (6 blocks to confirm) |
| 12 | `Close the channel from node 1 to node 2.` | `ln_closechannel` + mining to confirm |

### Bitcoin / regtest control

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 13 | `Mine 3 blocks to a new address from the shared bitcoin wallet.` | `btc_getnewaddress` → `btc_generatetoaddress` — the placeholder-chained two-step pattern (`$step1.result.payload`) |
| 14 | `What is the wallet balance of the shared bitcoin wallet?` | `btc_getbalance` — Bitcoin Core wallet read |

### End-to-end golden path

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 15 | `Have node 2 create an invoice for 10000 msat and pay it from node 1. Verify the payment succeeded.` | `ln_invoice` → `ln_pay` (full payment round-trip, requires an open channel) |
| 16 | `Open a channel from node 1 to node 2 with 500000 sats, then send 10000 msat from node 1 to node 2.` | Full connect → open → mine → invoice → pay chain |

### Intent discrimination (info vs. action)

These confirm the translator and planner route info-requests and explanation-requests correctly without executing actions.

| # | Prompt | Expected behavior |
|---|--------|-------------------|
| 17 | `Decode this scenario: how would I open a channel from node 1 to node 2 with 100000 sats?` | Translator → `intent_type=informational`; planner returns an empty plan; summarizer produces a conditional-tense walkthrough. **No channels are opened.** |
| 18 | `Walk me through the process of paying a BOLT11 invoice.` | Same — explanation only, no `ln_pay` call |
| 19 | `Explain how I would close a channel on node 1.` | Same — explanation only, no `ln_closechannel` call |

### Memory / recall

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 20 | `What did I run in the last few prompts?` | `memory_lookup` — pipeline history recall |
| 21 | `Did the last payment succeed?` | `memory_lookup` with `outcome` filter |

### Cross-machine connectivity (advanced)

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 22 | `Make node 1 reachable from another machine on the LAN.` | `sys_netinfo` → `ln_node_stop` → `ln_node_start` with `bind_host=0.0.0.0` + `announce_host=<LAN IP>` → `ln_getinfo` for the pubkey |

### Expanded coverage (Initiative 1 Phase 1 + Phase 2)

These exercise tools added to the battery in the 2026-04 expansion. Most are one-shot deterministic reads driven by Translator `read_target` tags; the last three are the Phase 2 state-change additions.

> Note: the numbers below are runbook reference IDs and do NOT align with battery prompt IDs (the battery uses its own 1–25 sequence). Use the battery driver flags (`--phase`, `--only`) to run them automatically.

| # | Prompt | What it exercises |
|---|--------|-------------------|
| 23 | `What fee should I use for the next 6 blocks?` | `btc_estimatefee` — `read_target=fee_estimate` with `conf_target=6` |
| 24 | `Show me the current mempool size.` | `btc_getmempoolinfo` — `read_target=mempool_info` |
| 25 | `What are node 1's lightning fee rates?` | `ln_feerates` — `read_target=ln_fee_rates` |
| 26 | `Give me a fresh lightning deposit address for node 1.` | `ln_newaddr` — `read_target=ln_address` |
| 27 | `List all forwarded HTLCs on node 1.` | `ln_listforwards` — `read_target=forward_list` (counter-example for payment_list) |
| 28 | `Show me the recent on-chain transactions in the shared wallet.` | `btc_listtransactions` — `read_target=btc_tx_list` |
| 29 | `List the lightning transactions on node 1.` | `ln_listtransactions` — `read_target=ln_tx_list` (counter-example for fund_list) |
| 30 | `What BOLT12 offers does node 1 have?` | `ln_listoffers` — `read_target=offer_list` (counter-example for invoice_list) |
| 31 | `Stop node 2.` | `ln_node_stop` — `action_target=stop_node` |
| 32 | `Restart node 2.` | `ln_node_stop` → `ln_node_start` chain — `action_target=restart_node` (leaves node 2 running) |
| 33 | `Have node 1 sign the message 'runbook-sig-<TS>'.` | `ln_signmessage` — `action_target=sign_message`. Use a unique `<TS>` so each run produces a distinct signature in the trace. |

### How to automate a battery

A persisted driver lives at `ln-ai-network/scripts/tools/run_battery.py`. It enqueues prompts via the file-based inbox (not HTTP), polls the outbox for a `pipeline_report` with the matching `request_id`, and scores each response against an `expected_tools` / `forbidden_tools` pair. A one-shot retry cushions single-sample LLM flakiness.

```bash
# Boot the stack first
./scripts/1.start.sh 2

# Run the full battery (currently 25 prompts: 10 baseline + 12 Phase 1 + 3 Phase 2)
python ln-ai-network/scripts/tools/run_battery.py

# Or run a phase slice
python ln-ai-network/scripts/tools/run_battery.py --phase 0   # prompts 1–10 (baseline)
python ln-ai-network/scripts/tools/run_battery.py --phase 1   # prompts 11–22 (Phase 1 reads)
python ln-ai-network/scripts/tools/run_battery.py --phase 2   # prompts 23–25 (Phase 2 state-change)

# Or cherry-pick
python ln-ai-network/scripts/tools/run_battery.py --only 14,18,25

# Per-prompt timeout in seconds (default 120)
python ln-ai-network/scripts/tools/run_battery.py --timeout 90

# Audit mode — forces PLANNER_FORCE_LLM=1 so deterministic dispatches are
# skipped and every freeform request goes through the Planner LLM
python ln-ai-network/scripts/tools/run_battery.py --force-llm
```

Exit code is the count of hard failures (PASS + RETRY both count as 0). Verdicts print live as each prompt runs:

```
  [# 1] ✓ PASS    5.2s  'Check the network health and report the status...'
  [# 5] ~ RETRY   9.8s  'How many channels does node 1 currently have?'
  [#14] ✗ FAIL    3.1s  'List all forwarded HTLCs on node 1.'
        → expected one of ['ln_listforwards'], got ['ln_listpays']
```

For each prompt, the driver reads from the outbox entry:
- `success` (bool), `stage_failed` (translator/planner/executor/null)
- `plan.steps[*].tool` — the tools the Planner picked (verdict key)
- `step_results[*].ok` — executor-level success per step
- `content` / `human_summary` — the Summarizer's user-facing answer

A response is **correct** when (a) the Planner picked at least one tool from the `expected_tools` set and no `forbidden_tools` fired, (b) every executed step reports `ok=True` (or is intentionally `skipped`), and (c) the summary only mentions values actually present in the step results (no hallucinated balances or counts).

If a prompt fails, capture `ln-ai-network/runtime/agent/trace.log` before the next one — the trace resets per request.

## Troubleshooting

See [Troubleshooting](../1.Setup/TROUBLESHOOTING.md) for common failure modes and fixes.
