---
title: AI Agent Internals
parent: Architecture
nav_order: 2
---

# AI Agent Internals

This page is the pipeline deep-dive. It is the right level of detail for a reader who wants to understand, modify, or write tests against the controllers in `ai/controllers/`.

For the layer overview, see [Architecture](../2.Architecture/). For the by-tool reference, see [Components → Tools](../3.Components/TOOLS.md). For the files-and-data-flow map, see [System Design](../2.Architecture/system-design.md).

---

## What the agent is

The agent is a **prompt-driven pipeline**: a natural-language request enters on one side and an executed plan plus human-readable answer leaves on the other side. There is no single "LLM loop" — the system is four cooperating controllers, each of which produces a typed artefact consumed by the next:

```
User prompt
     │
     ▼
[1. Translator] — LLM call
     │  IntentBlock { goal, intent_type, context, success_criteria, … }
     ▼
[2. Planner]    — LLM call
     │  ExecutionPlan { steps: [ PlanStep(tool, args, on_error, …), … ] }
     ▼
[3. Executor]   — no LLM calls
     │  Runs each MCP tool call, resolves placeholders, applies retry policy,
     │  verifies goal after state-changing steps
     ▼
[4. Summarizer] — LLM call
     │  Human-readable answer + ✓/✗ success verdict
     ▼
PipelineResult → outbox.jsonl → Web UI
```

## Key properties

**Tool-only execution.** The agent can call exactly the 47 tools exposed by the MCP server. It cannot run shell commands, read arbitrary files, or interact with any other system.

**Structured intermediate state.** Each stage produces a frozen dataclass (`IntentBlock`, `PlanStep`, `StepResult`) defined in `ai/models.py`. Parsing failures trigger a self-correcting retry before the stage aborts.

**Multi-turn context.** The last `PIPELINE_HISTORY_MAX` (default 4) prompt/response pairs are included in the Translator's system prompt. Follow-ups like *"now pay that invoice"* or *"what was the balance change?"* resolve naturally because the Translator sees both the previous answer and the new request.

**Value chaining.** The Executor resolves `$stepN.result.payload.field` placeholders in plan arguments at runtime. The LLM does not need to copy values like bolt11 strings or pubkeys through its context — the Executor wires them in.

**Goal verification.** After a state-changing intent (`pay_invoice`, `open_channel`, `rebalance`, `set_fee`, …) completes, the Executor automatically runs a read-only MCP call (`ln_listfunds`, `ln_listchannels`, etc.) to confirm the state change actually happened. A successful tool call is not the same as a successful goal.

**Deterministic Executor.** The Executor makes zero LLM calls. Given a plan and an infrastructure state it produces the same `StepResult` list every time. This is the property that makes the smoke-test battery (see [Runbooks](../4.Runbooks/)) meaningful.

---

## Stage-by-stage responsibilities

### Stage 1 — Translator

**Input:** raw user text + last N history turns.

**Output:** `IntentBlock(goal, intent_type, context, success_criteria, clarifications_needed, human_summary, raw_prompt)`.

**Behaviour:**

- One LLM call at a low temperature (default `TRANSLATOR_TEMPERATURE=0.1`).
- On JSON parse failure, the error is appended to the conversation and the LLM is asked to self-correct, up to `TRANSLATOR_MAX_RETRIES` additional attempts (default 2).
- `ai/intent_validate.py` rejects any intent whose `context` contains shell metacharacters, path traversal, raw `http://` URLs, or `sudo`. A rejected intent aborts the pipeline before the Planner stage sees it.
- Extracts common entities into `context` — node IDs, amounts in msat/sat/BTC, BOLT11 / BOLT12 strings, labels — so the Planner doesn't have to re-parse them out of the original prompt.

### Stage 2 — Planner

**Input:** `IntentBlock` + the tool schema string from `ai/tools.py`.

**Output:** `ExecutionPlan(steps=[PlanStep(step_id, tool, args, depends_on, on_error, max_retries, expected_outcome)], plan_rationale, intent)`.

**Behaviour:**

- One LLM call. For a small set of common single-tool patterns (`read_target=block_height`, `action_target=stop_node`, …) the Planner also has **deterministic dispatch short-circuits** that bypass the LLM entirely. These are workarounds for weak local models — see `PLAN.md` → Initiative 2 for the demotion plan once a stronger model can handle the same cases unaided. The short-circuits can be disabled with `PLANNER_FORCE_LLM=1` for auditing.
- Plan steps can contain `"$stepN.result.payload.path.to.field"` or `"$context.field"` placeholders; the Executor resolves them at runtime.
- Each step carries an `on_error` policy — `abort`, `retry`, or `skip` — and a `max_retries` count.
- `plan_rationale` is shown to the user in the Pipeline tab so the plan is inspectable before execution.

### Stage 3 — Executor

**Input:** `ExecutionPlan`.

**Output:** `List[StepResult]`.

**Behaviour:**

For each step in order:

1. Resolve `$stepN.…` and `$context.…` placeholders against prior results and the intent context.
2. Normalise arguments (`_normalize_tool_args` in `ai/tools.py`): unwrap nested `{"args": …}`, coerce string integers to ints, validate required keys, check `node` / `from_node` values against `runtime/node_count`.
3. JSON-RPC call the MCP server via `ai/mcp_client.py`. A hard per-call timeout (`MCP_CALL_TIMEOUT_S`, default 30 s) is enforced by a background thread.
4. On tool error: apply the step's `on_error` policy. `retry` re-runs the step after a fixed backoff (up to `max_retries` extra attempts). `skip` records `skipped=True` and continues. `abort` stops the plan.
5. Oscillation detection: a read-only tool called with identical args since the last state change is blocked after `MAX_CONSEC_READ_ONLY` consecutive read-only calls.
6. After any state-changing step, run a read-only verification call (`ln_listchannels` after `ln_openchannel`, `ln_listfunds` after `ln_pay`, …) to confirm the intent's `success_criteria` actually hold.

x402 auto-pay integration: if an HTTP tool call returns a `402 Payment Required` response with a BOLT11 invoice and `EXECUTOR_X402_AUTO_PAY=1`, the Executor pays the invoice via `ln_pay` and retries the tool call with the payment preimage.

### Stage 4 — Summarizer

**Input:** `IntentBlock` + `List[StepResult]` + `goal`.

**Output:** human-readable answer string + success verdict (boolean).

**Behaviour:**

- One LLM call. The Summarizer is the only stage whose output is shown directly to the user.
- Its system prompt includes a state-change guard: if the intent implied a state-changing verb (`pay`, `open`, `close`, `fund`, …) but no state-changing tool ran, the Summarizer surfaces the discrepancy rather than silently claiming success.

---

## Source layout

| File | Purpose |
|------|---------|
| `ai/pipeline.py` | Main loop: inbox polling, stage orchestration, history, goal verification, archive write. |
| `ai/models.py` | Dataclasses for all structured pipeline state. |
| `ai/tools.py` | Tool schemas (for LLM prompts), `TOOL_REQUIRED`, `READ_ONLY_TOOLS`, `STATE_CHANGING_TOOLS`. |
| `ai/controllers/translator.py` | Stage 1 — text → `IntentBlock`. |
| `ai/controllers/planner.py` | Stage 2 — `IntentBlock` → `ExecutionPlan`. |
| `ai/controllers/executor.py` | Stage 3 — execute the plan. |
| `ai/controllers/summarizer.py` | Stage 4 — results → human answer. |
| `ai/controllers/shared.py` | JSON repair, parsing helpers used by all four stages. |
| `ai/mcp_client.py` | JSON-RPC client for the MCP server. |
| `ai/intent_validate.py` | Safety gate for stage-1 output. |
| `ai/core/` | Config, scheduler, registry, rate limiter, backoff, token estimator, concurrency gate. |
| `ai/llm/` | `LLMBackend` protocol + `create_backend_for_role` factory + `GuardedBackend` wrapper + 4 adapters. |

## LLM backends

| Backend | `LLM_BACKEND` | Notes |
|---------|---------------|-------|
| OpenAI | `openai` | Default. Requires `OPENAI_API_KEY`. |
| Anthropic Claude | `claude` | Requires `ANTHROPIC_API_KEY`. |
| Google Gemini | `gemini` | Requires `GEMINI_API_KEY`. |
| Ollama | `ollama` | Local, no API key. Needs Ollama running and a tool-calling model pulled. |

Each stage can override the global `LLM_BACKEND` with `TRANSLATOR_LLM_BACKEND`, `PLANNER_LLM_BACKEND`, or `SUMMARIZER_LLM_BACKEND`. Per-role model overrides use `{ROLE}_{BACKEND}_MODEL` — e.g. `PLANNER_OPENAI_MODEL=gpt-4o`.

Set `LLM_TEMPERATURE=0` for deterministic, reproducible tool-calling behaviour during experiments.

---

## Running the test suite

```bash
cd ln-ai-network
source .venv/bin/activate
python -m pytest ai/tests/ -v
```

791 tests, 3 skipped (live-infra-only), 0 failing. All tests run offline with no API keys or live infrastructure.

---

## Related

- [MCP Tools reference](../3.Components/TOOLS.md)
- [System design](../2.Architecture/system-design.md)
- [Runbooks](../4.Runbooks/) — smoke-test prompts, restart procedures, debugging traces
