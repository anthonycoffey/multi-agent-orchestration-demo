---
service: 'yardmaster'
repo: 'anthonycoffey/yardmaster'
updated: 2026-11-04
---

# Agent Brief: yardmaster

> Context for AI agents working in this repo. Keep it updated as the codebase evolves.

## Purpose

yardmaster is a **control plane for a fleet of coding agents**. It watches a target GitHub
repository, treats issues as a work queue, and decides:

- which issues are ready and safe to automate, and which need a human first
- which issues may run concurrently without colliding in the same files
- how much money and wall time each issue is allowed to consume
- what the agent is permitted to touch, and what happens when it fails
- in what order the resulting pull requests integrate against a moving `main`

It then leases the actual code editing to a vendored coding agent that it treats as an
untrusted, replaceable black box.

The first target repository is `anthonycoffey/coffey.codes`. Nothing in the design is
specific to it.

**What this is not:** a coding agent. The industry has several good ones. See
[ADR-001](../../specs/adrs/ADR-001-control-plane-not-coding-agent.md).

## Tech Stack

| Layer                | Technology                                      |
| -------------------- | ----------------------------------------------- |
| Language             | Python 3.12                                     |
| Orchestration        | LangGraph                                       |
| Data contracts       | Pydantic v2                                     |
| Model access         | Anthropic SDK, behind a `ModelProvider` protocol |
| Coding worker        | Claude Agent SDK (default driver)               |
| Database / Storage   | In-memory (default) · Postgres (opt-in profile) |
| Observability        | OpenTelemetry + structured JSON logs            |
| Hosting / Deployment | CLI first; container image for scheduled runs   |
| Package Manager      | uv                                              |

## Key Dependencies

- **LangGraph** — the orchestration substrate. Chosen for its explicit graph (deterministic
  and assertable), built-in checkpointing (durable resume), and native `interrupt()` for
  human-in-the-loop. See [ADR-002](../../specs/adrs/ADR-002-langgraph-orchestration-substrate.md).
- **Claude Agent SDK** — the default coding worker driver, run in an ephemeral sandbox with
  scoped credentials. Swappable; `claude-code-action` is a supported alternative driver.
- **Pydantic v2** — every boundary is a model. The `WorkOrder` contract is the spine of the
  whole system.
- **GitHub REST/GraphQL API** — the work queue, via an adapter with a full fake for tests.

## Entry Points

- `src/yardmaster/cli.py` — the CLI. Subcommands: `watch`, `run`, `simulate`, `replay`, `ledger`.
- `src/yardmaster/graph.py` — LangGraph assembly. The authoritative picture of the pipeline.
- `src/yardmaster/models/work_order.py` — the central typed contract. Read this first.
- `src/yardmaster/intake/triage.py` — the quarantine boundary. The most security-sensitive
  file in the repo.
- `src/yardmaster/scheduler/scheduler.py` — concurrency, conflict and budget admission.

## API / Interface Contracts

Four protocols define every boundary that gets faked in tests:

| Protocol         | Where                  | Real implementation      | Test double            |
| ---------------- | ---------------------- | ------------------------ | ---------------------- |
| `ModelProvider`  | `providers/base.py`    | `providers/anthropic.py` | `providers/scripted.py`|
| `WorkerDriver`   | `workers/base.py`      | `workers/agent_sdk.py`   | `workers/scripted.py`  |
| `GitHubGateway`  | `gateways/github.py`   | same                     | `gateways/fake_github.py` |
| `Store`          | `store/base.py`        | `store/postgres.py`      | `store/memory.py`      |

`WorkOrder` is the one contract that matters most: it is the *only* thing that crosses out
of triage. It carries a normalized title, acceptance criteria, labels, file hints, a
readiness score, and provenance — never raw issue prose.

## Environment Variables

| Variable                 | Required | Description                                             |
| ------------------------ | -------- | ------------------------------------------------------- |
| `ANTHROPIC_API_KEY`      | yes      | Model access                                            |
| `YM_GITHUB_TOKEN`        | yes      | Scoped PAT for the target repo. Must not grant `workflow`. |
| `YM_TARGET_REPO`         | yes      | `owner/name` of the repository under management          |
| `YM_ADMISSION_LABEL`     | no       | Label that admits an issue. Default `agent-ready`.       |
| `YM_AUTHOR_ALLOWLIST`    | no       | Comma-separated logins permitted to admit work           |
| `YM_MAX_CONCURRENT`      | no       | Concurrent worker leases. Default 3.                     |
| `YM_BUDGET_PER_ISSUE_USD`| no       | Hard per-issue ceiling. Default 2.00.                    |
| `YM_BUDGET_FLEET_DAILY_USD` | no    | Fleet-wide daily ceiling; trips the circuit breaker.     |
| `YM_STORE_DSN`           | no       | Postgres DSN. Absent means the in-memory store.          |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | no  | Trace export. Absent means spans are logged only.        |

## Common Tasks

### Add a new pipeline stage

1. Write the spec first. No exceptions.
2. Decide whether the stage is a **policy** (pure, returns a decision) or an **effect**
   (touches the world). Policies go in the owning module as pure functions; effects go
   behind a protocol in `gateways/` or `workers/`.
3. Add the Pydantic models for its inputs and outputs in `models/`.
4. Write layer 0 tests for the policy. Write layer 1 tests for the stage inside the graph,
   including at least one failure path.
5. Register the node in `graph.py` and assert the new topology in the topology test.
6. Emit events through `obs/events.py` — never `print`, never a bare logger call.

### Run locally

```bash
uv sync
cp .env.example .env     # then fill in ANTHROPIC_API_KEY and YM_GITHUB_TOKEN
uv run pytest -m "not cassette and not eval"
uv run yardmaster simulate --issues 50      # no API calls, fully deterministic
uv run yardmaster run --issue 123 --dry-run # real triage, no writes
```

## Known Gotchas

- **The quarantine is load-bearing, and easy to break by accident.** Any change that widens
  what triage returns is a security change and needs the injection corpus re-run. Adding a
  `raw_body` field to `WorkOrder` would defeat the entire design.
- **`workflow` scope is forbidden on the GitHub token.** An agent that can edit
  `.github/workflows/**` can exfiltrate every secret in the target repo's CI. The path
  deny-list is a second layer, not the first.
- **The target repo auto-deploys `main` on Vercel.** A merge is a production deploy. The
  final merge gate is human, by design, not because the merge train cannot do it.
- **LangGraph state is append-mostly.** Reducers must be associative; a reducer that
  overwrites rather than merges will silently lose a parallel branch's result. Covered by a
  layer 0 reducer test for every state field.
- **Tests must never hit the network.** A fixture fails the run if a real socket is opened
  outside the `cassette` marker.
- **Cost accounting is in integer micro-dollars**, not floats. Float drift in a budget
  ledger produces off-by-a-cent bugs that are miserable to chase.

## Related Docs

- [ADR-001](../../specs/adrs/ADR-001-control-plane-not-coding-agent.md) — control plane, not a coding agent
- [ADR-002](../../specs/adrs/ADR-002-langgraph-orchestration-substrate.md) — LangGraph as substrate
- [ADR-003](../../specs/adrs/ADR-003-untrusted-issue-input-trust-boundary.md) — the trust boundary
- [ROADMAP.md](../../specs/plans/ROADMAP.md) — phased delivery plan
- [development-standards.md](../development-standards.md) — process and the five test layers
