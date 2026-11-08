# yardmaster

A control plane for running a fleet of coding agents against a GitHub repository.

> **Status: pre-implementation.** The architecture, decision records, and the first spec are
> written; no source code exists yet. This repository follows a spec-first workflow, so the
> documents come first by design rather than by accident. Start with
> [docs/specs/plans/ROADMAP.md](docs/specs/plans/ROADMAP.md).

## The argument

Multi-agent systems fail in production for operational reasons, not model reasons.

The code-editing agent is close to a solved, commoditized problem — Copilot's coding agent,
Devin, Codex cloud tasks, and `anthropics/claude-code-action` all do it, and most do it
better than a hand-rolled version would. What is *not* solved is everything above the agent:

- Most issues are not fit for automation, and nothing decides that for you.
- Two agents editing the same file produce a conflict neither can resolve.
- Issue B depends on issue A's interface change, and nothing sequences them.
- A dozen concurrent agents is a surprising amount of money, with no attribution.
- N open agent pull requests against a moving `main` is an integration problem.
- An agent that cannot finish should stop, clean up, and explain itself. Most just retry.
- Issue bodies on a public repository are written by strangers, and they are fed to a process
  holding repository write access and CI credentials.

Those are admission control, scheduling, budget governance, integration, and security
problems. They are also almost entirely *deterministic* problems — which is what makes a
real test-driven workflow possible here.

yardmaster owns that layer and treats the coding agent as a replaceable black box behind an
interface. See [ADR-001](docs/specs/adrs/ADR-001-control-plane-not-coding-agent.md).

## How you test a system that calls a language model

By shrinking the nondeterministic surface until most of the system is ordinary code, then
testing around what remains:

| Layer | What                                                                           | CI        |
| ----- | ------------------------------------------------------------------------------ | --------- |
| 0     | Pure logic: scoring, dependency graph, conflict prediction, scheduling, budgets | Every PR  |
| 1     | Whole-graph runs against a scripted model. All fault injection lives here.      | Every PR  |
| 2     | Schema contracts, event-log golden files, adapter parity, crash-and-resume      | Every PR  |
| 3     | Recorded interactions against the real Anthropic and GitHub APIs                | Nightly   |
| 4     | Graded scenarios scored against a rubric, with a quality floor                  | Scheduled |

Three node types in the whole system reason with a model: triage, review, and the vendored
worker. Everything else is a pure function of state.

Detail: [development-standards.md](docs/documentation/development-standards.md).

## Security posture

GitHub issue content is treated as untrusted input throughout. Containment is layered and
does not depend on model compliance: an admission gate that verifies the *actor* who applied
the label, a triage quarantine whose only output channel is a schema an attacker cannot
widen, a token without `workflow` scope, a path deny-list, an egress allowlist, pre-PR diff
scanning, and a human merge gate.

The residual risk is documented rather than hidden — see the Edge cases section of
[SPEC-001](docs/specs/active/SPEC-001-work-intake-and-trust-boundary.md).

Full threat model: [ADR-003](docs/specs/adrs/ADR-003-untrusted-issue-input-trust-boundary.md).

## Documentation

| Document                                                                                   | What it covers                           |
| ------------------------------------------------------------------------------------------ | ---------------------------------------- |
| [ROADMAP.md](docs/specs/plans/ROADMAP.md)                                                  | Ten phases, what each one proves          |
| [SPEC-001](docs/specs/active/SPEC-001-work-intake-and-trust-boundary.md)                   | Work intake and the trust boundary        |
| [ADR-001](docs/specs/adrs/ADR-001-control-plane-not-coding-agent.md)                       | Why this is not a coding agent            |
| [ADR-002](docs/specs/adrs/ADR-002-langgraph-orchestration-substrate.md)                    | LangGraph, and what was rejected          |
| [ADR-003](docs/specs/adrs/ADR-003-untrusted-issue-input-trust-boundary.md)                 | The trust boundary and threat model       |
| [development-standards.md](docs/documentation/development-standards.md)                    | Spec-first workflow and the test layers   |
| [agents/yardmaster.md](docs/documentation/agents/yardmaster.md)                            | Repo context for AI agents                |

## Stack

Python 3.12 · LangGraph · Pydantic v2 · Anthropic SDK · Postgres (opt-in) · OpenTelemetry ·
uv · pytest · ruff · mypy

## License

MIT
