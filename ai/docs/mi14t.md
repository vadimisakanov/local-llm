# Mi 14T — Pocket LLM Phone

Xiaomi Mi 14T, the mobile tier of the setup: on-device GGUF inference via PocketPal AI, and a potential thin client to the [Powerserver](powerserver.md).

*Source: [README.md](../../README.md)* · *See also: [Overview](overview.md), [MacBook](macbook.md)*

## Specifications

| Component | Details |
|---|---|
| Model | Xiaomi Mi 14T (2024) |
| SoC | MediaTek Dimensity 8300-Ultra (4 nm, octa-core; Mali-G615 GPU + NPU) |
| RAM | 12 GB LPDDR5X |
| "Extended" RAM | +6 GB storage-backed memory extension (the "12+6" in the inventory) |
| Disk | 512 GB UFS |
| Preferred software | PocketPal (README) |

## Component notes (local-LLM perspective)

### SoC — Dimensity 8300-Ultra
- CPU ≈ 1×3.35 GHz + 3×3.2 GHz (A715-class) + 4×2.2 GHz (A510-class); GPU Mali-G615 (modest — llama.cpp-based apps on Android mostly use the CPU backend).
- LPDDR5X bandwidth is a small fraction of even the [MacBook's](macbook.md) ~150 GB/s — expect memory-bound generation; token speeds will be modest for 7–8B.

### RAM — 12 GB physical (+6 GB "extension")
- Treat the device as **12 GB** for inference planning. The "+6 GB" is Xiaomi's memory extension: storage-backed swap, orders of magnitude slower than RAM — useless for model weights beyond avoiding hard OOM.
- Android low-memory pressure: a foreground app can realistically use roughly half the physical RAM before the killer steps in; keep background apps cleared when loading bigger models.

### PocketPal
- Open-source llama.cpp-based app: runs GGUF models fully on-device; downloads models in-app.
- Also supports remote OpenAI-compatible endpoints — can be pointed at the [Powerserver](powerserver.md) API over the LAN to use the V100 from the phone.

## Model capacity estimates (12 GB, phone-class)

| Model size | Quantization | Fits? | Notes |
|---|---|---|---|
| 1.5–3 B | Q4–Q8 | ✅ comfortable | Practical pocket chat; responsive |
| 7–8 B | Q4_K_M | ⚠️ tight | ~4.5–5 GB weights; works but slow, needs headroom (close background apps) |
| 12 B+ | any | ❌ | Not realistic on this device |

*Estimates only — no measured benchmarks yet. Verify with actual loads and record results here.*

## Open questions

1. Measured tokens/s for representative models in PocketPal?
2. Used standalone, as a Powerserver API client, or both?
3. Which models are actually loaded on the phone?

---
*LLM-maintained wiki — see [index](index.md) and [log](log.md).*
