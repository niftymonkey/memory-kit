# memory-kit — Exploration Summary

## Problem / Opportunity

I routinely capture useful information in daily life that gets lost: who replaced the water heater, when I last changed the furnace filter, what restaurant was recommended, what errands I mentioned, what commitments I have for next weekend.

I want an AI-backed system that captures these via voice-first notes, extracts structured meaning, and lets me interrogate my own memory using natural language — "who fixed my plumbing?", "what's happening next weekend?", "do any commitments conflict with me going skiing this Saturday?".

This project is R&D toward **Mnemo**, a private AI assistant with long-term memory. The memory layer is the hard part of Mnemo; this project builds and dogfoods that layer in isolation so patterns prove before being pulled into Mnemo itself.

## Target Users

- **Primary (MVP):** me, single-user, dogfood-driven
- **Downstream:** Mnemo (consumes `notes-api` as its memory substrate)
- **Potential:** other niftymonkey projects that need a memory layer

## Core Requirements

### Must-have (MVP)

- **Voice-first web capture** (MediaRecorder → cloud Whisper → transcription), reusing the review-kit pattern
- **LLM extraction pipeline** — single structured-output call per note (Vercel AI SDK `generateObject` + Zod) producing mentions, events, commitments, preferences
- **Hybrid storage** — Postgres + pgvector + Drizzle. Raw note text + embedding always retained; derived structured rows linked via FK
- **Time normalization** — LLM resolves with captured_at + timezone context; `chrono-node` validates deterministically. Phrase, precision, and resolver preserved per date field
- **Chat-style NL query UI** — streaming answers, source citations inline
- **Lightweight bounded retrieval agent** — tool set: `searchNotes`, `searchEvents`, `searchCommitments`, `searchPreferences`, `getNoteById`. Max 5 tool calls per query
- **Correction semantics** — raw text immutable; corrections are new notes with `corrects_note_id` FK + type + summary. Derived tables mutable in place
- **Retention** — everything stays. Status flags filter at query time. Stale-item review UI, no auto-decay
- **Outbox pattern** — every mutation emits an append-only event to an `outbox` table for future consumers
- **Versioned HTTP API** (`/v1/*`) — stable contract for Mnemo consumption
- **Docker Compose containerized** — matches Mnemo's architectural pattern for 1:1 portability

### Nice-to-have / Explicitly deferred

- **Phase 2 (same repo):** Expo mobile app, Graphiti evaluation via `apps/graph-sync` reading the outbox, voice-driven corrections
- **Phase 3+:** MCP server / CLI for AI-harness ingestion (so Claude Code sessions can push into memory-kit)
- **Not MVP:** canonical entity resolution (this is Graphiti's job), auto-archival, summarization compression, auth

## Key Decisions

1. **Stack:** TypeScript pnpm monorepo + Docker Compose. Mirrors Mnemo's architectural pattern exactly.
2. **Repo location:** standalone at `~/dev/niftymonkey/memory-kit/`. Mirrors pattern without coupling to Mnemo's git/deploy.
3. **Product boundary:** the memory service is the product (drop-in for Mnemo), rooted at `apps/notes-api/` but free to decompose into multiple containers (API + background workers + outbox relay, etc.) as implementation needs dictate. `apps/web-ui` is dogfood/R&D surface only.
4. **LLM provider topology:** provider-agnostic via Vercel AI SDK. Config switch between cloud (default) and Mnemo's `ai-gateway` (for local-model evaluation). No own Ollama in this project.
5. **Whisper:** cloud API, direct call (review-kit pattern). No whisper-api service in this project.
6. **Capture surfaces:** web voice + typing in MVP. CLI deferred (would be MCP-style ingestion for AI harnesses later). Mobile (Expo) is Phase 2.
7. **Query UX:** chat thread with streaming answers and source citations. Voice input supported (same MediaRecorder pattern). Notes list / dashboard as secondary sanity-check surface.
8. **Storage model (hybrid):** structured facts + vector-indexed raw notes, cross-linked. No `entities` table — mentions live as JSONB on notes. Canonical entity resolution is Graphiti's territory and belongs in Phase 2.
9. **Retrieval strategy:** lightweight bounded agent (tool-calling) via Vercel AI SDK. Max ~5 tool calls, bounded timeout.
10. **Time normalization:** hybrid — LLM + `chrono-node` validation. Preserve phrase, precision (exact/day/week/month/approx/unresolvable), and resolver (both_agree / library_override / llm_only / unresolved) per date field.
11. **Corrections:** append-only notes with flat FK fields (`corrects_note_id`, `correction_type`, `correction_summary`) for DB-level integrity. Derived tables mutable with `last_correction_note_id` FK.
12. **Retention:** additive only. No auto-decay or destruction. Stale-item review surface in UI. Outbox enables future archival, cold storage, and summarization tiers without schema rewrites.
13. **Project name:** `memory-kit` — matches `review-kit` / `ai-toolkit` pattern; "kit" signals reusable drop-in.
14. **Shared types delivery:** HTTP-only for MVP. Extract `packages/memory-schema` only if a consumer (e.g., Mnemo) wants compile-time type safety over the API contract and HTTP-only proves insufficient. Default assumption: the versioned HTTP contract is the contract.

## Constraints / Boundaries

- **Additive, never destructive.** Every mutation preserves history. Reversibility is a first-class property.
- **Raw source always retained.** `notes.raw_text` is immutable; derived views are rebuildable from it + outbox.
- **12-factor compliance** per Mnemo's standard.
- **Single-user MVP.** API designed to accept an auth header later but no auth in MVP.
- **No own GPU / Ollama** — cloud for fast iteration; Mnemo's running Ollama reusable via config flag for local-model evaluation.
- **Versioned HTTP API from first commit** — Mnemo depends on the contract.

## Open Questions

Some questions below have a *starting stance* recorded, but the real answer requires implementation and dogfood data.

- **Graphiti evaluation** (Phase 2): what operational cost/benefit does it show in practice? Does the outbox → graph-sync pattern work as expected?
- **Noise threshold tuning.** *Starting stance:* permissive — extract anything with discernible signal; tune threshold based on dogfood noise. Open: what does "discernible signal" look like in practice, and how often does extraction falsely skip useful notes?
- **Retrieval agent prompt engineering.** *Starting stance:* terse tool descriptions, 5-iteration hard cap, ~30s timeout per query. Open: how verbose do tool descriptions need to be in practice? How often does the agent hit the iteration cap?
- **Stale-commitment review UX:** weekly review ritual, dashboard banner, or notification? Dogfood will tell.
- **Voice-driven corrections** (Phase 2+): how do we resolve "correct my note about Sarah" when there are multiple Sarahs?
- **Graphiti adoption path** for Mnemo: direct replacement of parts of memory-kit, or strict additive layer on top?

## Final Schema Snapshot

```
notes              id, raw_text, captured_at, source, embedding,
                   extraction_status, mentions JSONB,
                   corrects_note_id FK, correction_type,
                   correction_summary

events             id, description, event_type,
                   occurred_at + phrase + precision + resolver,
                   note_id FK, embedding,
                   last_correction_note_id FK, updated_at

commitments        id, description, commitment_type, status,
                   due_at + phrase + precision + resolver,
                   note_id FK, embedding,
                   last_correction_note_id FK, updated_at

preferences        id, subject, stance, context, note_id FK,
                   embedding, last_correction_note_id FK,
                   updated_at

outbox             id, event_type, payload JSONB, emitted_at,
                   consumed_by JSONB
```

## Workspace Shape

```
memory-kit/
├── apps/
│   ├── notes-api/          # the product — containerized TS service
│   ├── web-ui/             # dogfood/R&D surface — Next.js
│   └── graph-sync/         # Phase 2 — Graphiti consumer (deferred)
├── packages/
│   └── shared/             # Drizzle schema, Zod schemas, shared types
├── scripts/
│   └── test-fixtures.ts    # R&D test harness, not a shipped CLI
├── docker-compose.yml
├── docker-compose.dev.yml
└── pnpm-workspace.yaml
```

> **Note:** the memory service may start as a single container (`apps/notes-api`) or be decomposed into sibling apps (e.g., `notes-worker` for extraction, `outbox-relay` for downstream consumers) as implementation needs warrant. The product boundary is the *memory service*, not any single container.
