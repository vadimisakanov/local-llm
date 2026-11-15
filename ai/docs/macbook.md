# MacBook — Local LLM Notebook

MacBook Pro 16", the portable machine of the setup. Runs models on-device via Apple's unified memory architecture.

*Source: [README.md](../../README.md)* · *See also: [Overview](overview.md), [Powerserver](powerserver.md)*

## Specifications

| Component | Details |
|---|---|
| Model | MacBook Pro 16" (M3 Pro) |
| SoC | Apple M3 Pro |
| RAM | 18 GB unified memory |
| Disk | 512 GB NVMe SSD |

## Component notes (local-LLM perspective)

### SoC — Apple M3 Pro
- CPU and GPU share the 18 GB unified memory pool; there is no separate VRAM.
- M3 Pro memory bandwidth: ~150 GB/s — the main limit on token generation speed (roughly 1/6 of the [Powerserver's V100](powerserver.md) HBM2 bandwidth, but 3–5× a typical Intel laptop with DDR4/DDR5).
- Default M3 Pro GPU configuration in the 16" chassis: 14-core GPU (the 12-core CPU / 18-core config shipped in the 16"). Exact bin for this unit is **unverified** — check `system_profiler SPHardwareDataType` or *About This Mac*.

### RAM — 18 GB unified
- macOS caps the GPU's wired-memory allocation to ~70–75% by default (~11–13 GB usable for models).
- The cap can be raised via `sudo sysctl iogpu.wired_limit_mb=<MB>` (reverts on reboot). Beyond ~2/3 of total RAM, system instability/swap risk grows — leave headroom for the OS and apps.
- Practical consequence: models beyond ~14B Q4 will swap or not load; the realistic ceiling here is 12–14B quantized.

### Disk — 512 GB
- Model files consume real space (7B Q4 ≈ 4–5 GB each). A handful of models plus apps/OS will fill the drive; plan for external storage or selective model loading.

## Model capacity estimates (M3 Pro 18 GB)

| Model size | Quantization | Fits? | Notes |
|---|---|---|---|
| 3–4 B | Q8 / FP16 | ✅ | Snappy, good for quick tasks |
| 7–8 B | Q4_K_M | ✅ comfortably | Sweet spot for this machine |
| 7–8 B | Q8 | ✅ tight | ~8–9 GB with KV cache; watch the GPU cap |
| 12–14 B | Q4_K_M | ⚠️ tight | Needs GPU wired-limit raised; expect slowness on CPU-bound ops |
| 30 B+ | any | ❌ | Won't fit meaningfully |

*Estimates only — no measured benchmarks yet. Verify with actual loads and record results here.*

## Open questions

1. Exact M3 Pro bin (CPU/GPU core counts) of this unit?
2. Which runtime is installed (Ollama, LM Studio, llama.cpp, MLX)?
3. Current `iogpu.wired_limit_mb` setting, if adjusted?
4. Measured tokens/s for the primary travel model (e.g. 8B Q4)?
5. Does this MacBook connect to the [Powerserver](powerserver.md) over the LAN for large-model inference (client to its OpenAI-compatible API)?

---
*LLM-maintained wiki — see [index](index.md) and [log](log.md).*
