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
- **Sr. Engineers** build the **How**: execution machinery, IO pipelines, integrations, APIs.

This maximizes speed while ensuring correctness in the most expensive-to-fix layer: **data modeling + workflow meaning**.

---

## Team Scaling Phases

This ownership map adapts as the team grows. We start with founder-only, then progressively delegate.

| Phase | Headcount | Ownership Model |
|-------|-----------|-----------------|
| **Bootstrap** | David only | David owns all; doc serves as future contract |
| **First Hire** | David + 1 Sr. Eng | Collapsed map (see below) |
| **Full Team** | David + 2 Sr. Engs | Full ownership map activated |

### Bootstrap Phase (Current)

David builds the Phase 1 foundation (TenantContext, PermissionService, base executor patterns). This establishes the "trusted patterns" that the first hire will extend rather than architect from scratch.

### First Hire Collapse

When one Sr. Engineer joins, ownership collapses to:

| Domain | David | Sr. Engineer |
|--------|-------|--------------|
| Business Kernel (`network/`, `insurance/`, `assets/`) | **Primary** | Contributor (follows patterns) |
| Platform Kernel (`common/`, `access/`, `core/brain/`) | Reviewer | **Primary** |
| IO Layer (`documents/`, `communication/`, APIs) | Reviewer | **Primary** |

The first hire effectively becomes "Sr. Eng A + B" until headcount grows.

---

## Ownership Map (Full Team)

### Shared Decision Rights

- **Changes to Golden Rules** (tenancy, ACL, entity resolution, brain determinism) require Architect sign-off after hearing objections. Architect has final call.
- Any change that impacts **tenant isolation** or **PII** requires Architect approval.

### Chief Architect / Domain Expert (David) — Business Kernel Owner

**Primary owner**:
- `network/` (Entity, Identifier normalization, merge/claim semantics)
- `insurance/` (Policy/Term/Coverage/Submission semantics)
- `assets/` (Driver/Vehicle domain rules, PII constraints)
- `apps/*/models/` where business semantics live (e.g., `apps/agency_os/models/`)

**Also owns**:
- Workflow semantics and plan intent mapping (Loss Run, Quote Submission, Renewal, etc.)
- "What must be true" invariants: uniqueness constraints, canonicalization rules, lifecycle states

**Approval required**:
- Any schema change in the above domains
- Any change that alters business meaning (e.g., what "Submission" grants, what "Claimed" means)

**Trusted Patterns (No Approval Needed)**:
- Adding a nullable field to an existing model that follows established naming conventions
- Adding a new `@property` or helper method that doesn't change persistence
- Adding indexes for query performance
- Extending enum choices where the pattern is established (e.g., adding a new `DocumentType`)

### Sr. Engineer A (Platform) — Execution & Security Kernel Owner

**Primary owner**:
- `common/` (BaseModel patterns, ContextVars, scoped managers)
- `access/` (Resource, AccessGrant, PermissionService, relationship edges)
- `core/brain/` (BrainTask/Plan plumbing, executor, check-execute-commit, crash recovery)

**Accountable for**:
- Deterministic executor behavior (idempotency, retries, step insertion semantics)
- Tenant isolation correctness (TenantContext in workers + brain execution)
- Observability (trace_id propagation), PII-safe logs

**Approval required**:
- Any changes under `common/`, `access/`, `core/brain/`

**Trusted Patterns (No Approval Needed)**:
- Adding new log statements (if PII-safe)
- Adding new metrics/traces that follow existing conventions
- Bug fixes that don't change executor state machine semantics

### Sr. Engineer B (Integrator) — IO, Integrations, API Surface Owner

**Primary owner**:
- `documents/` (S3 storage, classification/extraction pipeline, linking tables)
- `communication/` (email/sms/voice gateways)
- External integrations (Gmail/Resend/RingCentral/Stripe/portals)
- REST APIs + serializers + views for consuming the domains cleanly

**Accountable for**:
- Reliable ingestion (email/PDF → Document → links → events)
- Integration idempotency and retry safety
- Clean API contracts that respect ACL and tenancy

**Trusted Patterns (No Approval Needed)**:
- Adding new API endpoints that follow established serializer patterns
- Adding retry/backoff logic to existing integrations
- Adding new webhook handlers that follow existing patterns

---

## Collaboration Rules (Non-Negotiable)

### PR Review Rules

**Architect must approve** any PR that:
- Changes domain schemas/rules in `network/`, `insurance/`, `assets/`, `apps/*/models/`
- Changes anything that affects "business truth" (policy ownership, grants, claiming)

**Platform owner must approve** any PR that:
- Touches `common/`, `access/`, `core/brain/`
- Alters executor semantics (dynamic step injection, idempotency, status transitions)

**Integrator owner must approve** any PR that:
- Touches `documents/`, `communication/`, external gateways, or public API surfaces

### Async Approval Protocol

To prevent bottlenecks (especially while David is fundraising):

| Approval Type | Timeout | Default if No Response |
|---------------|---------|------------------------|
| Trusted Pattern changes | None | Auto-merge if tests pass |
| Standard changes | 24 hours | Approved if tests pass + no objections |
| Golden Rule / PII / Tenant isolation | No timeout | Must get explicit approval |
| Schema migrations | No timeout | Must get explicit approval |

**Exception**: During active fundraising sprints, David may designate a "delegate reviewer" for standard domain changes.

### "Contracts Before Code"

Before implementing a cross-domain feature:
- Write a **1-page contract**: inputs, outputs, required IDs, tenant context assumptions.
- Confirm **which model is the source of truth** (Entity vs Tenant vs Policy vs Submission).

### "No Hidden Tenancy"

- Background jobs must accept `tenant_id` and set `TenantContext` explicitly.
- Domain queries must use `TenantScopedManager` where applicable; global models must use ACL services.

### "No Legacy Imports"

- Brain + new domain code must not import legacy `core.models.Org`.
- Everything new keys off **`identity.Tenant`**.

---

## Delivery Sequencing (3 Phases)

Designed to unblock parallel work while keeping foundational correctness first.

### Phase 1 — Security + Tenancy Foundation

**Owner:** David (bootstrap), then Sr. Eng A (Platform) extends. Architect reviews.

**Deliverable (Definition of Done)**:
- `TenantContext` exists and is required for scoped queries.
- `PermissionService` + `AccessGrant` enforced patterns exist.
- A "hello world" worker and brain tool can run inside tenant context.
- Trusted patterns documented for Phase 2 contributors.

**Hiring Gate**: Phase 1 core must be complete before first engineer starts. They extend, not architect.

### Phase 2 — Domain Models + Ingestion Primitives

**Owner:** Architect (domain models), Sr. Eng B (documents ingestion), Sr. Eng A supports.

**Deliverable**:
- Canonical `network.Entity` + `EntityIdentifier` normalization implemented.
- Insurance contract models exist with constraints and clear ownership semantics.
- Documents can be stored, classified, and linked to Entities/Policies with ACL anchors.

### Phase 3 — Brain Workflows + Marketplace

**Owner:** Architect defines workflow semantics + plan steps; Sr. Eng A implements executor; Sr. Eng B wires IO triggers.

**Deliverable**:
- Loss Run workflow runs end-to-end with deterministic executor rules (including **step insertion**, not append).
- Marketplace gating exists (tenant subscriptions → allowed plans/capabilities).
- Human-in-the-loop flows supported (`NEEDS_REVIEW` status, clear remediation path).

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
4. **Post-mortem**: Platform Owner documents root cause within 48 hours.

### Decision Deadlock Resolution

If Architect and Platform Owner disagree on whether something is "domain" vs "platform":

1. Both write a 1-paragraph position (async, in PR comments).
2. If still unresolved after 24 hours: **Architect decides** (domain correctness > platform elegance).
3. Dissenting opinion logged in ADR (Architecture Decision Record) for future reference.

---

## Onboarding Protocol (First Hire)

### Week 1: Shadow & Absorb

- Day 1-2: Read this doc + `ARCHITECTURE.md` + Brain spec. Ask questions async in dedicated Slack channel.
- Day 3-4: Shadow David on one domain task (e.g., reviewing a model change) and one platform task (e.g., debugging TenantContext).
- Day 5: Ship a "hello world" PR that touches Platform layer (add a log line, new test, etc.). Get review feedback.

### Week 2: Take Ownership

- Take primary ownership of Platform Kernel (`common/`, `access/`, `core/brain/`).
- David shifts to reviewer role for Platform; remains primary on Business Kernel.
- First real deliverable: extend executor with one new capability (e.g., step timeout handling).

### Week 3+: Full Velocity

- Engineer owns Platform + IO layers (collapsed Sr. Eng A+B role).
- Weekly 30-min sync with David: review open PRs, surface domain questions, adjust ownership boundaries as needed.

---

## References

- Architecture spec: `mw-dj6-v2/ARCHITECTURE.md` (renamed from `fromscratch.md`)
- Brain rewrite spec: `.cursor/plans/masterplan/aibrain/nuke.md`
- Loss Run workflow narrative: `.cursor/plans/masterplan/aibrain/lossrun.md`

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 4.2.0 | Initial | Original ownership map |
| 4.2.1 | — | Added team scaling phases, trusted patterns, async approval protocol, failure modes, onboarding protocol, deadlock resolution |
