# Session 2026-11-15/16 — V100 Above-4G firmware triage

*Participants: user + Kilo agents (ask → code). Continues [2026-11-16-wiki-bootstrap-v100-bios-plan](2026-11-16-wiki-bootstrap-v100-bios-plan.md).*

## User prompts (condensed)

1. What firmware does the Powerserver need to make the Tesla V100 work on the Huanan X79 motherboard?
2. (README updated with PSU ZALMAN TeraMax II SE 1000 W + GPU cooler ID-COOLING FX240 PRO.) Enabling Above 4G Decoding → board doesn't boot, a few beeps at startup only.
3. Granted code mode. Test A (CSM off) passed; Test B (CSM off + Above 4G, no V100) passed; with CSM off + Above 4G on [and the V100 installed] it doesn't boot. Questions: (a) is it true, per ai/plan/bios.md + ai/sessions/, that ReBarUEFI can't be used on this setup? (b) for the stock-BIOS route, how to check which chipset variant (B75/Q65) the board has?
4. Save the session log and summary to ai/sessions. (Session ended at Q1/Q2 answers: ReBarUEFI confirmed usable; chipset check via `lspci` LPC controller.)

## Findings

- **ReBarUEFI is usable** — the plan's "Resizable BAR is not needed / X79 doesn't have it" refers to the *feature*, not the tool. ReBarUEFI's UEFIPatch explicitly ships an **"X79 Above 4G Decoding fix"**, plus 64-bit BAR patches (size-limit removal, 64→32-bit downgrade prevention, Sandy/Ivy MMIO-high extension — may need DSDT mod on some boards). Preferred fix: dump **this board's own BIOS**, patch, flash — sidesteps the chipset-variant dump-matching problem. ReBAR itself is irrelevant (Volta has no ReBAR; the V100's native ~16–32 GB BAR1 just needs working 64-bit allocation), so ReBarState/DXE module can be skipped.
- **Test matrix results** (recorded in [powerserver.md](../docs/powerserver.md)): GT 730 has UEFI GOP; Above-4G option present and functional without the card; **V100 installed = beep(s), no POST** → stock large-BAR allocator broken. `pci=realloc` Plan B dead (kernel never runs).
- **Companion chipset identification** (for stock-dump fallback): `lspci -nn | grep -iE 'isa|lpc'` — the LPC controller names it (B75 / Q65 / C204 Express LPC Controller). Clarified earlier session's "ISA bridge is not the chipset" note: that applies to CPU-side X79 root ports; the LPC bridge IS the companion PCH. Cross-checks: `dmidecode -t 2`, BIOS main-page string, board silkscreen/sticker, HWiNFO (Windows).
- Stock 2.49P dumps (e5450.com): B75 variant orig 06-26-2017 + 04-18-2018 build; Q65 variant 04-24-2018. 6 MB → AFUDOS, 8 MB → FPT. Match chipset + closest build date.
- **Correction**: V100 PCIe power = single **8-pin EPS (CPU-style)** connector, 250 W (NVIDIA product brief, TechPowerUp) — not 2× PCIe 8-pin. Dedicated EPS cable advised.
- Beep reference used: 1 long + 2 short / 8 short ≈ VGA init failure (AMI).

## Actions (code agent)

- `ai/docs/powerserver.md`: PSU + AIO rows/notes; V100 bring-up status section; open questions updated (5 resolved/narrowed, 2 added: companion PCH, BIOS version).
- `ai/docs/log.md`: two entries (hardware ingest; triage findings).
- `ai/plan/bios.md`: open questions 2 & 4 answered; Plan B marked not viable; execution log added; ReBarUEFI clarification on the "ReBAR not needed" bullet.

## Next steps

1. Record BIOS version (`dmidecode -s bios-version`), back up own BIOS dump (AFUDOS save / FPT / CH341A).
2. Patch own dump with ReBarUEFI UEFIPatch (verify pad files unchanged) → flash → CMOS clear → CSM off + Above 4G on → install V100 → verify (`lspci -vv`, `nvidia-smi`).
3. Fallback: stock 2018 dump matched by companion PCH.
4. After success: wiki aftermath per plan (BIOS version, flash method, load temps).
