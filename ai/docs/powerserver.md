# Powerserver — Local LLM Server

DIY tower server, the high-capacity inference machine of the setup. Built on a Chinese X79 dual-era platform with a datacenter Tesla V100 as the inference engine.

*Source: [README.md](../../README.md)* · *See also: [Overview](overview.md), [MacBook](macbook.md)*

## Specifications

| Component | Details |
|---|---|
| Motherboard | Huanan X79 v2.49P Green Deluxe (LGA 2011, Chinese clone board) |
| CPU | Intel Xeon E5-1650 v2 |
| RAM | 64 GB ECC DDR3 (4 × 16 GB) |
| Disk | 256 GB NVMe SSD + 4 TB HDD |
| GPU 1 | NVIDIA GeForce GT 730, 4 GB VRAM, PCIe |
| GPU 2 | NVIDIA Tesla V100, 32 GB VRAM, PCIe |

## Component notes (local-LLM perspective)

### CPU — Xeon E5-1650 v2
- Ivy Bridge-EP (2013): 6 cores / 12 threads, 3.5 GHz base, ~3.9 GHz turbo, 130 W TDP.
- 40 PCIe 3.0 lanes — enough to host both GPUs at x16/x8 without lane starvation.
- DDR3 quad-channel controller; with 4 DIMMs this should run quad-channel, which matters for CPU-offloaded inference layers.
- Main role: hosting the API service, prompt preprocessing, and any CPU-offloaded layers. Not a fast inference CPU by modern standards.

### RAM — 64 GB ECC DDR3
- Ample for loading large model files from disk and memory-mapping (mmap) quantized GGUF files.
- ECC is valuable for long-running server duty (24/7 API host).

### Disk — 256 GB NVMe + 4 TB HDD
- Suggested layout (not confirmed as configured): OS + model hot-set on NVMe, model archive on the 4 TB HDD.
- A 30B Q4 model is ~18–20 GB; 13B Q8 ~14 GB — the NVMe can hold a working set of several models.

### GPU 1 — NVIDIA GeForce GT 730 (4 GB)
- Display card only. The V100 has no video outputs, so the GT 730 drives the monitor.
- ~4 GB DDR3, very low memory bandwidth (~25–40 GB/s depending on variant). **Not useful for inference**: its compute capability (3.5 or 2.1 depending on Kepler/Fermi variant) is no longer supported by current CUDA/PyTorch builds.
- Keep it on a minimal driver for display; do not budget inference work to it.

### GPU 2 — NVIDIA Tesla V100 (32 GB, PCIe)
- Volta GV100 datacenter card: 5,120 CUDA cores + 640 Tensor Cores, compute capability 7.0.
- **32 GB HBM2 with ~900 GB/s bandwidth** — the key metric for token generation speed. Comparable to a desktop RTX 3090 (~936 GB/s), but with double the VRAM; generation speed scales with memory bandwidth, not CUDA-core count.
- ~250 W TDP, **passive cooling** — requires directed chassis airflow; this card is not designed to sit in a quiet desktop case without a fan shroud.
- No display outputs, no NVLink (PCIe version only).
- Still supported by modern CUDA 12.x / llama.cpp (compute 7.0 ≥ minimum), including FlashAttention. Tensor-core (FP16) paths help prompt processing.
- Supported by vLLM (Volta is the oldest supported architecture), with limitations vs. newer GPUs.

## Model capacity estimates (V100 32 GB)

| Model size | Quantization | Fits? | Notes |
|---|---|---|---|
| 7–8 B | FP16 / Q8 | ✅ easily | Fastest generation; headroom for long context |
| 13–14 B | FP16 | ✅ | ~26–28 GB — near the limit with long context |
| 13–14 B | Q8 / Q6 | ✅ comfortably | Good quality/speed balance |
| 30–34 B | Q4_K_M | ✅ | ~18–20 GB + context KV cache |
| 70 B+ | Q4 | ❌ | Does not fit in 32 GB; would need CPU offload (slow on this platform) |

*Estimates only — no measured benchmarks yet. Verify with actual loads and record results here.*

## Open questions

1. Chassis and airflow solution for the passive V100 — confirmed working under load?
2. Which CUDA stack / inference engine is installed (llama.cpp, Ollama, vLLM)?
3. Is the RAM actually running in quad-channel mode?
4. Measured tokens/s for a target model (e.g. Qwen/Llama 14B Q6) at realistic context lengths?
5. Power draw under full inference load; PSU wattage and headroom?

---
*LLM-maintained wiki — see [index](index.md) and [log](log.md).*
