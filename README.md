# memory-kit

Voice-first memory capture with natural-language recall. Built for daily personal use, architected so its patterns transfer into **Mnemo** — a private AI assistant with long-term memory.

## What it is

A containerized memory service (`notes-api`) plus a dogfood web UI (`web-ui`). Record voice or text notes; an LLM extracts structured meaning (events, commitments, preferences, mentions); everything is stored in Postgres + pgvector. A chat UI lets you interrogate your own memory in natural language — *"who fixed the plumbing?"*, *"what's happening next weekend?"*, *"do any commitments conflict with me going skiing this Saturday?"*

The memory service is the portable piece — designed to drop into Mnemo later. The web UI is the app for daily use.

## Status

Early — planning docs in place; no code yet.

## Design in brief

- TypeScript pnpm monorepo, Docker Compose orchestration.
- Postgres + pgvector + Drizzle for hybrid structured/vector storage.
- Vercel AI SDK against an OpenAI-compatible base URL — cloud or local Ollama swappable via config.
- Single LLM call per note for structured extraction (Zod-schema `generateObject`).
- Lightweight bounded tool-calling agent for retrieval; streaming answers with inline source citations.
- Additive-never-destructive: corrections are new notes with an FK to the original; outbox pattern from day one.
- No canonical entity resolution (deferred to Graphiti in Phase 2).
