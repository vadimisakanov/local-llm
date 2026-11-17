# Session 2026-11-08/09 — Wiki bootstrap, V100/BIOS planning, Mi 14T ingest

*Filed 2026-11-16. Participants: user + Kilo agents (docs-specialist → ask → plan → code).*

## User prompts (condensed)

1. Bootstrap an LLM-wiki in `ai/docs/` following the [Karpathy llm-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f); add a link to it in `AGENTS.md`.
2. Q&A: how to make the Tesla V100 32GB work; Google says the motherboard BIOS needs "4G decoding" support — which chipset to look at, "ISA bridge" or "PCI bridge"?
3. Make a plan for the BIOS update and put it in `ai/plan/`.
4. Continue refining the plan; do not implement. (Answered interviewer question: **Linux** is installed on the Powerserver.)
5. Copy the refined plan to `ai/plan/bios.md`.
6. Ingest the new README section: Xiaomi Mi 14T phone.
7. Save this session summary to `ai/sessions/` and a short log entry to `ai/docs/log.md`.

## What was done

- **Wiki bootstrap (2026-11-09)**: created `ai/docs/` pages — [index](../docs/index.md), [log](../docs/log.md), [overview](../docs/overview.md), [powerserver](../docs/powerserver.md), [macbook](../docs/macbook.md); `AGENTS.md` gained the Wiki schema section (README = immutable raw source; wiki = LLM-owned layer).
- **Chipset answer**: "ISA bridge"/"PCI bridge" are `lspci` device classes, not chipsets. Platform is fixed: Intel X79 PCH + CPU-integrated PCIe root ports. BIOS selection is by exact board model (Huanan X79 v2.49P Green Deluxe). The V100 needs **Above 4G Decoding** (its ~16 GB BAR1 can't fit below the 4 GB legacy PCI window); Resizable BAR is not needed.
- **BIOS plan** → `ai/plan/bios.md`: Phase 0 assess → 0.5 Linux dry-run (`pci=realloc pci=nocrs`) → 1 prepare → 2 flash → 3 configure → 4 verify; no-flash Plan B confirmed viable (Linux installed); failure modes incl. CH341A SPI recovery; V100 power = 8-pin EPS 250 W.
- **Mi 14T ingest (2026-11-09)**: created [mi14t](../docs/mi14t.md), updated overview/index; "+6 GB" identified as storage-backed memory extension — phone treated as 12 GB for inference.

## Key decisions / knowledge

- README.md is never modified; the wiki in `ai/docs/` is the LLM-owned layer (per AGENTS.md).
- V100: Above 4G Decoding required; don't use `flashrom` for the main flash; EPS 8-pin power; passive card needs directed airflow.
- Phone "+6 GB" RAM extension is storage swap — not inference-usable.

## Open threads

- Execute `ai/plan/bios.md` starting at Phase 0; record results back into the wiki (log entry + `powerserver.md` updates).
- Unresolved open questions on device pages: BIOS version present on board, Above-4G availability in official BIOS, PSU/EPS status, measured tok/s baselines per device, cooling under load.
