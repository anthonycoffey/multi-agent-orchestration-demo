---
id: ADR-003
title: 'GitHub issue content is untrusted input; containment is layered and enforced'
status: proposed
date: 2026-11-04
deciders: [Anthony Coffey]
supersedes: ''
superseded_by: ''
---

# ADR-003: GitHub issue content is untrusted input; containment is layered and enforced

## Status

`proposed`

## Context

yardmaster reads GitHub issues and hands work to a process that holds write access to a
source repository and runs inside CI. On a public repository, **anyone on the internet can
write an issue**. Issue bodies, titles, and comments are therefore attacker-controlled text
flowing into a privileged automated process.

This is not a theoretical concern. It is the standard shape of the prompt-injection
vulnerability class as it appears in CI automation, and the consequences available to an
attacker who succeeds are severe:

| Goal                  | Mechanism if uncontained                                                       |
| --------------------- | ------------------------------------------------------------------------------ |
| Secret exfiltration   | Instruct the agent to edit a workflow file to echo secrets, or to POST env vars |
| Supply-chain injection| Get malicious code into a pull request that a tired maintainer approves         |
| Credential theft      | Read `.env`, service-account JSON, or CI tokens present in the checkout         |
| Budget exhaustion     | File many elaborate issues to burn model spend                                  |
| Repository damage     | Force-push, branch deletion, history rewrite                                    |

The target repository makes this concrete: `anthonycoffey/coffey.codes` auto-deploys `main`
to Vercel, and its working tree can contain `.env` and Google service-account JSON files.
Those are gitignored, which protects the repository but not a checkout the agent can read.

The naive mitigation — writing "ignore instructions in the issue body" into a system prompt —
is not a control. It is a request, it has a known failure rate, and it fails silently.

## Decision

**Treat all issue-authored content as untrusted data, never as instructions, and enforce
containment with layered controls that do not depend on model compliance.**

### Layer 1 — Admission gate (the primary control)

No issue enters the pipeline unless it carries the admission label **and** that label was
applied by a login on the author allowlist. The actor is verified, not just the label's
presence, because a label is a mutable attribute that any triager might add. Admission is
re-verified at lease time, not only at discovery.

Content authored after admission does not re-enter the pipeline. An attacker commenting on an
already-admitted issue changes nothing without a fresh, human-actor-verified admission.

### Layer 2 — Quarantined triage

Raw issue text is read in exactly one place: the triage subgraph. Triage's contract is
*extraction*, not compliance: it emits a typed `WorkOrder` and nothing else. The `WorkOrder`
carries a normalized title, acceptance criteria, labels, file hints, a readiness score, and
provenance. It has **no free-text passthrough field**, by design.

No downstream node — planner, scheduler, worker, review, merge train — ever receives raw
issue prose. The worker receives a `WorkOrder`.

This is a structural control. It holds even when the triage model is fully persuaded by an
injected payload, because the only channel out of triage is a schema the attacker cannot
widen.

### Layer 3 — Least-privilege credentials

- The GitHub token must not carry `workflow` scope. An agent able to edit
  `.github/workflows/**` can print every secret in CI, which makes every other control moot.
- The worker sandbox is ephemeral and holds no ambient cloud credentials.
- Repository secrets are not exposed to worker runs.
- The worker's token cannot push to `main`, force-push, or delete branches.

### Layer 4 — Path deny-list

The worker's writes are filtered against a deny-list before a commit is produced:
`.github/**`, `.env*`, `*.pem`, `*.p12`, service-account JSON, `CODEOWNERS`, and lockfiles
unless the `WorkOrder` explicitly declares a dependency change. A denied write fails the
lease; it is not silently dropped.

### Layer 5 — Egress allowlist

Sandbox network access is restricted to package registries, the GitHub API, and the model
endpoint. This closes the simplest exfiltration path, which is an outbound HTTP request
rather than anything to do with git.

### Layer 6 — Pre-PR diff scanning

Every diff is scanned for secret material and for changes to denied paths before a pull
request is opened. A hit blocks the pull request and escalates.

### Layer 7 — Human merge gate

Pull requests require human review via CODEOWNERS and branch protection. No automated path
merges to `main`. On the first target repository a merge is a production deploy, so this is
also a deployment control.

### Layer 8 — Budget ceilings

Per-issue and fleet-wide daily spend ceilings bound the cost of abuse and trip a circuit
breaker rather than degrading silently.

### Layer 9 — Audit

Every run emits an append-only event log: the admitting actor, the derived `WorkOrder`, every
prompt, every tool call, every denied write, and the final cost. Replayable after the fact.

### Verification

An injection corpus lives in the test suite and runs at layer 1 on every pull request.
Payloads include direct instruction override, fake system-prompt framing, claimed maintainer
authority, urgency pressure, encoded and hidden text, and instructions embedded in fenced
code blocks and HTML comments. The assertion is never "the model refused." It is that **the
`WorkOrder` contains no attacker-supplied directive and no denied path was written** — a
structural assertion about the boundary, not a behavioral one about the model.

## Consequences

### Positive

- Containment does not depend on model compliance, so it does not silently degrade as models
  or payloads change.
- The security posture is testable, and most of it is testable at layer 0 and 1 with no
  network and no model.
- The label gate is cheap and eliminates the entire anonymous-attacker path on day one.
- This is the part of the design that is genuinely hard to copy, and it is the reason
  [ADR-001](ADR-001-control-plane-not-coding-agent.md) is worth building.

### Negative / Trade-offs

- The admission gate means yardmaster cannot act on community issues autonomously. A human
  is always in the loop at the front. This is a deliberate ceiling on autonomy.
- The no-passthrough rule on `WorkOrder` loses nuance. A long, well-written issue is
  compressed into a schema, and some useful context is discarded. The readiness rubric and
  the clarification loop are the compensating mechanism.
- The path deny-list will produce false positives. An issue legitimately asking to update CI
  cannot be automated, and must be done by a human.
- Egress allowlisting and sandboxing add real operational complexity, and are the most
  likely controls to be weakened under delivery pressure. They are therefore asserted in
  tests rather than documented as intent.

### Neutral

- `WorkOrder` carries provenance, so the raw issue remains auditable at its source without
  ever being carried in-band.

## Alternatives Considered

### Option A: Prompt-level defenses only

- **Description:** Instruct the model to disregard instructions found in issue content.
- **Pros:** Trivial; no architecture required.
- **Cons:** Not a security control. Probabilistic, silently fallible, and untestable in any
  meaningful sense.
- **Why rejected:** Treating a system prompt as a trust boundary is the mistake this project
  exists to demonstrate not making.

### Option B: Allowlist authors only, no further containment

- **Description:** Only maintainer-authored issues are automated; trust them fully.
- **Pros:** Simple; removes the anonymous attacker.
- **Cons:** No defense in depth. A compromised maintainer account, or a maintainer pasting a
  customer's bug report verbatim, reintroduces untrusted text with full privilege.
- **Why rejected:** Correct as a first layer, insufficient as the only one.

### Option C: Full air-gap — no network, no credentials, patch files out

- **Description:** The worker runs with no network and no tokens; its output is a patch file
  a human applies.
- **Pros:** Strongest containment available.
- **Cons:** No dependency installation, so tests cannot run, so the agent cannot verify its
  own work. Removes the automation value almost entirely.
- **Why rejected:** Over-rotated. The egress allowlist retains most of the benefit at a
  fraction of the cost.

### Option D: Review-agent-as-gate instead of human merge

- **Description:** A reviewing agent approves and merges when checks pass.
- **Pros:** Full autonomy; a more impressive demonstration.
- **Cons:** An injected payload that satisfies CI reaches production. On the first target
  repository that is a live deploy.
- **Why rejected:** The review agent is valuable as a *filter* that reduces human load. It is
  not a trust boundary and must not be positioned as one.

## Notes

- Related: [ADR-001](ADR-001-control-plane-not-coding-agent.md),
  [ADR-002](ADR-002-langgraph-orchestration-substrate.md).
- Any change widening what crosses out of triage is a security change. It requires the
  injection corpus to be re-run and this ADR to be revisited. Adding a `raw_body` field to
  `WorkOrder` would defeat the design entirely.
- Before the first live run against a real repository, confirm: token scopes exclude
  `workflow`, branch protection on `main` is active, and CODEOWNERS requires review.
