---
title: System Design
parent: Architecture
nav_order: 1
---

# System Design

A deeper look at how a prompt travels through the system, what files are touched along the way, and how the four pipeline stages interact. Read [Architecture](index.md) first for the top-level layer diagram and glossary.

---

## The full data flow

```
User types prompt
        │
        ▼
POST /api/ask  ──►  runtime/agent/inbox.jsonl (append one JSON line)
        │
        │  Pipeline polls the inbox every PIPELINE_POLL_INTERVAL_S seconds
        ▼
ai/pipeline.py PipelineCoordinator reads the new line
        │
        ├─► [Stage 1 — Translator]
        │       Input:  prompt text + last N history turns
        │       Call:   LLM (one call, up to TRANSLATOR_MAX_RETRIES on parse failure)
        │       Output: IntentBlock
        │                 { goal, intent_type, context, success_criteria,
        │                   clarifications_needed, human_summary, raw_prompt }
        │       Safety: ai/intent_validate.py rejects intents containing shell
        │               metacharacters, path traversal, sudo, or raw URLs.
        │
        ├─► [Stage 2 — Planner]
        │       Input:  IntentBlock + tool schema + planner system prompt
        │       Call:   LLM (one call, plus a deterministic dispatch short-circuit
        │               for common single-tool patterns — see PLAN.md Initiative 2)
        │       Output: ExecutionPlan
        │                 { steps: [ PlanStep(step_id, tool, args, depends_on,
        │                                     on_error, max_retries), … ],
        │                   plan_rationale, intent }
        │       Placeholders: PlanStep.args can reference prior step output as
        │                     "$stepN.result.payload.path.to.field" or
        │                     "$context.field"; the Executor resolves both at
        │                     runtime.
        │
        ├─► [Stage 3 — Executor]
        │       Input:  ExecutionPlan
        │       For each step in order:
        │         1. Resolve "$stepN.…" and "$context.…" placeholders
        │         2. Normalize args (coerce "1" → 1, unwrap nested {"args":…},
        │            validate node indices against runtime/node_count)
        │         3. JSON-RPC call to the MCP server (stdin/stdout)
        │         4. MCP server runs the appropriate bitcoin-cli/lightning-cli
        │         5. Collect result, apply on_error policy (abort/retry/skip)
        │       After state-changing steps: run a read-only verification call
        │       (e.g. ln_listchannels after ln_openchannel) to confirm the
        │       intended state change actually happened.
        │       Output: List[StepResult]
        │
        └─► [Stage 4 — Summarizer]
                Input:  List[StepResult] + IntentBlock + goal
                Call:   LLM (one call)
                Output: human-readable answer with explicit success/failure
                        verdict

Coordinator assembles everything into a PipelineResult
        │
        ▼
runtime/agent/outbox.jsonl (append one JSON line)
        │
        │  UI server file-watches the outbox and pushes an SSE event
        ▼
Web UI renders the Summary card, Pipeline cards, and trace panel
```

Three LLM calls per successful run (Translator, Planner, Summarizer). The Executor itself makes **zero** LLM calls — it is a deterministic dispatch loop that reads a plan and runs it.

---

## Pipeline source layout

| File | Purpose |
|------|---------|
| `ai/pipeline.py` | The coordinator. Polls the inbox, runs the four stages, writes the outbox, maintains history, verifies goals. |
| `ai/models.py` | Frozen dataclasses for every pipeline artefact: `IntentBlock`, `PlanStep`, `ExecutionPlan`, `StepResult`, `PipelineResult`. |
| `ai/tools.py` | Tool registry: `READ_ONLY_TOOLS` / `STATE_CHANGING_TOOLS` sets, `TOOL_REQUIRED` argument map, `_normalize_tool_args()` argument coercer, LLM schema generators. |
| `ai/mcp_client.py` | JSON-RPC client that talks to the MCP server over stdio. `FastMCPClientWrapper` adds a timeout and a per-instance lock. |
| `ai/intent_validate.py` | Safety gate: rejects intents containing shell metacharacters, path traversal, raw URLs, or sudo. |
| `ai/controllers/translator.py` | Stage 1 — prompt → `IntentBlock`, including JSON repair and self-correcting retry. |
| `ai/controllers/planner.py` | Stage 2 — `IntentBlock` → `ExecutionPlan`, including the deterministic dispatch short-circuits for common patterns. |
| `ai/controllers/executor.py` | Stage 3 — executes the plan with placeholder resolution, retry/skip/abort policy, x402 auto-pay, and goal verification. |
| `ai/controllers/summarizer.py` | Stage 4 — tool results → human answer. |
| `ai/controllers/shared.py` | JSON repair, code-fence stripping, and LLM parsing helpers shared by all four stages. |
| `ai/core/config.py` | `AgentConfig` — every pipeline env var, parsed once at startup. |
| `ai/core/scheduler.py` | `DeterministicScheduler` — tick-driven inbox polling loop with no wall-clock dependency. |
| `ai/core/registry.py` | `AgentRegistry` — file-backed record of which pipeline processes are running (used by multi-agent mode). |
| `ai/core/rate_limiter.py` | `DualRateLimiter` — per-backend rate limiting for LLM calls. |
| `ai/core/backoff.py` | `DeterministicBackoff` — fixed-schedule retry with a circuit breaker. |
| `ai/core/token_estimation.py` | `HeuristicTokenEstimator` — conservative cost estimator for LLM-call quoting. |
| `ai/core/concurrency.py` | `ConcurrencyGate` — bounds the number of simultaneous LLM calls. |
| `ai/llm/base.py` | `LLMBackend` protocol and the `LLMRequest` / `LLMResponse` dataclasses. |
| `ai/llm/factory.py` | `create_backend_for_role()` — reads env vars and returns the right adapter per pipeline stage. |
| `ai/llm/guarded_backend.py` | `GuardedBackend` — wraps any adapter with the rate limiter, backoff, and concurrency gate. |
| `ai/llm/adapters/openai_backend.py` | OpenAI Chat Completions + function-calling. |
| `ai/llm/adapters/claude_backend.py` | Anthropic Messages API + tool use. |
| `ai/llm/adapters/gemini_backend.py` | Google Gemini generateContent + function-calling. |
| `ai/llm/adapters/ollama_backend.py` | Local Ollama HTTP API. |

---

## MCP tool server

`mcp/ln_mcp_server.py` is the execution boundary. It:

- Reads newline-delimited JSON requests on stdin.
- Runs `bitcoin-cli` or `lightning-cli` subprocess calls with validated arguments.
- Returns `{"ok": bool, "payload": ...}` on stdout.

The pipeline starts the server as a subprocess via `FastMCPClient` and keeps the connection open for the life of the agent. Each tool call is synchronous with a hard timeout (`MCP_CALL_TIMEOUT_S`, default 30 s) enforced by a background thread. If the timeout fires, the Executor treats it as a tool failure and applies the step's `on_error` policy.

The server is the **only** component in the whole system that runs `bitcoin-cli` or `lightning-cli`. Nothing in `ai/` touches the infrastructure directly. This is deliberate: it is the property that makes the safety claim "the LLM cannot run arbitrary commands" true.

---

## Runtime layout

Everything that the running system writes lives under `ln-ai-network/runtime/` and is gitignored. A fresh start recreates it from scratch.

```
ln-ai-network/runtime/
  node_count              — number of Lightning nodes active this run (e.g. "2")
  mcp.pid                 — MCP server PID
  ui_server.pid           — UI server PID
  registry.jsonl          — agent-registry file for multi-agent mode
  rpc_auth.conf           — Bitcoin RPC credentials (mode 0600)
  agent.pid · agent.log   — top-level agent process PID and stdout/stderr
  ui_server.log           — UI server log
  agent/
    agent.pid             — pipeline process PID
    pipeline.lock         — single-instance advisory lock (contains "pid=NNNN")
    inbox.jsonl           — incoming prompts (append-only)
    inbox.offset          — byte-offset read cursor into inbox.jsonl
    outbox.jsonl          — completed PipelineResults (append-only)
    history.jsonl         — rolling last-N exchanges used for multi-turn context
    archive.jsonl         — permanent episodic log consumed by the memory_lookup tool
    trace.log             — live per-request trace (reset on each new prompt)
    stream.jsonl          — live LLM token stream for the UI /api/tokens endpoint
  bitcoin/
    shared/               — bitcoind data directory
      bitcoin.conf
      regtest/            — blockchain, wallet, chainstate
  lightning/
    node-1/               — lightningd data for node 1
      config
      hsm_secret          — node private key
      regtest/            — channel database, gossip, …
    node-2/
      ...
```

The byte-offset cursor in `inbox.offset` is how the pipeline survives a restart without re-processing old prompts: it records how many bytes of `inbox.jsonl` have already been consumed, and a clean restart resumes from that exact offset.

---

## Multi-turn conversation history

After every completed pipeline run, the prompt and the Summarizer's answer are appended to `runtime/agent/history.jsonl`. On subsequent prompts, the last `PIPELINE_HISTORY_MAX` turns (default 4) are included in the Translator's system prompt as prior conversation. This is how follow-ups like *"now pay that invoice"* or *"what was the balance change?"* resolve — the Translator sees both the old answer and the new request when it produces the next `IntentBlock`.

History is also persisted to `archive.jsonl` (without the truncation the rolling window imposes), which the `memory_lookup` MCP tool queries when the user asks questions like *"what did I run in the last few prompts?"*.

---

## SSE streaming

The web UI connects to `/api/stream`, a Server-Sent Events endpoint served by `scripts/ui_server.py`. The server pushes three event types:

| Event | When it fires | Payload |
|-------|---------------|---------|
| `status` | Every ~2 seconds, and immediately when the outbox changes | Agent lock PID, last outbox entry, inbox/outbox counts, last 10 inbox/outbox lines |
| `pipeline_result` | On every new outbox line | Full serialised `PipelineResult` |
| `trace` | On every new `trace.log` append | Batch of trace events since the last push |

The LLM token-level stream is available separately on `/api/tokens`, which tails `runtime/agent/stream.jsonl`. Most UI panels consume `pipeline_result` and `trace`; only the "live tokens" display uses `/api/tokens`.

---

## Security boundary

The MCP server is the only component with shell access. The AI pipeline cannot run arbitrary commands — it can only call the 47 named tools in `ln_mcp_server.py`. Each tool:

- Has a fixed signature validated by the server before the subprocess runs.
- Calls exactly one `bitcoin-cli` or `lightning-cli` invocation.
- Returns structured JSON; no raw shell output is passed back.

The two passthrough tools (`ln_raw`, `btc_raw`) exist for completeness — they let a plan call a CLI command that doesn't have a first-class wrapper — but they enforce their own per-tool blocklists. For example, `btc_raw` refuses `dumpprivkey`, `encryptwallet`, `importwallet`, and the full list is visible in [`../3.Components/TOOLS.md`](../3.Components/TOOLS.md#passthrough-tools).

The HTTP layer has its own security stack: session authentication, CSRF tokens, rate limiting, RBAC, optional TLS, and audit logging.
