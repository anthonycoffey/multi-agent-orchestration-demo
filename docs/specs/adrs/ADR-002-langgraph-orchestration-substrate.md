---
id: ADR-002
title: 'LangGraph as the orchestration substrate'
status: proposed
date: 2026-11-04
deciders: [Anthony Coffey]
supersedes: ''
superseded_by: ''
---

# ADR-002: LangGraph as the orchestration substrate

## Status

`proposed`

## Context

[ADR-001](ADR-001-control-plane-not-coding-agent.md) places the orchestration layer at the
centre of this project. That layer needs:

1. **An explicit, inspectable topology.** We want to assert on the shape of the pipeline in
   a test, not infer it from control flow scattered across a loop.
2. **Durable execution.** A fleet run that dies halfway must resume without redoing paid
   work or duplicating side effects. "Crash and resume to an identical terminal state" is a
   headline claim of this project and must be a real test, not a paragraph.
3. **First-class human-in-the-loop.** Work pauses before any irreversible action, surfaces a
   typed diff of what it intends to do, and resumes on approval.
4. **Deterministic testability.** The orchestration must be runnable end to end with no
   network access and no model, so that failure injection is cheap.
5. **Bounded parallelism with safe state merging**, for concurrent specialist and worker
   branches.

A secondary, non-technical force: this repository is partly a credential. Choosing a
substrate that technical readers recognize reduces the explanation burden, provided the
choice is also defensible on the merits.

## Decision

**Use LangGraph as the orchestration substrate**, with the Anthropic SDK behind a
`ModelProvider` protocol rather than called directly.

Specifically:

- The pipeline is a compiled `StateGraph`. Topology is asserted in a layer 0 test.
- Graph state is a Pydantic model with explicit, associative reducers per field.
- Durability uses LangGraph's checkpointer interface, behind our own `Store` protocol with
  two implementations: in-memory (default, zero setup) and Postgres (opt-in profile).
- Human-in-the-loop uses LangGraph's `interrupt()` before every irreversible stage.
- Model calls go through `ModelProvider`. The real implementation wraps the Anthropic SDK;
  the scripted implementation returns predetermined turns and powers all of layer 1.
- Durable orchestration at the *workflow* level stops here for v1. Temporal is not adopted.

The model provider indirection matters as much as the graph choice: it is what keeps layer 1
hermetic, and it is the seam that would let a different model vendor or a local model drop in
without touching orchestration code.

## Consequences

### Positive

- Checkpointing and `interrupt()` arrive as substrate features rather than hand-rolled
  machinery we would then have to defend in review.
- The explicit graph makes topology, and therefore the pipeline documentation, verifiable.
- Streaming and per-node tracing hooks are available without extra plumbing.
- Recognizable to readers assessing Python multi-agent experience.

### Negative / Trade-offs

- A framework dependency on a fast-moving library. Breaking changes between minor versions
  are a live risk, mitigated by pinning and by keeping framework types out of our domain
  models.
- LangGraph's checkpointer is per-run. Fleet-level state — the dependency DAG, leases, the
  budget ledger — lives in our own store, so there are two persistence concerns to reason
  about rather than one.
- Some LangGraph idioms encourage putting logic inside nodes. Our code style pushes the
  opposite way: nodes stay thin and call pure policy functions. This has to be enforced in
  review, since the framework will not enforce it.
- An ecosystem pull toward LangChain abstractions we do not want. We depend on LangGraph,
  not on LangChain's model wrappers.

### Neutral

- Observability is vendor-neutral OpenTelemetry rather than LangSmith. LangSmith remains
  available and is not wired in by default.

## Alternatives Considered

### Option A: Hand-rolled event-sourced kernel

- **Description:** A small deterministic scheduler and event log written from scratch,
  roughly 700 lines, with the Anthropic SDK behind an adapter.
- **Pros:** Demonstrates the mechanics rather than API familiarity; zero framework risk;
  completely self-contained and reviewable in one sitting.
- **Cons:** Durability, resume semantics and interrupt handling all have to be built and
  defended; reviewers reasonably ask why a maintained option was not used; more surface
  where our own bugs live rather than upstream's.
- **Why rejected:** Strong portfolio argument, weaker engineering argument. The
  hand-rolled-kernel insight is preserved by keeping our domain logic framework-free, so the
  substrate remains replaceable.

### Option B: Claude Agent SDK as the orchestrator

- **Description:** Use the Agent SDK's own subagent and hook mechanisms as the pipeline.
- **Pros:** Closest to Anthropic-native patterns; built-in MCP and tool handling; less code.
- **Cons:** Control flow lives inside the agent loop, so the topology is implicit and hard
  to assert; admission, conflict scheduling and budget governance are not what it is for;
  testing the orchestration deterministically means fighting the harness.
- **Why rejected:** Wrong altitude for a control plane. It remains the default *worker*
  driver, which is the role it suits.

### Option C: Temporal for durability, LangGraph for reasoning

- **Description:** Workflow durability, retries and timers in Temporal; per-issue reasoning
  in LangGraph activities.
- **Pros:** The most production-credible durability story available; battle-tested retry and
  timer semantics; the honest answer for a real multi-tenant deployment.
- **Cons:** Substantial infrastructure to stand up; a reviewer cannot run the project in one
  command; two programming models to learn before reading our actual logic.
- **Why rejected for v1:** Deferred, not rejected. The `Store` protocol and the pure
  scheduler are deliberately shaped so a Temporal workflow could drive them later. A
  follow-up ADR covers that migration if and when fleet size justifies it.

### Option D: A task queue (Celery, RQ, Inngest)

- **Description:** Model each stage as a queued task.
- **Pros:** Operationally familiar; good concurrency primitives.
- **Cons:** No notion of a graph, so topology and state merging are ours to invent; HITL
  pauses become awkward; resume semantics are per-task, not per-run.
- **Why rejected:** Solves distribution, not orchestration. The interesting problems stay
  unaddressed.

## Notes

- Pin LangGraph to an exact version. Treat a minor bump as a `chore` with the full test
  suite as the gate.
- Keep framework types out of `models/`. If a LangGraph type appears in a domain model, the
  substitution seam described here has already been lost.
- Revisit this ADR when either fleet concurrency exceeds what a single process should own, or
  a LangGraph major version forces a migration.
