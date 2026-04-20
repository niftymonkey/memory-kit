# memory-kit → Mnemo: R&D Carry-Forward

## Purpose

This document is orthogonal to `exploration-summary.md`. That doc captures *what* memory-kit is. This doc captures *why* memory-kit exists as an R&D vehicle for Mnemo, and *how* its outputs flow forward into Mnemo's production memory layer.

## Stance

This project values **learning over shipping**. When the two are in tension — when there's a shortcut that would produce working code but weaker R&D yield — **learning wins**. The output of memory-kit is not a notes app; it's a set of validated patterns, measured findings, and a drop-in memory service that Mnemo can consume.

"Does the code run?" is necessary but not sufficient. *"Did we learn what we came to learn?"* is the real definition of done, as formalized in the Success Criteria section below.

If this stance ever shifts (e.g., the project pivots toward product delivery), both `exploration-summary.md` and this document must be revisited before implementation continues.

## Core Hypothesis

> A memory service built on Postgres + pgvector + LLM extraction, *without* canonical entity resolution, can cover roughly 80% of natural-language memory interrogation. Graphiti is worth bolting on later specifically for the remaining 20% — entity-centric aggregation and cross-note relationships. Everything else is achievable with hybrid-DB + agent retrieval alone.

memory-kit's job is to validate this hypothesis with real dogfood data. If the 80% holds, Mnemo's memory layer has a known-good foundation. If it doesn't hold, we learn where and why *before* building it into Mnemo.

## What memory-kit Produces for Mnemo

1. **The memory service** (rooted at `apps/notes-api/`) — production-candidate, packaged as one or more containers depending on implementation needs. Drops into Mnemo's `docker-compose.yml` alongside `web-ui`, `ai-gateway`, `ollama`, `whisper-api`.
2. **Drizzle schema** — hybrid structured + vector memory, potentially extractable as `packages/memory-schema` for Mnemo import.
3. **Extraction pipeline** — prompt + Zod schemas, battle-tested on real captured notes.
4. **Retrieval tool surface** — the lightweight bounded agent and its tool definitions, portable to Mnemo's chat flow.
5. **Evidence** — instrumented dogfood data answering specific Mnemo design questions (see below).
6. **Graphiti adoption playbook** (Phase 2 output) — a worked example of outbox → graph-sync → retrieval that de-risks Graphiti adoption in Mnemo.

## Specific Questions memory-kit Answers for Mnemo

Each question has a corresponding instrumentation target in the memory-kit implementation, so we get real answers, not guesses.

| Question | Instrumentation |
|---|---|
| How far does hybrid-DB + vector + LLM reasoning get without entity resolution? | Dogfood for N weeks; tag queries that fail or degrade; categorize failure modes |
| What's the right structured vs vector split in practice? | Log which tool the retrieval agent called for each query; analyze frequency and success |
| Does LLM-alone time normalization work, or is chrono-node validation worth it? | `occurred_at_resolver` / `due_at_resolver` field logs which path resolved each date |
| What tools does the retrieval agent actually call most? Which never get called? | Tool-call telemetry on every query; analyze N=hundreds |
| How often do corrections happen in real capture? | Count of notes with `corrects_note_id IS NOT NULL`, bucketed by correction_type |
| How do cloud vs local (via Mnemo ai-gateway) extraction models compare on structured-output quality? | Same notes through both paths; compare extraction schemas on identical input |
| What's the write-path throughput? Query latency budget? | Basic timing metrics on `/v1/notes` POST and `/v1/query` POST |
| Does the outbox pattern work as a clean integration surface? | Phase 2 graph-sync implementation stress-tests it; document what broke or was unclear |

## Drop-in Path to Mnemo

When memory-kit's MVP has dogfooded long enough to trust the patterns, adoption into Mnemo looks like this:

1. **Add the memory service's container(s) to Mnemo's `docker-compose.yml`.** Own Postgres container or shared with Mnemo's Postgres via schema namespacing (decision TBD during adoption — not blocking).
2. **Mnemo's `ai-gateway` (or its chat layer) calls memory-kit's retrieval tools alongside its existing LLM tools.** Memory-augmented chat becomes real.
3. **Versioned HTTP API (`/v1/*`) is the contract.** Mnemo depends on stability; breaking changes require version bumps.
4. **Mnemo's chat UI stays separate.** memory-kit's `web-ui` is dogfood-only and doesn't travel.
5. **Optionally: Mnemo adopts `packages/memory-schema`** for type safety on the client side, if extracted as a shared package.

## Phase 2 — Graphiti Evaluation (This Repo)

Once MVP is stable and dogfooded, Phase 2 adds `apps/graph-sync` reading the outbox and building a Graphiti knowledge graph. This produces:

- A worked integration (how does Graphiti actually consume outbox events?)
- Performance data (query latency, ingestion throughput)
- Operational cost (deployment complexity, memory footprint, reliability)
- **A decision doc** on when and how Mnemo should adopt Graphiti, backed by evidence rather than speculation

**This is the "app for Graphiti evaluation" that was otherwise going to be a separate project.** Keeping it in this repo means the full path (memory → memory + graph) is one coherent artifact.

## Design Principles Proven Here, Carrying to Mnemo

These should become Mnemo's baseline principles for anything memory- or persistence-related:

1. **Additive, never destructive.** Every mutation preserves history. Reversibility is a first-class property of a memory system, not an afterthought.
2. **Raw source always retained.** Raw transcription + embedding is the source of truth. Derived views are rebuildable. No information is ever irrecoverable.
3. **Outbox from day 1.** Event log enables downstream consumers (Graphiti, analytics, backups, replication) without retrofitting. Zero cost today; unlocks everything later.
4. **Provider-agnostic LLM access via OpenAI-compatible gateway.** Cloud/local model swap is a config change, not a code change. Mnemo's `ai-gateway` is doing this work already — memory-kit validates it's the right interface.
5. **Structured-where-valuable + vector-where-fuzzy + LLM-at-query.** Three modalities cooperate. Don't force one to do all three jobs.
6. **Instrumentation is the R&D artifact.** Every resolution path (time, extraction, corrections, retrieval) logs which tier or tool won. The logs are the deliverable, not just the code.
7. **Voice-first for both capture and query.** Both surfaces are conversational. This carries directly to Mnemo's existing chat UI philosophy.
8. **Versioned HTTP API from the first commit.** Contract stability for downstream consumers. No "we'll version it later."

## Non-Goals — What memory-kit Will NOT Answer

To stay honest about the R&D scope:

- **How Mnemo's UI surfaces memory in conversation.** That's a Mnemo design problem. memory-kit provides the API; the UX belongs to Mnemo.
- **Multi-user / team memory.** Single-user only.
- **Long-term (multi-year) corpus behavior.** Dogfood window is weeks-to-months; true longitudinal behavior is a Mnemo-era problem.
- **Performance at scale.** Single-user dogfood is not a load test.
- **Security / auth architecture.** Designed to accept auth but not implementing it.
- **Mobile-specific capture UX.** Deferred to Phase 2 of memory-kit itself.
- **Entity resolution correctness.** Intentionally avoided; Graphiti's domain.

## Success Criteria for This Project

memory-kit is a successful R&D project when:

1. It runs reliably enough that I actually use it for real captures for at least 6-8 weeks.
2. The retrieval agent answers realistic queries from my brief's examples with correctness I trust.
3. The captured dogfood data has enough breadth to answer at least 5 of the 8 "Specific Questions memory-kit Answers" above with measured evidence.
4. `apps/notes-api` is container-portable with a documented HTTP API contract, ready to add to Mnemo's compose.
5. Phase 2 Graphiti evaluation has enough scaffolding to start (even if not completed) — outbox works, consumer pattern is proven viable.

## Post-Project Handoff

When this project graduates:

- The HTTP API contract, schema, and extraction prompts become inputs to a Mnemo integration spec.
- The instrumented findings feed a "memory layer learnings" document that informs Mnemo's `database-architecture.md` Phase 1-3 (or replaces parts of it with evidence-backed revisions).
- The Phase 2 Graphiti findings (if pursued) feed a separate "Graphiti adoption decision" document for Mnemo.
