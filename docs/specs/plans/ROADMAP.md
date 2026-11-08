---
id: PLAN-001
title: 'yardmaster phased delivery plan'
status: draft
created: 2026-11-04
author: Anthony Coffey
---

# yardmaster — Phased Delivery Plan

## Thesis

Multi-agent systems fail in production for operational reasons, not model reasons.

yardmaster is built to make that argument concretely: the control plane around a fleet of
coding agents is deterministic, fully unit-testable, budgeted, traced, and resumable — and
the code-editing agent itself is a replaceable commodity behind an interface.

See [ADR-001](../adrs/ADR-001-control-plane-not-coding-agent.md) for scope,
[ADR-002](../adrs/ADR-002-langgraph-orchestration-substrate.md) for the substrate, and
[ADR-003](../adrs/ADR-003-untrusted-issue-input-trust-boundary.md) for the trust boundary.

## Phases

| Phase | Spec     | Deliverable                                            | Demonstrates                               |
| ----- | -------- | ------------------------------------------------------ | ------------------------------------------ |
| 1     | SPEC-001 | Intake spine: admission, rubric, quarantined triage    | The trust boundary; the test harness       |
| 2     | SPEC-002 | Budget ledger, cost attribution, circuit breaker       | Fleet cost governance                      |
| 3     | SPEC-003 | Planner: dependency DAG, file-conflict prediction      | Why dozens of agents normally collide      |
| 4     | SPEC-004 | Scheduler and worker leases; simulation command        | Fleet-scale decisions, zero model calls    |
| 5     | SPEC-005 | Worker drivers, blast-radius controls, approval gate   | Containment and human-in-the-loop          |
| 6     | SPEC-006 | Integration: PR, CI, review, merge train, escalation   | Knowing when to stop; integration ordering |
| 7     | SPEC-007 | Observability: OTel, redacted logs, replay CLI         | Operability and post-mortems               |
| 8     | SPEC-008 | Postgres store, crash-and-resume                       | Durable execution, proven not claimed      |
| 9     | SPEC-009 | Eval harness and scorecard                             | Quality measurement over time              |
| 10    | —        | README, case study, long-form writeup                  | The argument, told well                    |

**v1 cut line is end of phase 8.** Phases 1–8 are the complete argument: a CLI-driven control
plane that admits work safely, schedules it without collisions, spends within a ceiling,
contains an untrusted worker, integrates results in order, stands down gracefully, and
survives being killed. Phases 9–10 follow.

## Phase detail

### Phase 1 — Intake spine (SPEC-001)

The spine everything else hangs from. Establishes the `WorkOrder` contract, the admission
gate, the readiness rubric, the quarantined triage subgraph, and the layer 0–2 test harness
including the scripted model provider, the fake GitHub gateway, and the injection corpus.

**Done looks like:** `yardmaster run --issue N --dry-run` emits either a validated
`WorkOrder` or a posted clarification request. The injection corpus is green. No network
access in the PR test suite.

### Phase 2 — Ledger (SPEC-002)

Integer micro-dollar cost accounting, per-issue and fleet-daily ceilings, attribution by
issue and label, and a circuit breaker. Pure logic; property-tested invariants.

**Done looks like:** `yardmaster ledger` reports spend by issue and label. A run that exceeds
its ceiling stands down cleanly with partial state persisted.

### Phase 3 — Planner (SPEC-003)

The repository file graph, the issue dependency DAG, and predicted file-overlap conflict
between candidate work orders. This is the phase that answers "why can't I just run twenty
agents at once."

**Done looks like:** given a set of work orders, the planner returns a conflict matrix and a
dependency ordering, with no model involvement and deterministic output.

### Phase 4 — Scheduler and leases (SPEC-004)

Bounded concurrency subject to conflicts, dependencies and budget. Lease lifecycle with
timeouts and cancellation. Ships the simulation command.

**Done looks like:** `yardmaster simulate --issues 200` runs the full admission and
scheduling logic against synthetic issues, deterministically, with no API calls — and the
output shows admitted, deferred, and rejected work with reasons.

### Phase 5 — Worker and containment (SPEC-005)

The `WorkerDriver` boundary with the Agent SDK sandbox driver, the path deny-list, the egress
allowlist, pre-PR diff scanning, and the `interrupt()` approval gate before any write.

**Done looks like:** a real pull request opened against the target repository from a real
issue, with the approval gate showing a typed diff of intended writes beforehand, and a
denied-path attempt failing the lease in a test.

### Phase 6 — Integration (SPEC-006)

Pull request lifecycle, CI wait, the review node, merge train ordering against a moving
`main`, bounded retries, and escalation with a written failure summary.

**Done looks like:** two concurrent work orders integrate in a correct serialized order; a
work order that fails CI twice closes its branch and posts a writeup.

### Phase 7 — Observability (SPEC-007)

OpenTelemetry spans per node and tool call, structured JSON logs with secret and PII
redaction, the append-only event log, and a replay CLI for post-mortems.

**Done looks like:** `yardmaster replay <run-id>` reconstructs a completed run from its event
log, and a trace shows per-node latency and cost.

### Phase 8 — Durability (SPEC-008)

The Postgres store implementation behind the `Store` protocol, adapter parity tests against
the in-memory implementation, and the crash-and-resume test.

**Done looks like:** a run killed mid-flight resumes from its checkpoint to an identical
terminal state, with no duplicated side effects. Both store implementations pass the same
contract suite.

### Phase 9 — Evals (SPEC-009)

Graded scenario suite with a rubric, scored as a scorecard with a quality floor, on a
schedule rather than in the pull request gate.

### Phase 10 — The writeup

README leading with the thesis and the test-layer table, a case study on coffey.codes, and a
long-form piece on testing nondeterministic systems. The dogfooding record — real issues,
real pull requests, real costs — is the evidence.

## Sequencing notes

- Phases 2–4 are pure logic and contain no model calls at all. They are the fastest to build
  and carry the most of the project's argument.
- A minimal event log ships in phase 1 because layer 1 and 2 tests assert on emitted events.
  Phase 7 makes it operable rather than introducing it.
- Phase 5 is the first phase that can cost money or touch a real repository. Everything
  before it runs hermetically.
- Dogfooding begins at phase 5 and compounds: by phase 10 the repository's own pull request
  history is the demonstration.

## Open questions

- Repository name. `yardmaster` is a placeholder chosen for the rail-yard metaphor that the
  merge train already borrows. Renaming before the first public push is a single `sed`.
- Whether phase 5 dogfoods against `coffey.codes` directly or against a scratch repository
  first. Recommendation: a scratch repository for the first live lease, then `coffey.codes`.
- Whether the review node (phase 6) uses a different model tier than triage. Likely yes;
  decide with cost data from phase 2 rather than by guessing.
