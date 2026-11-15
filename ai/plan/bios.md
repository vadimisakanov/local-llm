# Plan: Huanan X79 v2.49P BIOS update — enable Above 4G Decoding for Tesla V100

Target machine: Powerserver (Huanan X79 v2.49P Green Deluxe, Xeon E5-1650 v2, Tesla V100 32GB PCIe + GT 730 display card). This is the repo copy (`ai/plan/bios.md`), synced from the refined planning-agent version on 2026-11-08.

## Goal

Make the Tesla V100 32GB usable by enabling 64-bit PCI BAR assignment ("Above 4G Decoding") on the Huanan X79 v2.49P, via BIOS update if the option isn't already available — without bricking the board.

**Success criteria**

- System POSTs with the V100 installed in a CPU-attached x16 slot
- `lspci -vv -s <slot>` shows the V100's large region (~16 GB BAR1) assigned a real 64-bit address, not unassigned
- `dmesg | grep -iE 'bar|nvidia'` shows no `BAR 1: no space` / allocation errors
- `nvidia-smi` reports `Tesla V100-PCIE-32GB` with ~32510 MiB

## Background

- The V100 exposes a ~16 GB BAR1; legacy firmware can only place BARs below 4 GB, so without Above 4G Decoding the card hangs POST or stays invisible. Resizable BAR is **not** needed (and X79 doesn't have it).
- "Above 4G Decoding" (a.k.a. "4G Decode" — *decoding*, not encoding) lives in BIOS Setup → Advanced → PCI Subsystem Settings on AMI Aptio firmware. On many Huanan BIOSes it is hidden or absent.
- The ISA-bridge/PCI-bridge entries from `lspci` are device classes, not chipsets — the platform chipset is fixed (Intel X79 + CPU-integrated PCIe root ports). BIOS selection is by **exact board model/revision**, not by bridge class.
- Entry point depends on current machine state:
  - **POSTs** with the V100 installed (display from GT 730) but the card is invisible/mis-BAR'd → start at Phase 0.
  - **Does not POST** with the V100 installed → power off, remove the card, run Phases 0–2 without it, reinstall at Phase 3.

## Risks & constraints

| Risk | Mitigation |
|---|---|
| Wrong BIOS image (other Huanan variant / X99) = brick | Match exact model: `dmidecode -t baseboard` + board silkscreen "v2.49P Green Deluxe"; verify before flashing |
| Power loss mid-flash = brick (no dual BIOS on these boards) | Flash only on stable power / UPS; never interrupt; last-resort recovery is CH341A SPI clip on the SOIC-8 flash chip |
| Official BIOS may not expose the option | Fallback order: official update → community-modded (AMIBCP-unlocked) BIOS → Plan B (Linux `pci=realloc`, no flash) |
| V100 needs 250 W via 8-pin **EPS** (CPU-style) connector | Verify PSU has EPS 8-pin + headroom (≥550–600 W quality unit for 250 W GPU + 130 W CPU) |

## Steps

### Phase 0 — Assess (no changes)

1. Record current BIOS: `dmidecode -s bios-version && dmidecode -t bios`.
2. Record board identity: `dmidecode -t baseboard` — confirm "v2.49P Green Deluxe".
3. Enter BIOS Setup → Advanced → PCI Subsystem Settings → look for "Above 4G Decoding"/"4G Decode".
   - **If present**: enable, save, reboot → skip to Phase 4 (verification). No flash needed.
   - **If absent/hidden**: continue.
4. OS confirmed: **Linux** (2026-11-08) — Linux paths below are primary; Windows steps removed.

### Phase 0.5 — Dry-run first (recommended; only if the machine POSTs with the V100 installed)

With Linux installed this costs nothing: add `pci=realloc pci=nocrs` to the kernel cmdline of the installed system (or a live USB), GT 730 primary. If the V100 then shows correct BAR assignment in `lspci -vv` (Region 1 ≈ [size=16G], assigned), the hardware is proven — slot, EPS power, card alive — and **Plan B is a working steady state**; flashing becomes optional. If the machine won't POST with the card in, skip: hardware validation happens after Phase 3.

### Phase 1 — Prepare

1. Power off, **remove the V100** (flash with GT 730 only, in the primary CPU-attached x16 slot).
2. Select BIOS image, in order of preference:
   a. Official Huanan download for X79 v2.49/v2.49P (latest dated build);
   b. Community BIOS known-good for Tesla P100/V100 on Huanan X79 (r/LocalLLaMA, Level1Techs threads; usually AMIBCP-unlocked with Above-4G forced on).
3. Validate the image: re-download and compare hashes; confirm it targets X79 v2.49 (not X99/other); read the package README/instructions.
4. If possible, back up the current firmware with the flasher's save function (AFUWIN "Save" / FPT dump).
5. Prepare a FAT32 USB stick with the BIOS image.
6. Photograph every BIOS Setup page (settings reset to defaults after flash/CMOS clear).
7. In Setup, load default settings ("Load Optimized Defaults", F9) and save — flashing from a clean-default state avoids bricks from incompatible settings.
8. Flash routes on Linux: the Setup-integrated flasher (UEFI) or a FreeDOS boot USB with the package's AFUDOS. **Do not use Linux `flashrom` for the main flash** — Intel descriptor/protected-region handling on these boards makes it a brick risk.
9. Order a CH341A SPI programmer + SOIC-8 clip in advance as cheap insurance (~$5). Caution: many clone CH341A boards feed the chip 5 V — get a 3.3 V-fixed version.

### Phase 2 — Flash

1. Boot with GT 730 only, minimal peripherals, stable power.
2. Flash using the utility shipped with the BIOS package (Huanan packages typically use AFUDOS/AFUWIN or the Setup-integrated flash tool). If the flasher refuses (write-protected regions), fall back to the Setup-integrated utility. **Do not cut power** until the board reboots itself and POSTs.
3. After POST, settings are at defaults → re-enter Setup.

### Phase 3 — Configure

1. Enable **Above 4G Decoding** (Advanced → PCI Subsystem Settings).
   - If the official BIOS still doesn't expose it → decision gate: re-flash with community-modded BIOS (repeat Phase 2), or switch to **Plan B** (no flash).
2. Restore essential settings (boot order, AHCI, DRAM profile, fan curves).
3. Power off. Install V100 in a **CPU-attached x16 slot** (root port shows as "Xeon E7 v2/E5 v2… PCI Express Root Port"), GT 730 stays primary/display. Connect the 8-pin EPS power to the V100 and confirm seating.

### Phase 4 — Verify

1. POST check with both cards installed.
2. `lspci | grep -i nvidia`; `lspci -vv -s <slot>` (Region 1 ≈ [size=16G] with an assigned 64-bit address); `dmesg | grep -iE 'bar|nvidia'` clean; `cat /proc/iomem | grep -i nvidia` should show mappings above 4 GB (≥ 0x100000000).
3. Driver: blacklist nouveau permanently (GRUB: append `modprobe.blacklist=nouveau` to `GRUB_CMDLINE_LINUX_DEFAULT`, then `update-grub`); install the NVIDIA driver (V100 = compute 7.0; supported on CUDA 12.x branches — CUDA 13 drops Volta, so pin driver/CUDA accordingly); `nvidia-smi` → `Tesla V100-PCIE-32GB`, ~32510 MiB.
4. Smoke test under load: run a short 8B-Q4 generation (llama.cpp/Ollama); watch `nvidia-smi` temps — the V100 is passively cooled; verify chassis airflow holds it under ~83 °C.

### Plan B — no-flash route (zero flash risk, Linux-only)

Keep GT 730 primary; make it permanent in GRUB: append `pci=realloc` (add `pci=nocrs` if BARs still fail to assign) to the kernel cmdline. Verify via Phase 4. **Confirmed available — Linux is installed.** Acceptable steady-state if flashing is deemed too risky.

## Failure modes

| Symptom | Action |
|---|---|
| No POST after flash | Clear CMOS (jumper/battery, ~10 min), retry with minimal config (one RAM stick, GT 730 only) |
| Still dead | External re-flash: locate SOIC-8 SPI flash (usually Winbond, near PCIe slots), CH341A + clip, dump before write; or replace board (cheap) |
| V100 visible, `nvidia-smi` fails | Reseat EPS power; check `dmesg` for BAR errors → back to Phase 3.1 |
| POST hangs with V100 in one specific slot | That slot is likely PCH-fed — move V100 to CPU-attached x16 |

## Open questions (record answers during execution)

1. Current BIOS version/date on this board?
2. Does the official v2.49P BIOS expose Above 4G Decoding?
3. ~~Which OS is installed on the Powerserver?~~ **Answered: Linux (2026-11-08).**
4. PSU model/wattage; EPS 8-pin connector available?

## Aftermath — wiki update (per AGENTS.md)

After execution (whichever path succeeded), a write-capable agent should:
- Append `## [YYYY-MM-DD] ingest | V100 BIOS bring-up results` to `ai/docs/log.md`
- Update `ai/docs/powerserver.md`: BIOS version found, Above-4G status/flash method used, PSU/EPS and cooling findings; resolve corresponding open questions
