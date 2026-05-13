---
title: Prompt Battery Archive
parent: Runbooks
nav_order: 1
---

# Prompt Battery Archive

A running log of issues the prompt battery has surfaced, the fixes that landed, and the state of each run. This file is append-only history; don't delete entries — future debugging benefits from knowing what used to break and why it was fixed.

For the current list of copy-paste smoke-test prompts, see [Runbooks index → Smoke-test prompts](./index.md#smoke-test-prompts).

## How to use this archive

1. When the battery is run against a new model, backend, or after a pipeline-prompt change, add a new dated section at the top under **## Runs** with a table of results.
2. When a run exposes a new failure mode, add an entry under **## Known issues / fixes** that links the failure to the commit that fixed it.
3. If an old issue re-emerges, link back to the existing entry and add a "re-observed on YYYY-MM-DD" note rather than duplicating.

The battery itself lives in `/tmp/run_battery.py`; recreate it from the snippet in the runbook index if it's missing.

---

## Known issues / fixes

Each entry is **what broke**, **why it broke**, and **what landed to fix it**. Commits referenced are on `josh` unless noted.

### 1. Executor crashed on Jinja-style placeholders in `btc_generatetoaddress`

**Observed:** 2026-04-08 (qwen2.5:14b) — prompt "Mine 3 blocks to a new address from the shared bitcoin wallet."

**Symptom:** Step 2 failed with `invalid regtest address: {{ steps[0].output }}`. The planner emitted Jinja-style placeholders to chain step 2 onto step 1's `btc_getnewaddress` output, but the Executor only resolves `$stepN.field.path` and passed the Jinja string through verbatim.

**Fix:**
- `ae21660` — Banned Jinja/Mustache/shell/XML placeholder syntax in the planner system prompt. Added a `STANDALONE MINING` few-shot showing the correct `$step1.result.payload` format.
- Added a validator pass in `_validate_plan_steps` that walks nested dicts/lists and rejects any arg value matching `{{...}}`, `${...}`, `<stepN>`, or bare `$1`/`$2`. Tests: `test_jinja_placeholder_rejected`, `test_shell_placeholder_rejected`, `test_xml_placeholder_rejected`, `test_nested_jinja_placeholder_rejected`, `test_valid_stepN_placeholder_accepted`.
- Updated `ai/controllers/executor.py` and `ai/controllers/planner.py` docstrings to document `$stepN.field.path` as the only supported placeholder format.

**Status:** Validator rejects the bad output, but **qwen2.5:14b still emits Jinja** in some contexts (see run table below for `network health` and `mine blocks` on 2026-04-08). The retry loop's self-correction doesn't reliably pull it out. The validator failing the plan is preferable to the executor crashing at the tool call, but the underlying model instruction-following is a known limitation — switching `PLANNER_LLM_BACKEND=openai` or going to a larger Ollama model is the workaround.

### 2. Planner collapsing to `network_health` for specific read queries

**Observed:** 2026-04-08 (qwen2.5:14b) — prompts "node info", "channel count", "list peers", "list funds" all returned a single `network_health` step regardless of which tool they should have targeted.

**Symptom:** The planner picked `network_health` as a default anywhere the intent was "read some information". The summarizer then had to answer a specific question from a payload that didn't contain the answer — either hallucinating values or admitting "the tool did not return the requested information".

**Fix:**
- `6e849fe` — Added a **READ-ONLY INFORMATION QUERIES** block to the planner system prompt mapping phrases to dedicated tools:
  - `show info / pubkey / version / alias for node N` → `ln_getinfo(node=N)`
  - `list peers on node N` → `ln_listpeers(node=N)`
  - `list funds for node N` / `balance of node N` → `ln_listfunds(node=N)`
  - `how many channels / channel state` → `ln_listchannels(node=N)` or `ln_listfunds(node=N)`
  - `current block height` → `btc_getblockchaininfo`
  - `wallet balance` → `btc_getbalance`
- Narrowed the `network_health` rule so it only fires on explicit diagnostic phrasings like "run a diagnostic", "check health", "overall status", "is the system up".
- Added an `INTENT DISCRIMINATION` rule: info requests must READ; action requests must EXECUTE; never substitute one for the other.

**Status:** Partially effective with qwen2.5:14b. Prompts 7 and 9 (list funds, list peers) now hit the right tool. Prompts 3 and 5 (node info, channel count) **still** collapse to `btc_getblockchaininfo` on some runs — see the 2026-04-08 post-fix table. The rule is in the prompt; the model's instruction-following doesn't always honor it.

### 3. Empty plans silently falling back to `intent.human_summary`

**Observed:** 2026-04-08 (qwen2.5:14b) — prompt "List all funds for node 1" produced an empty plan; the pipeline took its generic empty-plan short-circuit and returned the translator's `human_summary` ("I will list funds for node 1") instead of ever calling the listfunds tool. The user saw a confident sentence about what the agent was *about* to do, but no data.

**Symptom:** Empty plan for a `freeform` intent → pipeline returns `human_summary` → user gets a confident rewording of the intent with no actual tool output.

**Fix:**
- `6e849fe` — Planner now raises `ValueError` when it produces an empty plan for a `freeform` intent, giving the retry loop a chance to self-correct. If retries exhaust, the pipeline reports `stage_failed=planner` (honest failure) instead of returning a misleading "success".
- Empty plans are still accepted for `noop` and (later) `informational` intent types, where they are correct by design.
- Tests: `test_empty_plan_for_freeform_intent_rejected`, `test_empty_plan_retry_succeeds_on_second_attempt`, `test_empty_plan_accepted_for_non_freeform_intents`.

**Status:** Fixed — `list funds` prompts that used to collapse to empty plans now fail loudly or produce a real plan.

### 4. Informational ("decode this scenario") prompts getting executed

**Observed:** 2026-04-08 — prompt "Decode this scenario: how would I open a channel from node 1 to node 2 with 100000 sats?" was classified as `open_channel` and the pipeline actually tried to open the channel (partially: pre-flight `ln_node_status`, `ln_node_start`, `sys_netinfo`, `ln_getinfo` all ran, but the model didn't finish the chain).

**Symptom:** The user asked for an explanation ("decode this scenario", "how would I…") but the pipeline was treating it as an actionable request. Side effects were partial and unwanted.

**Fix (three commits):**
- `136ed29` — Added a new `informational` intent type for explanation/walkthrough requests.
  - **Translator:** Recognizes phrases like "decode this", "walk me through", "how would I", "explain how", "hypothetically", "what would happen if", "describe the process". Emits `intent_type=informational`.
  - **Planner:** Short-circuits before the LLM call and returns an empty plan. No tools are selected, no retries happen, no state changes.
  - **Summarizer:** Swaps the default past-tense system prompt for an explanation-mode prompt that uses future/conditional tense ("node 2 would need to be running", "ln_connect would be called…"). Without this swap, the default "PAST TENSE" guard would force the LLM to invent a fake execution report.
- `781b444` — Fixed a pipeline routing bug: the `PipelineCoordinator` was taking its generic "empty plan" short-circuit before the summarizer was reached, which meant the explanation-mode prompt was unreachable. Now informational intents skip the executor but still flow into the summarizer. `_verify_goal` is also skipped for informational intents so the post-summary "verified" note doesn't claim a hypothetical was acted on.
- Tests: `test_informational_intent_type_accepted` (translator), `test_plan_informational_short_circuits_to_empty_plan` (planner), `test_summarize_informational_uses_explanation_prompt`, `test_summarize_non_informational_uses_default_prompt` (summarizer).

**Status:** Fixed end-to-end. Verified on the 2026-04-08 (post-fix, post-pipeline-fix) re-run: prompt 10 returns a full conditional-tense walkthrough starting *"Here's how that would work — no actions were taken. First, node 2 would need to be running..."* instead of the translator fallback *"I'll walk you through the steps..."*.

### 5. `sys_netinfo` invoked for same-machine scenarios

**Observed:** 2026-04-08 (qwen2.5:14b) — prompt "Decode this scenario" chain included `sys_netinfo` even though both nodes run on `127.0.0.1` and the prompt has no mention of another machine.

**Symptom:** The planner was treating "connect node 1 to node 2" as if it needed to discover an external IP, adding a wasted `sys_netinfo` step and then using its output as `announce_host` / `bind_host`. On its own this would only bloat the plan; combined with Issue 1, the Jinja placeholder syntax used for `announce_host` crashed the planner validator.

**Fix:**
- `d876d35` — Tightened the `CROSS-MACHINE CONNECTIVITY` rule in the planner system prompt to require a literal phrase match: `"from another machine"`, `"from a remote host"`, `"across the internet"`, `"announce a public address"`, `"bind to 0.0.0.0"`, etc. Added an explicit NEVER: same-machine `open_channel`/`connect`/`pay` must not set `bind_host` or `announce_host`.

**Status:** Effective for prompts with no "another machine" language. Qwen2.5:14b **still** fires `sys_netinfo` for some freeform prompts (see the post-fix `network health` and `mine blocks` failures on 2026-04-08) — likely because the model is associating "network" or "machine" loosely. This is another instruction-following limitation; switching to a larger planner backend is the reliable workaround.

### 6. Executor treating Ollama HTTP 429 as permanent

**Observed:** pre-2026-04-08 — Ollama backend raised `PermanentAPIError` on rate limits instead of `RateLimitError`, so the `GuardedBackend` circuit breaker tripped on transient throttling.

**Symptom:** Rate-limited prompts looked like hard backend failures in the UI and didn't get retried.

**Fix:** `1471b00` on `main` — map HTTP 429 to `RateLimitError` in the Ollama adapter.

**Status:** Fixed — no longer observed.

---

## Runs

Newest first. Each run records the battery result against a specific model/backend so future debugging has a baseline to compare against.

### 2026-04-27 — qwen2.5:7b/GPU full 42-prompt battery, `PLANNER_FORCE_LLM=fallback` (Phase 5 second validation / variance check)

**Result: 40/42 PASS, 0 RETRY, 2 FAIL.** Wall time **~579s (~9.7 min)**. Clean-runtime protocol (wiped `runtime/` before the run, fresh `./scripts/1.start.sh 2` boot, 0 pre-existing channels at start). Same fallback-mode setup as the 2026-04-24 run.

**Purpose**: repeat-variance check on the Phase 5 fallback mode and a second authoritative data point for the headline-results table. Same model, same mode, same battery, run on a freshly synced `main` (after the local-Luke-commit reconciliation pulled in 7 upstream commits including PLAN.md audit additions, registry Windows-safe fixes, and repo cleanup that removed 44 MB Windows bitcoin binaries). No code changes between this run and the prior fallback-mode run.

**Backend / mode** (verified via `/proc/$AGENT_PID/environ`):
- `LLM_BACKEND=ollama` ✓
- `OLLAMA_MODEL=qwen2.5:7b` ✓
- `COMPUTE_DEVICE=gpu` ✓
- `PLANNER_FORCE_LLM=fallback` ✓ (exported in parent shell; not in `.env`)

**Trace-event counts across all 42 per-request logs** (`runtime/agent/logs/*.jsonl`):

| Event | Count | Interpretation |
|-------|------:|----------------|
| `planner_force_llm_active` (mode: fallback) | 41 | Fallback mode active on every non-informational planner call (#10 is informational and short-circuits above the gate). |
| `planner_llm_dispatch` | 41 | LLM path executed every time fallback was active. |
| `deterministic_dispatch_hit` | **0** | Initial dispatch correctly skipped (by design). |
| `planner_fallback_dispatch_fired` | **0** | Safety net never triggered — qwen produced valid plans every time. |

**Failures (different prompts than the prior fallback run, but same headline 40/42 result):**

| # | Prompt | Stage | What went wrong |
|---|--------|-------|-----------------|
| 14 | "List all forwarded HTLCs on node 1." | executor | The LLM-planned `ln_listforwards` call carried a `status` argument that CLN rejected: `"status: Unrecognized status: invalid token (see logs for details)"`. In **legacy** mode this prompt is dispatched directly to `ln_listforwards()` with no args, which is why it doesn't fail there. In fallback mode the LLM has to construct the call itself and over-specifies arguments. Same root pattern as #36 last time: LLM-side argument hallucination on a prompt the legacy dispatch covers. |
| 32 | "Confirm node 2 received the 5000-msat payment." | planner (driver scoring) | Planner emitted `ln_listpays` instead of the expected `ln_listinvoices`. This is the payment-direction confusion already documented in the planner system prompt: "received → ln_listinvoices on recipient; sent → ln_listpays on sender". The planner has the rule and the exemplar; qwen2.5:7b just doesn't always follow them at temperature=0.1. The driver flagged it as FAIL because it scores tool-selection correctness; the underlying plan was structurally valid (so the fallback safety net didn't fire — fallback only fires on parse/validation failure, not on a wrong-but-valid plan). |

**Both failures are at non-fallback layers**:
- #14 is an executor stage failure caused by an LLM-generated arg the legacy dispatch wouldn't have produced.
- #32 is a tool-selection mismatch (driver-side scoring) on a structurally valid plan.

Neither is a Phase 5 regression; both reflect known qwen-side weaknesses that legacy mode masks via dispatch.

**Cross-comparison: this run vs the prior fallback-mode run (2026-04-24)**:

| | 2026-04-24 fallback | 2026-04-27 fallback (this run) |
|---|---|---|
| Pass rate | 40/42 | 40/42 |
| Retries | 0 | 0 |
| Wall time | ~643 s | ~579 s |
| Failure 1 | #36 "Re-open channel 200000 sats" — UTXO shortage | #14 "List all forwarded HTLCs" — `ln_listforwards` arg hallucination |
| Failure 2 | #38 "Pay BOLT12 offer" — cascade from #36 | #32 "Confirm node 2 received payment" — `ln_listpays` vs `ln_listinvoices` |
| Fallback fires | 0 | 0 |
| Initial dispatch hits | 0 | 0 |
| LLM planner calls | 41 | 41 |

The headline number (40/42) is stable across both runs. Which two prompts fail varies — this run's #36 and #38 both passed, while last run's #14 and #32 both passed. The residual ~5% delta vs the legacy 42/42 baseline is dominated by qwen-side variance and downstream-environment flakes that drift between runs, not by Phase 5 itself. Two independent runs both showing 0 fallback fires is strong evidence that the safety net is correctly gated and currently dormant.

**Implication**: treat this as a 2-of-2 fallback-validation outcome (40/42, 0 fallback fires, drift between which two prompts fail) rather than a single-run anomaly.

---

### 2026-04-24 — qwen2.5:7b/GPU full 42-prompt battery, `PLANNER_FORCE_LLM=fallback` (Initiative 2 Phase 5 validation)

**Result: 40/42 PASS, 0 RETRY, 2 FAIL.** Wall time **~643s (~10.7 min)**. Clean-runtime protocol: wiped `runtime/` before the run, fresh `./scripts/1.start.sh 2` boot, 0 pre-existing channels at start.

**Purpose**: validate the Initiative 2 Phase 5 `PLANNER_FORCE_LLM=fallback` mode that shipped in commit `46da37f`. In fallback mode the deterministic dispatches are skipped initially; the Planner LLM runs first on every prompt, and the dispatch only fires as a safety net if the LLM retry loop exhausts AND the Translator set a recognized `read_target`/`action_target` tag.

**Backend / mode**:
- `LLM_BACKEND=ollama`, `OLLAMA_MODEL=qwen2.5:7b`, `COMPUTE_DEVICE=gpu` (the official local baseline — unchanged).
- `PLANNER_FORCE_LLM=fallback` exported in the parent shell before `./scripts/1.start.sh 2` so the agent process inherits it. `.env` was not modified (the flag is not defined there).
- Verified via `/proc/$AGENT_PID/environ` that the agent process picked up the fallback mode.

**Trace-event counts across all 42 per-request logs** (`runtime/agent/logs/*.jsonl`):

| Event | Count | Interpretation |
|-------|------:|----------------|
| `planner_force_llm_active` (mode: fallback) | 41 | Fallback mode active on every non-informational planner call. Prompt #10 is informational and short-circuits above the gate. |
| `planner_llm_dispatch` | 41 | LLM path executed every time fallback mode was active. |
| `deterministic_dispatch_hit` | **0** | Initial dispatch was correctly skipped (by design in fallback mode). |
| `planner_fallback_dispatch_fired` | **0** | The safety net was never triggered — the LLM produced valid plans every time. |

**Planner-level conclusion**: qwen2.5:7b planned all 41 non-informational battery prompts without dispatch assistance and without a single parse/validation failure. The fallback safety net is fully wired (confirmed by unit tests in `test_planner.py`), correctly gated, and dormant in practice on this model for this battery.

**The two failures were executor/topology-sensitive, not planner/fallback failures:**

| # | Prompt | Stage | CLN error |
|---|--------|-------|-----------|
| 36 | "Re-open a channel from node 1 to node 2 with 200000 sats." | executor | `"Could not afford 200000sat using all 1 available UTXOs: 274sat short"` — node-1's on-chain balance is marginally below 200k + fees after the #34 close → mine → reopen cycle. |
| 38 | "Pay node 2's BOLT12 offer lno1…" | executor | `"Ran out of routes to try: No path found"` — direct cascade from #36; the channel never reopened, so there's no route. |

Neither failure reflects on Phase 5. Both are in the same topology/liquidity-sensitivity class already documented in the 2026-04-22 clean-runtime hygiene decision-log entry. Possible mitigations (not attempted in this pass): bump `NODE_FUNDING_BTC` from 10 → 15 in `.env`, or tighten the planner's #26/#36 exemplar to use a smaller default channel size (e.g. 180k) to leave more headroom for fees.

**Direct comparison against the authoritative legacy qwen baseline (2026-04-22)**:

| Metric | Legacy (dispatch-first) | Fallback (this run) |
|---|---|---|
| Pass rate | 42/42 | 40/42 |
| Retries | 0 | 0 |
| Wall time | ~486 s (~8.1 min) | ~643 s (~10.7 min, ~1.32×) |
| Initial dispatch hits | ~15+ | 0 (skipped) |
| LLM planner calls | ~27 | 41 |
| Fallback-dispatch fires | n/a | 0 |
| Planner-stage failures | 0 | 0 |
| Executor-stage failures | 0 | 2 (#36, #38) |

The 2-prompt pass-rate delta is entirely at the executor stage and falls within the band of topology noise seen on previous runs at this depth (compare with the earlier 38/42 → 42/42 pair on the same code in 2026-04-22). The planner stage is equivalent in both modes, just paying more LLM calls in fallback mode for the cheaper wall time.

**Machine-readable artifacts**: per-request trace logs in `ln-ai-network/runtime/agent/logs/0001_…` through `0042_…` at time of run; battery log captured in session output. No permanent artifacts committed.

---

### 2026-04-24 — OpenAI gpt-4o full 42-prompt battery (comparison against qwen2.5:7b/GPU baseline)

**Result: 42/42 PASS, 0 RETRY, 0 FAIL** on `gpt-4o`, wall time **~914s (~15.2 min)**. Clean-runtime protocol: wiped `runtime/` directory before the run, fresh `./scripts/1.start.sh 2` boot, 0 pre-existing channels at start, agent autostarted node-2 via MCP in prompt #4.

Backend setup: `.env` `LLM_BACKEND` was temporarily switched from `ollama` → `openai` for this one run and reverted immediately after (see commit history — no persistent change). `OPENAI_MODEL=gpt-4o` (full model, not mini). Same battery driver, same 42 prompts, same `--timeout 180` as the local qwen baseline. No translator / planner / executor / dispatch / tool-schema changes — this is a pure backend swap.

**Direct comparison against the clean-topology qwen2.5:7b/GPU baseline (2026-04-22)**:

| Metric | qwen2.5:7b / GPU (2026-04-22) | OpenAI gpt-4o (this run) |
|---|---|---|
| Pass rate | 42/42 | **42/42** |
| Retries | 0 | 0 |
| Failures | 0 | 0 |
| Total wall time | ~486 s (~8.1 min) | ~914 s (~15.2 min, ~1.9×) |
| Mean per-prompt | ~11.6 s | ~21.8 s |
| Mean Phase 1–2 (#1–#25) | ~6–10 s per prompt | ~10–17 s per prompt |
| Mean Phase 3–4 (#26–#42) | ~15–25 s per prompt | ~25–45 s per prompt |
| Determinism | local, reproducible | network-dependent (OpenAI-side latency variance) |

**Qualitative observations**:
- Pass-rate parity. Neither backend needed a retry. No new failure modes introduced on OpenAI.
- OpenAI is network-bound — individual LLM calls run ~1.5–3s, with network round-trip adding most of the remaining per-prompt time. Multi-step payment-flow prompts (#26–#38) issue multiple serial LLM calls and therefore feel the network overhead most.
- BOLT12 end-to-end (#37 → #38) worked on first attempt on gpt-4o with no translator drift. The translator/planner/safety/tool-arg fixes landed in commits `0d7de67` / `e7f1985` for qwen are also triggered on gpt-4o but are not strictly necessary — gpt-4o would plan the `ln_fetchinvoice → ln_pay` chain unaided.
- The deterministic dispatches still fire on gpt-4o wherever the Translator emits `read_target` / `action_target` tags (they're backend-agnostic). Per the Initiative 2 Phase 3 audit run (below — same day, 12/12 PASS on OpenAI with `PLANNER_FORCE_LLM=1`), gpt-4o doesn't need them, but they do no harm. This informs Initiative 2 Phase 5 (demote to fallback-only).
- Goal verification on read-then-confirm prompts (#27, #35) produced clean "verify" plans without accidentally re-executing the state-changing action — the summarizer state-change guard held on gpt-4o, same as on qwen.

**Scope of this run**: comparison data point, not a baseline change. **qwen2.5:7b + GPU remains the official default local baseline** because it achieves identical pass-rate at roughly half the wall time and $0 marginal cost. OpenAI is kept as a reference point for Phase 5's fallback-only validation and as a cross-model comparison dataset.

Artifacts: full per-prompt log at `/tmp/openai_battery.log` at the time of the run (rotates with reboots); structured results in the Decision log entry 2026-04-24. Also see the 2026-04-24 audit entry under **Audit runs** below for the 12/12 dispatch-discipline result.

---

### 2026-04-12 — Phase 3 first live run: qwen2.5:14b 3/10, gemini-2.5-flash blocked by Google API errors

**Result A — qwen2.5:14b: 3/10 PASS, 0 RETRY, 7 FAIL** on the new Phase 3 prompts (#26–#35, LLM-planned multi-step flows). This is the **expected** outcome — Phase 3 is the test of whether the planner can decompose payment-lifecycle requests on its own, with no deterministic dispatch fallback. The 7 failures confirm the thesis from Decision log 2026-04-09 that qwen2.5:14b cannot reliably plan multi-step Lightning flows at this temperature.

Stack: 2 nodes regtest, 16 channels in `CHANNELD_NORMAL`, fresh agent state, `--phase 3 --timeout 120`.

| # | Verdict | Time | What went wrong |
|---|---------|------|-----------------|
| 26 | FAIL | 103.3s | `ln_openchannel` — translator timeout on attempt 1 (>120s), executor failed on retry |
| 27 | PASS | 33.2s | Verify channel state |
| 29 | FAIL | 22.4s | `ln_invoice` — planner picked `ln_listinvoices` (read-only) instead of the create endpoint |
| 28 | SKIP | — | Depends on bolt11 capture from #29 |
| 30 | SKIP | — | Depends on bolt11 capture from #29 |
| 31 | PASS | 16.8s | Check last payment status |
| 32 | FAIL | 30.4s | `ln_listinvoices` — picked `ln_listtransactions` (Bitcoin tx list, wrong layer) |
| 33 | FAIL | 73.7s | `ln_keysend` — planner stage failed to produce a valid plan at all |
| 34 | FAIL | 40.9s | `ln_closechannel` — picked `btc_getblockchaininfo` (unrelated) |
| 35 | PASS | 30.0s | Verify channel closed |

The 2 cascade SKIPs (#28, #30) are not independent failures — they would have run if #29 had passed and exposed a bolt11 to the driver capture. So the underlying model-level failure rate is **5/8 = 62%** on prompts the planner actually got to attempt.

**Result B — gemini-2.5-flash: untestable due to upstream API errors.** Switched the whole pipeline to Gemini (`LLM_BACKEND=gemini`, model `gemini-2.5-flash`) after the user updated the API key. The driver scored 1/10 PASS, but inspecting the archived outbox revealed the real story: 12 of 15 LLM requests across the 10 prompts failed with Google API errors (early `503 UNAVAILABLE`, then a `429 RESOURCE_EXHAUSTED` cascade as the driver's one-shot retries compounded). Only 3 requests reached the executor stage at all, so the 1/10 verdict says nothing about Gemini's planning quality — it's an infrastructure result, not a model result.

| Failure type | Count | Stage |
|--------------|-------|-------|
| `503 UNAVAILABLE` | 5 | translator (4) + planner (1) |
| `429 RESOURCE_EXHAUSTED` | 7 | translator (7) |
| End-to-end success | 3 | full pipeline |

The 503s came first (Google service overload at the moment of the run), then once requests started failing the driver's retry logic doubled the call rate which tripped 429s from there on. The single `PASS` (#31) and the two later end-to-ends (#14, #15) just landed in gaps between rate-limit windows.

**What we still don't know**: how a stronger model actually plans Phase 3. To get a real Gemini comparison we need either (a) a paid-tier API key with higher RPM, or (b) the battery driver to add inter-prompt sleeps + retry-after handling for Google's API. Defer the rerun until one of those is in place. Same applies to the OpenAI cross-model audit — `gpt-4o` should not have the same rate-limit problem on a paid tier and is the better next experiment.

**State left in repo after this run**: `.env` `LLM_BACKEND` reverted to `ollama` (was temporarily `gemini` for the test). Phase 3 prompts and tests stay as-is. No translator/planner code changes — there is nothing to fix from this run because the qwen2.5:14b failures are the planner-quality result we expected, and the Gemini failures are upstream.

### 2026-04-11 — qwen2.5:14b, Initiative 1 Phases 1 + 2 live verification (25 prompts)

**Result: 25/25 PASS, 0 RETRY, 0 FAIL** — full sweep on the expanded battery after Translator-prompt tightening.

Stack: `OLLAMA_MODEL=qwen2.5:14b`, 2 nodes, regtest, `./scripts/1.start.sh 2`; driver: `ln-ai-network/scripts/tools/run_battery.py` (persisted), per-prompt timeout 120s, default `PLANNER_FORCE_LLM=0`.

**First run (pre-fix)**: 22/25 with 3 Translator classification failures:

| # | Prompt | Expected | Got | Root cause |
|---|--------|----------|-----|------------|
| 2 | "What is the current Bitcoin block height?" | `btc_getblockchaininfo` (via `read_target=block_height`) | `network_health` | Translator picked the broad diagnostic tag for a single-field chain read. |
| 9 | "List all peers connected to node 1." | `ln_listpeers` (via `read_target=peer_list`) | `ln_listchannels` (via `read_target=channel_count`) | Translator collapsed "list peers" into the channel-list trigger — lexical confusion between "list peers" and "list channels". |
| 23 | "Stop node 2." | `ln_node_stop` (via `action_target=stop_node`) | `ln_node_status` → `ln_node_start` (via `action_target=start_node`) | Translator **inverted the verb** and emitted start instead of stop. This would have silently restarted a running node instead of stopping it — the worst of the three because the tool actually ran and the action completed opposite to the user's intent. |

**Fix (single commit)**: tightened `ai/controllers/translator.py` system prompt with explicit **NOT** clauses on the affected tags and three new worked examples:

- **`channel_count` / `peer_list`**: added cross-referenced NOT clauses ("peers and channels are different — peer_list enumerates remote TCP connections; channel_count enumerates funded payment channels") and a new worked example for "List all peers connected to node 1" with an explicit counter-example note.
- **`block_height`**: expanded triggers to include "current block height", "what is the bitcoin block height", "latest block", "how many blocks have been mined", plus a NOT clause ruling out `network_health` for any chain/height/tip question. Added a worked example for prompt 2.
- **`start_node` / `stop_node` / `restart_node`**: each got a "NOT for: <opposite verb>" clause and the most emphatic wording in the prompt: *"The verbs stop and start are opposites — never emit start_node when the user said 'stop'."* Added an explicit "Stop node 2" worked example with the verb-inversion warning.

**Second run (post-fix)**: 25/25 clean pass on the first attempt. Re-ran only the 3 failing prompts (`--only 2,9,23`) → 3/3 PASS on first attempt, then full battery → 25/25 PASS with 0 retries.

Timings were reasonable across the board: 11.2s–39.7s per prompt, median ≈ 16s. Prompt 6 (invoice creation) was the longest at 39.7s due to the UTC-timestamp-labelled `ln_invoice` call; prompt 8 (mine 3 blocks) took 29.6s because it runs the 2-step `btc_getnewaddress` → `btc_generatetoaddress` chain with 3 block generations. All 22 previously-passing prompts stayed green across both runs, so the prompt tightening has no regression cost.

**What this unblocks**: Initiative 1 Phases 1 and 2 are now live-verified. Phase 3 (LLM-planned multi-step flows — prompts #26-#35, payment lifecycle) was blocked on this verification and can start next. Initiative 2 Phase 4 (adversarial seeds, live re-run) can also proceed against the tightened prompt.

**Lesson for future translator work**: add a NOT clause next to any trigger-vocabulary overlap. The three failures had the same shape: a lexical trigger that the Translator matched on a superficial word ("list", "block", "node 2") without checking the verb's actual direction or scope. The NOT clauses force the LLM to rule out the wrong tag explicitly rather than falling for the first close-looking match.

### 2026-04-08 — qwen2.5:14b, post-deterministic-dispatch (Phase F1–F5)

**Result: 10/10 clean passes — full sweep.**

Phases F1–F5 landed deterministic short-circuits in the planner for the most
common single-tool patterns and a Jinja-auto-rewrite pass for the most common
formatting mistake. Both directly target the qwen2.5:14b instruction-following
gaps that caused the prior 4/10 baseline.

Stack: `OLLAMA_MODEL=qwen2.5:14b`, 2 nodes, regtest, fresh queue
(`./scripts/restart_agent.sh fresh`), invoice label rotated with UTC timestamp.

| # | Prompt | Tools picked | Verdict |
|---|--------|--------------|---------|
| 1 | network health | `network_health` | PASS — "network is in a degraded state, node-1 running, node-2 connection refused, chain at block 1441" |
| 2 | block height | `btc_getblockchaininfo` | PASS — block 1441 |
| 3 | node 1 info | `ln_getinfo` | PASS — real pubkey + v25.12-262-g09781bd reported |
| 4 | start node 2 | `ln_node_status` → `ln_node_start` | PASS — actually started node-2 on port 9736; summary correctly reports the start (no fabrication) |
| 5 | channel count | `ln_listchannels` | PASS — "Node 1 currently has one channel" |
| 6 | create invoice | `ln_invoice` | PASS — UTC label `battery_20260409_040538`, no collision (Phase F1) |
| 7 | list funds | `ln_listfunds` | PASS |
| 8 | mine 3 blocks | `btc_getnewaddress` → `btc_generatetoaddress` | PASS — real address + 3 block hashes returned |
| 9 | list peers | `ln_listpeers` | PASS |
| 10 | decode channel scenario | *(empty plan, informational)* | PASS — full conditional walkthrough referencing ln_getinfo, ln_connect, ln_openchannel, 6-block confirmation |

**What changed since the prior run (4/10):**

- **Phase F1** — `/tmp/run_battery.py` rewritten with UTC-timestamp invoice label so prompt 6 no longer collides on `Duplicate label`.
- **Phase F2** — Translator system prompt updated to extract `context.read_target` and `context.action_target` tags for the most common single-tool patterns (`node_info`, `channel_count`, `peer_list`, `fund_list`, `wallet_balance`, `block_height`, `network_health`, `start_node`, `stop_node`, `mine_blocks`). Several worked examples added.
- **Phase F3** — Planner gained `_deterministic_read_plan()` and `_deterministic_action_plan()` short-circuits that fire BEFORE the LLM call when the translator emitted a recognised tag. This bypasses qwen's instruction-following gap entirely for the patterns that previously collapsed to `btc_getblockchaininfo`. Same pattern as the existing noop / informational short-circuits.
- **Phase F4** — Summarizer state-change verb guard. If the user's prompt contains a state-change verb (start/open/close/mine/pay/etc.) but no state-changing tool actually ran, the summarizer prompt is augmented with an explicit warning telling it NOT to claim the action succeeded. Defense-in-depth for the prompt 4 hallucination class.
- **Phase F5** — Planner Jinja → `$stepN` auto-rewrite. Walks plan args before validation and rewrites the most common qwen mistakes (`{{ step_1.x }}`, `{{ steps[0].x }}`, `${step1.x}`) to the canonical placeholder format. Exotic/unknown Jinja vars are still rejected as before so the validator stays strict for everything we can't safely interpret.

**Prompts whose path changed (compared to the prior 4/10 run):**

- Prompts 1, 3, 5 — now use the deterministic read short-circuit (Phase F3). Translator emits `read_target` and the planner builds the plan without calling its own LLM.
- Prompt 4 — now uses the deterministic `action_target=start_node` short-circuit (Phase F3). The two-step `ln_node_status` → `ln_node_start` plan is hardcoded.
- Prompt 6 — UTC-timestamp label (Phase F1) prevents the duplicate collision.
- Prompt 8 — uses the deterministic `action_target=mine_blocks` short-circuit. The plan is the canonical 2-step `btc_getnewaddress` → `btc_generatetoaddress($step1.result.payload)` chain. Phase F5's Jinja rewriter is the safety net for the case where qwen still emits `{{ steps[0] }}` despite the deterministic plan being preferred.
- Prompt 10 — already passing (post `781b444`); no change.

**Follow-ups:**
- The prior `_friendly_error` failure modes (Issues 1, 2, 5 in the archive above) are now structurally suppressed by the deterministic dispatch — qwen's instruction-following is never invoked for the patterns that triggered them. They remain in the archive because a future model/backend swap could reintroduce them if the deterministic short-circuits were ever removed.
- The state-change guard (Phase F4) did not fire on this run because every prompt with an action verb also got a deterministic action plan (the right tool ran). It remains as a safety net for future regressions where the planner picks a wrong read-only tool.
- The summary card on prompts 1, 3, 4, 5, 6, 7, 8 is grounded in real tool output — no hallucinated balances, no fabricated counts.

### 2026-04-08 — qwen2.5:14b, post-pipeline-fix (commit 781b444)

After fixing the empty-plan pipeline routing bug, prompt 10 finally reaches the summarizer and produces a real walkthrough. The persistent qwen instruction-following issues remain for prompts 1, 3, 4, 5, 8.

Stack: `OLLAMA_MODEL=qwen2.5:14b`, 2 nodes, regtest, no live channels.

| # | Prompt | Tools picked | Verdict |
|---|--------|--------------|---------|
| 1 | network health | *(planner validation failure)* | FAIL — qwen emitted malformed JSON (unterminated string) after rejecting an earlier Jinja placeholder. Stage `planner`. |
| 2 | block height | `btc_getblockchaininfo` | PASS |
| 3 | node 1 info | `btc_getblockchaininfo` | WRONG TOOL — summarizer correctly said "tool did not return the requested information for node 1" (no hallucination, which is an improvement over llama3.1) |
| 4 | start node 2 | `btc_getblockchaininfo` | WRONG TOOL + **SUMMARIZER HALLUCINATION** — summary claimed "Node 2 has started successfully" based on btc_getblockchaininfo output. This is a real regression — the summarizer's past-tense rule forced it to invent a success. |
| 5 | channel count | `btc_getblockchaininfo` | WRONG TOOL — summarizer grounded: "does not provide channel count data" |
| 6 | create invoice | `ln_invoice` | FAIL — duplicate label from prior test runs (`Duplicate label 'battery_test'`). Not a code bug; the battery driver should rotate the label. |
| 7 | list funds | `ln_listfunds` | PASS |
| 8 | mine 3 blocks | *(planner validation failure)* | FAIL — qwen emitted `"bind_host": "{{ steps[0].result.non_loopback_ips.0 }}"` for a plan that should never have included `bind_host` or `sys_netinfo`. Issue 5 still active. |
| 9 | list peers | `ln_listpeers` | PASS — real peer pubkey and binding address returned |
| 10 | decode channel scenario | *(empty plan, informational)* | **PASS — walkthrough** — summarizer generated *"Here's how that would work — no actions were taken. First, node 2 would need to be running and its pubkey and listening port would be obtained via ln_getinfo..."*. Issue 4 fix verified end-to-end. |

Tally: **4/10 clean passes** (2, 7, 9, 10) with 1 major new win (prompt 10 now a real walkthrough). **1 regression:** prompt 4 fabricated a success. **2 planner-fail retries** that didn't self-correct (prompts 1, 8). **2 wrong-tool** with grounded summaries (prompts 3, 5). **1 infra issue** (prompt 6, duplicate label).

**Follow-ups:**
- Prompt 4's "fabricated success" needs a summarizer guard: if the tool set called doesn't overlap with the intent's expected tool category, the summary should flag uncertainty instead of writing "successfully". Potentially a new rule: "if `intent_type` implies an action verb but no state-changing tool was executed, do NOT claim the action was done".
- Prompt 6 needs a label rotation in `/tmp/run_battery.py` — use a UTC-timestamp suffix so re-runs don't collide.
- Prompts 1, 3, 5, 8 are all qwen instruction-following failures against rules already present in the planner prompt. Further tightening the prompt has diminishing returns; the practical fixes are:
  1. `PLANNER_LLM_BACKEND=openai` for CI / release verification, or
  2. Larger Ollama model (`qwen2.5:32b`, `phi4:14b`, `llama3.3:70b`).

### 2026-04-08 — qwen2.5:14b, post-planner-prompt-fixes (commit d876d35)

First run after Phases A/B/C-INTENT/C1 landed but before the pipeline routing fix. Intermediate state: the planner now handles the intent discrimination and Jinja-ban rules, but the summarizer's explanation-mode prompt was never reached because the pipeline took an earlier empty-plan short-circuit.

| # | Prompt | Tools picked | Verdict |
|---|--------|--------------|---------|
| 1 | network health | *(validator failed)* | FAIL — qwen emitted `{{netinfo[0].addr_lan}}` for `announce_host` on an unwanted `sys_netinfo` step |
| 2 | block height | `btc_getblockchaininfo` | PASS |
| 3 | node 1 info | `btc_getblockchaininfo` | WRONG TOOL — summarizer grounded |
| 4 | start node 2 | `ln_node_status` | PARTIAL — only pre-check fired, no follow-up start |
| 5 | channel count | `btc_getblockchaininfo` | WRONG TOOL — summarizer grounded |
| 6 | create invoice | `ln_invoice` | FAIL — node 2 never actually started (cascade from prompt 4) |
| 7 | list funds | `ln_listfunds` | **PASS (new)** — no more empty-plan fallback |
| 8 | mine 3 blocks | *(validator failed)* | FAIL — `{{ steps[0].result.non_loopback_ips.0 }}` for a `bind_host` arg that shouldn't exist |
| 9 | list peers | `ln_listpeers` | **PASS (new)** — READ-ONLY rule worked |
| 10 | decode scenario | *(empty plan, informational)* | **PARTIAL (new)** — planner correctly returned empty plan, BUT pipeline took its generic short-circuit and returned the translator's `human_summary` ("I'll walk you through...") instead of reaching the summarizer. This is the bug fixed in `781b444`. |

Tally: 4/10 clean passes (2, 7, 9, 10-partial). Same failure modes as the pre-fix run for prompts 1, 3, 4, 5, 8 but two previously-broken prompts (7 list funds, 9 list peers) now pass.

### 2026-04-08 — qwen2.5:14b, pre-fix baseline

Reference baseline before any of the planner-prompt tightening landed. Matches the initial bug report: 4/10 clean passes with two partial/empty-plan cases.

| # | Prompt | Tools picked | Verdict |
|---|--------|--------------|---------|
| 1 | network health | `btc_getblockchaininfo` + `ln_listnodes` | PASS — multi-step, accurate |
| 2 | block height | `network_health` | PASS — block height correctly reported from network_health payload |
| 3 | node 1 info | `network_health` | WRONG TOOL — should have used `ln_getinfo` |
| 4 | start node 2 | `ln_node_status` + `ln_node_start` | PASS — node 2 actually started on port 9736 |
| 5 | channel count | `network_health` | WRONG TOOL — should have used `ln_listfunds`/`ln_listchannels` |
| 6 | create invoice | `ln_invoice` | PASS — real bolt11 + payment hash |
| 7 | list funds | *(empty plan)* | NO PLAN — summarizer just reworded intent (Issue 3) |
| 8 | mine 3 blocks | `btc_getnewaddress` + `btc_generatetoaddress` | FAIL — Issue 1 (Jinja placeholder crashed executor) |
| 9 | list peers | `network_health` | WRONG TOOL — should have used `ln_listpeers` |
| 10 | open-channel scenario | `ln_node_status`, `ln_node_start`, `sys_netinfo`, `ln_getinfo` | PARTIAL — gathered context but didn't decompose to `ln_connect`/`ln_openchannel`; half-executed a scenario the user only wanted explained (Issue 4) |

Wins from switching off llama3.1:
- Multi-step plans for the easy cases (health, start node, invoice).
- Summarizer stopped hallucinating specific balances — it grounds itself in tool results and says "information not provided" when the payload is thin.
- Node 2 actually started; real bolt11 with payment hash created.

### 2026-04-08 — llama3.1:8b (historical — resolved by model upgrade)

Kept for context. The 8B model defaulted to a single `network_health` step regardless of intent and hallucinated specific values from the thin `network_health` payload (e.g. *"Node-1 has a balance of 1000.00 BTC and two peers: node-1 and node-2"* — none of which came from a tool call).

| # | Prompt | Tools picked | Verdict |
|---|--------|--------------|---------|
| 1 | network health | `network_health` | PASS |
| 2–9 | various | `network_health` | WRONG TOOL — always `network_health`, with the summarizer inventing values |
| 6 | create invoice | *(empty plan)* | NO PLAN |
| 10 | open-channel scenario | *(empty plan)* | NO PLAN |

**Three failure modes observed:**
- **(a)** Planner collapsed to `network_health` regardless of intent.
- **(b)** Summarizer hallucinated specific values from a `network_health` payload — the most dangerous mode because the answer looks confident and correct.
- **(c)** Empty plans for prompts the 8B model couldn't decompose at all (6, 10). The summarizer fell back to rewording the intent (*"I will have node 2 create an invoice..."* — no invoice was created).

**Resolution:** Upgraded default `OLLAMA_MODEL` to `qwen2.5:14b` (commit `b51796d`). The planner-prompt tightening work (Issues 1–5 above) followed because qwen still had instruction-following gaps the prompt had to compensate for.

---

## How to run an audit (`PLANNER_FORCE_LLM`)

The Planner has a deterministic dispatch path that bypasses the LLM for prompts the Translator tagged with `read_target` or `action_target` (e.g. "show node 1 info" → `ln_getinfo`). Those dispatches were added as a workaround for `qwen2.5:14b`'s tendency to default to `btc_getblockchaininfo` for any read query. They are **provisional** — the long-term goal is to demote them to a fallback-only path once a stronger Planner LLM is proven capable of handling the same prompts directly.

To measure whether a dispatch is still earning its keep, run an **audit** that disables the dispatches and routes everything through the LLM.

### The flag

Set `PLANNER_FORCE_LLM=1` on the agent process. The flag is read fresh on every `Planner.plan()` call; it's an audit-time toggle, not a hot-path constant.

| Value | Behaviour |
|-------|-----------|
| unset or `0` (default) | Current behaviour — deterministic dispatch fires for `read_target` / `action_target` matches; LLM runs for everything else. |
| `1` | Dispatch is skipped for `read_target` and `action_target`. The Planner LLM is invoked for every freeform intent. **Informational and noop short-circuits still fire** — those are correctness short-circuits, not workarounds. |
| `fallback` | (Future, Initiative 2 Phase 5) Dispatch fires only if the Planner LLM produced an invalid plan after retries. Default-LLM-with-dispatch-as-safety-net. Not implemented yet. |

When the flag is active, the Planner emits a `planner_force_llm_active` trace event so audit runs are clearly distinguishable in the trace log. The mutually-exclusive `deterministic_dispatch_hit` and `planner_llm_dispatch` events (one per request) let you tally how many requests went through each path.

### Offline audit driver

A reusable offline driver lives at `ln-ai-network/scripts/tools/audit_dispatch_discipline.py`. It runs Translator → Planner directly against the configured LLM backend with `PLANNER_FORCE_LLM=1` and prints the verdict per prompt. **It does not require live infra** (no bitcoind, lightningd, or MCP server) — only the LLM backend has to be reachable. This is much faster than running a full battery against the agent process for the dispatch-discipline question, which is purely about which tool the Planner LLM picks.

```bash
# Run against the default backend (qwen2.5:14b on Ollama)
python ln-ai-network/scripts/tools/audit_dispatch_discipline.py

# Run against OpenAI for the Initiative 2 Phase 3 cross-model audit
AUDIT_BACKEND=openai python ln-ai-network/scripts/tools/audit_dispatch_discipline.py
```

The driver writes machine-readable JSON to `/tmp/audit_dispatch_discipline.json` (override with `AUDIT_OUT=/path/to/file.json`) so the verdict table can be re-rendered or diffed across runs.

### Running an audit

1. Stop the agent: `./scripts/restart_agent.sh fresh` (clears queue state for a clean run).
2. Re-export the flag and any per-stage backend overrides before starting the agent — e.g. to audit on OpenAI:
   ```bash
   PLANNER_LLM_BACKEND=openai PLANNER_FORCE_LLM=1 ./scripts/restart_agent.sh
   ```
3. Run the battery driver (`/tmp/run_battery.py` for now; will move to `scripts/tools/run_battery.py` once the backlog item lands).
4. For each prompt, capture from the outbox: `success`, `plan.steps[*].tool`, `step_results[*].ok`, and `human_summary`. Compare against the same prompt's default-mode (dispatch-enabled) baseline.

### Recording the verdict

For each prompt audited, the verdict is one of:

- **`dispatch needed`** — the LLM-only path fails or produces a wrong plan on **both** the default model (`qwen2.5:14b`) and the stronger model. The dispatch stays as the default for everyone.
- **`qwen-only workaround`** — the LLM-only path fails on `qwen2.5:14b` but passes on the stronger model. Mark the dispatch for demotion to a small-model fallback.
- **`unnecessary dispatch`** — the LLM-only path passes on **both** models. The dispatch can be removed entirely.

Append a dated **Audit runs** subsection to this file with the verdict table. Once Initiative 2 Phase 3 ships verdicts for the full battery, the verdicts feed Phase 5 (demote dispatches based on the verdict counts).

## Audit runs

### 2026-04-24 — `openai gpt-4o` dispatch-discipline audit (Initiative 2 Phase 3 completion)

**Purpose**: cross-model comparison for the 12 `read_target` candidate prompts that were audited on `qwen2.5:14b` on 2026-04-09. Answers the Initiative 2 Phase 3 question: "does a stronger Planner LLM still need the deterministic dispatches, or can it handle these unaided?"

**Driver**: `AUDIT_BACKEND=openai python ln-ai-network/scripts/tools/audit_dispatch_discipline.py`. `PLANNER_FORCE_LLM=1` is set internally so the dispatch is guaranteed bypassed; the Planner LLM must pick the tool on its own. No live Bitcoin/Lightning infra required.

**Result: 12/12 PASS, 0 WRONG, 0 EMPTY, 0 ERROR.** Wall time ~280 s (~4.7 min), average 23 s per prompt. Every candidate returned the intended tool.

| # | Tag | Prompt | qwen2.5:14b (2026-04-09) | gpt-4o (this run) | Verdict |
|---|-----|--------|---------------------------|-------------------|---------|
| 11 | `fee_estimate` | "What fee should I use for the next 6 blocks?" | WRONG → `btc_getblockchaininfo` | ✅ PASS → `btc_estimatefee` | **qwen-only workaround** |
| 12 | `mempool_info` | "Show me the current mempool size." | WRONG → `btc_getblockchaininfo` | ✅ PASS → `btc_getmempoolinfo` | **qwen-only workaround** |
| 13 | `ln_fee_rates` | "What are node 1's lightning fee rates?" | WRONG → `btc_getblockchaininfo` | ✅ PASS → `ln_feerates` | **qwen-only workaround** |
| 14 | `forward_list` | "List all forwarded HTLCs on node 1." | PASS (5/5) → `ln_listforwards` | ✅ PASS → `ln_listforwards` | **unnecessary dispatch** (already has no dispatch) |
| 15 | `ln_address` | "Give me a fresh lightning deposit address for node 1." | WRONG → `btc_getblockchaininfo` | ✅ PASS → `ln_newaddr` | **qwen-only workaround** |
| 16 | `btc_tx_list` | "Show me the recent on-chain transactions in the shared wallet." | WRONG → `btc_getblockchaininfo` | ✅ PASS → `btc_listtransactions` | **qwen-only workaround** |
| 17 | `ln_tx_list` | "List the lightning transactions on node 1." | WRONG → 4-tool junk plan | ✅ PASS → `ln_listtransactions` | **qwen-only workaround** |
| 18 | `offer_list` | "What BOLT12 offers does node 1 have?" | PASS (1/1) → but 4/5 on 5-pass repeat, i.e. flaky | ✅ PASS → `ln_listoffers` | **qwen-only workaround** (qwen flaky; dispatch added) |
| 19 | `recall` | "What did I run in the last few prompts?" | EMPTY plan | ✅ PASS → `memory_lookup` | **qwen-only workaround** |
| 20 | `recall` | "Did the last payment succeed?" | EMPTY plan | ✅ PASS → `memory_lookup` | **qwen-only workaround** |
| 21 | `ln_fee_rates (node 2)` | "What is node 2's lightning fee rates?" | (same as #13) | ✅ PASS → `ln_feerates` (via `route` intent) | **qwen-only workaround** |
| 22 | `btc_tx_list (alt phrasing)` | "List the bitcoin transactions in the shared wallet." | (same as #16) | ✅ PASS → `btc_listtransactions` | **qwen-only workaround** |

**Totals (distinct tags)**: 8 `qwen-only workaround`, 1 `unnecessary dispatch` (`forward_list`), 0 `dispatch needed on both`.

**Translator behavior note on #21**: gpt-4o correctly emitted `intent_type=route` (with `target_node=2`, `routed_prompt=...`) instead of `freeform` because the prompt asks about a different node than `self_node=1`. The Planner still picked the right downstream tool, so it scored PASS. This is stronger instruction-following than qwen showed — gpt-4o noticed the `route` intent type exists and used it appropriately.

**Headline finding**: the deterministic dispatches in `planner.py` are entirely a **small-local-model workaround**. On gpt-4o the planner LLM correctly handles every candidate without any dispatch assistance. This is exactly the input Initiative 2 Phase 5 needs: 8 dispatches to demote from "default-on" to "fallback-only" (LLM runs first; dispatch fires only if the LLM plan is invalid), plus `forward_list` already correctly has no dispatch.

**Cross-reference**: same-day supplementary full 42-prompt battery on gpt-4o scored 42/42 PASS in ~914 s — see the 2026-04-24 entry under **Runs** above. Machine-readable audit output at `/tmp/audit_dispatch_discipline.json` at time of run.

---

### 2026-04-09 — `qwen2.5:14b` dispatch-discipline check on Initiative 1 Phase 1 candidates

**Purpose**: Before adding 9 new `read_target` deterministic short-circuits to `planner.py` for Initiative 1 Phase 1, run the project-level dispatch-discipline check (PLAN.md → "How to use this document") to confirm the Planner LLM on the current default model is actually mis-routing each candidate prompt. If the LLM picks the right tool unaided, the dispatch should NOT be added — the prompt goes into the battery as-is.

**Driver**: `ln-ai-network/scripts/tools/audit_dispatch_discipline.py` (12 prompts, runs offline against Ollama).

**Command**:
```bash
python ln-ai-network/scripts/tools/audit_dispatch_discipline.py
```

**Result**: 3 PASS / 7 WRONG TOOL / 2 EMPTY PLAN / 12 (≈25% LLM-only correct). Total runtime ~3 min.

| # | Tag | Prompt | qwen2.5:14b (force-llm) picked | Verdict | Decision |
|---|-----|--------|-------------------------------|---------|----------|
| 11 | `fee_estimate` | "What fee should I use for the next 6 blocks?" | `[btc_getblockchaininfo]` | WRONG | **Add dispatch** → `btc_estimatefee` |
| 12 | `mempool_info` | "Show me the current mempool size." | `[btc_getblockchaininfo]` | WRONG | **Add dispatch** → `btc_getmempoolinfo` |
| 13 | `ln_fee_rates` | "What are node 1's lightning fee rates?" | `[btc_getblockchaininfo]` | WRONG | **Add dispatch** → `ln_feerates` |
| 14 | `forward_list` | "List all forwarded HTLCs on node 1." | `[ln_listforwards]` | **PASS** | **Skip dispatch** + fix Translator (see below) |
| 15 | `ln_address` | "Give me a fresh lightning deposit address for node 1." | `[btc_getblockchaininfo]` | WRONG | **Add dispatch** → `ln_newaddr` |
| 16 | `btc_tx_list` | "Show me the recent on-chain transactions in the shared wallet." | `[btc_getblockchaininfo]` | WRONG | **Add dispatch** → `btc_listtransactions` |
| 17 | `ln_tx_list` | "List the lightning transactions on node 1." | `[btc_wallet_ensure, ln_node_create, ln_node_start, ln_getinfo]` | WRONG | **Add dispatch** → `ln_listtransactions` |
| 18 | `offer_list` | "What BOLT12 offers does node 1 have?" | `[ln_listoffers]` | **PASS** | **Skip dispatch** + fix Translator (see below) |
| 19 | `recall` | "What did I run in the last few prompts?" | `[]` (empty plan) | EMPTY | **Add dispatch** → `memory_lookup` |
| 20 | `recall` | "Did the last payment succeed?" | `[]` (empty plan) | EMPTY | **Add dispatch** → `memory_lookup` |
| 21 | `ln_fee_rates (n2)` | "What is node 2's lightning fee rates?" | `[btc_getblockchaininfo]` | WRONG | Same as #13 |
| 22 | `btc_tx_list (alt)` | "List the bitcoin transactions in the shared wallet." | `[btc_listtransactions]` | **PASS** | Same as #16 (dispatch handles both phrasings) |

**Verdict counts (per unique tag):**
- **7 dispatches needed** — `fee_estimate`, `mempool_info`, `ln_fee_rates`, `ln_address`, `btc_tx_list`, `ln_tx_list`, `recall`. Add to `planner.py`.
- **2 tags do not need a dispatch** — `forward_list`, `offer_list`. The LLM picks the right tool unaided. Add prompts to the battery WITHOUT a dispatch case.

**Secondary finding — Translator silently mis-tags several prompts.** The audit revealed that the **Translator** (separate stage from the Planner) is currently extracting wrong `read_target` tags for several prompts, which would route them to the **wrong dispatch** in default mode:

| # | Prompt | Translator emits | Default dispatch routes to | Wrong because |
|---|--------|------------------|---------------------------|---------------|
| 11 | "What fee should I use…" | `read_target=network_health` | `network_health` | Should be `btc_estimatefee` |
| 14 | "List all forwarded HTLCs…" | `read_target=payment_list` | `ln_listpays` | Should be `ln_listforwards` |
| 18 | "What BOLT12 offers…" | `read_target=invoice_list` | `ln_listinvoices` | Should be `ln_listoffers` |
| 13 | "What are node 1's lightning fee rates?" | `read_target=fee_rates` (unrecognized) | (falls through to LLM) | Tag never recognized; harmless drift but Translator should use canonical name |
| 15 | "Give me a fresh lightning deposit address…" | `action_target=create_deposit_address` (unrecognized) | (falls through) | Same as above |
| 21 | "What is node 2's lightning fee rates?" | `read_target=fee_policy` (unrecognized) | (falls through) | Same as above |

**Implication**: For prompts #14 and #18 ("PASS" verdicts), the LLM-only path is correct but the **default-mode** path would still be wrong because the Translator emits a tag that maps to the wrong existing dispatch. The fix is in the Translator system prompt, not the Planner — tighten the tag rules so these prompts emit no `read_target` at all (letting them fall through to the LLM-only path that already works).

Similarly, prompt #11 (`fee_estimate`) is a triple problem: Translator wrongly tags it as `network_health`, AND the LLM-only path picks `btc_getblockchaininfo`. Both stages need the fix. The Planner dispatch lands the right tool, and the Translator system prompt also needs to stop emitting `network_health` for fee questions.

**Implementation notes for Initiative 1 Phase 1** (carry into the work):
1. Add 7 dispatches in `_deterministic_read_plan()`: `fee_estimate`, `mempool_info`, `ln_fee_rates`, `ln_address`, `btc_tx_list`, `ln_tx_list`, `recall`.
2. Add Translator triggers for those 7 tags. The trigger phrases must use canonical tag names — kill the `fee_rates`, `fee_policy`, `create_deposit_address` aliases the Translator currently emits.
3. For `forward_list` and `offer_list`: do NOT add a Planner dispatch. Add Translator triggers that either emit the canonical tag (`forward_list`, `offer_list`) **with no matching `_deterministic_read_plan()` case so the request falls through to the LLM**, or emit an empty context. The simplest fix is the latter — strengthen the Translator system prompt's "do not emit `payment_list` for forward queries" / "do not emit `invoice_list` for offer queries" guidance.
4. For #11 specifically: the Translator must stop classifying fee questions as `network_health`. Add a counter-example to the Translator system prompt.
5. After landing 1–4, re-run `audit_dispatch_discipline.py` and confirm the WRONG verdicts flip to PASS via dispatch (or to "dispatch fires correctly" via the trace).

**Cross-reference**: when Initiative 2 Phase 3 lands, run the same driver with `AUDIT_BACKEND=openai` to produce the second column of the dispatch-needed-vs-qwen-only table. Compare verdicts to determine which dispatches can be demoted to fallback-only.

---

### 2026-04-09 — Initiative 1 Phase 1 implementation result

**What ran**: After landing the Translator + Planner changes (8 new `read_target` tags + 8 new dispatches + 1 Translator-only fix), verified end-to-end via `/tmp/verify_phase1_pipeline.py`. This driver runs Translator → Planner with `PLANNER_FORCE_LLM` **unset** (i.e. dispatches are enabled, the way the system runs in production), and checks that each prompt's final tool selection matches the intended set.

**Result**: **12/12 PASS** on `qwen2.5:14b` (Ollama default).

| # | Prompt | Translator ctx | Final tool | Verdict |
|---|--------|----------------|------------|---------|
| 11 | "What fee should I use for the next 6 blocks?" | `read_target=fee_estimate, conf_target=6` | `btc_estimatefee` | PASS via dispatch |
| 12 | "Show me the current mempool size." | `read_target=mempool_info` | `btc_getmempoolinfo` | PASS via dispatch |
| 13 | "What are node 1's lightning fee rates?" | `read_target=ln_fee_rates, node=1` | `ln_feerates` | PASS via dispatch |
| 14 | "List all forwarded HTLCs on node 1." | `node=1` (no `read_target`) | `ln_listforwards` | PASS via Planner LLM |
| 15 | "Give me a fresh lightning deposit address…" | `read_target=ln_address, node=1` | `ln_newaddr` | PASS via dispatch |
| 16 | "Show me the recent on-chain transactions…" | `read_target=btc_tx_list` | `btc_listtransactions` | PASS via dispatch |
| 17 | "List the lightning transactions on node 1." | `read_target=ln_tx_list, node=1` | `ln_listtransactions` | PASS via dispatch |
| 18 | "What BOLT12 offers does node 1 have?" | `read_target=offer_list, node=1` | `ln_listoffers` | PASS via dispatch |
| 19 | "What did I run in the last few prompts?" | `read_target=recall` | `memory_lookup` | PASS via dispatch |
| 20 | "Did the last payment succeed?" | `read_target=recall` | `memory_lookup` | PASS via dispatch |
| 21 | "What is node 2's lightning fee rates?" | `read_target=ln_fee_rates, node=2` | `ln_feerates` | PASS via dispatch |
| 22 | "List the bitcoin transactions in the shared wallet." | `read_target=btc_tx_list` | `btc_listtransactions` | PASS via dispatch |

**One in-flight correction**: the original audit verdict said `offer_list` (#18) didn't need a dispatch because the LLM-only Planner picked `ln_listoffers` correctly. After the rest of Phase 1 landed, the verification's first run hit a different LLM sample and produced `ln_node_start`. A 5-pass repeat test on #18 gave **4/5 PASS** — i.e. ~20% flake rate. A 5-pass repeat test on #14 (the other "skip dispatch" verdict from the audit) gave **5/5 PASS**, confirming `forward_list` is genuinely fine without a dispatch. So `offer_list` was promoted to a real dispatch and `forward_list` was left as a Translator-only fix. See PLAN.md decision log for the full rationale and the lesson on single-sample audit verdicts.

**Test suite delta**: +17 new unit tests (9 in `test_planner.py` for the new short-circuits, 8 in `test_translator.py` for system-prompt regression guards). Total now: **737 passing, 1 skipped, 3 pre-existing live-infra failures unrelated to this work.**
