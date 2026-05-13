---
title: LLM Results Summary
parent: Runbooks
nav_order: 2
---

# LLM & Battery Results Summary

> **What this is**: the curated reference of LLM evaluation results
> for `lightning-network-ai-agents`. It gives a single comparable view of
> every authoritative run we've done.
>
> **What this is not**: the raw experiment log. The full chronological journal
> (every dated run, including failed/exploratory runs, intermediate fix cycles,
> per-prompt failure traces, and CLN error text) lives in
> [`prompt-battery.md`](prompt-battery.md). Every row in the headline table
> below links back to its canonical source subsection there.

---

## Headline comparison table

| Date | Model / backend | Mode | Scope | Result | Retries | Wall time | Fallback / dispatch behaviour | Interpretation | Source |
|------|-----------------|------|-------|--------|---------|-----------|-------------------------------|----------------|--------|
| 2026-04-22 | qwen2.5:7b on local GPU (RTX 5070, `num_gpu=999`) | Legacy (dispatch-first, default) | Full 1–42 prompt battery, clean-runtime protocol | **42/42 PASS, 0 FAIL** | 0 | ~486 s (~8.1 min) | Dispatch fired ~15+ times for tagged prompts (cheap path); LLM ran for the rest | **Authoritative local baseline.** A clean wiped-runtime + fresh stack boot is mandatory for reproducible scores. | [`prompt-battery.md` → 2026-04-22 entry](prompt-battery.md) |
| 2026-04-24 | OpenAI `gpt-4o` (cloud) | Legacy (dispatch-first, default) | Full 1–42 prompt battery, clean-runtime protocol | **42/42 PASS, 0 FAIL** | 0 | ~914 s (~15.2 min) | Dispatch still fires on tagged prompts (backend-agnostic); LLM ran for the rest | **Cross-model parity.** Pass-rate parity with the local qwen baseline; ~1.9× slower end-to-end because every LLM call is network-bound. Backend-agnostic pipeline. | [`prompt-battery.md` → 2026-04-24 OpenAI full-battery entry](prompt-battery.md) |
| 2026-04-24 | OpenAI `gpt-4o` (cloud) | `PLANNER_FORCE_LLM=1` (force-LLM, no dispatch) | 12-prompt dispatch-discipline audit, offline (no live infra) | **12/12 PASS** | n/a | ~280 s (~4.7 min) | All dispatches bypassed by design; LLM picked the right tool every time | **Initiative 2 Phase 3 verdict**: 8 of 9 dispatch tags are `qwen-only workaround`, 1 (`forward_list`) is `unnecessary dispatch` (already dispatch-free). On a frontier model the dispatches are not needed. | [`prompt-battery.md` → 2026-04-24 OpenAI dispatch audit](prompt-battery.md) |
| 2026-04-24 | qwen2.5:7b on local GPU | `PLANNER_FORCE_LLM=fallback` | Full 1–42 prompt battery, clean-runtime protocol | **40/42 PASS, 2 FAIL** (#36 UTXO shortage, #38 cascade) | 0 | ~643 s (~10.7 min) | 41 LLM-only planner calls; **0 `planner_fallback_dispatch_fired`** events; 0 initial dispatch hits | **Initiative 2 Phase 5 first validation.** qwen planned everything LLM-only without needing the safety net. The 2 failures were executor / topology-sensitive, not planner regressions. | [`prompt-battery.md` → 2026-04-24 qwen fallback-mode entry](prompt-battery.md) |
| 2026-04-27 | qwen2.5:7b on local GPU | `PLANNER_FORCE_LLM=fallback` | Full 1–42 prompt battery, clean-runtime protocol | **40/42 PASS, 2 FAIL** (#14 `ln_listforwards` arg hallucination, #32 `ln_listpays` vs `ln_listinvoices` direction) | 0 | ~579 s (~9.7 min) | 41 LLM-only planner calls; **0 `planner_fallback_dispatch_fired`** events; 0 initial dispatch hits | **Initiative 2 Phase 5 second validation, repeat-variance check.** Same headline result as the prior run (40/42, 0 fallback fires) but the two failures landed on different prompts. Confirms (a) fallback machinery is stable, (b) the residual ~5% delta vs the legacy baseline is dominated by qwen-side variance and downstream-environment flakes that drift between runs, not by Phase 5 itself. | [`prompt-battery.md` → 2026-04-27 qwen fallback-mode entry](prompt-battery.md) |

**Glossary** (for readers approaching the table cold):

- **Legacy mode** — the default behaviour: when the Translator emits a `read_target` / `action_target` tag, the Planner short-circuits to a hardcoded one-tool plan instead of calling the LLM. Cheap, deterministic, but bypasses the model on those prompts.
- **`PLANNER_FORCE_LLM=1`** (audit mode) — every freeform request is routed through the Planner LLM; the dispatch is skipped entirely. Used for dispatch-discipline audits.
- **`PLANNER_FORCE_LLM=fallback`** (Phase 5 mode) — LLM-first planning. The dispatch fires *only* when the LLM retry loop exhausts AND the Translator set a recognised tag. Default-LLM-with-dispatch-as-safety-net mode.
- **Clean-runtime protocol** — wipe `runtime/{bitcoin,lightning,agent,…}` then fresh `./scripts/1.start.sh 2`. Required for reproducible payment-flow results because accumulated topology / liquidity state distorts later prompts.

---

## Adversarial Translator-classification (Initiative 2 Phase 4)

Smaller comparison: an adversarial set of 10 multi-step / conditional prompts (seeds A–J) tests whether the Translator drifts and emits a single-tool dispatch tag for prompts that should be planner-decomposed.

| Date | Model | Variant | Result | Source |
|------|-------|---------|--------|--------|
| 2026-04-23 | qwen2.5:7b on local GPU | Baseline system prompt (counter-examples A/B/C only) | **4/10 PASS, 6/10 DRIFT** | (live throwaway, results in `/tmp/phase4_live_rerun.json` at the time) |
| 2026-04-23 | qwen2.5:7b on local GPU | + counter-examples D/E/F (additive prompt hardening) | **5/10 PASS, 5/10 DRIFT** | (same throwaway driver) |
| 2026-04-23 | qwen2.5:7b on local GPU | + structural reorder (prohibition before tag list) | **5/10 PASS, 5/10 DRIFT** (one seed flipped each direction; net unchanged) | (same throwaway driver) |
| 2026-04-23 | hermes3:8b on local GPU | Translator-only swap (Planner+Summarizer stayed on qwen) | **6/10 PASS, 4/10 DRIFT** (one extraction-quality regression on seed J) | (same throwaway driver) |

**Headline finding (Phase 4)**: the qwen2.5:7b Translator hits a 5/10 ceiling on this adversarial set with prompt-only hardening. Drift clusters: question-then-conditional-action, read-verb-and-plural-filtered-action, state-change-then-read. Hermes3:8b improves by one seed but introduces an extraction-quality regression. **None of the 10 adversarial prompts appear in the production 1–42 battery**, so this is a latent-risk surface, not a production regression.

---

## Key findings

1. **qwen2.5:7b on a single consumer GPU achieves 42/42 on the production battery.** With the deterministic dispatches and counter-example-tightened system prompts in place, a local 8 GB-VRAM-fit model is sufficient for end-to-end Lightning payment, channel, and informational workflows on regtest. The official baseline is qwen2.5:7b + GPU.
2. **Pipeline is backend-agnostic.** Switching `LLM_BACKEND=openai` with no other changes produces 42/42 on `gpt-4o`, with no prompt edits and no schema edits. The cost is ~1.9× wall time (network-bound) and a real API bill.
3. **The deterministic dispatches are local-small-model workarounds, not universal requirements.** A 12-prompt dispatch-discipline audit shows 8 of 9 distinct tags are `qwen-only workaround` (gpt-4o handles them unaided); the 9th (`forward_list`) is `unnecessary dispatch` (already correctly has no dispatch case in the planner). On a stronger model the dispatches are dead weight.
4. **Phase 5 reframes the dispatches as a safety net.** `PLANNER_FORCE_LLM=fallback` makes the LLM the primary planner and only triggers the dispatch when the LLM retry loop exhausts on a recognised tag. Across two clean-runtime fallback-mode 42-prompt runs (2026-04-24 and 2026-04-27), the safety net never triggered: qwen planned all 41 non-informational prompts LLM-only. The dispatches now exist as an honest correctness backstop, not as the primary planning path.
5. **Clean-runtime hygiene is non-optional for reproducible battery scores.** A run starting from accumulated `runtime/` state (extra channels, spent liquidity, stale agent queue) can score 38/42 where a wiped-runtime run scores 42/42 — same code, same prompts. Always wipe before benchmarking.
6. **Translator-classification has a residual 5/10 ceiling on qwen2.5:7b for adversarial multi-step phrasings** that don't appear in production. Prompt-only hardening hit diminishing returns; Hermes3:8b is marginally better (6/10) but introduces extraction-quality regressions. This is a documented latent risk, not a production failure.

---

## How to update this file

This file is the **curated summary** of authoritative runs. The maintenance rule:

1. **Authoritative runs go in both places.** Whenever you add a dated entry to [`prompt-battery.md`](prompt-battery.md) for an authoritative run (one that's stable, reproducible, and worth citing), append a new row to the **Headline comparison table** above pointing back to that subsection.
2. **Exploratory / debugging runs stay in `prompt-battery.md` only.** Don't pollute the headline table with intermediate fix cycles, single-prompt rerun probes, or stack-debug runs. Those belong in the raw log.
3. **The Glossary stays in sync.** If a new mode is introduced (e.g. a future `PLANNER_FORCE_LLM=warn` or a new model-routing flag), add a glossary entry the same day the mode ships.
4. **Key findings are durable claims, not status-tracking.** Update them only when a finding is materially different (e.g. a new finding lands, an old finding is invalidated). Don't add per-run noise here.
5. **Adversarial Translator runs share a sub-table** because they evaluate a separate dimension of the system.
6. **Source-link every row.** Every table row points back to its `prompt-battery.md` subsection so a reader can drill into the raw evidence.

The corresponding rule in `PLAN.md`'s workflow section says the same thing from the planning side: future authoritative runs must update both `prompt-battery.md` and this file in the same commit.
