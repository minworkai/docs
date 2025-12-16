# Loss Run Workflow — Deep Dive 

> **Purpose:** This document walks through the Loss Run workflow step-by-step to help engineers understand how the new `core.brain` architecture works in practice.
<img width="800" height="533" alt="image" src="https://github.com/user-attachments/assets/6882f78d-77a0-40b5-9cea-676c62d4fea0" />

---

## 🔴 Critical Implementation Rules

Before reading the workflow steps, understand these **non-negotiable rules** from `nuke.md`:

### Rule 1: Dynamic Steps are INSERTED, not APPENDED

When a tool returns `new_steps`, they must be **inserted immediately after the current step**, not appended to the end.

```python
# ❌ WRONG (from nuke.md pseudo-code - this is a BUG)
task.frozen_plan.extend(new_steps)  # Adds to END of plan

# ✅ CORRECT (actual implementation)
if new_steps:
    # Insert steps at current_index + 1 so they run NEXT
    for i, step in enumerate(new_steps):
        task.frozen_plan.insert(step_index + 1 + i, step)
    task.save()
```

**Why?** If your plan is `[Assess, CreateRequests, Notify]` and `Assess` injects `[GenerateLOA]`:
- ❌ `extend()` → `[Assess, CreateRequests, Notify, GenerateLOA]` → LOA generated AFTER requests fail
- ✅ `insert()` → `[Assess, GenerateLOA, CreateRequests, Notify]` → LOA generated BEFORE requests

### Rule 2: Use `Tenant`, NOT `Org`

All new Brain models and tools use `identity.Tenant`, NOT legacy `core.Org`.

```python
# ❌ WRONG (legacy model)
request_batch = RequestBatch.objects.create(org=deal.org, deal=deal)

# ✅ CORRECT (v4.2 identity model)
request_batch = RequestBatch.objects.create(tenant=context['_tenant'], deal=deal)
```

**Why?** The entire point of the "Nuke" is to break free from the old dependency graph. Linking new models to `Org` re-introduces the spaghetti we're deleting.

---

### 1. "This is a complete rewrite and we are in the initial phase"
**Correct.** We're building `core.brain/` from scratch. We do NOT wrap, call, or integrate with old `CopilotService`. We build fresh, test E2E, then point entry points at Brain and delete old code.

### 2. "Monolith vs Microservices"
**Good analogy.** The Brain is a shared kernel (monolith core) with domain-specific plugins (tools). Each namespace (insurance, fleet, freight) registers its own tools at startup. The orchestration engine is shared; the business logic is modular.

### 3. "Deterministic vs Probabilistic"
**You're right.** Here's the distinction:

| Layer | Type | Example |
|-------|------|---------|
| **LLM Calls** | Probabilistic | "What intent is this email?" → 85% confidence |
| **Workflow Engine** | Deterministic | "Step 3 completed, run Step 4" → Always same order |

The architecture **isolates** the probabilistic parts (Extraction, Intent Detection, Planning) behind **validation gates** (confidence thresholds, human review). Once the plan is set, execution is deterministic.

### 4. Where Non-Deterministic AI Agents Fit (The "Waiter vs Kitchen" Model)

The Brain is the **Kitchen** — deterministic, process-driven, executes workflows exactly the same way every time. But users don't talk to the kitchen directly. They talk to the **Waiter** — an AI Agent that handles the conversational interface.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ARCHITECTURE LAYERS                              │
└─────────────────────────────────────────────────────────────────────────┘

     ┌─────────────┐
     │    User     │  "Get loss runs for ACME"
     └──────┬──────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    AGENT LAYER (Non-Deterministic)                       │
│  • Conversational interface                                              │
│  • Context awareness ("We talked about ACME yesterday")                  │
│  • Clarifies missing info ("I need the policy number")                   │
│  • Translates Brain status to natural language                           │
└─────────────────────────────────────────────────────────────────────────┘
            │
            │  Structured Request: {entity_id, intent, payload}
            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    BRAIN LAYER (Deterministic)                           │
│  • Workflow execution engine                                             │
│  • Tools (domain-specific actions)                                       │
│  • State management (DB writes)                                          │
│  • Safety gates (validation, human review)                               │
└─────────────────────────────────────────────────────────────────────────┘
            │
            │  Status: {COMPLETED, NEEDS_INFO, FAILED}
            ▼
     ┌─────────────┐
     │   Database  │  LossRunRequest, Deal, Entity...
     └─────────────┘
```

**Example: The Missing Info Loop**

```
1. User → Agent:     "Get loss runs for ACME."
2. Agent (Context):  "I see ACME in your history, but I don't have a policy number."
3. Agent → Brain:    Tries to start workflow...
4. Brain:            "Error: Missing required field `policy_number`. PAUSED."
5. Agent:            "I can start that, but I need the policy number first."
6. User → Agent:     "It's POL-123."
7. Agent → Brain:    Updates task input & Resumes workflow.
8. Brain:            Executes steps 1-10 successfully.
9. Agent:            "Done! I've sent the requests to Progressive and Travelers."
```

**Why This Separation is Powerful:**

| Concern | Agent Handles | Brain Handles |
|---------|---------------|---------------|
| **Conversation** | ✅ "What do you mean by loss run?" | ❌ |
| **Context Memory** | ✅ "We talked about ACME yesterday" | ❌ |
| **State Changes** | ❌ | ✅ All DB writes |
| **Workflow Execution** | ❌ | ✅ Step-by-step with checkpoints |
| **Safety Validation** | ❌ | ✅ Confidence gates, human review |

**Key Insight:** The Agent **cannot** hallucinate a database write. It *must* go through the Brain to change state. The Brain doesn't know how to "chat" — it just reports status (`MISSING_INFO`, `NEEDS_REVIEW`), and the Agent translates that into natural language.

This means users can build custom AI agents (with their own personality, domain expertise, language) that all talk to the same deterministic Brain. The Agent is the **interface**; the Brain is the **execution engine**.

---

## The Loss Run Workflow — Step by Step

### Scenario
**Input:** An email arrives from a broker ("ABC Insurance Agency") with a PDF attachment (a "Dec Page" showing ACME Trucking's current policies). They want us to request loss runs from the insurance carriers.

**End Result:** `LossRunRequest` records created in our DB, one per carrier, with the workflow ready to chase them.

---

## Step 1: Email Received → Task Created

**What happens:**
1. Webhook receives email from Resend/Postmark
2. `InboundEmail` record created with attachments
3. `BrainService.create_email_task(inbound_email)` called

**Code (New Architecture):**
```python
# core/brain/services/brain_service.py
def create_email_task(self, inbound_email: InboundEmail) -> BrainTask:
    # Generate idempotency key from email message ID
    idempotency_key = f"email:{inbound_email.message_id}"
    
    # Check for duplicate (prevents double-processing)
    existing = BrainTask.objects.filter(idempotency_key=idempotency_key).first()
    if existing:
        return existing  # Skip duplicate
    
    task = BrainTask.objects.create(
        tenant=inbound_email.org.tenant,  # v4.2: Uses Tenant, not Org
        channel=BrainTask.Channel.EMAIL,
        status=BrainTask.Status.PENDING,
        idempotency_key=idempotency_key,
        input_text=inbound_email.body_text[:5000],
        inbound_email=inbound_email,
        trace_id=str(uuid4()),  # For distributed tracing
    )
    
    # Dispatch to Celery worker AFTER commit
    transaction.on_commit(lambda: process_brain_task.delay(str(task.id)))
    return task
```

**Tables Populated:**
- `brain_tasks` (new task record)

**Deterministic Guarantee:** If the same email webhook fires twice, the second one is skipped instantly via `idempotency_key`.

---

## Step 2: Worker Picks Up Task → Extraction

**What happens:**
1. Celery worker calls `BrainService.process_task(task)`
2. Worker acquires row lock (`select_for_update`)
3. Attached PDFs are extracted using Two-Stage AI

**Two-Stage Extraction (Why?):**
A single "God Prompt" asking for ALL possible fields fails on complex documents. Data gets mixed up, fields get hallucinated.

```
┌─────────────────────────────────────────────────────────────┐
│  STAGE 1: Content Analysis (Fast, Cheap)                    │
│  "What kind of document is this? What data sections exist?" │
│  Output: { document_type: "DEC_PAGE", has_coverages: true } │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 2: Targeted Extraction (Dynamic Schema)              │
│  Only asks for data that Stage 1 detected.                  │
│  Output: { insured: {...}, coverages: [...] }               │
└─────────────────────────────────────────────────────────────┘
```

**Example Extraction Output (stored in `task.execution_context['state']`):**
```json
{
  "extractions": [{
    "media_id": "abc-123",
    "document_type": "DEC_PAGE",
    "confidence": 0.92,
    "extracted_data": {
      "insured": {
        "name": "ACME Trucking LLC",
        "dot_number": "1234567",
        "address": "123 Main St, Fresno CA",
        "email": "john@acmetrucking.com"
      },
      "coverages": [
        {
          "type": "Auto Liability",
          "insurer_name": "Progressive",
          "policy_number": "POL-001",
          "effective_date": "2024-01-01",
          "expiration_date": "2025-01-01"
        },
        {
          "type": "Cargo",
          "insurer_name": "Travelers",
          "policy_number": "POL-002",
          "effective_date": "2024-01-01",
          "expiration_date": "2025-01-01"
        }
      ]
    }
  }]
}
```

**Tables Populated:**
- `brain_task_log_entries` (step completion log)
- `task.execution_context` updated (JSONB field)

**Nothing written to business tables yet!** This is key — extraction is probabilistic, so we don't commit to DB until validated.

---

## Step 3: Intent Detection

**What happens:**
1. LLM classifies the intent based on email content + extraction results
2. Confidence score calculated

**Prompt (simplified):**
```
User Request: "Please request loss runs for ACME Trucking"
Attached Files: dec_page.pdf (DEC_PAGE detected)

Available Intents:
- insurance.loss_run_request: Process dec pages to create loss run requests
- insurance.new_lead: Process new business lead
- general_instruction: General chat

Return: {"intent": "...", "confidence": 0.0-1.0, "reasoning": "..."}
```

**Output:**
```json
{
  "intent": "insurance.loss_run_request",
  "confidence": 0.95,
  "reasoning": "Email contains dec page attachment with coverage information"
}
```

**Human-in-the-Loop Gate:**
```python
if task.confidence_score < task.confidence_threshold:  # default 0.8
    pause_for_review(task, reason="Low confidence")
    return  # Task pauses until human approves
```

If confidence is 0.65, the UI shows a "Review Required" screen. Human fixes it, clicks "Resume", and processing continues with corrected data.

---

## Step 4: Plan Resolution

**What happens:**
1. Find the best `BrainWorkflowPlan` for this intent
2. Snapshot the plan steps to `task.frozen_plan` (immutable)

**Plan Selection Priority:**
1. Tenant-specific plan (if exists)
2. Global plan
3. Within each: higher priority wins

**Example Plan (`insurance.loss_run_pipeline`):**
```json
{
  "key": "insurance.loss_run_pipeline",
  "intent_key": "insurance.loss_run_request",
  "domain": "insurance",
  "steps": [
    {"id": "normalize_email", "tool": "normalize_email_input"},
    {"id": "extract_docs", "tool": "insurance.extract_documents"},
    {"id": "resolve_entity", "tool": "resolve_entity", "args": {
      "name": "{{ context.state.insured.name }}",
      "dot_number": "{{ context.state.insured.dot_number }}",
      "entity_type": "motor_carrier"
    }},
    {"id": "recall_dossier", "tool": "core.recall_dossier"},
    {"id": "create_deal", "tool": "insurance.create_deal"},
    {"id": "assess_plan", "tool": "insurance.assess_and_plan"},
    {"id": "notify", "tool": "notify_agent", "stop_on_failure": false}
  ]
}
```

**Tables Populated:**
- `task.frozen_plan` (JSONB snapshot)
- `task.selected_plan` (FK for audit)

---

## Step 5: Entity Resolution (The Anchor)

**What happens:**
1. Look up `network.Entity` by DOT number or name
2. Enrich with any new contact info from extraction
3. Create "Shadow Entity" if not found

**Tool: `resolve_entity`**
```python
def resolve_entity(context, name, dot_number, entity_type, email=None, phone=None, **kwargs):
    # 1. Try DOT number match (most reliable)
    entity = Entity.objects.filter(
        identifiers__identifier_type='DOT',
        identifiers__normalized_value=dot_number
    ).first()
    
    # 2. Fallback to name match
    if not entity:
        entity = Entity.objects.filter(name__iexact=name).first()
    
    # 3. Create Shadow Entity if not found
    if not entity:
        entity = Entity.objects.create(
            name=name,
            entity_type=entity_type,
            owner_tenant=context['_tenant'],  # Shadow until claimed
        )
        EntityIdentifier.objects.create(
            entity=entity,
            identifier_type='DOT',
            value=dot_number,
        )
        return {'success': True, 'data': {'entity_id': str(entity.id), 'entity_is_new': True}}
    
    # 4. ENRICHMENT: Update empty fields with new data
    updates = []
    if not entity.email and email:
        entity.email = email
        updates.append('email')
    if not entity.phone and phone:
        entity.phone = phone
        updates.append('phone')
    if updates:
        entity.save(update_fields=updates)
    
    return {
        'success': True,
        'data': {
            'entity_id': str(entity.id),
            'entity_name': entity.name,
            'entity_is_new': False,
            'entity_was_enriched': bool(updates),
        }
    }
```

**Tables Populated:**
- `network_entities` (created or enriched)
- `network_entity_identifiers` (DOT number lookup)

---

## Step 6: Memory Recall (The Librarian)

**What happens:**
1. Load SQL state: "How many pending requests exist for this entity?"
2. Load Vector memory: "Any past issues with this entity?"

**Tool: `core.recall_dossier`**
```python
def recall_dossier(context, entity_id, **kwargs):
    from core.memory.services.dossier import DossierService
    
    dossier = DossierService.build_dossier(
        tenant=context['_tenant'],
        entity_id=entity_id
    )
    
    return {
        'success': True,
        'data': {'dossier': dossier},
        'llm_summary': f"Entity has {dossier['state']['pending_requests']} pending requests."
    }
```

**Example Dossier Output:**
```json
{
  "entity": {"id": "...", "name": "ACME Trucking LLC"},
  "state": {
    "pending_requests": 0,
    "open_deals": 1,
    "last_contact": "2024-12-01"
  },
  "insights": [
    "Warning: Progressive sends bad VINs",
    "Broker prefers email over fax"
  ],
  "recent_events": [
    "2024-11-15: Loss run received from Travelers"
  ]
}
```

---

## Step 7: Dynamic Planning (The Domain Brain)

**What happens:**
1. AI examines the Dossier and extraction data
2. Decides what steps are needed based on context
3. Injects new steps into the plan dynamically

**Tool: `insurance.assess_and_plan`**
```python
def assess_and_plan(context, **kwargs):
    dossier = context['state'].get('dossier', {})
    deal = context['state'].get('deal')
    
    reasoning = []
    new_steps = []
    
    # Decision 1: Too many pending requests?
    if dossier['state']['pending_requests'] > 2:
        reasoning.append("3+ pending requests exist. Switching to Chase mode.")
        new_steps.append({'id': 'dyn_chase', 'tool': 'insurance.send_chase_email'})
        return {'success': True, 'data': {'new_steps': new_steps, 'reasoning': reasoning}}
    
    # Decision 2: Is this a Renewal? (No LOA needed)
    if deal and deal.pipeline_type == 'RENEWAL':
        reasoning.append("Renewal deal - skipping LOA generation.")
        new_steps.append({'id': 'dyn_create_requests', 'tool': 'insurance.create_loss_run_requests'})
    
    # Decision 3: New Business without LOA
    elif not _check_loa_exists(deal):
        reasoning.append("New Business without signed LOA. Generating LOA first.")
        new_steps.append({'id': 'dyn_gen_loa', 'tool': 'insurance.generate_loa'})
        new_steps.append({'id': 'dyn_email_loa', 'tool': 'insurance.email_loa_request'})
        new_steps.append({'id': 'dyn_create_requests', 'tool': 'insurance.create_loss_run_requests',
                         'args': {'requires_loa': True}})
    
    # Decision 4: Memory warning - add validation
    if 'bad VINs' in str(dossier.get('insights', [])):
        reasoning.append("Memory warning: History of bad VINs. Adding validation step.")
        new_steps.insert(0, {'id': 'dyn_validate_vins', 'tool': 'fleet.validate_vins_human_review'})
    
    return {
        'success': True,
        'data': {'new_steps': new_steps, 'reasoning': '\n'.join(reasoning)},
        'llm_summary': f"Injected {len(new_steps)} steps based on context analysis."
    }
```

**⚠️ CRITICAL: Executor uses INSERT, not APPEND (see Rule 1 above)**

When `assess_plan` returns `new_steps`, the Executor **inserts** them immediately after the current step:

```python
# In core/brain/services/executor.py
new_steps = result.get('data', {}).get('new_steps', [])
if new_steps:
    # INSERT at current position + 1 (so they run NEXT)
    for i, step in enumerate(new_steps):
        task.frozen_plan.insert(step_index + 1 + i, step)
    task.save()
```

**After Dynamic Injection, `task.frozen_plan` looks like:**
```json
[
  {"id": "normalize_email", "tool": "normalize_email_input"},
  {"id": "extract_docs", "tool": "insurance.extract_documents"},
  {"id": "resolve_entity", "tool": "resolve_entity"},
  {"id": "recall_dossier", "tool": "core.recall_dossier"},
  {"id": "create_deal", "tool": "insurance.create_deal"},
  {"id": "assess_plan", "tool": "insurance.assess_and_plan"},  // ← We are HERE (step_index = 5)
  // ↓ INSERTED AT step_index + 1 ↓
  {"id": "dyn_gen_loa", "tool": "insurance.generate_loa"},
  {"id": "dyn_email_loa", "tool": "insurance.email_loa_request"},
  {"id": "dyn_create_requests", "tool": "insurance.create_loss_run_requests"},
  // ↑ INSERTED BEFORE notify ↑
  {"id": "notify", "tool": "notify_agent"}  // ← Original step 6, now step 9
]
```

**Execution Order:** `assess_plan` → `dyn_gen_loa` → `dyn_email_loa` → `dyn_create_requests` → `notify`

This ensures the LOA is generated and sent **before** we try to create loss run requests.

---

## Step 8: Create Loss Run Requests

**What happens:**
1. For each carrier in the extraction, create a `LossRunRequest`
2. Use deduplication: If a DRAFT request already exists for (Deal + Carrier), UPDATE instead of CREATE
3. Create `ActionRequestPolicy` records for UI visibility

**Tool: `insurance.create_loss_run_requests`**

**⚠️ CRITICAL: Uses `Tenant`, NOT `Org` (see Rule 2 above)**

```python
def create_loss_run_requests(context, requires_loa=False, **kwargs):
    extracted_policies = context['state'].get('extractions', [{}])[0].get('extracted_data', {})
    deal_id = context['state'].get('deal_id')
    tenant = context['_tenant']  # ✅ v4.2: Use Tenant from context
    
    deal = Deal.objects.get(id=deal_id)
    request_batch = RequestBatch.objects.create(tenant=tenant, deal=deal)  # ✅ Tenant, not Org
    
    created_count = 0
    updated_count = 0
    request_ids = []
    
    for coverage in extracted_policies.get('coverages', []):
        carrier_name = coverage.get('insurer_name')
        carrier = Carrier.objects.filter(name__icontains=carrier_name).first()
        
        # DEDUPLICATION: Check for existing DRAFT request
        existing = LossRunRequest.objects.filter(
            deal=deal,
            target_carrier=carrier,
            status=LossRunRequest.Status.DRAFT
        ).first()
        
        if existing:
            # UPDATE existing request (merge data)
            existing.props['policy_numbers'].append(coverage.get('policy_number'))
            existing.save()
            updated_count += 1
            request_ids.append(str(existing.id))
        else:
            # CREATE new request
            request = LossRunRequest.objects.create(
                tenant=tenant,  # ✅ Tenant, not Org
                deal=deal,
                request_batch=request_batch,
                target_carrier=carrier,
                status=LossRunRequest.Status.DRAFT,
                requires_loa=requires_loa,
                props={'policy_numbers': [coverage.get('policy_number')]}
            )
            created_count += 1
            request_ids.append(str(request.id))
    
    return {
        'success': True,
        'data': {
            'created_request_ids': request_ids,
            'created_count': created_count,
            'updated_count': updated_count,
            'deal_id': str(deal.id),
            'request_batch_id': str(request_batch.id),
        },
        'llm_summary': f"Created {created_count}, updated {updated_count} loss run requests."
    }
```

**Tables Populated:**
- `insurance_loss_run_requests` (one per carrier, linked to `Tenant`)
- `insurance_request_batches` (groups requests, linked to `Tenant`)
- `insurance_action_request_policies` (for UI)
- `insurance_activities` (audit log)

**Note:** All new records are linked to `Tenant` (v4.2), NOT legacy `Org`. The `TenantScopedManager` automatically filters queries by the active `TenantContext`.

---

## Step 9: Workflow Delegation (Temporal Handoff)

**What happens (if LOA required):**
1. The workflow starts a Temporal workflow to chase signatures
2. Brain marks task as `DELEGATED` (not `COMPLETED`)
3. When Temporal finishes, it calls back to update the task

**Why DELEGATED status?**
If we mark `COMPLETED` immediately, the user sees "Success" but nothing actually happened yet. `DELEGATED` is honest: "We started the work, waiting for completion."

---

## Step 10: Task Completion

**Final State:**
```json
{
  "status": "COMPLETED",
  "execution_context": {
    "state": {
      "entity_id": "abc-123",
      "entity_name": "ACME Trucking LLC",
      "deal_id": "def-456",
      "request_ids": ["req-001", "req-002"],
      "created_count": 2,
      "updated_count": 0
    }
  },
  "completed_step_ids": [
    "normalize_email",
    "extract_docs", 
    "resolve_entity",
    "recall_dossier",
    "create_deal",
    "assess_plan",
    "dyn_create_requests",
    "notify"
  ]
}
```

**Tables Populated (Summary):**
| Table | Records Created |
|-------|-----------------|
| `brain_tasks` | 1 |
| `brain_task_log_entries` | ~8-10 (one per step) |
| `network_entities` | 0-1 (if new) |
| `insurance_deals` | 1 |
| `insurance_request_batches` | 1 |
| `insurance_loss_run_requests` | 2 (one per carrier) |

---

## Crash Recovery

**What if the worker dies mid-execution?**

1. Task has `completed_step_ids = ['normalize_email', 'extract_docs']`
2. Zombie cleanup beat task detects stuck task (locked_at > 30 min ago)
3. Resets task to `PENDING`
4. New worker picks up, skips completed steps, resumes from `resolve_entity`

**Step IDs not Indices:** If you hot-fix a plan to insert a step at index 0, resuming at "index 1" would execute the wrong step. Step IDs are stable strings.

---

## Old vs New Architecture Comparison

| Aspect | Old (`CopilotService`) | New (`core.brain`) |
|--------|------------------------|-------------------|
| **Memory** | None (stateless) | Dossier (SQL + Vector) |
| **Planning** | Hardcoded if/else | Dynamic step injection |
| **Safety** | Fire and pray | Check-Execute-Commit |
| **Retry** | Re-run everything | Resume from checkpoint |
| **Human Review** | None | Pause/Resume on low confidence |
| **Logging** | JSONField (TOAST bloat) | Append-only table |
| **Multi-domain** | Insurance-only | Registry pattern (any domain) |

---

## Ready to Implement

The plan is solid. Start with:

1. **Day 1:** Scaffold `core.brain/` app + models + migrations
2. **Day 2-3:** `BrainService` (create_* methods, pipeline skeleton)
3. **Day 3-4:** Executor (Check-Execute-Commit, Jinja2 templates)
   - ⚠️ **Implement INSERT logic for dynamic steps, not APPEND** (Rule 1)
4. **Day 5-6:** Core tools (normalize, extract, resolve_entity)
5. **Day 6-7:** Insurance tools + E2E test (loss_run_request scenario)
   - ⚠️ **All models use `Tenant`, not `Org`** (Rule 2)

### Pre-Flight Checklist

Before marking a Day complete, verify:

| Day | Critical Check |
|-----|----------------|
| Day 3-4 | Dynamic steps are **inserted** after current step (not appended) |
| Day 5-7 | All new models link to `Tenant`, not `Org` |
| Day 5-7 | All tools use `context['_tenant']`, not `task.org` or `deal.org` |
| Day 7 | E2E test: LOA steps run BEFORE create_loss_run_requests |



