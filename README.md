# Personal AI Platform — Technical Architecture Overview

A personal AI platform built with a local-first, privacy-conscious
architecture. This document describes the engineering architecture and
implementation patterns behind the project — it intentionally omits product
scope, feature details, and domain-specific design, which are outside the
purpose of this overview.

The system is built as a set of independently testable, provider-agnostic
services rather than a single script wrapping an LLM API. Two full
implementations of the core extraction service currently coexist — one in
TypeScript/Node.js, one in Python/FastAPI — sharing the same schema contract
and behavior, built in parallel to prove the architecture isn't locked to a
single runtime.

## Monorepo structure

```
packages/
├── knowledge-contract/       # Shared, versioned schema definitions (TypeScript + Zod) —
│                              # the typed contract every service validates its I/O against
├── understanding-worker/     # TypeScript service: LLM-based structured data extraction
└── understanding-worker-py/  # Python/FastAPI port of the same service — polyglot parity by design
```

Each package is independently buildable, lintable, and testable. Services
communicate only through the shared schema contract — never through
implicit shared state or direct imports of one another's internals.

## Core pattern: provider-agnostic structured extraction

The central architectural pattern is a **structured-output provider
abstraction**. Every LLM vendor integration implements the same interface:

```
generate(system_prompt, input, output_schema) -> ValidatedInstance
```

No calling code ever touches a vendor SDK, a tool-calling convention, or a
JSON-mode quirk directly — those differences are fully encapsulated inside
each provider implementation. Two providers are implemented today (Anthropic
Claude, Google Gemini), each using that vendor's native structured-output
mechanism rather than free-text parsing or regex extraction on either side.

Consequences of this design:
- Swapping or A/B-testing providers is a one-class change, not a rewrite.
- The system has no hard dependency on any single LLM vendor.
- New providers are additive — they implement the interface, nothing else changes.

### Retry logic with error categorization

A generic retry helper distinguishes *why* a call failed and reacts
differently per category, instead of blindly retrying on any exception:

- **Validation failure** (output failed schema or cross-field domain-rule
  validation) → retried once, with the specific validation issues fed back
  into the next prompt as a correction hint, so the model can self-correct
  on the retry rather than repeating the same mistake blindly.
- **No structured output returned** (provider produced nothing usable) →
  treated as a distinct failure mode, not retried the same way, and surfaced
  as a different error at the API layer.

This keeps failure modes observable and distinct instead of collapsing every
LLM failure into one generic "retry and hope" path.

### Schema-based extraction & validation

- Every extraction is validated at two levels: **shape** (Pydantic in
  Python, Zod in TypeScript) at the individual field level, and **cross-field
  domain rules** (referential integrity between related fields, chronological
  consistency, duplicate-identifier detection) that no single-field schema
  can express on its own.
- Schemas are the single source of truth: static types are inferred from
  them, never hand-duplicated, so runtime validation and compile-time types
  can't drift apart.
- LLM output is never trusted or persisted directly — it is validated
  immediately, before anything downstream ever sees it.

## Tech stack

- **Languages**: Python 3.12, TypeScript 5.9, Node.js
- **API layer**: FastAPI (Python)
- **Validation**: Pydantic v2, Zod v4
- **LLM providers**: Anthropic Claude (native structured outputs), Google Gemini (schema-constrained generation)
- **Persistence (planned)**: PostgreSQL + pgvector
- **Tooling**: uv, pytest, ruff, mypy (Python) · npm workspaces, ESLint, Prettier, native Node test runner (TypeScript)

**Testing philosophy**: fully offline test suites. All LLM provider SDKs are
mocked or faked at the unit level, so correctness is verified deterministically,
without live API calls, flakiness, or cost.

## Technical roadmap

- **Retrieval worker** — a semantic retrieval layer over PostgreSQL +
  pgvector, exposed as a first-class service in the same provider-agnostic
  style as the extraction layer.
- **Multi-agent orchestration** — a LangGraph-based orchestration layer to
  coordinate multiple specialized workers (extraction, retrieval, reasoning)
  as the system grows beyond single-pass extraction.
- **Formalized service boundaries** — evolving the current Python/TypeScript
  split into independently deployable services communicating over the shared
  schema contract, rather than in-process calls.

## Why this architecture

- **LLMs are a replaceable implementation detail, not the architecture.**
  Every AI call sits behind an interface; system correctness never depends
  on a specific model or vendor.
- **Structure over free text.** Model output is validated against explicit
  schemas at both the field and cross-field level before anything downstream
  consumes it — never trusted, never stored raw.
- **Single-responsibility, isolated services.** Each worker answers exactly
  one question and has no knowledge of how its output is used downstream,
  keeping the system composable and independently testable.
- **Call discipline.** LLM calls are used only where they add value a
  deterministic function cannot — not as a default tool for every problem.
- **Built for polyglot longevity.** The same contract and business logic
  exist in two languages today, by design — proof the architecture isn't
  locked to one runtime and can be extended in any stack without
  re-deriving the domain rules from scratch.

## Engineering practices

- Test-first development with atomic, conventional-commit git history (one concern per commit)
- Strict typing end-to-end: `mypy --strict` (Python), TypeScript strict mode
- Zero live network calls in test suites — full offline determinism via provider mocking/fakes
- Cross-language schema parity maintained by design, not by codegen — the
  contract is the interface, not a generated artifact

---

This is a private portfolio project. Product scope and functional details
are intentionally not included in this technical overview.
