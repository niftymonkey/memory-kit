# memory-kit — Product Requirements Document

## Problem Statement

I continuously encounter moments in daily life where useful information needs capturing — who replaced the water heater, when I last changed the furnace filter, what restaurant was recommended, what commitments I've already made for next weekend — but these facts get lost in the normal flow of a day. When I later need them (*"who did that sprinkler job last year?"*), I can't find them, or the recall takes more effort than the fact was worth.

I want a voice-first capture tool that remembers these fragments and lets me interrogate my own memory in natural language. And I want to build it as R&D — one that validates the memory-layer patterns that will eventually power **Mnemo**, a private AI assistant with long-term memory.

Full context: `docs/exploration-summary.md` and `docs/mnemo-carry-forward.md` (the latter is load-bearing; its stance — *learning over shipping* — is an implicit requirement on every implementation decision below).

## Solution

A containerized memory service (`notes-api`) plus a dogfood web UI (`web-ui`) that together:

1. Capture voice or typed notes via a web UI, transcribing voice via cloud Whisper (no audio retained).
2. Extract structured meaning from each note — mentions, events, commitments, preferences — via a single LLM structured-output call per note.
3. Store facts in a Postgres + pgvector hybrid schema: raw notes with embeddings + derived structured rows cross-linked by foreign key.
4. Let me interrogate my memory via a chat UI — natural-language query, streaming answer, inline source-note citations, powered by a bounded tool-calling retrieval agent.
5. Preserve every capture as append-only; support corrections as *new* notes with a foreign key to the original; keep derived rows mutable so queries stay fast; emit every mutation to an `outbox` table for future consumers (Graphiti evaluation in Phase 2).

The memory service is the product; the web UI is a dogfood surface.

## User Stories

1. As a user, I want to record a voice note from the web UI, so that I can capture information in the moment without typing.
2. As a user, I want to type a note as an alternative to voice, so that I can capture in contexts where speaking aloud is impractical.
3. As a user, I want every captured note to appear in a chronological list, so that I can sanity-check what's been captured.
4. As a user, I want each note in the list to show the structured data extracted from it (events, commitments, preferences, mentions), so that I can verify extraction quality during dogfood.
5. As a user, I want to ask natural-language questions in a chat interface, so that I can recall information without remembering which note it came from.
6. As a user, I want answers to stream token-by-token, so that the UI feels responsive.
7. As a user, I want each answer to cite the source note(s) it drew from, so that I can verify the answer and click through to the original.
8. As a user, I want the chat to preserve conversation context across turns, so that I can follow up without restating context.
9. As a user, I want time-based queries (*"what's on next weekend?"*, *"what events happened last week?"*) to return accurate date-filtered results, so that relative-time queries work.
10. As a user, I want entity-loose queries (*"who fixed the plumbing?"*) to succeed via vector search + LLM synthesis, so that I don't need to remember exact names.
11. As a user, I want to voice-record a query as an alternative to typing, so that query uses the same input modality as capture.
12. As a user, I want to edit a note when I realize I said something wrong, so that my memory stays accurate over time.
13. As a user, I want corrections to create a new note that references the original (not overwrite it), so that capture history is preserved.
14. As a user, I want corrections to update the derived extracted rows, so that queries return the current state.
15. As a user, I want a commitment's status to change from pending to done or cancelled, so that *"what's pending?"* queries remain meaningful over time.
16. As a user, I want to see a list of stale pending commitments (older than 30 days), so that I can close or reaffirm them during a weekly review.
17. As a developer, I want to swap the extraction LLM between cloud providers and Mnemo's local `ai-gateway` via config, so that I can empirically compare structured-output quality across providers.
18. As a developer, I want the memory service to expose a versioned HTTP API, so that Mnemo can consume it as a sibling container without version fragility.
19. As a developer working on Mnemo later, I want to drop the memory service's container(s) into Mnemo's `docker-compose.yml` and have the API be immediately usable, so that adoption has low friction.
20. As a developer, I want every mutation to emit an outbox event, so that a future graph-sync consumer (Phase 2) can build a Graphiti knowledge graph without retrofitting.
21. As an R&D practitioner, I want Evalite evals for extraction, time resolution, and retrieval, so that I can measure LLM-behavior changes with evidence instead of intuition.
22. As an R&D practitioner, I want a discover-candidates script using pickai v2, so that I can shortlist models to evaluate before running full evals.
23. As an R&D practitioner, I want every resolution path (time, extraction, tool-call, correction) to log which tier or tool "won," so that dogfood data produces the findings the carry-forward doc expects.
24. As an R&D practitioner, I want the memory service instrumented so that the *"Specific Questions memory-kit Answers for Mnemo"* table in the carry-forward doc can be answered with measured evidence, so that the project's R&D yield is real, not claimed.

## Implementation Decisions

### Architectural principles (guardrails on every decision that follows)

Lifted from `docs/mnemo-carry-forward.md` § "Design Principles Proven Here." Enforced during implementation; not optional.

- **Additive, never destructive.** No deletes. Retractions are corrections with status flips.
- **Raw source always retained.** `notes.raw_text` is immutable once captured.
- **Outbox from day 1.** Every mutation emits an event to the `outbox` table inside the same transaction.
- **Versioned HTTP API from the first commit.** `/v1/*` stable; breaking changes require a version bump.
- **Provider-agnostic LLM access via an OpenAI-compatible interface.** Cloud vs local-Ollama swap is a config change, not a code change.
- **Instrumentation is the deliverable.** Every resolution path (time-resolver tier, extraction model, tool calls, correction type) is logged.

### Module decomposition inside the memory service

Five deep modules, each behind a small typed interface. Interface shapes are defined in a shared monorepo package consumed by all apps; Zod schemas where input/output validation matters, TypeScript interfaces where validation is internal.

1. **Extraction** — raw note + context → structured output (`{ mentions, events, commitments, preferences, isNoise, modelUsed, promptVersion }`). Single LLM call per note via Vercel AI SDK's `generateObject` + a Zod schema. LLM backend is swappable via config.
2. **TimeResolver** — post-extraction arbitration. Input: phrase + LLM-produced ISO candidate + reference date + timezone. Output: final ISO + precision + resolver tag (`both_agree`, `library_override`, `llm_only`, `unresolved`). Internal logic: `chrono-node` re-parse; chrono wins on disagreement; LLM stands when chrono can't resolve; nothing resolves → `unresolved` + null ISO with phrase preserved. Async interface for future-proofing.
3. **Storage** — Drizzle-backed repository surface over Postgres. One repository per table (`notes`, `events`, `commitments`, `preferences`, `outbox`). Every mutation emits an outbox row inside the same transaction — this is enforced by the repository implementation, not the caller.
4. **Retrieval** — user query + chat history → streaming answer with citations and tool-call trace. Internally runs a bounded tool-calling agent loop (default: max 5 iterations, 30s timeout; both configurable). Tools are composed internally from Storage repositories; the agent does not see Postgres directly.
5. **Embedder** — text → vector. Thin wrapper over the configured embedding provider. Supports single and batch operations.

### Write path — capture

1. `POST /v1/notes` with `{ rawText, capturedAt, timezone, source: "voice" | "text" }`. Whisper transcription happens in the capture pipeline before this call; no audio is stored.
2. Service embeds the raw text via the Embedder module.
3. Extraction module produces the structured output.
4. For every event/commitment with a date phrase, the TimeResolver arbitrates to a final ISO + precision + resolver.
5. Storage writes the notes row + derived rows + outbox rows in one transaction.
6. Response: `{ note, extracted: { events, commitments, preferences } }`.

### Write path — correction

1. `POST /v1/notes/:id/correct` with `{ rawText, capturedAt, timezone, source, correctionType: "amendment" | "retraction" | "reschedule" | "status_change" }`.
2. Extraction runs with correction context — the original note's raw text is passed to the prompt so the LLM can identify what's changing.
3. Extraction output identifies target derived rows (by `note_id` FK) plus field-level changes.
4. Storage inserts a new `notes` row with `corrects_note_id` FK set; updates the identified derived rows in place, setting `last_correction_note_id` FK.
5. Outbox emits one `note_created` event plus one `*_updated` event per affected derived row.

### Query path

1. `POST /v1/query` with `{ query, chatHistory? }`. Response streams in the Vercel AI SDK streaming format.
2. Retrieval loops the bounded agent against a tool set wired from Storage repositories:
   - `searchNotes(queryText, dateRange?, hasMentions?)` — vector + structured filter
   - `searchEvents(queryText?, eventType?, dateRange?)`
   - `searchCommitments(queryText?, status?, dueBefore?, dueAfter?)`
   - `searchPreferences(queryText?, subject?)`
   - `getNoteById(id)`
3. Streams tokens back with structured citation metadata interleaved.

### Other MVP endpoints

- `GET /v1/notes`, `GET /v1/notes/:id` — list and read-by-id
- `GET /v1/events`, `GET /v1/events/:id` — search/list and read
- `GET /v1/commitments`, `PATCH /v1/commitments/:id` — PATCH is specifically for status updates (`pending` → `done` | `cancelled`); other changes go through the correction flow
- `GET /v1/preferences`
- `GET /v1/outbox` — dev/debug only; feature-flagged off by default in MVP
- `GET /v1/health`, `GET /v1/version` — ops

All endpoint request/response shapes are defined as Zod schemas in the shared package; the schema is the source of truth. Changes to `/v1/*` shapes require a version bump.

### Schema

See `docs/exploration-summary.md` § "Final Schema Snapshot" for the canonical schema. The schema lives in code as Drizzle migrations — not duplicated here because it drifts. MVP requires no schema additions beyond what that snapshot specifies.

### LLM provider topology

All LLM calls (Extraction, Retrieval tool loop, any future TimeResolver fallback) go through the Vercel AI SDK configured against an OpenAI-compatible base URL.

- Default: cloud provider (Anthropic or OpenAI) for speed-of-iteration and best-in-class structured output accuracy.
- Alternative: Mnemo's running `ai-gateway` via Docker network — enables local-model evaluation across the same capture/query workload.
- Configurable per call site so Extraction and Retrieval can be swapped independently.

Whisper transcription is a direct cloud call from the Next.js server action (review-kit pattern). No `whisper-api` container in this project.

### Outbox pattern

- Columns: `id`, `event_type`, `payload` (JSONB — full row snapshot), `emitted_at`, `consumed_by` (JSONB — `{ consumerName: timestamp }`).
- MVP event types: `note_created`, `note_corrected`, `event_created`, `event_updated`, `commitment_created`, `commitment_updated`, `preference_created`, `preference_updated`.
- No consumers in MVP. Phase 2's `graph-sync` (Graphiti evaluation) reads and marks consumption.

### Container shape

`apps/notes-api` is the root of the memory service. For MVP it may ship as a single container. Sibling apps (`notes-worker` for extraction batching, `outbox-relay` for downstream consumers, etc.) may be introduced if implementation needs warrant decomposition. **The product boundary is the memory service, not any one container.**

Postgres runs as its own container with the `pgvector` extension enabled.

### Config — all via env vars (12-factor)

`DATABASE_URL`, `LLM_MODE` (`cloud` | `local`), `LLM_PROVIDER_BASE_URL`, `LLM_MODEL_EXTRACTION`, `LLM_MODEL_RETRIEVAL`, `EMBEDDING_PROVIDER_BASE_URL`, `EMBEDDING_MODEL`, `WHISPER_API_KEY`, `NODE_ENV`, `LOG_LEVEL`, plus Evalite-side `EVAL_OPENROUTER_API_KEY`.

## Testing Decisions

Two-track strategy: **deterministic code → Vitest; LLM behavior → Evalite.**

### What makes a good test in this project

- **External behavior only.** If a test would break on a correct refactor, it's testing the wrong thing.
- **Invariants over narrow behaviors.** The bugs that would hurt are invariant violations — FK integrity, correction chain walkability, outbox consistency inside transactions, API contract drift — not UI regressions.
- **Evals measure LLM quality with ground truth, not anecdote.** Curated fixtures, scorers split into **gate** (must-pass) and **ranking** (quality), and cross-model comparisons are first-class.

### Per-module coverage

| Module | Vitest unit | Vitest integration | Evalite |
|---|---|---|---|
| Storage | Yes | Yes (real Postgres via testcontainer or shared dev DB) | — |
| TimeResolver | Yes (chrono-node path) | — | Yes (LLM path + arbitration outcomes) |
| Extraction | Minimal (shape transforms only) | — | Yes |
| Retrieval | Yes (iteration cap, timeout enforcement) | Yes (seeded DB fixture + deterministic tool responses) | Yes (agent quality, tool-call appropriateness) |
| Embedder | Skip | — | — |

### API contract tests

Every `/v1/*` endpoint gets a contract test: request shape, response shape, status codes. Schemas derived from the shared Zod definitions, so drift breaks the test.

### Evalite setup

Mirroring the review-kit layout: an **evals package** in the monorepo with its own minimal Evalite config (timeout + concurrency tuned to cloud runs; bumped for local-Ollama runs), one `.eval.ts` file per LLM module (extraction, time-resolver, retrieval), reusable scorer modules, JSON fixtures organized by scenario category, and a `discover-candidates` script built on pickai v2.

- **Purpose profile for memory-kit**: reasoning (3), instruction-following (3), cost (2), structured-output required, min 32K context. Output tiers: speed-first, balanced, quality.
- **Gate scorers (examples)**: Zod conformance on Extraction output; valid ISO format from TimeResolver; citations present on Retrieval answers; answer returned within iteration budget.
- **Ranking scorers (examples)**: field-level accuracy vs fixture ground truth; ISO day-bucket accuracy (exact = 1.0, ±1 day partial); tool-call appropriateness; answer correctness (LLM-as-judge for subjective queries, deterministic for fact-lookups).
- **Provider isolation**: `EVAL_OPENROUTER_API_KEY` keeps eval costs separate from runtime LLM costs.
- **Not CI-wired.** Evals are a manual/dev-time R&D workflow.

### Prior art (reference implementations)

- `niftymonkey/champ-sage` — fixtures + Evalite + pickai pattern for coaching recommendations.
- `niftymonkey/review-kit` — fixtures + Evalite + pickai pattern for review generation. Closer structural match (also pnpm monorepo).
- Borrow gate/ranking scorer split, fixture organization, pickai purpose profile pattern, and OpenRouter API-key isolation from these.

### R&D harness (optional, reduced scope)

A thin CLI to eyeball a single note through the full pipeline may be kept for one-off ad-hoc captures during prompt tuning. Evalite's `eval:watch` and `eval:serve` cover most of what this was originally scoped to do.

## Out of Scope

MVP will not include (references: `docs/exploration-summary.md` § "Nice-to-have / Explicitly deferred" and `docs/mnemo-carry-forward.md` § "Non-Goals"):

- **Expo mobile app** — Phase 2.
- **Graphiti evaluation via `apps/graph-sync`** — Phase 2 (outbox readiness is in MVP; the consumer is not).
- **Voice-driven corrections inside the chat UI** — Phase 2+.
- **MCP server / CLI for AI-harness ingestion** — Phase 3+.
- **Canonical entity resolution** — Graphiti's domain, intentionally deferred.
- **Auto-archival, auto-decay, cold storage tiers, summarization compression** — all reachable as additive post-MVP moves thanks to the outbox + additive-never-destructive principle.
- **Auth architecture** — single-user MVP; API designed to accept an auth header later but no auth enforcement.
- **Multi-user / team semantics.**
- **Performance at scale, load testing, longitudinal corpus behavior.** Single-user dogfood is the test.
- **E2E UI tests (Playwright).** Dogfood replaces this.
- **Shared types package (`packages/memory-schema`) extraction** — HTTP contract is the default; extract only if a consumer can't realistically use HTTP.

## Further Notes

### Relationship to the other planning documents

This PRD sits on top of two load-bearing docs in the same repository:

- `docs/exploration-summary.md` — decisions locked from the explore-idea session. *What* to build.
- `docs/mnemo-carry-forward.md` — *why* the project exists, what must be measured, what must not be built. Guardrails.

The PRD is **derived from both**. It must be read alongside them, not as a standalone spec. Non-goals are lifted from the carry-forward doc; design principles from the carry-forward doc are implicit requirements on every implementation decision here. The carry-forward stance — *learning over shipping* — applies: when tension arises between shipping fast and producing measurable R&D yield, R&D yield wins.

When a major decision in this PRD is revisited (for example, deciding to build canonical entity resolution after all), both `exploration-summary.md` and `mnemo-carry-forward.md` are revisited in the same pass. The three documents stay synchronized or they are all considered stale.

### Success criteria for MVP

Lifted from `docs/mnemo-carry-forward.md` § "Success Criteria for This Project":

1. Runs reliably enough to dogfood for 6–8 weeks with real captures.
2. Retrieval agent answers realistic queries from the original brief with correctness I trust.
3. Dogfood data breadth can answer at least 5 of the 8 *"Specific Questions memory-kit Answers for Mnemo"* with measured evidence.
4. The memory service (however many containers) is portable into Mnemo's `docker-compose.yml` with a documented HTTP API contract.
5. Phase 2 Graphiti evaluation scaffolding is viable — outbox works, consumer pattern is proven conceptually viable even if not yet implemented.

### Open questions (deferred, not oversights)

Lifted from `docs/exploration-summary.md` § "Open Questions." These are empirical questions whose answers require implementation + dogfood data:

- **Noise threshold tuning.** Starting stance: *permissive* — extract anything with discernible signal; tune threshold based on dogfood noise.
- **Retrieval agent prompt engineering.** Starting stance: terse tool descriptions, 5-iteration hard cap, ~30s timeout.
- **Stale-commitment review UX.** Dogfood will reveal the right cadence and surface (weekly ritual vs dashboard banner vs notification).
- **Graphiti adoption path for Mnemo.** Phase 2 output.
- **Shared types package.** Default: HTTP-only.
