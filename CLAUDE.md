# memory-kit

R&D vehicle for Mnemo's memory layer. Voice-first note capture + chat-style natural-language interrogation, built to prove patterns that drop into Mnemo later.

## Stance

**Learning over shipping.** If a shortcut produces working code but weaker R&D yield, learning wins. The output is not a notes app — it's validated patterns, measured findings, and a drop-in memory service for Mnemo.

## Hard Rules (non-negotiable)

- Additive, never destructive. No deletes. Retractions flip status.
- `notes.raw_text` is immutable once captured.
- Every mutation emits an outbox event inside the same transaction — enforced at the repo layer.
- Versioned HTTP API (`/v1/*`) from day one. Breaking changes bump the version.
- LLM access is provider-agnostic via an OpenAI-compatible base URL. Cloud/local swap is a config change.
- Instrumentation is the deliverable. Every resolution path logs which tier or tool won.

## Non-Goals (MVP)

Don't build without re-scoping in `docs/mnemo-carry-forward.md` and `docs/exploration-summary.md`:

- Canonical entity resolution (Graphiti's domain; Phase 2).
- Expo mobile app · Graphiti `graph-sync` (Phase 2) · MCP ingestion CLI (Phase 3+).
- Auth enforcement · auto-decay / auto-archival / auto-deletion · E2E Playwright for the UI.

## Context — Load on Demand

Don't preload all three planning docs. Pull them into context when the task touches their scope.

- **`docs/mnemo-carry-forward.md`** — *why, stance, guardrails.* Load for architectural decisions or when recording R&D findings.
- **`docs/exploration-summary.md`** — *what.* Load for locked decisions or when resolving open questions.
- **`docs/prd.md`** — *how.* Load for implementation specifics: module interfaces, endpoint shapes, testing split.

**Tiebreaker when docs conflict:** carry-forward > exploration-summary > PRD. Intent outranks spec detail. If all three diverge, stop and escalate — don't pick a side silently.

## R&D Scorecard

`docs/mnemo-carry-forward.md` § *"Specific Questions memory-kit Answers for Mnemo"*. Every feature with a row there lands with its instrumentation implemented. As findings accumulate during dogfood, write the answers into that table in place.

## Prior Art (Sibling Projects)

- **`~/dev/niftymonkey/review-kit`** — closest architectural analog. Voice capture, Evalite, `discover-candidates` pattern.
- **`~/dev/niftymonkey/champ-sage`** — scorer conventions (gate vs ranking).
- **`~/dev/niftymonkey/mnemo`** — downstream consumer; architectural DNA (OpenAI-compatible `ai-gateway`, pgvector direction, containerized TS services).

## Project Defaults

pnpm monorepo · Docker Compose · TypeScript strict · Postgres + pgvector + Drizzle · Vercel AI SDK against an OpenAI-compatible base URL · cloud Whisper (review-kit pattern; no audio retained) · Vitest (unit + integration + contract) · Evalite (LLM behavior) · `@niftymonkey/ai-toolkit` for model catalog.

## Testing (TDD)

- Tests before implementation for testable code.
- **Red phase fails on assertions, not missing modules** — stub the module with empty returns so it compiles, then write failing tests, then implement.
- Test external behavior through public interfaces, not implementation details.
- Mock external dependencies (LLM, network). Don't mock internal modules.
- Factory functions for fixtures that might be mutated — no shared mutable state.
- **Bug found → failing test first, then fix.**
- LLM-behavior correctness lives in Evalite, not Vitest. Scorers split into gate (must-pass) and ranking (quality). Evals hit real models via isolated API key; not CI-wired.

## Scripts

No inline bash/tsx one-liners for anything worth running twice. Put it in `scripts/*.ts` with a matching `package.json` entry.

## Living Documents

Update the planning docs in the same commit as the code change:

- Findings → `docs/mnemo-carry-forward.md` § Specific Questions (answer in place, don't just collect).
- Resolved questions → `docs/exploration-summary.md` § Open Questions.
- Spec changes → `docs/prd.md`.

If a change shifts the stance, revisit all three in the same pass.

## Commands

*(Populated as code lands: `pnpm dev`, `pnpm test`, `pnpm typecheck`, `pnpm eval{,:watch,:serve,:discover}`, plus Docker Compose dev commands matching Mnemo's pattern.)*
