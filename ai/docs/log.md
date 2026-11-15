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
