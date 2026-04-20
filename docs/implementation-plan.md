# Plan: memory-kit MVP Implementation

> Source PRD: [`docs/prd.md`](./prd.md) — read alongside [`docs/exploration-summary.md`](./exploration-summary.md) (locked decisions) and [`docs/mnemo-carry-forward.md`](./mnemo-carry-forward.md) (guardrails + Specific Questions table that instrumentation must answer).

Tracer-bullet phasing: each phase is a thin vertical slice that cuts through all integration layers (schema → service → UI → tests → instrumentation) and is demoable on its own.

## Architectural decisions

Durable across all phases. Referenced here, not repeated per phase.

### Monorepo + orchestration

- **pnpm workspaces** for the monorepo.
- **Docker Compose** orchestration; memory service rooted at `apps/notes-api/` but free to decompose into sibling apps (`notes-worker`, `outbox-relay`, etc.) if implementation needs warrant.
- Workspace layout: `apps/{notes-api, web-ui}`, `packages/{shared, evals}`, `scripts/`.

### Routes (versioned from Phase 1)

- `POST /v1/notes` — capture (text in Phase 1, text+voice from Phase 2)
- `GET /v1/notes`, `GET /v1/notes/:id`
- `POST /v1/notes/:id/correct` — correction flow (lights up in Phase 5)
- `GET /v1/events`, `GET /v1/events/:id`
- `GET /v1/commitments`, `PATCH /v1/commitments/:id` (status; lights up in Phase 6)
- `GET /v1/preferences`
- `POST /v1/query` — retrieval (lights up in Phase 4; streaming)
- `GET /v1/outbox` — dev/debug, feature-flagged
- `GET /v1/health`, `GET /v1/version`

Any breaking change to `/v1/*` shapes requires a `/v2/*` bump.

### Schema (created whole in Phase 1; columns populate as features light up)

Five tables from Phase 1:

- `notes` — raw capture, embedding column (empty until Phase 4), mentions JSONB, correction FK fields (empty until Phase 5).
- `events` — derived; date fields include phrase, precision, resolver (resolver populated from Phase 3).
- `commitments` — derived; same date-field trio; status starts `pending` (transitions added in Phase 6).
- `preferences` — derived.
- `outbox` — event_type, payload JSONB, emitted_at, consumed_by; emission enforced at the repo layer from Phase 1.

Canonical shape: `docs/exploration-summary.md` § Final Schema Snapshot. Schema-as-code lives in Drizzle migrations.

### Modules (inside the memory service)

Five deep modules, each behind a typed interface in `packages/shared`:

- **Extraction** — raw note + context → structured output. Lands in Phase 1.
- **Storage** — repository surface; outbox emission in-tx enforced at this layer. Lands in Phase 1.
- **TimeResolver** — post-extraction arbitration. Lands in Phase 3.
- **Embedder** — text → vector. Lands in Phase 4.
- **Retrieval** — bounded tool-calling agent. Lands in Phase 4.

### External boundaries

- **LLM access:** Vercel AI SDK against an OpenAI-compatible base URL. `LLM_MODE={cloud,local}` swaps between cloud provider and Mnemo's `ai-gateway`. No own Ollama container in this project.
- **Whisper transcription:** cloud API, direct server-side call from the Next.js server action (review-kit pattern). No audio retained.
- **Embedding provider:** configured via env var.

### Testing strategy

- **Vitest** — unit, integration (real Postgres via testcontainer or shared dev DB), and API contract tests.
- **Evalite** — LLM-behavior evals in `packages/evals`. Scaffolded in Phase 1; grows per phase as LLM-using modules land. Never CI-wired; manual R&D workflow.
- **API contract tests** — Zod-derived request/response validation for every `/v1/*` endpoint.
- **No E2E Playwright** — dogfood replaces UI regression tests.

### Cross-cutting principles (from `mnemo-carry-forward.md`)

- Additive, never destructive. No deletes anywhere.
- `notes.raw_text` immutable once captured.
- Outbox emission in-transaction from Phase 1.
- Versioned HTTP API from the first commit.
- Instrumentation built in with each feature, not retrofitted. Every resolution path (time tier, tool call, correction type, model+prompt version) logs which tier/tool won.
- As findings accumulate, write answers into `mnemo-carry-forward.md` § Specific Questions table in place.

### Auth

Single-user MVP; no auth enforced. API accepts an optional auth header (ignored). No login UI.

### Out of scope for MVP (these get their own plans later)

- `apps/graph-sync` (Graphiti evaluation) — Phase 2 of the project.
- Expo mobile app — Phase 2 of the project.
- MCP ingestion CLI — Phase 3+ of the project.
- Entity resolution, auto-decay/archival, multi-user, load testing.

---

## Phase 1: Text capture + extraction + list view

**User stories:** 2, 3, 4, 17, 20, 21 (scaffold), 24 (partial)

### What to build

End-to-end write path for typed notes. A user types a note in the web UI; the service runs LLM extraction; raw + derived rows are written transactionally with outbox emission; the notes list displays raw text alongside extracted mentions, events, commitments, and preferences.

No voice, no time resolution, no embeddings, no query, no corrections yet. Date phrases from extraction are stored verbatim; precision/resolver fields remain null until Phase 3. Embedding columns exist but are empty until Phase 4.

Covers:

- Monorepo, Docker Compose, Postgres + pgvector container.
- Full Drizzle schema (all 5 tables created now — columns populate progressively).
- Extraction module (Vercel AI SDK `generateObject` + Zod).
- Storage module with in-transaction outbox emission.
- Notes API: `POST /v1/notes` (text), `GET /v1/notes`, `GET /v1/notes/:id`, `GET /v1/health`, `GET /v1/version`.
- Notes list UI with expandable extraction panel per row.
- Evalite package scaffolding with an initial `extraction.eval.ts` + starter fixtures + gate (Zod conformance) and first ranking scorers (field presence, noise-classification sanity).

### Acceptance criteria

- [ ] `pnpm dev` brings up Postgres, notes-api, and web-ui with hot reload.
- [ ] `pnpm deploy:local` (or equivalent) runs production containers identically (Mnemo parity).
- [ ] Typing a note in the web UI → note appears in the chronological list with raw text + extracted structure visible.
- [ ] `POST /v1/notes` writes to all relevant tables + outbox events in a single transaction. Transaction rollback on failure verified by integration test.
- [ ] Every `/v1/*` endpoint has a contract test that fails on schema drift.
- [ ] Storage module has unit tests plus an integration test against a real Postgres.
- [ ] `pnpm eval` runs `extraction.eval.ts` and produces gate + ranking scores against fixtures. `pnpm eval:serve` opens the Evalite UI.
- [ ] Every captured note logs `modelUsed` and `promptVersion`. Log format is structured (JSON) so instrumentation can be aggregated later.
- [ ] `LLM_MODE=cloud|local` env var swaps providers without code changes; manual verification that pointing at Mnemo's `ai-gateway` works if Mnemo is running locally.
- [ ] `mnemo-carry-forward.md` § Specific Questions table gains a baseline entry under "How far does hybrid-DB + LLM reasoning get without entity resolution?" — blank with "TBD after dogfood" is fine.

---

## Phase 2: Voice capture

**User stories:** 1

### What to build

Add voice capture to the notes UI. Browser MediaRecorder records audio; server action uploads to cloud Whisper; transcribed text is fed into the existing `POST /v1/notes` from Phase 1. No audio is retained anywhere.

Reuses review-kit's voice pattern: press-to-record, release-to-stop, audio level visualization, auto-timeout.

### Acceptance criteria

- [ ] Record a voice note in-browser → note appears in the list with the same extraction quality as a typed note.
- [ ] Audio buffer is discarded immediately after transcription; no audio artifacts on disk or in DB.
- [ ] Recording UX gives clear visual feedback (recording indicator, level meter, auto-stop on long silences).
- [ ] Whisper API key config via env var; network failure falls back to a visible error state (capture not silently dropped).
- [ ] No new backend endpoints; `POST /v1/notes` unchanged.

---

## Phase 3: Time resolution

**User stories:** 9 (storage-side; full query-side in Phase 4), 23 (time resolver logging)

### What to build

TimeResolver module arbitrates between the LLM's date candidate (from extraction) and a `chrono-node` re-parse of the original phrase. Every event and commitment gains a final ISO timestamp + precision + resolver tag (`both_agree` / `library_override` / `llm_only` / `unresolved`) + verbatim phrase preserved.

Notes list UI surfaces the resolved date, the original phrase, a precision badge, and which resolver won (so the user can spot regressions during dogfood).

Evalite `time-resolver.eval.ts` lands here with fixtures across common and edge phrases, gate scorer (valid ISO format or explicit `unresolved`), and ranking scorers (exact-match rate, day-bucket accuracy, resolver distribution).

### Acceptance criteria

- [ ] A note containing "next Saturday" (relative phrase) produces an event/commitment whose `occurred_at`/`due_at` equals the actual next Saturday from `captured_at`.
- [ ] Every date-bearing derived row stores phrase, ISO, precision, and resolver tag.
- [ ] Unresolvable phrases ("after the storm") persist with `iso=null` and `precision='unresolvable'`; the note and phrase are still retained.
- [ ] Notes list shows resolved date + precision + resolver next to each event/commitment.
- [ ] TimeResolver unit tests cover the chrono-node path deterministically; `time-resolver.eval.ts` runs and produces scored output against LLM path.
- [ ] Aggregate logs show resolver distribution (what % resolved by which tier) — this becomes evidence for the carry-forward question "does LLM-alone time normalization work, or is chrono-node validation worth it?".

---

## Phase 4: Chat-based retrieval (text + voice query)

**User stories:** 5, 6, 7, 8, 9 (query-side), 10, 11, 23 (retrieval tool-call logging), 24 (full instrumentation)

### What to build

This is the fattest phase. Memory becomes interrogable.

- **Embedder module** integrated into the write path for new notes; one-time backfill script embeds all rows written in Phases 1-3.
- **Retrieval module** — bounded tool-calling agent (default: 5 iterations, 30s timeout). Tool surface composed from Storage repositories: `searchNotes`, `searchEvents`, `searchCommitments`, `searchPreferences`, `getNoteById`. Tool calls logged (trace, tool name, arguments, results summary).
- **`POST /v1/query`** — streaming response in the Vercel AI SDK format; citations interleaved as structured metadata.
- **Chat UI** — multi-turn thread with streaming tokens, inline clickable citations to source notes, conversation history preserved per-thread. Voice input for queries reuses Phase 2's capture pipeline (Whisper → query text → same `/v1/query`).
- **`retrieval.eval.ts`** — scored against fixture queries with expected answer shape and tool-call appropriateness. Gate scorers: citations present, answer returned within iteration budget. Ranking scorers: answer correctness (LLM-as-judge for subjective, deterministic for lookups), tool-call appropriateness.

### Acceptance criteria

- [ ] All Phase 1-3 notes are embedded via backfill; new notes are embedded as part of the write path.
- [ ] Asking "what's happening next weekend?" returns a streaming answer citing the relevant commitments/events with clickable note references.
- [ ] Asking "who fixed the plumbing?" returns an answer derived from vector-search notes + LLM synthesis, with source citations.
- [ ] Follow-up questions ("what else happened that day?") succeed with conversation context; retrieval doesn't restart from scratch.
- [ ] Iteration cap and timeout are enforced and logged when hit; the UI surfaces a clear "couldn't answer in budget" state.
- [ ] Every query logs its tool-call trace for instrumentation.
- [ ] Voice query produces the same answer quality as text query of the same phrase.
- [ ] `retrieval.eval.ts` runs and produces scored output.
- [ ] Instrumentation now supports answering the carry-forward questions about "what tools does the agent actually call most?" and "structured vs vector split in practice."

---

## Phase 5: Corrections

**User stories:** 12, 13, 14

### What to build

Notes become correctable without losing history. Editing a note in the UI invokes `POST /v1/notes/:id/correct` with a new raw text and a `correctionType`. A new notes row is inserted with `corrects_note_id` FK + correction metadata; the LLM extraction step runs with correction context (original note's text passed to the prompt); identified target derived rows are updated in place, their `last_correction_note_id` set; outbox emits `note_created` + one `*_updated` event per affected row.

Notes list renders correction chains visibly: originals show an "amended by …" badge, corrections show "corrects …", and the query layer surfaces current state by default but can show chain history on demand.

### Acceptance criteria

- [ ] Editing a note creates a new correction note with FK; original raw text is preserved byte-for-byte.
- [ ] Affected events/commitments/preferences update in place; `last_correction_note_id` set.
- [ ] Outbox contains both the new-note event and per-derived-row update events, all committed in the same transaction.
- [ ] Notes list shows correction chains with clear before/after.
- [ ] Retrieval answers default to post-correction state; retrieval can cite corrections when relevant.
- [ ] Integration test: multi-hop correction chain (note → correction → correction) walks correctly via FK traversal and outbox replay.
- [ ] `mnemo-carry-forward.md` § Specific Questions gains a data point for "how often do corrections happen in real capture?" — query ready, data accumulates during dogfood.

---

## Phase 6: Commitment status + stale review

**User stories:** 15, 16

### What to build

Commitments gain a usable lifecycle.

- **`PATCH /v1/commitments/:id`** for status transitions (`pending` → `done` | `cancelled`). Patch emits a `commitment_updated` outbox event with the old/new status in payload.
- **Notes list / dashboard surface** showing pending commitments with `due_at < now - 30 days`. Each stale item offers inline actions: mark done, mark cancelled, leave as-is. Bulk operations acceptable but not required.
- **Status-aware retrieval** — queries about "what's pending?" filter `status='pending'` by default; queries can opt in to include completed/cancelled.

### Acceptance criteria

- [ ] Toggling a commitment's status via UI reflects immediately in subsequent queries.
- [ ] Status PATCH emits the correct outbox event.
- [ ] Stale-review surface lists exactly the pending commitments older than 30 days (configurable threshold if trivial).
- [ ] Retrieval agent respects status in its default tool calls for "pending" queries.
- [ ] Dogfood value: the stale-review feature becomes the weekly ritual surface.

---

## Phase 7: Discover-candidates + cross-model eval sweep

**User stories:** 22, 17 (exercised end-to-end), 24 (final wiring)

### What to build

Close the R&D loop.

- **`scripts/discover-candidates.ts`** using pickai v2. Purpose profile for memory-kit: reasoning (3), instruction-following (3), cost (2), structured-output required, min 32K context. Output tiers: speed-first, balanced, quality. Mirrors the review-kit / champ-sage pattern.
- **Run one full eval sweep** comparing cloud provider(s) vs Mnemo's `ai-gateway` across `extraction.eval.ts`, `time-resolver.eval.ts`, and `retrieval.eval.ts` on the current fixture corpus.
- **Record findings** in `docs/mnemo-carry-forward.md` § Specific Questions table — especially the "cloud vs local extraction quality" row and the "does LLM-alone time normalization work?" row. Answers get written in place with the measured data and a one-line finding.

### Acceptance criteria

- [ ] `pnpm eval:discover` produces a human-readable shortlist of model candidates grouped by tier.
- [ ] Eval sweep runs against at least 3 models (e.g., one cloud default, one cloud budget, one local via `ai-gateway`) and produces comparable scored output.
- [ ] `mnemo-carry-forward.md` § Specific Questions has at least two answered rows with measurable findings.
- [ ] Findings committed in the same pass as the eval run (living-documents workflow).

---

## Post-MVP (separate plans when we get there)

Not scoped here. Each gets its own PRD/plan pass:

- **Phase 8 — Graphiti evaluation via `apps/graph-sync`** reading the outbox and building a knowledge graph. Produces the "Graphiti adoption decision" doc for Mnemo.
- **Phase 9 — Expo mobile app** (`apps/mobile`) for in-the-moment voice capture; reuses the same memory service API.
- **Phase 10 — MCP/ingestion surface** — AI-harness interface for pushing into memory-kit from Claude Code sessions or other agents.

Post-MVP triggers: MVP dogfooded for 6-8 weeks; carry-forward Success Criteria 1-3 met.
