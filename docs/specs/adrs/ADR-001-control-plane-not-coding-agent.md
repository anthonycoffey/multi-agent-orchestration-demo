---
id: ADR-001
title: 'The coding agent is a commodity; the control plane is the product'
status: proposed
date: 2026-11-04
deciders: [Anthony Coffey]
supersedes: ''
superseded_by: ''
---

# ADR-001: The coding agent is a commodity; the control plane is the product

## Status

`proposed`

## Context

The original framing of this project was "AI agents that read GitHub issues and resolve
them." That capability is the most heavily contested product category in the industry right
now. As of late 2026 it is served by GitHub Copilot's coding agent, Devin, Cursor background
agents, OpenAI Codex cloud tasks, Google Jules, OpenHands, SWE-agent, and others. Anthropic
itself ships `anthropics/claude-code-action`, a maintained GitHub Action that already
performs label-triggered issue-to-pull-request work using the same model family this project
would use.

A hand-rolled reimplementation of that capability would be strictly worse than the
maintained options, and would invite exactly one reaction from a technical reader: *this is a
worse version of a GitHub Action I can install in five minutes.* Everything else the
repository tries to demonstrate is discounted from that point on.

However, the forces that make the category crowded also create a gap. Single-agent,
single-issue execution is close to solved. What is **not** solved, and what every team
adopting these tools hits within a week, is the operational layer above the agent:

- Most issues are not fit for automation, and nothing decides that for you.
- Two agents editing the same file produce a merge conflict neither of them can resolve.
- Issue B depends on issue A's interface change, and nothing sequences them.
- Ten concurrent agents is a real and surprising amount of money, with no attribution.
- N open agent-authored pull requests against a moving `main` is an integration problem,
  not a code-generation problem.
- An agent that cannot finish needs to stop, clean up, and explain itself. Most just retry.
- Issue bodies on a public repository are written by strangers, and they are fed to a
  process holding repository write access and CI credentials.

Those are scheduling, admission control, budget governance, integration, and security
problems. They are also, almost entirely, *deterministic* problems — which makes them
testable in a way that the code-editing step is not.

## Decision

**yardmaster owns the control plane and explicitly does not own code editing.**

In scope:

- Admission control — readiness scoring, label and author gating, clarification requests
- Planning — issue dependency DAG, predicted file-level conflict between candidate work
- Scheduling — bounded concurrency subject to conflicts, dependencies, and budget
- Budget governance — per-issue and fleet ceilings, circuit breaking, cost attribution
- Worker lifecycle — leases, sandboxing, credential scoping, timeouts, cancellation
- Integration — merge train ordering, re-verification, conflict handling
- Escalation — bounded retries, standing down, and writing up the failure for a human
- Observability — event log, traces, replay, ledger

Out of scope, permanently:

- Editing source code
- Reading or reasoning about the target repository's code semantics beyond what conflict
  prediction requires
- Being a better coding agent than the vendored one

The code-editing step sits behind a `WorkerDriver` protocol. The default implementation
drives the Claude Agent SDK inside an ephemeral sandbox, because that gives yardmaster
direct control over the credential and filesystem boundary. A second implementation drives
`claude-code-action` for teams already standardized on it. Both are replaceable, and the
test suite exercises the pipeline against a scripted driver that writes no code at all.

## Consequences

### Positive

- The project competes in a category that is under-built rather than over-built.
- The hard engineering lands in deterministic logic, so the majority of the system is
  covered by fast layer 0 tests. This is what makes a spec-first TDD workflow honest here
  rather than aspirational.
- The vendored agent can be swapped when something better ships, which it will. The value
  built here does not depreciate with the model.
- The security story becomes central rather than incidental: owning the boundary around an
  untrusted worker *is* the product.

### Negative / Trade-offs

- The repository does not contain an impressive "watch it write code" demo. The impressive
  part is a scheduler and a ledger, which require a reader who gets it.
- yardmaster's observable output quality is bounded by the vendored agent's. A bad worker
  makes the control plane look bad even when it behaves correctly.
- Treating the worker as a black box forfeits optimizations available to a tighter
  integration, such as steering the agent mid-task based on partial diffs.

### Neutral

- The `WorkerDriver` boundary means local development and the whole PR test suite run with
  no coding agent present at all.
- "Control plane" framing invites comparison to Kubernetes-style vocabulary. Useful as
  shorthand, and a liability if it becomes a promise about scale we have not demonstrated.

## Alternatives Considered

### Option A: Build our own coding agent

- **Description:** Implement the edit-test-fix loop directly, no vendored worker.
- **Pros:** Full control; a single self-contained artifact; a more obviously impressive demo.
- **Cons:** Competes directly with multiple well-funded products; the resulting agent would
  be worse; the work depreciates every time a better model or harness ships.
- **Why rejected:** Highest effort, lowest differentiation, shortest shelf life.

### Option B: Use `claude-code-action` as the entire system

- **Description:** Configure the existing Action with labels and trust it.
- **Pros:** Nearly zero effort; genuinely the correct choice for a small team today.
- **Cons:** Demonstrates no engineering. Provides no admission control, no conflict-aware
  scheduling, no fleet budget, no merge ordering, and no containment of untrusted issue text.
- **Why rejected:** It is the right answer to a different question. It is also the baseline
  this project must be measured against, which is why it ships as a supported worker driver.

### Option C: Fork an open-source agent platform (OpenHands, SWE-agent)

- **Description:** Adopt an existing platform and extend it with the control-plane features.
- **Pros:** Inherits a working execution loop and an existing community.
- **Cons:** Inherits a large unfamiliar codebase and its architectural assumptions; the
  control-plane concerns would be bolted onto someone else's state model; attribution of
  what we actually built becomes muddy in a portfolio context.
- **Why rejected:** The valuable work would be buried inside a fork diff.

### Option D: Control plane plus a thin steering layer into the worker

- **Description:** As decided, but with the ability to interrupt and redirect the worker
  mid-task.
- **Pros:** Better results on partially-wrong attempts; a richer demonstration.
- **Cons:** Couples us to a specific worker's internals, undermining the replaceability that
  ADR-001 is built on.
- **Why rejected for now:** Deferred rather than rejected. Revisit once the `WorkerDriver`
  boundary has survived two real implementations.

## Notes

- Related: [ADR-002](ADR-002-langgraph-orchestration-substrate.md) (substrate choice),
  [ADR-003](ADR-003-untrusted-issue-input-trust-boundary.md) (the trust boundary this
  decision makes central).
- The README must state the `claude-code-action` comparison explicitly and early. A reader
  who wonders "why not just use the Action?" and does not find an answer will assume there
  isn't one.
