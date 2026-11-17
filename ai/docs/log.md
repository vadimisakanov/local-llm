# Log

Append-only chronological record. Entries start with `## [YYYY-MM-DD] <operation> | <subject>` so they can be parsed with `grep "^## \[" log.md | tail -5`.

## [2026-11-09] ingest | Initial hardware inventory (README.md)

- Ingested [README.md](../../README.md) (raw source): two machines — Powerserver and MacBook.
- Created pages: [overview.md](overview.md), [powerserver.md](powerserver.md), [macbook.md](macbook.md).
- Added component notes and model-capacity estimates from hardware knowledge. **All estimates flagged unverified** — no benchmarks exist yet.
- Open questions logged: inference stack per machine, airflow for passive V100, GPU wired-memory limit on macOS, measured tok/s baselines, PSU/power budget.

## [2026-11-09] ingest | Mi 14T mobile phone (README.md update)

- Ingested README.md change: new device 3/ — Xiaomi Mi 14T (12+6 GB RAM, 512 GB, preferred software: PocketPal).
- Created [mi14t.md](mi14t.md); updated [overview.md](overview.md) (three-device setup, complementarity) and [index.md](index.md).
- Noted: the "+6 GB" is Xiaomi's storage-backed memory extension — treat the phone as 12 GB for inference.

## [2026-11-16] ingest | PSU & GPU AIO added to Powerserver (README update)

- Ingested README.md change: `GPU Cooler: ID-COOLING FX240 PRO` and `PSU: ZALMAN TeraMax II SE 1000W`.
- Updated [powerserver.md](powerserver.md): spec table rows + component notes. PSU verdict: ample (~2× headroom). AIO: GPU die cooling addressed; VRM/HBM2 airflow + load temps unverified. Open question on PSU wattage resolved; cooling question narrowed.

## [2026-11-16] update | V100 Above-4G bring-up triage

- Test results recorded in [powerserver.md](powerserver.md): GT 730 has UEFI GOP (CSM-off boots); Above 4G Decoding option present and boots fine without the V100; **with the V100 installed → beep(s), no POST** → stock BIOS large-BAR allocation broken.
- Finding: [ReBarUEFI](https://github.com/xCuri0/ReBarUEFI) UEFIPatch ships a dedicated **"X79 Above 4G Decoding fix"** + 64-bit BAR/MMIO patches → preferred fix (patch this board's own dump; avoids chipset-variant matching). ReBAR feature itself not needed/unsupported on Volta.
- Fallback: stock 2.49P dumps selected by companion PCH (B75/Q65/C204, identified via LPC controller in `lspci`).
- Ruled out: Linux `pci=realloc` (Plan B in [ai/plan/bios.md](../plan/bios.md)) — board never POSTs with the card, kernel never runs.
- Correction recorded: V100 PCIe power = single **8-pin EPS** (CPU-style), not 2× PCIe 8-pin.

## [2026-11-16] session | Session summary filed (wiki bootstrap, V100 BIOS planning, Mi 14T)

- Filed session summary to [ai/sessions/2026-11-16-wiki-bootstrap-v100-bios-plan.md](../sessions/2026-11-16-wiki-bootstrap-v100-bios-plan.md).
- Covers the 2026-11-08/09 session: wiki bootstrap, V100/Above-4G chipset Q&A, BIOS-update plan (`ai/plan/bios.md`, Linux confirmed), Mi 14T ingest.
- Pending: execute the BIOS plan from Phase 0 and record results back into the wiki.

## [2026-11-16] ingest | BIOS & baseboard identity recorded (Phase 0)

- BIOS (via `dmidecode`): **4.6.5** (rev 4.6), **2018-04-24**, **6 MB ROM**, UEFI. Baseboard: HUANANZHI V2.49P confirmed (SMBIOS), s/n **MB-201804242018** — serial encodes the BIOS date → factory image, never updated.
- Key implication: installed build = newest known stock (Q65-variant 04-24-2018 per e5450 catalog) → **stock-dump fallback dead; ReBarUEFI patch of this board's own dump is the only firmware route**. 6 MB → AFUDOS-family flash; true chip size to be confirmed from the backup dump.
- Updated [powerserver.md](powerserver.md): BIOS spec row, Phase 0 note in bring-up section, open questions 6 (hinted Q65, near-moot) & 7 (recorded; backup dump pending). Updated [ai/plan/bios.md](../plan/bios.md): OQ 1 answered, Phase 1 step 2a marked exhausted, execution-log entry added.
- Session log: [ai/sessions/2026-11-16-bios-phase0-execution.md](../sessions/2026-11-16-bios-phase0-execution.md).

## [2026-11-16] ingest | First POST with V100 installed (stock BIOS); OS identified as Proxmox VE

- **Breakthrough**: board POSTs and enters Setup with the V100 installed on the unpatched stock BIOS — config: V100 upper x16, GT 730 lower x16, CSM **enabled**, boot options UEFI-only, Above 4G Decoding on. Attribution unknown (CSM off→on and slot swap both changed vs the 2026-11-15 beep tests). POST ≠ working — BAR assignment unverified; CSM-on may leave Above-4G partially inert.
- **Plan B (`pci=realloc`) conditionally revived** — its kill condition ("never POSTs with V100") no longer holds; test after boot repair before any flash.
- **OS identified: Proxmox VE** (GRUB 2.06-13+pmx6; GPT disk: gpt1 BIOS-boot, gpt2 ESP, gpt3 LVM pve-root/pve-swap). Recorded in [powerserver.md](powerserver.md) spec table; open question 2 narrowed (host vs VM passthrough decision).
- **Current blocker**: boot drops to `grub>` (core.img runs, grub.cfg not found — likely NVRAM/boot-path reset from the BIOS settings change). Recovery recorded in plan/session: manual boot from `(lvm/pve-root)` → `update-grub` → `grub-install`/`efibootmgr` as needed.
- ReBarUEFI usage re-confirmed against upstream README: UEFIPatch 0.28.0 + repo `patches.txt`; relevant rules = X79 Above-4G fix, <4/<16 GB BAR limit removal, 64→32-bit downgrade prevention, Sandy/Ivy MMIO 16→64 GB (DSDT mod may be needed); skip ReBarDxe/ReBarState; check pad files unchanged.
- Updated [powerserver.md](powerserver.md) (bring-up table row, update paragraph, OS row, OQ 2), [ai/plan/bios.md](../plan/bios.md) (Plan B revival note, execution-log entry), session log extended.
