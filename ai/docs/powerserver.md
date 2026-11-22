# Powerserver — Local LLM Server

DIY tower server, the high-capacity inference machine of the setup. Built on a Chinese X79 dual-era platform with a datacenter Tesla V100 as the inference engine.

*Source: [README.md](../../README.md)* · *See also: [Overview](overview.md), [MacBook](macbook.md)*

## Specifications

| Component | Details |
|---|---|
| Motherboard | Huanan X79 v2.49P Green Deluxe (LGA 2011, Chinese clone board) |
| BIOS | 4.6.5 (2018-04-24), 6 MB ROM, UEFI — factory image (SMBIOS: HUANANZHI V2.49P, s/n MB-201804242018 encodes the same date) |
| CPU | Intel Xeon E5-1650 v2 |
| RAM | 64 GB ECC DDR3 (4 × 16 GB) |
| Disk | 256 GB NVMe SSD + 4 TB HDD |
| OS | Proxmox VE (PVE), GRUB 2.06, LVM boot (pve-root, pve-swap) — identified 2026-11-16 |
| GPU 1 | NVIDIA GeForce GT 730, 4 GB VRAM, PCIe |
| GPU 2 | NVIDIA Tesla V100, 32 GB VRAM, PCIe |
| GPU cooling | ID-COOLING FX240 PRO, 240 mm AIO (on the V100) |
| PSU | ZALMAN TeraMax II SE, 1000 W |

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
- **UEFI GOP confirmed** (system boots with CSM disabled) — verified 2026-11-15.

### GPU 2 — NVIDIA Tesla V100 (32 GB, PCIe)
- Volta GV100 datacenter card: 5,120 CUDA cores + 640 Tensor Cores, compute capability 7.0.
- **32 GB HBM2 with ~900 GB/s bandwidth** — the key metric for token generation speed. Comparable to a desktop RTX 3090 (~936 GB/s), but with double the VRAM; generation speed scales with memory bandwidth, not CUDA-core count.
- ~250 W TDP, **passive cooling** — requires directed chassis airflow; this card is not designed to sit in a quiet desktop case without a fan shroud.
- No display outputs, no NVLink (PCIe version only).
- Still supported by modern CUDA 12.x / llama.cpp (compute 7.0 ≥ minimum), including FlashAttention. Tensor-core (FP16) paths help prompt processing.
- Supported by vLLM (Volta is the oldest supported architecture), with limitations vs. newer GPUs.
- Powered by a **single 8-pin EPS (CPU-style) connector**, 250 W max (NVIDIA product brief / TechPowerUp) — use a dedicated EPS cable from the PSU, not a PCIe daisy-chain. Native BAR1 is **32 GB — confirmed assigned above 4 GB on the stock BIOS** (`lspci -vv`, 2026-11-21; see [V100 bring-up status](#v100-bring-up-status-2026-11-16)).

### GPU cooling — ID-COOLING FX240 PRO (240 mm AIO)
- Added 2026-11-15 (README update): 240 mm AIO fitted to the V100, replacing reliance on directed chassis airflow for the GPU die.
- **Unverified under load**: AIO brackets typically cool the die only — check that the VRM and HBM2 areas still get airflow, and record load temps (target < ~83 °C).

### PSU — ZALMAN TeraMax II SE (1000 W)
- Ample for the build: V100 250 W + E5-1650 v2 130 W + GT 730 ~50 W + AIO/fans ≈ 500–550 W worst case, ~2× headroom.
- Use separate, dedicated cables for the V100's EPS 8-pin and the CPU EPS.

## V100 bring-up status (2026-11-16)

Making the V100's ~16–32 GB BAR1 assignable above 4 GB ("Above 4G Decoding"). Working plan: [ai/plan/bios.md](../../ai/plan/bios.md).

| Step | Result |
|---|---|
| CSM off, Above 4G off, GT 730 only | ✅ boots — GT 730 has UEFI GOP |
| CSM off + Above 4G on, V100 removed | ✅ boots — the option exists and the board trains normally |
| CSM off + Above 4G on, V100 installed | ❌ beeps at POST, no boot — firmware fails to allocate the 64-bit BAR |
| CSM **on** + UEFI-only, Above 4G on, V100 upper x16 + GT 730 lower x16 (stock BIOS, not patched) | ✅ **POSTs, enters Setup** (2026-11-16) — first POST with the V100 installed; BAR assignment still unverified |
| Same config, booted to Proxmox (2026-11-21) | ✅ **BAR1 [size=32G] assigned a 64-bit address above 4 GB, dmesg clean** — no `pci=realloc`, no firmware patch; hardware bring-up complete |

Conclusion: the stock BIOS exposes Above 4G Decoding but its large-BAR allocation is broken (known X79 firmware flaw, confirmed in ReBarUEFI issue tracker). The Linux `pci=realloc` fallback (old Plan B in the plan file) **cannot help** — the board never POSTs with the card installed, so the kernel never runs. A firmware fix is mandatory.

**Update 2026-11-16 (later):** the board **does POST with the V100 installed** on the stock BIOS — V100 in the upper x16, GT 730 in the lower x16, CSM enabled, boot filter UEFI-only, Above 4G on. Two variables changed vs the beep tests (CSM off→on, slot swap), so attribution is unknown — needs a controlled retest. The no-POST kill of `pci=realloc` is therefore **retracted pending verification**; CSM-on can also leave Above-4G allocation partially inert, so CSM-off remains the target end state. Current blocker: boot drops to a `grub>` prompt (Proxmox, GRUB 2.06) after the BIOS settings change — manual boot from `(lvm/pve-root)`, then `update-grub` (+ `grub-install`/`efibootmgr` if the NVRAM entry is gone).

**Resolution (2026-11-21):** boot fixed, then **the BAR assigns correctly on the stock BIOS — no flash, no `pci=realloc`**. GRUB root cause: the ESP's FAT32 was malformed — Linux vfat and the AMI firmware read it, but GRUB's strict parser rejected it (hence the original `grub>`: the ESP's `grub.cfg` stub was unreadable, and `grub-install` failed with "unknown filesystem"). Fix: backup → reformat nvme0n1p2 as clean FAT32 → `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=proxmox` → add the missing `/boot/efi` fstab entry. With CSM on + Above 4G on, `lspci -vv` shows the V100's Region 1 **[size=32G] at a 64-bit address ≥ 0x100000000** and dmesg is clean. `/proc/iomem | grep nvidia` is empty only because no driver has claimed the regions yet. Remaining: NVIDIA driver on the PVE host, `nvidia-smi`, load/thermal smoke test.

Fix paths, preferred first:

1. **ReBarUEFI UEFIPatch applied to this board's own BIOS dump** — the project ships an explicit "X79 Above 4G Decoding fix" patch, plus 64-bit-BAR handling patches (BAR size limit removal, prevent 64→32-bit downgrade, Sandy/Ivy MMIO-high window extension). Patching our own dump sidesteps the companion-chipset variant problem entirely. The Resizable BAR *feature* itself is neither needed nor supported by Volta — skip ReBarState/the DXE module; the patches are the fix.
2. **Newer stock 2.49P dump** (e.g. e5450.com collection) — must match the companion PCH (B75 / Q65 / C204) and, ideally, build date. Identify via `lspci -nn | grep -iE 'isa|lpc'` (the LPC controller names the companion chipset), board silkscreen/sticker, or `dmidecode`.

**Phase 0 update (2026-11-16):** current BIOS recorded — **4.6.5, 2018-04-24, 6 MB ROM, UEFI**; baseboard SMBIOS confirms HUANANZHI V2.49P (s/n MB-201804242018, date-matching the BIOS → factory image, never updated). The installed build already equals the newest known stock dump (Q65-variant date per the e5450 catalog), so fix path 2 offers nothing newer — **patching this board's own dump (path 1) is the only firmware route**. 6 MB ROM → AFUDOS-family flash; verify the true chip size from the backup dump before patching.

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

1. V100 cooling: FX240 PRO AIO added 2026-11-15 — load temps and VRM/HBM2 airflow still unverified.
2. OS confirmed: Proxmox VE (2026-11-16). Open: inference on the PVE host vs a VM with V100 PCIe passthrough (needs VT-d/IOMMU + `vfio-pci`, which itself needs working Above-4G); driver/CUDA stack choice (llama.cpp/Ollama/vLLM) undecided.
3. Is the RAM actually running in quad-channel mode?
4. Measured tokens/s for a target model (e.g. Qwen/Llama 14B Q6) at realistic context lengths?
5. PSU known (TeraMax II SE 1000 W, ample); measured wall draw under full inference load still pending.
6. Which companion PCH does this board use — B75, Q65, or C204? Hint: our 04-24-2018 BIOS date matches the Q65-variant stock build. Near-moot now — only relevant to the stock-dump fallback, which offers no newer build than the installed one.
7. ~~Current BIOS version/date~~ Recorded 2026-11-16: 4.6.5, 2018-04-24, 6 MB, UEFI. Backup dump no longer required — the stock BIOS assigns the V100's BAR correctly; ReBarUEFI patch route stays on standby only if the config regresses.

---
*LLM-maintained wiki — see [index](index.md) and [log](log.md).*
