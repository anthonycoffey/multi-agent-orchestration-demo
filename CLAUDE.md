# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

**yardmaster** is a control plane for running a fleet of coding agents against a GitHub
repository. GitHub issues are the work queue; yardmaster decides which issues are safe and
ready to automate, in what order, under what budget, and with what blast radius — then
leases the actual code-editing work to a vendored agent it treats as a black box.

The coding agent is deliberately *not* the product. Read
[ADR-001](docs/specs/adrs/ADR-001-control-plane-not-coding-agent.md) before proposing any
work that blurs that line.

## Documentation

Read before starting any non-trivial task:

- **Agent brief** (start here): `docs/documentation/agents/yardmaster.md`
- **Development standards**: `docs/documentation/development-standards.md` — spec-first
  workflow, the five-layer test strategy, code style
- **Roadmap**: `docs/specs/plans/ROADMAP.md` — what is built, what is next
- **ADRs**: `docs/specs/adrs/` — permanent decision records
- **Docs structure**: `docs/README.md`

## Commands

```bash
uv sync                                          # Create/refresh the environment
uv run pytest -m "not cassette and not eval"     # Layers 0-2 — the PR gate
uv run pytest -m unit                            # Layer 0 only (fastest)
uv run pytest                                    # Everything
uv run pytest --cov                              # Coverage
uv run ruff check . && uv run ruff format .      # Lint + format
uv run mypy src/                                 # Static types
uv run yardmaster simulate --issues 200          # Deterministic fleet-scale simulation
```

## Architecture

**Stack:** Python 3.12 · LangGraph · Pydantic v2 · Anthropic SDK · Postgres (opt-in) ·
OpenTelemetry · uv · pytest · ruff · mypy

The pipeline, front to back:

```
GitHub issue (UNTRUSTED)
  → admission gate      label + author allowlist; nothing enters without it
  → triage subgraph     QUARANTINED: raw text in, typed WorkOrder out
  → planner             issue dependency DAG + predicted file-overlap conflicts
  → scheduler           admit <= N concurrent, respecting conflicts and fleet budget
  → worker lease        ephemeral sandbox, scoped credentials, vendored agent
  → PR -> CI -> review  pass: merge train. fail twice: escalate with a writeup.
  → ledger              tokens, cost, wall time, attributed per issue
```

### The load-bearing idea

Every stage except triage, review, and the worker itself is **pure deterministic logic**.
That is what makes this project testable, and it is the reason the module layout separates
policy from I/O so aggressively. When adding a feature, the default question is: *can this
be a pure function that takes state and returns a decision?* If yes, it must be.

### Key directories

| Path                        | Purpose                                                        |
| --------------------------- | -------------------------------------------------------------- |
| `src/yardmaster/models/`    | Pydantic contracts — `WorkOrder` is the central one             |
| `src/yardmaster/intake/`    | Admission gate, readiness rubric, quarantined triage           |
| `src/yardmaster/planner/`   | Repo graph, conflict prediction, dependency DAG                 |
| `src/yardmaster/scheduler/` | Concurrency, conflict and budget admission; worker leases       |
| `src/yardmaster/workers/`   | `WorkerDriver` implementations — the vendored agent boundary    |
| `src/yardmaster/integration/`| Merge train, review node, escalation policy                   |
| `src/yardmaster/ledger/`    | Budget ceilings, circuit breaker, cost attribution             |
| `src/yardmaster/providers/` | `ModelProvider` implementations incl. the scripted test double  |
| `src/yardmaster/gateways/`  | GitHub API adapter and its fake                                |
| `src/yardmaster/store/`     | Checkpointer/store adapters (memory, Postgres)                  |
| `src/yardmaster/obs/`       | Tracing, structured logging with redaction, event log           |
| `tests/`                    | Mirrors `src/`; one directory per test layer                    |

### Rules that are not negotiable

1. **Raw issue text never leaves the triage quarantine.** Downstream nodes receive a typed
   `WorkOrder` and nothing else. If you find yourself passing an issue body deeper into the
   graph, you have introduced the vulnerability this project exists to demonstrate handling.
2. **No node pushes to `main`.** Branch and PR only, always.
3. **Time, randomness, and network access are injected.** Logic code never reads the clock,
   generates a UUID, or opens a socket directly.
4. **Every side-effecting operation takes an idempotency key** derived from run and step, so
   replay is safe.
5. **Nothing is coded without a `ready` spec.** See the development standards.

## Development workflow

Spec-first, then TDD:

1. **Plan** — understand the problem, identify unknowns
2. **Spec** — `/new-spec` or `/new-bug` into `docs/specs/active/` (status: `draft`)
3. **Approve** — spec reaches `ready` before any code
4. **Test (RED)** — failing tests first, at the right layer
5. **Implement (GREEN)** — minimum code to pass
6. **Refactor** — clean up, tests stay green
7. **PR → Ship** — merge, spec → `complete`, move to archive

Full detail: `docs/documentation/development-standards.md`

## Version control

- **Strategy:** trunk-based — short-lived branches off `main`, deleted after merge
- **Branch naming:** `feature/`, `fix/`, `chore/`, `refactor/`
- **Commit format:** `<type>: <description> [SPEC-XXX]`
- **NEVER commit directly to `main`**

## Custom slash commands

- `/new-spec` → feature spec in `docs/specs/active/`
- `/new-bug` → bug report in `docs/specs/active/`
- `/new-adr` → ADR in `docs/specs/adrs/`
- `/new-agent-brief` → agent brief in `docs/documentation/agents/`
