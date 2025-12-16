# Team Ownership & Division of Labor (v4.2 Rewrite)

This document defines **code ownership, decision rights, and delivery sequencing** for the v4.2 rewrite.

---

## Goals

- **Prevent domain drift**: the system must encode the business correctly (insurance + logistics reality).
- **Enable parallel execution** without merge chaos.
- Keep the **Brain deterministic** while allowing probabilistic AI at the edges (LLM calls + validation gates).
- Maintain the v4.2 non-negotiables: **`identity.Tenant`**, **ACL-first**, **TenantContext**, **no legacy `core.models.Org`**.

---

## Strategy: "Model-First" (Domain Expert owns the "What")

- **Chief Architect (Domain Expert)** defines the **What**: schemas, invariants, business rules, workflow semantics.
- **Founding Engineers** own the **How**: execution machinery, IO pipelines, integrations, APIs.

This maximizes speed while ensuring correctness in the most expensive-to-fix layer: **data modeling + workflow meaning**.

---

## AI-First Development Model

We don't hand-code. We **plan, review, and direct AI agents**.

### The Stack

| Tool | Role |
|------|------|
| **Cursor** | Primary coding agent — latest agent mode for implementation |
| **Claude** | Plan review, architecture validation, complex reasoning |
| **GitHub Copilot** | Inline suggestions during review (optional) |

### The Workflow: Plan → Review → Implement → Validate

```
┌─────────────────────────────────────────────────────────────────┐
│  1. PLAN (Human + AI Agent)                                                │
│     Engineer writes a plan: what to build, constraints,         │
│     acceptance criteria, affected files                         │
├─────────────────────────────────────────────────────────────────┤
│  2. REVIEW PLAN (Human + Human + AI Agent)                                 │
│     Peer reviews plan for correctness, scope creep,             │
│     domain alignment. Architect reviews if domain-touching.     │
├─────────────────────────────────────────────────────────────────┤
│  3. IMPLEMENT (AI Agent)                                        │
│     Cursor agent executes the approved plan.                    │
│     Engineer monitors, course-corrects, doesn't hand-code.      │
├─────────────────────────────────────────────────────────────────┤
│  4. VALIDATE (Human + Gemini Deep think or GPT5.2 Pro Extended)                                            │
│     Engineer reviews output: tests pass, code matches plan,     │
│     no regressions. PR opened for final review.                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why This Works

- **Speed**: What takes a team weeks, we ship in days. What takes days, we ship in hours.
- **Quality**: Plans are reviewed before a single line is written — catches errors at the cheapest point.
- **Leverage**: Engineers are multiplied 10x by directing agents instead of typing.
- **Consistency**: AI follows patterns perfectly once shown; humans drift.

### Rules for AI-First Development

1. **No hand-coding for new features.** Use Cursor agent mode. Hand-editing only for surgical fixes during review.
2. **Plans are artifacts.** Store them in `.cursor/plans/` or PR descriptions. They're the source of truth.
3. **Context is king.** Feed the agent the right files, specs, and examples. Bad context = bad output.
4. **Review the output, not the process.** Don't watch the agent type. Review the diff.
5. **If the agent struggles 3x, stop.** Rewrite the plan or break the task smaller.

---

## Team Scaling Phases

This ownership map adapts as the team grows.

| Phase | Headcount | Ownership Model | Timeline |
|-------|-----------|-----------------|----------|
| **Bootstrap** | David only | David owns all; doc serves as future contract | Current |
| **Founding Team** | David + 2 Founding Engs | Full founding ownership map | Day 1 ready |
| **Post-Foundation** | Founding Team + N | Vertical engineers spin up new OS apps | After Phase 3 |

### Bootstrap Phase (Current)

David builds the Phase 1 foundation (TenantContext, PermissionService, base executor patterns). This establishes the "trusted patterns" that founding engineers will extend rather than architect from scratch.

### Founding Team Phase

When the two founding engineers join, ownership splits:

| Domain | David (Architect) | Founding Eng A (Platform) | Founding Eng B (Integrator) |
|--------|-------------------|---------------------------|----------------------------|
| Business Kernel (`network/`, `insurance/`, `assets/`) | **Primary** | Contributor | Contributor |
| Platform Kernel (`common/`, `access/`, `core/brain/`) | Reviewer | **Primary** | Contributor |
| IO Layer (`documents/`, `communication/`, APIs) | Reviewer | Contributor | **Primary** |

The founding team's mission: **build the platform that makes spinning up new verticals trivial.**

### Post-Foundation: Vertical Engineers

Once the founding team has delivered the core platform (Phases 1-3 complete), we add **Vertical Engineers** who can independently build new OS apps:

**What Vertical Engineers Own:**
- Their entire `apps/{vertical}_os/` directory (models, views, workflows, templates)
- Vertical-specific Brain tools registered under their namespace (e.g., `fleet.*`)
- Vertical-specific Temporal workflows and Celery tasks

**What Vertical Engineers Do NOT Touch (without founding team review):**
- Core domains (`network/`, `insurance/`, `assets/`)
- Platform kernel (`common/`, `access/`, `core/brain/`)
- Shared integrations (`documents/`, `communication/`)

**The Contract:**
Vertical Engineers build on top of the platform. They consume `Entity`, `Tenant`, `PermissionService`, and `BrainWorkflowPlan` — they don't modify them. If they need a new core capability, they request it from the founding team.

---

## Ownership Map (Founding Team)

### Shared Decision Rights

- **Changes to Golden Rules** (tenancy, ACL, entity resolution, brain determinism) require Architect sign-off after hearing objections. Architect has final call.
- Any change that impacts **tenant isolation** or **PII** requires Architect approval.

### Chief Architect / Domain Expert (David) — Business Kernel Owner

**Primary owner**:
- `network/` (Entity, Identifier normalization, merge/claim semantics)
- `insurance/` (Policy/Term/Coverage/Submission semantics)
- `assets/` (Driver/Vehicle domain rules, PII constraints)
- `apps/*/models/` where business semantics live

**Also owns**:
- Workflow semantics and plan intent mapping (Loss Run, Quote Submission, Renewal, etc.)
- "What must be true" invariants: uniqueness constraints, canonicalization rules, lifecycle states

**Plan approval required**:
- Any plan that changes domain schemas/rules
- Any plan that alters business meaning (e.g., what "Submission" grants, what "Claimed" means)

**Trusted Patterns (No Plan Review Needed)**:
- Adding a nullable field following established naming conventions
- Adding a new `@property` or helper method that doesn't change persistence
- Adding indexes for query performance
- Extending enum choices where the pattern is established

### Founding Engineer A (Platform) — Execution & Security Kernel Owner

**Primary owner**:
- `common/` (BaseModel patterns, ContextVars, scoped managers)
- `access/` (Resource, AccessGrant, PermissionService, relationship edges)
- `core/brain/` (BrainTask/Plan plumbing, executor, check-execute-commit, crash recovery)

**Accountable for**:
- Deterministic executor behavior (idempotency, retries, step insertion semantics)
- Tenant isolation correctness (TenantContext in workers + brain execution)
- Observability (trace_id propagation), PII-safe logs

**Plan approval required**:
- Any plan touching `common/`, `access/`, `core/brain/`

**Trusted Patterns (No Plan Review Needed)**:
- Adding new log statements (if PII-safe)
- Adding new metrics/traces that follow existing conventions
- Bug fixes that don't change executor state machine semantics

### Founding Engineer B (Integrator) — IO, Integrations, API Surface Owner

**Primary owner**:
- `documents/` (S3 storage, classification/extraction pipeline, linking tables)
- `communication/` (email/sms/voice gateways)
- External integrations (Gmail/Resend/RingCentral/Stripe/portals)
- REST APIs + serializers + views

**Accountable for**:
- Reliable ingestion (email/PDF → Document → links → events)
- Integration idempotency and retry safety
- Clean API contracts that respect ACL and tenancy

**Trusted Patterns (No Plan Review Needed)**:
- Adding new API endpoints that follow established serializer patterns
- Adding retry/backoff logic to existing integrations
- Adding new webhook handlers that follow existing patterns

---

## Collaboration Rules (Non-Negotiable)

### Plan Review Rules

**Before AI implements**, plans must be reviewed:

| Plan Type | Reviewer Required | Turnaround |
|-----------|-------------------|------------|
| Domain schema / business logic | Architect | < 2 hours |
| Platform kernel changes | Platform Owner | < 2 hours |
| IO / API changes | Integrator Owner | < 2 hours |
| Trusted pattern changes | None — self-review | Immediate |
| Golden Rule / PII / Tenant isolation | Architect (explicit approval) | No timeout |

**Async Review Protocol:**
- Post plan in Slack or PR draft
- Reviewer has 2 hours during work hours
- No response + tests pass = approved for trusted patterns only
- Golden Rule changes: must get explicit 👍

### PR Review Rules (Post-Implementation)

After AI implements:

| PR Type | Reviewer | Focus |
|---------|----------|-------|
| Any PR | Plan author | Output matches plan intent |
| Domain-touching | Architect | Business correctness |
| Platform-touching | Platform Owner | Isolation, determinism |
| API-touching | Integrator Owner | Contract stability |

### "Contracts Before Code"

Before implementing a cross-domain feature:
- Write a **1-page contract**: inputs, outputs, required IDs, tenant context assumptions.
- Confirm **which model is the source of truth** (Entity vs Tenant vs Policy vs Submission).
- Get plan approved, then unleash the agent.

### "No Hidden Tenancy"

- Background jobs must accept `tenant_id` and set `TenantContext` explicitly.
- Domain queries must use `TenantScopedManager` where applicable; global models must use ACL services.

### "No Legacy Imports"

- Brain + new domain code must not import legacy `core.models.Org`.
- Everything new keys off **`identity.Tenant`**.

---

## Delivery Sequencing (3 Phases)

Compressed timelines. What normally takes teams months, we ship in weeks.

### Phase 1 — Security + Tenancy Foundation

**Owner:** David (bootstrap), then Founding Eng A (Platform) extends.
**Timeline:** 2-3 days with founding team

**Deliverable (Definition of Done)**:
- `TenantContext` exists and is required for scoped queries.
- `PermissionService` + `AccessGrant` enforced patterns exist.
- A "hello world" worker and brain tool can run inside tenant context.
- Trusted patterns documented for Phase 2 contributors.

**Founding Gate**: Phase 1 core complete before founding engineers start. They extend, not architect.

### Phase 2 — Domain Models + Ingestion Primitives

**Owner:** Architect (domain models), Founding Eng B (documents ingestion), Founding Eng A supports.
**Timeline:** 3-5 days

**Deliverable**:
- Canonical `network.Entity` + `EntityIdentifier` normalization implemented.
- Insurance contract models exist with constraints and clear ownership semantics.
- Documents can be stored, classified, and linked to Entities/Policies with ACL anchors.

### Phase 3 — Brain Workflows + Marketplace

**Owner:** Architect defines workflow semantics + plan steps; Founding Eng A implements executor; Founding Eng B wires IO triggers.
**Timeline:** 5-7 days

**Deliverable**:
- Loss Run workflow runs end-to-end with deterministic executor rules (including **step insertion**, not append).
- Marketplace gating exists (tenant subscriptions → allowed plans/capabilities).
- Human-in-the-loop flows supported (`NEEDS_REVIEW` status, clear remediation path).

**Total: ~2 weeks to full platform with founding team.**

---

## Failure Modes & Incident Response

### Ownership of Production Incidents

| Incident Type | Primary Responder | Escalation |
|---------------|-------------------|------------|
| Tenant data leak / PII exposure | Platform Owner → Architect | Immediate all-hands |
| Brain workflow stuck / crash | Platform Owner | Architect if business logic unclear |
| Integration failure (email/OCR/webhook) | Integrator Owner | Platform Owner if idempotency broken |
| Wrong business outcome (bad policy data, incorrect grants) | Architect | — |
| API down / performance degradation | Integrator Owner | Platform Owner |

### Executor Crash Recovery Runbook

When the Brain executor crashes mid-workflow:

1. **Identify**: Check `BrainTask.status` — if `EXECUTING` with no heartbeat > 5 min, assume crash.
2. **Diagnose**: Pull `trace_id`, check logs for last successful step.
3. **Recover**: 
   - If step was idempotent: re-run from crashed step.
   - If step had side effects (email sent, external API called): check idempotency key, decide skip vs retry.
   - If unclear: set status to `NEEDS_REVIEW`, alert Architect.
4. **Post-mortem**: Platform Owner documents root cause within 24 hours.

### Decision Deadlock Resolution

If Architect and Platform Owner disagree on whether something is "domain" vs "platform":

1. Both write a 1-paragraph position (async, in Slack or PR comments).
2. If still unresolved after 4 hours: **Architect decides** (domain correctness > platform elegance).
3. Dissenting opinion logged in ADR for future reference.

---

## Onboarding Protocol

### Founding Engineers (Day 1-3)

**Day 1: Absorb & Setup**
- Morning: Read this doc + `ARCHITECTURE.md` + Brain spec.
- Afternoon: Set up Cursor, clone repo, run local environment.
- End of day: Post 3 questions in Slack. David answers async.

**Day 2: First Plan**
- Write your first plan for a small task in your layer (Platform or IO).
- Get plan reviewed by David (< 2 hours).
- Execute plan with Cursor agent. Ship PR by EOD.

**Day 3+: Full Velocity**
- Own your layer end-to-end.
- Daily async standup (Slack): what you shipped, what you're planning, blockers.
- Weekly 30-min founding team sync: coordinate cross-layer work.

### Vertical Engineers (Post-Foundation)

**Day 1: Platform Orientation**
- Read `ARCHITECTURE.md` + this doc (focus on "Vertical Engineer" section).
- Review one existing `apps/*_os/` as a reference pattern.
- Scope call with founding team: define your vertical's MVP.

**Day 2: Build**
- Scaffold your `apps/{vertical}_os/` directory.
- Write plan for first feature. Get reviewed by any founding team member.
- Execute with Cursor. Ship PR.

**Day 3+: Independent Execution**
- Own your vertical end-to-end.
- Only escalate when you need a new core capability.
- Weekly demo to founding team to stay aligned.

---

## References

- Architecture spec: `mw-dj6-v2/ARCHITECTURE.md`
- Brain rewrite spec: `.cursor/plans/masterplan/aibrain/nuke.md`
- Loss Run workflow narrative: `.cursor/plans/masterplan/aibrain/lossrun.md`

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 4.2.0 | Initial | Original ownership map |
| 4.2.1 | — | Added team scaling phases, trusted patterns, async approval protocol, failure modes, onboarding protocol |
| 4.2.2 | — | Changed "hires" to "founding engineers"; added vertical engineer scaling model |
| 4.2.3 | — | Added AI-first development model (Cursor agent workflow); compressed all timelines (weeks → days); updated onboarding to Day 1-3 |
