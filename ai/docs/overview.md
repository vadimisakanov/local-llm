# Overview — Local LLM Inference Setup

The setup consists of three devices with complementary roles: a DIY server built around a datacenter GPU, a laptop for portable on-device inference, and a phone for pocket-tier inference and remote access.

| Machine | Role | Inference hardware | VRAM / usable memory |
|---|---|---|---|
| [Powerserver](powerserver.md) | Dedicated local LLM server | NVIDIA Tesla V100 32GB (HBM2) | 32 GB VRAM |
| [MacBook](macbook.md) | Portable local LLM notebook | Apple M3 Pro GPU (unified memory) | 18 GB unified (macOS caps GPU share by default) |
| [Mi 14T](mi14t.md) | Pocket LLM phone (PocketPal) | MediaTek Dimensity 8300-Ultra (CPU/GPU/NPU) | 12 GB RAM (the "+6 GB" is storage swap, not inference-usable) |

## How the devices complement each other

- **Powerserver** = capacity and throughput. The V100's 32 GB of HBM2 with ~900 GB/s bandwidth is the backbone of the setup: it fits 30B-class models in 4-bit quantization, and 13B-class models in FP16. Suitable for hosting an OpenAI-compatible API (llama.cpp / Ollama / vLLM) for other devices on the LAN.
- **MacBook** = mobility and privacy on the go. The M3 Pro's unified memory lets the GPU use a slice of the 18 GB pool; practical for 7B–8B models at 4-bit, tight for 12–14B.
- **Mi 14T** = pocket tier. Small models (1.5–4B) run on-device in PocketPal; it can also act as a thin client to the Powerserver's OpenAI-compatible API over the LAN.

## Known limitations & open questions

- The [Powerserver](powerserver.md) CPU (Xeon E5-1650 v2, Ivy Bridge, 2013) is old; CPU inference and prompt processing will be slow when models partially spill from VRAM.
- The Tesla V100 PCIe has **no display outputs** — a consumer [GT 730](powerserver.md#gpu-1---nvidia-geforce-gt-730-4gb) handles the video signal.
- Cooling: the V100 PCIe is a passive (no fan) datacenter card and needs directed server-style airflow. Current chassis/airflow status is **unverified**.
- No measured performance baselines exist yet (tokens/s per model/quant per device). See [open questions](#open-questions).

## Open questions

1. What inference stack is installed on each machine (llama.cpp, Ollama, LM Studio, vLLM)?
2. Measured throughput (tok/s) per machine for representative models/quants?
3. Power budget and UPS situation for the Powerserver (V100 TDP alone is 250 W)?
4. Which models are actually in use / targeted (chat, coding, embeddings)?
5. Is the GT 730 the 4 GB DDR3 variant (Kepler/Fermi)? It is display-only either way — not usable for modern CUDA inference.

---
*Sources: [README.md](../../README.md). LLM-maintained wiki — see [index](index.md) and [log](log.md).*
