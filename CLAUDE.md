# memory-kit

R&D vehicle for Mnemo's memory layer. Voice-first note capture + chat-style natural-language interrogation, built to prove patterns that drop into Mnemo later.

## Load These Before Working

Every coding session MUST read all three, in this order:

1. **`docs/mnemo-carry-forward.md`** — *why this project exists, what must be learned, what not to build.* Guardrails + stance.
2. **`docs/exploration-summary.md`** — *what to build.* Decisions locked from the explore-idea session.
3. **`docs/prd.md`** — *how to build it.* Implementation-facing spec derived from the two above.

### Tiebreaker rule

When the docs conflict, precedence is: **carry-forward > exploration-summary > PRD.** This mirrors the stability gradient (intent rarely changes; implementation spec changes often). If all three diverge on something important, stop and escalate — don't pick a side silently.

## Stance

> This project values **learning over shipping**. When the two are in tension — when there's a shortcut that would produce working code but weaker R&D yield — **learning wins.** The output of memory-kit is not a notes app; it's a set of validated patterns, measured findings, and a drop-in memory service that Mnemo can consume.

(Verbatim from `mnemo-carry-forward.md` § Stance.)

## Hard Rules

Non-negotiable. From `mnemo-carry-forward.md` § Design Principles:

- **Additive, never destructive.** No deletes. Retractions are corrections that flip status; originals remain.
- **Raw source always retained.** `notes.raw_text` is immutable once captured.
- **Outbox emission in-transaction.** Every mutation emits an event to `outbox` in the same transaction. Enforced at the repository layer, not opt-in at the call site.
- **Versioned HTTP API (`/v1/*`) from the first commit.** Breaking changes require a version bump.
- **Provider-agnostic LLM access via OpenAI-compatible gateway.** Cloud/local swap is a config change, not a code change.
- **Instrumentation is the deliverable.** Every resolution path (time, extraction, tool-call, correction) logs which tier or tool won.

### Non-Goals (MVP)

Do not build these without explicit re-scoping in **both** `mnemo-carry-forward.md` and `exploration-summary.md`:

- Canonical entity resolution (Graphiti's domain, deferred)
- Expo mobile app (Phase 2)
- Graphiti integration / `apps/graph-sync` (Phase 2)
- MCP server / AI-harness ingestion CLI (Phase 3+)
- Auth enforcement (designed-for, not implemented)
- Auto-decay, auto-archival, auto-deletion anywhere
- E2E Playwright tests for the UI (dogfood replaces this)

## R&D Measurement

The *"Specific Questions memory-kit Answers for Mnemo"* table in `mnemo-carry-forward.md` is the R&D scorecard. Every feature that has a row in that table must land with its instrumentation target implemented. If a feature can't be meaningfully instrumented, that's a design conversation — not a shortcut to skip.

As findings accumulate during dogfood, write the answers into that table in place. The docs are living.

## Prior Art (Sibling Projects)

When a problem looks like one of these has solved before, read theirs first:

- **`~/dev/niftymonkey/review-kit`** — closest architectural analog (pnpm monorepo, web + mobile apps, Evalite + pickai evals). Primary reference for voice capture UX, Evalite setup, and the `discover-candidates` script pattern.
- **`~/dev/niftymonkey/champ-sage`** — Evalite + pickai pattern for structured-output AI. Secondary reference for scorer conventions (gate vs ranking).
- **`~/dev/niftymonkey/mnemo`** — the downstream consumer. Architectural DNA: containerized TS services, OpenAI-compatible `ai-gateway`, pgvector direction per `docs/database-architecture.md`. memory-kit's notes-api is built to drop into Mnemo's compose later.

## Project Defaults (Locked)

- **Monorepo:** pnpm workspaces; Docker Compose for service orchestration.
- **Language:** TypeScript, strict mode.
- **Database:** Postgres + pgvector extension + Drizzle ORM.
- **LLM access:** Vercel AI SDK against an OpenAI-compatible base URL (cloud default; Mnemo `ai-gateway` via config for local-model evaluation).
- **Voice transcription:** cloud Whisper called directly server-side (review-kit pattern). No audio retained.
- **Testing:** Vitest (unit + integration + API contract) + Evalite (LLM behavior evals). Real Postgres for integration tests.
- **Model catalog:** `@niftymonkey/ai-toolkit` for model metadata and provider availability.

## Code Conventions

### TypeScript

- No `any` except at genuinely untyped external boundaries; type the value at the boundary.
- No intersection hacks (`Type & { extra }`) — make a proper interface that owns the full shape.
- No unnecessary `as` casts. Structure code so types flow naturally. `as` is acceptable only at SDK/library boundaries returning generic types.
- Strict mode: no implicit any, no unused locals, no unused parameters.

### React (web-ui only)

- Default to Server Components; add `'use client'` only when needed.
- Keep client components small and low in the tree.
- Avoid `useEffect` unless syncing with an external system.
- Components: presentational < 50 lines, container < 200 lines.

### Testing (TDD)

- **Write tests before implementation.** Every time, for testable code.
- **Red phase must fail on assertions, not missing modules.** Stub the module with empty returns first so it compiles; then write failing tests against stubs; then implement until green.
- Test behavior through public interfaces, not implementation details.
- Mock external dependencies (LLM providers, network), not internal modules.
- Use factory functions for fixtures that might be mutated — no shared mutable state across tests.
- Invariants over narrow behaviors. The bugs that hurt are invariant violations (FK integrity, correction-chain walkability, outbox consistency), not UI regressions.
- **When a bug is found: write a failing test first, then fix.**

### LLM Behavior (Evalite)

- LLM-behavior correctness (extraction quality, time resolution, retrieval answers) is verified via Evalite, not unit tests.
- Scorers split into **gate** (must-pass: Zod conformance, valid ISO, citations present) and **ranking** (field-level accuracy, answer correctness, tool-call appropriateness).
- Fixtures live in the evals package; curated + categorized by scenario.
- Evals always hit real models via an isolated API key (`EVAL_OPENROUTER_API_KEY`), not CI-wired. Manual R&D workflow.
- Before starting a new eval sweep, run `discover-candidates` to refresh the model shortlist.

### Scripts

- No throwaway inline bash/tsx one-liners for debugging. If it's useful enough to run, make it a proper TypeScript file in `scripts/`.
- Every `scripts/*.ts` needs a corresponding `package.json` script entry so it runs from the repo root.

### General

- No throwaway/hardcoded data in production paths. Build with real infrastructure from the start.
- Prefer proper libraries over hand-rolled solutions (`chrono-node` over regex for date parsing, Drizzle over raw SQL templating, etc.).

## Living Documents — Update as You Work

The three planning docs are not write-once. As work lands:

- **`docs/mnemo-carry-forward.md`** § *"Specific Questions memory-kit Answers for Mnemo"* — answers get written in place as evidence lands. Don't just implement the instrumentation; record what it measured.
- **`docs/exploration-summary.md`** § *"Open Questions"* — resolved questions get resolved in place with their answers; remaining stays.
- **`docs/prd.md`** — major decision changes propagate here immediately. If a decision would shift the stance, revisit all three docs in the same pass.

When in doubt, this triad is the source of truth. Code implements the triad.

## Before Committing

Check whether the session produced:

- New findings that belong in `mnemo-carry-forward.md`'s Specific Questions table
- Resolved open questions that belong updated in `exploration-summary.md`
- Spec changes that belong in `docs/prd.md`

If yes, update the doc in the same commit as the code.

## Commands

*(Populated as code lands. Target surface area: `pnpm dev`, `pnpm test`, `pnpm typecheck`, `pnpm eval`, `pnpm eval:watch`, `pnpm eval:serve`, `pnpm eval:discover`, plus Docker Compose dev commands matching Mnemo's pattern — `pnpm dev:start`, `pnpm dev:rebuild`, `pnpm dev:logs`, `pnpm dev:stop`.)*
