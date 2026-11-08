# Development Standards

How we work on yardmaster — development process, testing philosophy, spec lifecycle, and
version control conventions.

Inherited from the coffey.codes standards and adapted for Python, and for a system whose
behavior is partly nondeterministic.

---

## Development Process

**Spec-first, plan-driven.** No feature or bug fix begins without a written spec.

```
1. Plan      → Understand the problem. Identify unknowns. Research alternatives.
2. Spec      → Write a spec (feature or bug) in docs/specs/active/.
3. Approve   → Get spec status to `ready` before writing code.
4. Test      → Write failing tests first (RED phase).
5. Implement → Write the minimum code to make tests pass (GREEN phase).
6. Refactor  → Clean up the implementation while keeping all tests green.
7. Review    → PR opened, spec status → `review-pending`.
8. Ship      → Merge. Spec status → `complete`. Move to archive.
```

Specs are the contract between intent and implementation. If you are writing code without
a spec, stop and write one first.

---

## Test-Driven Development

RED → GREEN → REFACTOR, as usual: write a test that describes the desired behavior, watch
it fail, write the minimum code to pass, then clean up with the test green.

The interesting question for this project is *how you do that at all* when part of the
system calls a language model.

### The answer: shrink the nondeterministic surface, then test around it

yardmaster is a control plane. Most of it — admission scoring, the dependency graph, file
conflict prediction, the scheduler, the budget ledger, merge serialization, escalation
policy — is **pure deterministic logic with no model involvement**. That code is tested
like any other code, and it is the majority of the system.

Only three node types reason with a model: triage, review, and the vendored coding worker.
Those sit behind interfaces so the orchestration around them stays testable.

### The five layers

| Layer | Marker     | What it covers                                                                                                                                           | CI        |
| ----- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| 0     | `unit`     | Pure logic: scoring, dependency graph, conflict prediction, scheduling, budget ledger, policies. Graph topology assertions. Property tests on invariants. | Every PR  |
| 1     | `harness`  | Whole-graph runs against a scripted model. **All fault injection lives here.**                                                                           | Every PR  |
| 2     | `contract` | Pydantic schema contracts, event-log golden files, storage-adapter parity, crash-and-resume.                                                              | Every PR  |
| 3     | `cassette` | Recorded HTTP interactions against the real Anthropic and GitHub APIs.                                                                                   | Nightly   |
| 4     | `eval`     | Graded scenarios scored against a rubric; reported as a scorecard with a quality floor.                                                                   | Scheduled |

Layers 0–2 are fast, hermetic, and must stay green on every PR. Layers 3–4 detect drift in
real model and API behavior and are deliberately not PR gates — a model provider changing
behavior overnight is not a reason to block an unrelated pull request.

### Layer 1 is where the real work happens

A scripted model provider returns a predetermined sequence of turns, so a full graph run is
deterministic and millisecond-fast. That makes the following *ordinary unit tests* rather
than things we merely hope about in production:

- A specialist times out → run degrades, gap is flagged, no crash
- The provider returns HTTP 429 repeatedly → backoff, then circuit break
- A tool receives malformed arguments → typed validation error, bounded retry
- An agent loops on the same tool call → loop detector trips
- The per-issue budget is exhausted mid-run → clean stand-down, partial state persisted
- An issue body carries a prompt-injection payload → quarantine holds, worker never sees it
- A run is interrupted and resumed → identical terminal state

Every one of those is a test file, not a paragraph in a README.

### What we do not do

- We do not assert on model prose. Assertions are on structured output, state transitions,
  emitted events, and tool calls.
- We do not put layer 3 or 4 in the PR gate.
- We do not mock our own code to make a test pass. Fakes exist at process boundaries
  (model provider, GitHub API, clock, sandbox) and nowhere else.

---

## Spec Lifecycle

```
draft → ready → in-progress → review-pending → complete
                                    ↓
                               deprecated (from any status)
```

| Status           | Meaning                                             | Who sets it       |
| ---------------- | --------------------------------------------------- | ----------------- |
| `draft`          | Being written; not ready for development            | Author            |
| `ready`          | Approved for development; requirements are complete | Author + reviewer |
| `in-progress`    | Development is underway                             | Developer         |
| `review-pending` | Implementation complete; awaiting code review       | Developer         |
| `complete`       | Shipped, verified, and closed                       | Reviewer          |
| `deprecated`     | No longer relevant; superseded or cancelled         | Author            |

**Rules:**

- Never start coding on a spec with status `draft`
- ADR status transitions: `proposed` → `accepted` | `deprecated` | `superseded`
- ADRs are never deleted or archived — they are permanent records

---

## Version Control Conventions

### Branch Naming

```
feature/short-description      # New feature
fix/short-description          # Bug fix
chore/short-description        # Non-functional changes (deps, tooling, docs)
refactor/short-description     # Code restructuring without behavior change
```

### Commit Messages

Imperative mood. Reference the spec ID when applicable.

```
<type>: <short description> [SPEC-XXX]

<optional body — what and why, not how>
```

**Types:** `feat`, `fix`, `chore`, `refactor`, `docs`, `test`, `style`

### Branching Strategy

Trunk-based. All branches cut from `main`, merged back via pull request, deleted after merge.

**CRITICAL RULE: NEVER commit directly to `main`.** No exceptions. The sole historical
exception is this repository's genesis commit.

This rule binds agents as well as humans, and for agents it is *enforced* rather than
requested — see [ADR-003](../specs/adrs/ADR-003-untrusted-issue-input-trust-boundary.md).

---

## Code Style

- **Formatting and linting:** `ruff` (format + lint). No competing formatter.
- **Typing:** `mypy --strict` on `src/`. Public functions are annotated; `Any` requires a
  comment explaining why.
- **Data shapes:** Pydantic v2 models at every boundary — tool arguments, tool results,
  graph state, persisted records. If data crosses a module boundary it is a model, not a dict.
- **Errors:** a typed error taxonomy, not bare `Exception`. Every error carries whether it
  is retryable.
- **No bare `except`.** No silent `pass` in an exception handler.
- **Time and randomness are injected**, never read from the ambient environment in logic
  code, so tests stay deterministic.
- **Docstrings** on public functions, explaining *why* where the code already says *what*.

---

## Tooling

| Tool         | Command                                        | Purpose                        |
| ------------ | ---------------------------------------------- | ------------------------------ |
| Install      | `uv sync`                                      | Create/refresh the environment |
| Tests (fast) | `uv run pytest -m "not cassette and not eval"` | Layers 0–2 — the PR gate       |
| Tests (all)  | `uv run pytest`                                | Everything                     |
| Coverage     | `uv run pytest --cov`                          | Coverage report                |
| Lint         | `uv run ruff check .`                          | Lint                           |
| Format       | `uv run ruff format .`                         | Format                         |
| Types        | `uv run mypy src/`                             | Static type check              |
| Simulation   | `uv run yardmaster simulate`                   | Deterministic fleet-scale sim  |

The coverage floor is enforced in CI and applies to `src/` excluding adapter shims.
