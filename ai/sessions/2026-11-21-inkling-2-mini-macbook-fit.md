# Session 2026-11-21 — Inkling-2-Mini on the MacBook: fit assessment

*Participants: user + Kilo agent (ask mode, consultation only — no wiki edits this session).*

## User prompts (condensed)

1. Planning to run `mlx-community/Inkling-2-Mini-oQ4e-mtp` (via oMLX, omlx.ai) on the MacBook as (a) a Kilocode/VSCode subagent and (b) a scheduled night-time "test engineer" creating and running tests across all repos. Good idea? Better quantisation?
2. Save the session to ai/sessions.

## Findings

- **Model** (HF card): Inkling-2-Mini = thinkingmachines' looped multimodal MoE — 26.4B total / 3.4B active, middle 12 layers run ×3 per token, text+image, 1M context. The `-mtp` bundle ships a DSpark draft for speculative decoding; needs **oMLX ≥ 0.8.1**.
- **Checkpoint sizes (from HF cards):**
  - `oQ4e-mtp`: 16.8 GB (15.7 GiB), ~4.9 bpw — experts 4–4.5-bit, attention/embeddings 8-bit, router+loop FP32, vision+draft BF16.
  - `oQ3e-mtp`: 13.7 GB (12.7 GiB), ~3.95 bpw — experts 3-bit.
  - `mlx-community/Inkling-2-Mini-4bit`: 17.3 GB (HF's "7B params" label is metadata confusion from the looped-MoE config; requires unmerged mlx-lm PR #2086, no draft model).
- **MacBook verdict (M3 Pro, 18 GB):** oQ4e at 16.8 GB does not fit — default GPU wired limit ~11–13 GB, and even raised, weights alone exceed a sane budget with VSCode+Kilo resident. Confirms the wiki's existing ceiling ([macbook.md](../docs/macbook.md): "realistic ceiling 12–14B Q4"). MoE memory scales with *total* params; all experts stay resident.
- **Only same-model option on MacBook:** oQ3e-mtp, with `iogpu.wired_limit_mb≈15000`, context capped ~16–32k, machine otherwise idle. Expect degraded tool-call reliability at 3-bit-class experts — risky for unattended agent work.
- **Better split of roles:**
  - MacBook → 7–8B dense Q4 (or ≤~10 GB small MoE) as interactive Kilocode subagent via OpenAI-compatible `/v1` (oMLX/LM Studio/mlx_lm.server all expose it).
  - Night test-engineer → Powerserver V100 32 GB with the **GGUF** build (unsloth/ggml-org) at Q4_K_M via llama.cpp — not the oQ MLX checkpoint (MLX is Apple-only; NVFP4 experts are Blackwell-only; GGUF converts to K-quants). Blocker: V100 still mid-bring-up (ReBarUEFI/GRUB, [powerserver.md](../docs/powerserver.md)).
  - Capability note: 3.4B-active is small for autonomous multi-file test synthesis; a 30B-class dense Q4 on the V100 (already in the capacity plan) would write better tests.
- Misc: 1M context irrelevant on 18 GB (KV won't fit); laptop night runs need `caffeinate -s` + charger (lid-close sleeps inference).

## Actions

- Session log saved (this file). No wiki changes — no hardware changed; findings recorded here for future ingest if the user adopts a setup.

## Next steps (if pursued)

1. MacBook subagent: pick + benchmark a 7–8B Q4 with a realistic Kilo-length prompt; record tok/s in macbook.md.
2. Night engineer: finish V100 bring-up, then load Inkling-2-Mini GGUF Q4_K_M (or 30B-class dense Q4) via llama.cpp; schedule runs (cron/launchd) against the repo list.
3. If trialing oQ3e-mtp on the MacBook anyway: install oMLX ≥ 0.8.1, raise wired limit, verify load + a long agentic prompt before scheduling anything unattended.
