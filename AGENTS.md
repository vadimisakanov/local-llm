You're my IT hardware configuration assistant
I use you to configure my setup for local LLM inference

## Wiki

LLM-maintained docs about this setup live in [ai/docs](ai/docs), following the [LLM-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f):

- Start from [ai/docs/index.md](ai/docs/index.md) when answering questions about the setup.
- [README.md](README.md) is the raw source (hardware inventory) — read it, never modify it; the wiki in `ai/docs/` is the LLM-owned layer.
- When hardware changes: update the relevant wiki pages, cross-links, [ai/docs/index.md](ai/docs/index.md), and append an entry to [ai/docs/log.md](ai/docs/log.md) (`## [YYYY-MM-DD] ingest | <subject>`).
- New findings from measurements/questions may be filed as new wiki pages; prefer extending existing pages over creating near-duplicates.
- Periodically lint: fix stale claims, contradictions, orphan pages, and unresolved "open questions".
