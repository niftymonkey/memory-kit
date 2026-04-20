# memory-kit

Voice-first memory capture with natural-language recall. R&D toward **Mnemo** — a private AI assistant with long-term memory.

## What it is

A containerized memory service (`notes-api`) plus a dogfood web UI (`web-ui`). Record voice or text notes; an LLM extracts structured meaning (events, commitments, preferences, mentions); everything is stored in Postgres + pgvector. A chat UI lets you interrogate your own memory in natural language — *"who fixed the plumbing?"*, *"what's happening next weekend?"*, *"do any commitments conflict with me going skiing this Saturday?"*

The memory service is the product. The web UI is a dogfood surface. Both are built to validate patterns that will later drop into `mnemo`.

## Status

Early R&D. Planning docs in place; no code yet.

## Stance

**Learning over shipping.** The output is not a notes app — it's validated patterns, measured findings, and a memory service that drops into Mnemo.

## Design in brief

- TypeScript pnpm monorepo, Docker Compose orchestration.
- Postgres + pgvector + Drizzle for hybrid structured/vector storage.
- Vercel AI SDK against an OpenAI-compatible base URL — cloud or local Ollama swappable via config.
- Single LLM call per note for structured extraction (Zod-schema `generateObject`).
- Lightweight bounded tool-calling agent for retrieval; streaming answers with inline source citations.
- Additive-never-destructive: corrections are new notes with an FK to the original; outbox pattern from day one.
- No canonical entity resolution (deferred to Graphiti in Phase 2).

## Read more

- [`CLAUDE.md`](./CLAUDE.md) — guardrails and conventions for AI coding agents working on this repo.
- [`docs/exploration-summary.md`](./docs/exploration-summary.md) — locked decisions from the design exploration.
- [`docs/mnemo-carry-forward.md`](./docs/mnemo-carry-forward.md) — why this project exists; what must be learned; what not to build.
- [`docs/prd.md`](./docs/prd.md) — implementation-facing spec.

## Related (sibling projects)

- `niftymonkey/mnemo` — downstream consumer; provides architectural DNA.
- `niftymonkey/review-kit` — closest architectural analog; voice capture + Evalite pattern.
- `niftymonkey/champ-sage` — Evalite scorer conventions.
