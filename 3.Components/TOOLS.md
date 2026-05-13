---
title: MCP Tools Reference
parent: Components
nav_order: 1
---

# MCP Tools Reference

The MCP server at `mcp/ln_mcp_server.py` exposes **47 tools** across 11 categories. The AI pipeline calls these tools through `ai/mcp_client.py` over JSON-RPC on stdio. The pipeline has no other mechanism to interact with Bitcoin Core or Core Lightning — every action goes through one of the tools on this page.

This document is the **user-facing** tool reference: it explains what each tool does, when to use it, and what it returns.

---

## Conventions

**Node indexing is 1-based.** Use `node=1`, `node=2`, `from_node=1`. Never `node=0`.

**Every tool returns a result envelope:**

```json
{ "ok": true,  "payload": { ... } }
{ "ok": false, "error":   "human-readable message", "payload": null }
```

The pipeline's Executor reads `ok` to decide whether a step succeeded and uses `payload` for placeholder chaining — e.g. `$step1.result.payload.bolt11` resolves to the `bolt11` field of the first step's payload.

**Arguments must be at the top level.** Pass them directly, not wrapped under an `args` key:

```json
// Correct
{"node": 1}

// Wrong — the pipeline's normalizer will unwrap this, but correct shape is preferred
{"tool": "ln_getinfo", "args": {"node": "1"}}
```

The pipeline normalises arguments before dispatch: it unwraps nested `{"args": …}`, coerces string integers (`"1"` → `1`), and validates node numbers against `runtime/node_count`. Even so, a correctly-shaped plan is easier to debug.

---

## 1. System / health (3 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `network_health` | — | — | Aggregate health check: queries `bitcoind` plus every Lightning node. Returns `"ok"`, `"degraded"`, or `"down"` with per-node detail. |
| `sys_netinfo` | — | — | Hostname, default outbound IP, and every non-loopback interface IP of the host machine. Use before `ln_node_start` when setting `bind_host` / `announce_host` for cross-machine peers. |
| `memory_lookup` | — | `query`, `last_n`, `outcome` | Queries the episodic archive (`runtime/agent/archive.jsonl`). `query` filters by keyword in the prompt/goal; `outcome` filters by `"ok"`, `"partial"`, or `"failed"`. Returns up to `last_n` entries (default 5). |

---

## 2. Bitcoin Core — read operations (6 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `btc_getblockchaininfo` | — | — | Chain info: network name, current height, sync progress, chainwork. |
| `btc_getbalance` | — | `wallet` | Confirmed balance (BTC) for the named wallet (default `shared-wallet`). |
| `btc_listtransactions` | — | `wallet`, `count` | Most recent `count` (default 10) wallet transactions: txid, amount, confirmations, category. |
| `btc_gettransaction` | `txid` | `wallet` | Full detail for a specific transaction: amount, fee, confirmations, block hash, raw hex. |
| `btc_getmempoolinfo` | — | — | Mempool statistics: pending transaction count, byte size, max size. |
| `btc_estimatefee` | — | `conf_target` | Fee-rate estimate (BTC/kvB) for confirmation within `conf_target` blocks (default 6). On regtest this returns `-1` until enough blocks have been mined to seed the estimator. |

---

## 3. Bitcoin Core — write operations (4 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `btc_wallet_ensure` | `wallet_name` | — | Creates or loads a Bitcoin wallet. Idempotent — safe to call repeatedly. |
| `btc_getnewaddress` | — | `wallet` | Generates a fresh bech32m address in the named wallet (default `shared-wallet`). |
| `btc_sendtoaddress` | `address`, `amount_btc` | `wallet` | Sends `amount_btc` (decimal string like `"0.001"`) from the named wallet. Returns the txid. |
| `btc_generatetoaddress` | `blocks`, `address` | — | Mines `blocks` regtest blocks with the coinbase going to `address`. Capped at 2,016 blocks per call for safety. |

---

## 4. Lightning node lifecycle (6 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `ln_listnodes` | — | — | Lists every configured Lightning node plus whether it's currently running (checks for socket file + `getinfo` reachability). |
| `ln_node_create` | `node` | — | Initialises the data directory and config file for node N. Does not start `lightningd`. |
| `ln_node_status` | `node` | — | Detailed status: process running, RPC reachable, data directory present, reason if not. |
| `ln_node_start` | `node` | `bind_host`, `announce_host` | Starts `lightningd` for node N. Polls for RPC readiness up to 30 s. Set `bind_host="0.0.0.0"` and `announce_host=<LAN IP>` for cross-machine peers. |
| `ln_node_stop` | `node` | — | Stops `lightningd` for node N via `lightning-cli stop`. Polls for process exit up to 30 s. |
| `ln_node_delete` | `node` | `force` | Removes the data directory for node N. Node must be stopped first unless `force=true`. **Irreversible.** |

---

## 5. Lightning — read operations (12 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `ln_getinfo` | `node` | — | Node identity: pubkey (`payload.id`), alias, color, block height, peers, channels. Auto-starts the node if not running. |
| `ln_listpeers` | `node` | — | Connected peers: pubkey, connection state, associated channels. |
| `ln_listfunds` | `node` | — | On-chain UTXOs (`payload.outputs`) and channel balances (`payload.channels`). |
| `ln_listchannels` | `node` | — | Local peer channels with state, capacity, and balances. Uses CLN's `listpeerchannels` (not `listchannels`, which returns gossip-only data). |
| `ln_newaddr` | `node` | — | New on-chain address for the node's internal wallet. The server adds a convenience `payload.address` field. |
| `ln_listforwards` | `node` | `status` | Forwarded payments. Optional `status`: `offered`, `settled`, `failed`, `local_failed`. |
| `ln_feerates` | `node` | `style` | On-chain fee-rate estimates. `style`: `"perkb"` or `"perkw"` (default). |
| `ln_listpays` | `node` | `bolt11`, `status` | Outgoing payments. Optional filter by `bolt11` string or `status` (`complete`, `pending`, `failed`). |
| `ln_decodepay` | `node`, `string` | — | Decodes a BOLT11 or BOLT12 string: payment hash, amount, description, expiry, route hints. |
| `ln_waitanyinvoice` | `node` | `lastpay_index` | Blocks until any invoice on the node is paid (up to 2× timeout). |
| `ln_listoffers` | `node` | — | All BOLT12 offers on the node. |
| `ln_listtransactions` | `node` | — | On-chain transactions for the Lightning node's internal wallet (wallet receives + channel closes). |

---

## 6. Lightning — actions (4 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `ln_connect` | `from_node`, `peer_id`, `host`, `port` | — | Opens a TCP peer connection from `from_node` to `peer_id@host:port`. Must succeed before `ln_openchannel`. |
| `ln_openchannel` | `from_node`, `peer_id`, `amount_sat` | — | Funds a new channel. Amount is clamped to a minimum of 100,000 sat. Peer must already be connected and the node must have enough on-chain balance. Returns the funding txid; call `btc_generatetoaddress` 6 times to confirm. |
| `ln_closechannel` | `from_node`, `peer_id` | `unilateral_timeout` | Cooperative close by default; force-closes after `unilateral_timeout` seconds if the peer is unresponsive. |
| `ln_setchannel` | `node`, `peer_id` | `feebase`, `feeppm`, `htlcmin`, `htlcmax` | Sets the channel's forwarding fee policy and HTLC limits. `feebase` is base fee in msat; `feeppm` is the proportional fee in parts-per-million. |

---

## 7. Payments (5 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `ln_invoice` | `node`, `amount_msat`, `label`, `description` | — | Creates a BOLT11 payment request. `label` must be unique on the node — CLN rejects duplicates. Returns `payload.bolt11`. |
| `ln_pay` | `from_node`, `bolt11` | `maxfee`, `retry_for` | Pays a BOLT11 invoice (or a BOLT12 invoice that starts with `lni1`). `maxfee` caps routing fee in msat. Returns the payment preimage on success. |
| `ln_keysend` | `from_node`, `destination`, `amount_msat` | — | Spontaneous payment — no invoice required. `destination` is the recipient pubkey. |
| `ln_listinvoices` | `node` | `label` | Lists invoices on the node, optionally filtered by `label`. |
| `ln_waitinvoice` | `node`, `label` | — | Blocks until the invoice identified by `label` is paid (up to 2× timeout). |

---

## 8. BOLT12 offers (2 tools)

BOLT12 offers are reusable payment requests — one offer can be paid many times. This is different from BOLT11, where each invoice is single-use.

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `ln_offer` | `node`, `amount_msat`, `description` | — | Creates a reusable BOLT12 offer. `amount_msat` can be an integer or `"any"` for any-amount offers. Returns `payload.offer` (an `lno1…` string). |
| `ln_fetchinvoice` | `node`, `offer` | `amount_msat` | Fetches a fresh BOLT12 invoice (`lni1…` string) from an offer. `amount_msat` is required when the offer was created with `"any"`. The returned invoice can be paid with `ln_pay`. |

---

## 9. On-chain / utility (3 tools)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `ln_withdraw` | `node`, `destination`, `amount_sat` | — | Withdraws on-chain funds from the node's internal wallet. `amount_sat` can be an integer or `"all"`. |
| `ln_listtransactions` | `node` | — | On-chain transactions for the node's internal wallet. (Same tool name as in category 5; listed here for completeness.) |
| `ln_signmessage` | `node`, `message` | — | Signs `message` with the node's secret key. Returns the signature in zbase32. |

---

## 10. Passthrough (2 tools)

For CLI commands without a first-class wrapper. Each passthrough runs one `lightning-cli` or `bitcoin-cli` subcommand and enforces a blocklist.

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `ln_raw` | `node`, `command` | `args` | Runs any `lightning-cli` command. **Blocked**: `stop`, `recover`, `emergencyrecover`, `plugin`, everything starting with `dev-` or `autoclean-`. |
| `btc_raw` | `command` | `args`, `wallet` | Runs any `bitcoin-cli` command. **Blocked**: `stop`, `dumpprivkey`, `importprivkey`, `encryptwallet`, `backupwallet`, `dumpwallet`, `importwallet`. |

Common useful passthrough calls include `listconfigs`, `getlog`, `checkmessage`, `datastore` (for `ln_raw`) and `getpeerinfo`, `getnetworkinfo`, `listunspent`, `decoderawtransaction` (for `btc_raw`).

---

## 11. HTTP (1 tool)

| Tool | Required args | Optional args | What it does |
|------|--------------|---------------|---------------|
| `http_fetch` | `url` | `method`, `headers`, `body` | Fetches a URL. If the server responds with `HTTP 402 Payment Required` containing a BOLT11 invoice, the response is returned unchanged so the Executor's x402 auto-pay can handle it. `method` defaults to `"GET"`. |


---

## Worked example: end-to-end payment

The "golden path" — fund two nodes, open a channel between them, invoice, pay, and verify — calls the tools roughly in this order:

1. `network_health` — confirm infra is up.
2. `ln_node_status(node=2)` → `ln_node_start(node=2)` if not running.
3. `ln_getinfo(node=2)` → read `payload.id`, `payload.binding[0].address`, `payload.binding[0].port`.
4. `ln_connect(from_node=1, peer_id=$step3.payload.id, host=…, port=…)`.
5. `btc_wallet_ensure(wallet_name="miner")`.
6. `ln_newaddr(node=1)` and `ln_newaddr(node=2)` — on-chain funding addresses.
7. `btc_sendtoaddress(…)` to each node + `btc_generatetoaddress(blocks=101, address=…)` to confirm (coinbase maturity).
8. `ln_openchannel(from_node=1, peer_id=…, amount_sat=500000)` + `btc_generatetoaddress(blocks=6, …)` to confirm the funding transaction.
9. `ln_invoice(node=2, amount_msat=10000, label="pay-demo-<ts>", description="…")`.
10. `ln_pay(from_node=1, bolt11=$step9.result.payload.bolt11)` — use the *exact* bolt11 string.
11. `ln_listfunds(node=2)` — verify the balance actually moved (this is the goal-verification step the Executor runs automatically after a `pay_invoice` intent).

Every step is a single MCP tool call. The Planner produces the sequence; the Executor dispatches it; the Summarizer explains the outcome.

---

## Tool categories at a glance

| Category | Count | Side effects? |
|----------|-------|---------------|
| System / health | 3 | No |
| Bitcoin read | 6 | No |
| Bitcoin write | 4 | Yes |
| Node lifecycle | 6 | Yes (except list/status) |
| Lightning read | 12 | No |
| Lightning actions | 4 | Yes |
| Payments | 5 | Yes (except list/wait) |
| BOLT12 | 2 | Yes |
| On-chain / utility | 3 | Yes (except list/sign) |
| Passthrough | 2 | Depends on the underlying command |
| HTTP | 1 | No (but may trigger x402 auto-pay) |
| **Total** | **47** | |

The pipeline uses the read-only / state-changing distinction to (a) suppress redundant read calls in the Executor's oscillation detector and (b) trigger goal-verification reads after state changes. The full categorisation lives in `ai/tools.py`.
