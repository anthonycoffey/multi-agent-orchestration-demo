# docs/

Single source of truth for all documentation, specifications, and architectural decisions for **yardmaster**.

## Folder Rules

| Folder                      | Purpose                                                     | Never put here           |
| --------------------------- | ----------------------------------------------------------- | ------------------------ |
| `templates/`                | Canonical document templates. Do not modify without review. | Project-specific content |
| `specs/plans/`              | Multi-phase project plans and roadmaps                      | Completed work           |
| `specs/active/`             | Current feature and bug specs (draft → complete)            | Archived work            |
| `specs/adrs/`               | Architecture Decision Records — permanent, never archived   | Anything other than ADRs |
| `specs/archive/`            | Completed or deprecated specs                               | Active work              |
| `documentation/agents/`     | AI agent briefs — repo context for Claude and other tools   | Specs                    |
| `documentation/guides/`     | Procedural how-to documentation                             | Deep technical analysis  |

## Spec Lifecycle

```
draft → ready → in-progress → review-pending → complete
                                    ↓
                               deprecated (from any status)
```

- **draft** — being written, not ready for implementation
- **ready** — spec is complete and approved for development
- **in-progress** — development is underway
- **review-pending** — implementation is complete, awaiting review
- **complete** — shipped and verified
- **deprecated** — no longer relevant or superseded

ADRs are permanent records. They are never archived or deleted.

## Workflow

```
1. Plan      → Understand the problem space, identify unknowns
2. Spec      → Write a spec in docs/specs/active/ (status: draft)
3. Review    → Get spec to status: ready before writing code
4. Test      → Write failing tests first (RED)
5. Implement → Make tests pass (GREEN)
6. Refactor  → Clean up while keeping tests green
7. Archive   → Move completed spec to docs/specs/archive/
```

Full detail, including the five-layer test strategy this project uses to test a
nondeterministic system: [development-standards.md](documentation/development-standards.md)

## Slash Commands

| Command            | Creates                      | Destination                  |
| ------------------ | ---------------------------- | ---------------------------- |
| `/new-spec`        | Feature spec                 | `docs/specs/active/`         |
| `/new-bug`         | Bug report                   | `docs/specs/active/`         |
| `/new-adr`         | Architecture Decision Record | `docs/specs/adrs/`           |
| `/new-agent-brief` | AI agent brief               | `docs/documentation/agents/` |

## Index

### Plans

- [ROADMAP.md](specs/plans/ROADMAP.md) — phased delivery plan, SPEC-001 onward

### ADRs

- [ADR-001](specs/adrs/ADR-001-control-plane-not-coding-agent.md) — The coding agent is a commodity; the control plane is the product
- [ADR-002](specs/adrs/ADR-002-langgraph-orchestration-substrate.md) — LangGraph as the orchestration substrate
- [ADR-003](specs/adrs/ADR-003-untrusted-issue-input-trust-boundary.md) — Treating GitHub issue bodies as untrusted input

### Agent Briefs

- [yardmaster.md](documentation/agents/yardmaster.md) — agent brief for this repo
