# MinuteWork v4.2: Technical Architecture Specification

## Executive Summary

MinuteWork is the **Operating System for Trust in Logistics**. We replace fragmented email/PDF workflows with a unified, permissioned graph where:
- **Insurance Agencies** manage policies and renewals efficiently.
- **Freight Brokers** verify compliance instantly.
- **Insurance Carriers** underwrite risk without data entry.
- **Fleets** own their data once and share it everywhere.

**Core Innovation:** The "Golden Record" + "Shadow Tenant" growth engine.

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Domain Definitions (The Dictionary)](#2-domain-definitions)
3. [The 7 Golden Technical Rules](#3-the-7-golden-technical-rules)
4. [Directory Structure](#4-directory-structure)
5. [Core Data Models](#5-core-data-models)
6. [Access Control Engine](#6-access-control-engine)
7. [Key Workflows](#7-key-workflows)
8. [Security Implementation](#8-security-implementation)
9. [Infrastructure](#9-infrastructure)
10. [Implementation Roadmap](#10-implementation-roadmap)

---

## 1. System Architecture

### High-Level Stack
- **Backend:** Django 5.x (Async-safe Modular Monolith)
- **Database:** PostgreSQL 16+ (Partitioned Audit/Events)
- **Workers:** Celery (Tasks <60s) + Temporal (Workflows >60s)
- **ACL Engine:** Resource-based Access Control + Materialized Relationship Graph
- **Search:** PostgreSQL Trigram indexes (gin_trgm_ops)
- **Storage:** S3 (Encrypted) for documents

### The Core Value Proposition

We create a **"Golden Record"** for logistics entities (Trucking Companies) that:
1. Eliminates duplicate data entry across parties.
2. Propagates verified updates instantly across the network.
3. Enforces compliance through permissioned access.

---

## 2. Domain Definitions (The Dictionary)

Strict naming conventions to prevent ambiguity.

| Concept | System Term (TenantType) | Definition | Primary Data Rights |
|---------|--------------------------|------------|---------------------|
| **Fleet** | `motor_carrier` | The trucking company. The Subject. | Owns Drivers, Vehicles, Safety Data |
| **Agency** | `insurance_agency` | The independent broker selling the policy | Owns the Policy Resource (Legal Owner) |
| **Carrier** | `insurance_carrier` | The underwriter taking risk (Travelers, Progressive) | Participant. Owns Loss Runs |
| **Broker** | `freight_broker` | The logistics middleman moving loads | Monitor. View access to COI only |
| **Shipper** | `shipper` | The goods owner (Retailer/Manufacturer) | Guest. View verification proofs |

---

## 3. The 7 Golden Technical Rules

1. **ACL First:** Every sensitive model (Policy, Driver, Document) MUST link to a Resource. No exceptions.

2. **ContextVars for Tenancy:** Thread-locals are banned. Use `common.context` for async-safe tenant context.

3. **Explicit Scope:** Background jobs must accept `tenant_id` as an argument and set context explicitly via `TenantContext`.

4. **Shadow Tenants:** We do not migrate data on signup. We create "Shadow Tenants" (owned by the creator) that users later "Claim."

5. **Normalized Identifiers:** DOT/MC numbers are normalized (stripped, uppercased) in `EntityIdentifier` before uniqueness checks.

6. **Agency Owns Policy:** The `insurance_agency` is the `owner_tenant` of a Policy. The `insurance_carrier` is a guest via `AccessGrant`.

7. **Visibility Drives Grants:** We do not compute permissions at runtime via joins. When a connection exists, we create explicit `AccessGrant` rows asynchronously.

---

## 4. Directory Structure

```text
minutework/
├── manage.py
├── pyproject.toml
│
├── config/                     # Django Project Settings
│   ├── settings/
│   │   ├── base.py
│   │   ├── local.py
│   │   └── production.py
│   ├── urls.py                 # /api/v1/ versioning
│   ├── celery.py               # Celery config
│   └── wsgi.py
│
├── common/                     # Shared Infrastructure
│   ├── models/
│   │   ├── base.py             # BaseModel (NO props, NO soft-delete by default)
│   │   └── mixins.py           # SoftDeleteMixin (opt-in)
│   ├── context/
│   │   └── tenant.py           # Async-safe tenant context (ContextVars)
│   ├── managers/
│   │   └── scoped.py           # TenantScopedManager, AccessControlledManager
│   ├── middleware/
│   │   └── tenant.py           # Request context injection
│   ├── events/
│   │   ├── outbox.py           # DomainEvent (transactional outbox)
│   │   └── processor.py        # Event processor (SKIP LOCKED)
│   └── utils/
│
├── identity/                   # [DOMAIN] Auth, Tenancy, RBAC
│   ├── models/
│   │   ├── user.py             # User (NO soft-delete, is_active only)
│   │   ├── tenant.py           # Tenant (with Shadow status)
│   │   ├── membership.py       # Membership
│   │   └── api_key.py          # APIKey
│   ├── services/
│   │   ├── auth.py
│   │   └── shadow.py           # Shadow Tenant creation logic
│   └── api/
│
├── access/                     # [DOMAIN] ACL Primitive (P0 Critical)
│   ├── models/
│   │   ├── resource.py         # Resource (ACL anchor)
│   │   ├── grant.py            # AccessGrant (with source_type/provenance)
│   │   └── relationship_edge.py # Materialized graph for performance
│   ├── services/
│   │   ├── permissions.py      # PermissionService (can_access, grant, revoke)
│   │   └── auto_grant.py       # Auto-Grant Service (connection → grants)
│   └── managers/
│       └── accessible.py       # AccessControlledQuerySet
│
├── network/                    # [DOMAIN] The Global Graph
│   ├── models/
│   │   ├── entity.py           # Entity (with canonical_entity)
│   │   ├── identifier.py       # EntityIdentifier (with normalized_value)
│   │   ├── profiles.py         # CarrierProfile, InsurerProfile, etc.
│   │   ├── tenant_view.py      # TenantEntityView
│   │   ├── connection.py       # Connection
│   │   ├── change_proposal.py  # EntityChangeProposal
│   │   ├── matching.py         # EntityMatch, EntityMerge
│   │   └── verification.py     # VerificationEvidence, EntityClaimRequest
│   ├── services/
│   │   ├── resolution.py       # Entity matching/deduplication
│   │   ├── claiming.py         # Claim verification workflow
│   │   └── merge.py            # Merge operations
│   └── api/
│
├── insurance/                  # [DOMAIN] Contracts (Source of Truth)
│   ├── models/
│   │   ├── policy.py           # Policy (with Resource)
│   │   ├── policy_term.py      # PolicyTerm (with revision)
│   │   ├── coverage.py         # PolicyCoverage
│   │   ├── certificate.py      # Certificate, CertificateLine
│   │   ├── claim.py            # Claim
│   │   └── submission.py       # Submission (workflow state)
│   ├── services/
│   └── api/
│
├── assets/                     # [DOMAIN] Physical World
│   ├── models/
│   │   ├── vehicle.py          # Vehicle (with Resource)
│   │   ├── driver.py           # Driver (PII, with Resource)
│   │   ├── commodity.py        # Commodity
│   │   └── location.py         # Location
│   ├── services/
│   └── api/
│
├── documents/                  # [DOMAIN] First-Class Trust Units
│   ├── models/
│   │   ├── media.py            # Media (S3 storage)
│   │   ├── document.py         # Document (with visibility)
│   │   ├── document_entity_link.py    # Link to Entity
│   │   ├── document_policy_link.py    # Link to Policy
│   │   └── document_share.py   # DocumentShare
│   ├── services/
│   │   └── classification.py   # AI classification
│   └── api/
│
├── core/brain/                 # [SERVICE] The Brain (AI Orchestration)
│   ├── models/                 # BrainTask, BrainWorkflowPlan, BrainArtifactLink
│   │   ├── task.py             # BrainTask (uses identity.Tenant, NOT Org)
│   │   ├── tool.py             # BrainTool (registry metadata)
│   │   ├── plan.py             # BrainWorkflowPlan
│   │   └── artifact.py         # BrainArtifactLink
│   ├── services/               # Orchestrator, Executor, IntentDetector
│   │   ├── brain_service.py    # Main orchestrator (wraps execution in TenantContext)
│   │   ├── executor.py         # Check-Execute-Commit pattern
│   │   └── intent_detector.py  # LLM intent classification
│   ├── tools/                  # Shared tools (normalize, extract_documents)
│   ├── registry.py             # ToolRegistry (decorator-based registration)
│   └── tasks.py                # Celery async worker
│
│   ├── marketplace/            # [SERVICE] Capability App Store
│   │   ├── models/
│   │   │   ├── capability.py   # BrainCapability (The Product)
│   │   │   ├── subscription.py # TenantCapabilitySubscription (The License)
│   │   │   └── connection.py   # CapabilityConnection (Orchestration Rules)
│   │   ├── services/
│   │   ├── tools/              # core.evaluate_connections
│   │   └── api/
│
├── communication/              # [SERVICE] Nervous System
│   ├── models/
│   │   ├── conversation.py
│   │   ├── message.py
│   │   └── notification.py
│   ├── services/
│   │   ├── email.py            # Resend gateway
│   │   ├── sms.py
│   │   └── voice.py            # RingCentral
│   └── api/
│
├── audit/                      # [SERVICE] Compliance Trail
│   ├── models/
│   │   └── audit_log.py        # AuditLog (partitioned by month)
│   ├── services/
│   │   └── audit_service.py    # log_read(), log_write()
│   └── middleware.py
│
│
│   # Domain-specific Brain tools live in their domains:
│   # insurance/domain/tools/ → @ToolRegistry.register("insurance.*")
│   # fleet/domain/tools/     → @ToolRegistry.register("fleet.*")
│   #
│   # CRITICAL INTEGRATION:
│   # - BrainTask.tenant → FK(identity.Tenant), NOT legacy Org
│   # - Executor wraps tool execution in TenantContext(task.tenant)
│   # - This enables TenantScopedManager in domain tools
│
├── apps/                       # [APPS] Tenant Workflows
│   ├── agency_os/              # Insurance Agency CRM
│   │   ├── models/
│   │   │   ├── deal.py
│   │   │   ├── submission.py   # Quote request workflow
│   │   │   ├── action_request.py
│   │   │   └── activity.py
│   │   ├── views/
│   │   ├── workflows/          # Temporal ONLY
│   │   └── tasks.py            # Celery ONLY
│   │
│   ├── carrier_os/             # Fleet Management
│   │   ├── models/
│   │   │   ├── onboarding.py
│   │   │   └── maintenance.py
│   │   └── views/
│   │
│   └── broker_os/              # Compliance Monitoring
│       ├── models/
│       │   ├── monitor.py
│       │   └── packet.py
│       └── views/
│
└── datalake/                   # [DATA] External Data (Separate DB)
    └── models/
        ├── job_run.py          # ETL job tracking
        ├── fmca/               # FMCSA data (Census, Safety, Compliance)
        │   ├── carrier.py      # CarrierSnapshot (census), CensusHistory
        │   ├── insurance.py    # Insurance, InsuranceActive, BOC3Filing
        │   ├── inspection.py   # Inspection, Violation, InspectionGeo
        │   ├── safety.py       # CarrierSafety, SmsInputCensus, LiveSafetyScore
        │   ├── crash.py        # CrashEvent, CrashDetail
        │   ├── authority.py    # AuthorityHistory, OutOfServiceOrder, RevocationEvent
        │   ├── vehicle.py      # VehicleInspection, VehicleLifecycle
        │   ├── geo.py          # ZipCodeCentroid
        │   └── linkage.py      # EntityLink, LinkOverride, FmcsaInsurerPartnerMap
        └── enums.py            # CarrierStatus, SafetyRating, FleetSize, etc.
```

---

## 5. Core Data Models

### 5.1. BaseModel (common/models/base.py)

```python
"""
BaseModel: Foundation for all domain models.

Design Decisions:
- UUID primary key (distributed safety)
- Audit fields (created_by, updated_by)
- NO props JSONField (banned - use domain-specific fields)
- NO deleted_at by default (opt-in via SoftDeleteMixin)
"""
from uuid import uuid4
from django.db import models


class BaseModel(models.Model):
    """Base model for all domain objects."""
    id = models.UUIDField(primary_key=True, default=uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    # Audit fields
    created_by = models.ForeignKey(
        'identity.User',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='+'
    )
    updated_by = models.ForeignKey(
        'identity.User',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='+'
    )
    
    class Meta:
        abstract = True


class SoftDeleteMixin(models.Model):
    """
    Opt-in soft delete.
    
    NOT for: User (use is_active)
    OK for: Tenant, Entity, Policy, etc.
    """
    deleted_at = models.DateTimeField(null=True, blank=True, db_index=True)
    
    class Meta:
        abstract = True
    
    def soft_delete(self, user=None):
        from django.utils import timezone
        self.deleted_at = timezone.now()
        if hasattr(self, 'updated_by'):
            self.updated_by = user
        self.save()
    
    @property
    def is_deleted(self):
        return self.deleted_at is not None
```

### 5.2. Tenant Context (common/context/tenant.py)

```python
"""
Async-safe tenant context using ContextVars.

WHY NOT thread locals:
- Django 5 supports async views
- Thread locals don't work in async contexts
- Celery/Temporal workers need explicit scoping

RULES:
1. Web requests: Middleware sets context
2. Background jobs: MUST pass tenant_id and use TenantContext
3. Queries: Use TenantScopedManager (raises if no context)
"""
from contextvars import ContextVar
from typing import Optional

_current_tenant: ContextVar[Optional['Tenant']] = ContextVar('current_tenant', default=None)
_current_user: ContextVar[Optional['User']] = ContextVar('current_user', default=None)


def get_current_tenant():
    """Get tenant from async-safe context."""
    return _current_tenant.get()


def set_current_tenant(tenant):
    """Set tenant in async-safe context."""
    return _current_tenant.set(tenant)


def get_current_user():
    """Get user from async-safe context."""
    return _current_user.get()


def set_current_user(user):
    """Set user in async-safe context."""
    return _current_user.set(user)


class TenantContext:
    """
    Context manager for explicit tenant scoping.
    
    Usage in Celery tasks:
        @shared_task
        def process_deal(tenant_id: str, deal_id: str):
            tenant = Tenant.objects.get(id=tenant_id)
            with TenantContext(tenant):
                deal = Deal.objects.get(id=deal_id)  # Auto-scoped
                ...
    
    Usage in Temporal activities:
        @activity.defn
        async def send_notification(tenant_id: str, ...):
            tenant = await sync_to_async(Tenant.objects.get)(id=tenant_id)
            with TenantContext(tenant):
                ...
    
    Usage in Brain Executor (CRITICAL):
        # In core/brain/services/brain_service.py
        def _execute_plan(self, task: BrainTask):
            with TenantContext(task.tenant, user=task.created_by):
                for step in task.frozen_plan:
                    # Domain tools use TenantScopedManager
                    result = tool_fn(context=context, **args)
    """
    def __init__(self, tenant, user=None):
        self.tenant = tenant
        self.user = user
        self._tenant_token = None
        self._user_token = None
    
    def __enter__(self):
        self._tenant_token = set_current_tenant(self.tenant)
        if self.user:
            self._user_token = set_current_user(self.user)
        return self
    
    def __exit__(self, *args):
        _current_tenant.reset(self._tenant_token)
        if self._user_token:
            _current_user.reset(self._user_token)


class TenantContextRequired(Exception):
    """Raised when a scoped query is executed without tenant context."""
    pass
```

### 5.3. TenantScopedManager (common/managers/scoped.py)

```python
"""
Managers for tenant-scoped queries.

CRITICAL: Returns exception, NOT empty queryset, when context missing.
This prevents silent bugs in background jobs.
"""
from django.db import models
from common.context.tenant import get_current_tenant, TenantContextRequired


class TenantScopedManager(models.Manager):
    """
    Manager that REQUIRES tenant context.
    
    Behavior:
    - If tenant context exists: Filter by tenant + exclude soft-deleted
    - If no context: RAISE TenantContextRequired (fail loud, not silent)
    
    Usage:
        class Deal(BaseModel, SoftDeleteMixin):
            tenant = FK(Tenant)
            objects = TenantScopedManager()
            all_objects = models.Manager()  # For admin use
    """
    def get_queryset(self):
        qs = super().get_queryset()
        tenant = get_current_tenant()
        
        if tenant:
            qs = qs.filter(tenant=tenant)
            # Exclude soft-deleted if model has the field
            if hasattr(self.model, 'deleted_at'):
                qs = qs.filter(deleted_at__isnull=True)
            return qs
        
        # Fail loud: Prevent silent bugs in background jobs
        raise TenantContextRequired(
            f"Cannot query {self.model.__name__} without tenant context. "
            f"Use TenantContext or query via all_objects."
        )


class UnscopedManager(models.Manager):
    """
    Explicit unscoped manager for admin/internal use.
    
    Use when:
    - Admin views
    - Background jobs with explicit tenant handling
    - System-level operations
    """
    def get_queryset(self):
        qs = super().get_queryset()
        # Exclude soft-deleted if model has the field
        if hasattr(self.model, 'deleted_at'):
            qs = qs.filter(deleted_at__isnull=True)
        return qs
```

---

## 6. Identity Domain

**Architecture Pattern: "Passport & Badge"**
- **Passport (`User`):** Identity, authentication, global profile. Never soft-deleted.
- **Badge (`Membership`):** Context, permissions, access to a specific Tenant. One user can have many badges.

### 6.1. User (NO Soft-Delete)

```python
"""
identity/models/user.py

User: Platform login. NEVER soft-deleted.

Security Rules:
- Use is_active=False to deactivate
- Email is never reused
- For GDPR: anonymize() personal fields, keep ID for audit
"""
from django.contrib.auth.models import AbstractUser
from django.db import models
from uuid import uuid4


class User(AbstractUser):
    """
    Platform user.
    
    Can belong to multiple tenants via Membership.
    NO soft delete (security + audit + email reuse prevention).
    """
    id = models.UUIDField(primary_key=True, default=uuid4, editable=False)
    
    email = models.EmailField(unique=True)  # Never reused
    phone = models.CharField(max_length=50, blank=True)
    
    # Override username to be optional
    username = models.CharField(max_length=150, blank=True)
    
    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []
    
    class Meta:
        db_table = 'identity_user'
    
    def deactivate(self):
        """Deactivate user (instead of deleting)."""
        self.is_active = False
        self.save(update_fields=['is_active', 'updated_at'])
    
    def anonymize(self):
        """
        GDPR compliance: Anonymize personal data but keep ID for audit trail.
        """
        self.email = f'anonymized-{self.id}@deleted.local'
        self.first_name = 'Deleted'
        self.last_name = 'User'
        self.phone = ''
        self.is_active = False
        self.save()
```

### 6.2. Tenant (with Shadow Status)

```python
"""
identity/models/tenant.py

Tenant: The subscription container.

Key Feature: Shadow Status
- When Agency creates a lead for "Bob's Trucking" (not on platform),
  we create a Tenant(status='shadow', type='motor_carrier')
- This Tenant owns the data until Bob claims it
- This is the viral growth engine
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class Tenant(BaseModel, SoftDeleteMixin):
    """
    Subscription container (organization).
    
    Tenant Types match Domain Definitions (Section 2).
    Shadow Tenants are pre-created for unclaimed entities.
    """
    class TenantType(models.TextChoices):
        MOTOR_CARRIER = 'motor_carrier', 'Motor Carrier (Fleet)'
        INSURANCE_AGENCY = 'insurance_agency', 'Insurance Agency'
        INSURANCE_CARRIER = 'insurance_carrier', 'Insurance Carrier'
        FREIGHT_BROKER = 'freight_broker', 'Freight Broker'
        SHIPPER = 'shipper', 'Shipper'
        VENDOR = 'vendor', 'Vendor'
        PLATFORM_ADMIN = 'platform_admin', 'Platform Admin'
    
    class Status(models.TextChoices):
        SHADOW = 'shadow', 'Shadow (Unclaimed)'          # Pre-created by another tenant
        ACTIVE = 'active', 'Active'                      # Claimed/paying
        TRIAL = 'trial', 'Trial'                         # Free trial period
        SUSPENDED = 'suspended', 'Suspended'             # Payment failed
        CANCELLED = 'cancelled', 'Cancelled'             # Churned
    
    class SubscriptionTier(models.TextChoices):
        FREE = 'free', 'Free'
        PRO = 'pro', 'Pro'
        ENTERPRISE = 'enterprise', 'Enterprise'
    
    name = models.CharField(max_length=255)
    slug = models.SlugField(max_length=100)
    
    tenant_type = models.CharField(
        max_length=30,
        choices=TenantType.choices,
        db_index=True
    )
    
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.ACTIVE,
        db_index=True
    )
    
    # Hierarchy (for agency networks)
    parent = models.ForeignKey(
        'self',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='children'
    )
    
    subscription_tier = models.CharField(
        max_length=20,
        choices=SubscriptionTier.choices,
        default=SubscriptionTier.FREE
    )
    
    # Link to Network Entity (if this tenant IS a business)
    entity = models.ForeignKey(
        'network.Entity',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='tenant_account'
    )
    
    # Shadow Tenant Tracking
    created_by_tenant = models.ForeignKey(
        'self',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='created_shadow_tenants',
        help_text="If Shadow: The tenant that created this"
    )
    
    class Meta:
        db_table = 'identity_tenant'
        constraints = [
            models.UniqueConstraint(
                fields=['slug'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_active_tenant_slug'
            )
        ]


class TenantSettings(BaseModel):
    """Tenant-specific configuration."""
    tenant = models.OneToOneField(Tenant, on_delete=models.CASCADE, related_name='settings')
    
    features = models.JSONField(default=dict)
    # {"ai_extraction": true, "carrier_portal": false}
    
    limits = models.JSONField(default=dict)
    # {"max_users": 10, "max_policies": 1000, "api_calls_per_day": 10000}
    
    branding = models.JSONField(default=dict)
    # {"logo_url": "...", "primary_color": "#..."}
    
    class Meta:
        db_table = 'identity_tenant_settings'
```

### 6.3. Membership

```python
"""
identity/models/membership.py

Membership: Links User to Tenant with role.

Note: Uses SoftDeleteMixin for audit trail, but also has is_active
for temporary suspension (different from deletion).
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class Membership(BaseModel, SoftDeleteMixin):
    """
    Links User to Tenant with role-based access.
    
    A user can belong to multiple tenants.
    """
    class Role(models.TextChoices):
        OWNER = 'owner', 'Owner'
        ADMIN = 'admin', 'Admin'
        AGENT = 'agent', 'Agent'
        VIEWER = 'viewer', 'Viewer'
    
    user = models.ForeignKey('identity.User', on_delete=models.CASCADE, related_name='memberships')
    tenant = models.ForeignKey('identity.Tenant', on_delete=models.CASCADE, related_name='memberships')
    
    role = models.CharField(max_length=20, choices=Role.choices, default=Role.AGENT)
    
    # Temporary suspension (different from soft delete)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        db_table = 'identity_membership'
        constraints = [
            models.UniqueConstraint(
                fields=['user', 'tenant'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_active_membership'
            )
        ]
```

---

## 7. Access Control Engine (The Security Core)

### 7.1. Resource (access/models/resource.py)

```python
"""
Resource: ACL anchor for permission checks.

Every sensitive model (Policy, Driver, Document) MUST have a Resource.

Lazy Creation Pattern:
- Resources are created on-demand (not in model save)
- Only created when first grant is issued OR model is marked non-private
- This avoids write amplification for bulk operations
"""
from django.db import models
from common.models.base import BaseModel


class Resource(BaseModel):
    """
    Universal ACL anchor.
    
    Design:
    - Every sensitive object links to Resource
    - Permissions checked via AccessGrant
    - Owner tenant has implicit full access
    """
    class ResourceType(models.TextChoices):
        POLICY = 'policy', 'Policy'
        POLICY_TERM = 'policy_term', 'Policy Term'
        DOCUMENT = 'document', 'Document'
        VEHICLE = 'vehicle', 'Vehicle'
        DRIVER = 'driver', 'Driver'
        ENTITY = 'entity', 'Entity'
        DEAL = 'deal', 'Deal'
    
    resource_type = models.CharField(max_length=30, choices=ResourceType.choices, db_index=True)
    
    # Who owns this resource (implicit full access)
    owner_tenant = models.ForeignKey(
        'identity.Tenant',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='owned_resources'
    )
    
    # Public access (rare - only for truly public data)
    is_public = models.BooleanField(default=False)
    
    class Meta:
        db_table = 'access_resource'


def get_or_create_resource(obj, resource_type: str, owner_tenant=None):
    """
    Lazy resource creation helper.
    
    Usage:
        resource = get_or_create_resource(policy, 'policy', policy.broker_tenant)
        policy.resource = resource
        policy.save(update_fields=['resource'])
    """
    if hasattr(obj, 'resource') and obj.resource:
        return obj.resource
    
    resource = Resource.objects.create(
        resource_type=resource_type,
        owner_tenant=owner_tenant,
    )
    return resource
```

### 7.2. AccessGrant (with Provenance)

```python
"""
access/models/grant.py

AccessGrant: Permission record with provenance tracking.

Provenance (source_type) tracks WHY access exists:
- Shadow_Creator: Agency created shadow tenant, gets admin access
- Visibility: Document marked public, auto-granted to connections
- Workflow: Submission workflow granted carrier access to quote
- Manual: User explicitly shared

This enables safe bulk revocation when connections end.
"""
from django.db import models
from common.models.base import BaseModel


class AccessGrant(BaseModel):
    """
    Permission grant for a Resource to a Tenant.
    
    With provenance tracking for safe revocation.
    """
    class Permission(models.TextChoices):
        VIEW = 'view', 'View'
        DOWNLOAD = 'download', 'Download'
        EDIT = 'edit', 'Edit'
        ADMIN = 'admin', 'Admin'  # Can grant to others
    
    class SourceType(models.TextChoices):
        MANUAL = 'manual', 'Manual Share'
        SHADOW_CREATOR = 'shadow_creator', 'Shadow Tenant Creator'
        VISIBILITY = 'visibility', 'Document Visibility Rule'
        WORKFLOW = 'workflow', 'Workflow Grant'
        CONNECTION = 'connection', 'Connection-Based'
    
    resource = models.ForeignKey(Resource, on_delete=models.CASCADE, related_name='grants')
    tenant = models.ForeignKey('identity.Tenant', on_delete=models.CASCADE, related_name='access_grants')
    
    permission = models.CharField(max_length=20, choices=Permission.choices)
    
    # Provenance
    source_type = models.CharField(max_length=30, choices=SourceType.choices)
    source_id = models.UUIDField(null=True, blank=True)
    # ID of Connection, Workflow, Document that triggered this grant
    
    # Who/Why
    granted_by = models.ForeignKey('identity.User', null=True, on_delete=models.SET_NULL)
    reason = models.CharField(max_length=255, blank=True)
    
    # Expiration
    expires_at = models.DateTimeField(null=True, blank=True, db_index=True)
    
    # Revocation
    revoked_at = models.DateTimeField(null=True, blank=True)
    revoked_by = models.ForeignKey(
        'identity.User',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='+'
    )
    
    class Meta:
        db_table = 'access_grant'
        constraints = [
            models.UniqueConstraint(
                fields=['resource', 'tenant'],
                condition=models.Q(revoked_at__isnull=True),
                name='uniq_active_grant'
            )
        ]
        indexes = [
            models.Index(fields=['tenant', 'resource']),
            models.Index(fields=['expires_at'], condition=models.Q(expires_at__isnull=False)),
        ]
```

### 7.3. RelationshipEdge (Materialized Graph)

```python
"""
access/models/relationship_edge.py

RelationshipEdge: Materialized graph for rapid "who can see what" lookup.

Purpose:
- Flattens complex graph queries
- Enables fast "accessible_to(tenant)" queries
- Updated via domain events (connection.created, entity.claimed)

Example:
- Agency XYZ brokers for ABC Trucking
- ABC Trucking has policy with Progressive
- Edge: (tenant=Agency XYZ, entity=ABC Trucking, type='brokers_for')
- Edge: (tenant=Agency XYZ, entity=Progressive, type='works_with')
"""
from django.db import models
from common.models.base import BaseModel


class RelationshipEdge(BaseModel):
    """
    Materialized relationship for performance.
    
    Maintained by:
    - network.services.connection (on Connection create/delete)
    - identity.services.shadow (on claim)
    - insurance.services.policy (on policy bind)
    """
    class EdgeType(models.TextChoices):
        OWNS = 'owns', 'Owns'                     # Tenant owns Entity (claimed)
        BROKERS_FOR = 'brokers_for', 'Brokers For'  # Agency brokers for Fleet
        MONITORS = 'monitors', 'Monitors'         # Freight Broker monitors Fleet
        INSURES = 'insures', 'Insures'            # Insurance Carrier insures Fleet
        WORKS_WITH = 'works_with', 'Works With'   # Generic connection
    
    tenant = models.ForeignKey('identity.Tenant', on_delete=models.CASCADE, related_name='relationship_edges')
    entity = models.ForeignKey('network.Entity', on_delete=models.CASCADE, related_name='relationship_edges')
    
    edge_type = models.CharField(max_length=30, choices=EdgeType.choices)
    
    # Effective period
    effective_date = models.DateField(null=True, blank=True)
    end_date = models.DateField(null=True, blank=True)
    
    # What triggered this edge
    source_connection = models.ForeignKey(
        'network.Connection',
        null=True,
        blank=True,
        on_delete=models.SET_NULL
    )
    source_policy = models.ForeignKey(
        'insurance.Policy',
        null=True,
        blank=True,
        on_delete=models.SET_NULL
    )
    
    class Meta:
        db_table = 'access_relationship_edge'
        constraints = [
            models.UniqueConstraint(
                fields=['tenant', 'entity', 'edge_type'],
                name='uniq_relationship_edge'
            )
        ]
        indexes = [
            models.Index(fields=['tenant', 'edge_type']),
            models.Index(fields=['entity', 'edge_type']),
        ]
```

### 7.4. PermissionService (access/services/permissions.py)

```python
"""
access/services/permissions.py

Central permission checking service.
"""
from access.models import Resource, AccessGrant
from django.utils import timezone
from django.db import models


class PermissionService:
    """
    Central ACL enforcement.
    
    Usage:
        if not PermissionService.can_access(policy.resource, tenant, 'view'):
            raise PermissionDenied()
    """
    
    PERMISSION_HIERARCHY = {
        'view': 0,
        'download': 1,
        'edit': 2,
        'admin': 3,
    }
    
    @classmethod
    def can_access(cls, resource: Resource, tenant, required_permission: str) -> bool:
        """
        Check if tenant can access resource with required permission.
        
        Returns True if:
        - Tenant owns the resource, OR
        - Resource is public and permission is 'view', OR
        - Valid AccessGrant exists
        """
        if not resource or not tenant:
            return False
        
        # Owner always has full access
        if resource.owner_tenant_id == tenant.id:
            return True
        
        # Public resources allow view
        if resource.is_public and required_permission == 'view':
            return True
        
        # Check grants
        now = timezone.now()
        grant = AccessGrant.objects.filter(
            resource=resource,
            tenant=tenant,
            revoked_at__isnull=True,
        ).filter(
            models.Q(expires_at__isnull=True) | models.Q(expires_at__gt=now)
        ).first()
        
        if not grant:
            return False
        
        # Check permission hierarchy
        required_level = cls.PERMISSION_HIERARCHY.get(required_permission, 999)
        granted_level = cls.PERMISSION_HIERARCHY.get(grant.permission, -1)
        
        return granted_level >= required_level
    
    @classmethod
    def grant_access(
        cls,
        resource: Resource,
        tenant,
        permission: str,
        source_type: str,
        granted_by=None,
        reason: str = '',
        expires_at=None,
        source_id=None
    ):
        """
        Grant access to a tenant.
        
        Args:
            resource: The resource to grant access to
            tenant: The tenant receiving access
            permission: Permission level (view, download, edit, admin)
            source_type: Why this grant exists (manual, workflow, etc.)
            granted_by: User who granted (if manual)
            reason: Human-readable reason
            expires_at: Expiration datetime
            source_id: ID of object that triggered grant (Connection, Workflow, etc.)
        """
        grant, created = AccessGrant.objects.update_or_create(
            resource=resource,
            tenant=tenant,
            revoked_at__isnull=True,
            defaults={
                'permission': permission,
                'source_type': source_type,
                'source_id': source_id,
                'granted_by': granted_by,
                'reason': reason,
                'expires_at': expires_at,
            }
        )
        
        # Emit domain event
        from common.events.outbox import emit_event
        emit_event(
            event_type='access.granted',
            aggregate_type='Resource',
            aggregate_id=resource.id,
            payload={
                'tenant_id': str(tenant.id),
                'permission': permission,
                'source_type': source_type,
            }
        )
        
        return grant
    
    @classmethod
    def revoke_access(cls, resource: Resource, tenant, revoked_by):
        """Revoke access from a tenant."""
        grants = AccessGrant.objects.filter(
            resource=resource,
            tenant=tenant,
            revoked_at__isnull=True,
        )
        
        now = timezone.now()
        grants.update(
            revoked_at=now,
            revoked_by=revoked_by,
        )
        
        # Emit domain event
        from common.events.outbox import emit_event
        emit_event(
            event_type='access.revoked',
            aggregate_type='Resource',
            aggregate_id=resource.id,
            payload={
                'tenant_id': str(tenant.id),
                'revoked_by': str(revoked_by.id) if revoked_by else None,
            }
        )
    
    @classmethod
    def bulk_revoke_by_source(cls, source_type: str, source_id, revoked_by=None):
        """
        Revoke all grants created by a specific source.
        
        Example:
            # When a Connection ends, revoke all grants it created
            PermissionService.bulk_revoke_by_source('connection', connection.id)
        """
        now = timezone.now()
        AccessGrant.objects.filter(
            source_type=source_type,
            source_id=source_id,
            revoked_at__isnull=True,
        ).update(
            revoked_at=now,
            revoked_by=revoked_by,
        )
```

### 7.5. AccessControlledManager

```python
"""
common/managers/scoped.py (continued)

AccessControlledManager: Filters queryset by Resource permissions.
"""
from django.db import models
from common.context.tenant import get_current_tenant, TenantContextRequired


class AccessControlledManager(models.Manager):
    """
    Manager that filters by Resource permissions.
    
    For models that:
    - Have resource FK
    - Are global (not tenant-scoped)
    - Need permission checking
    
    Usage:
        class Policy(BaseModel):
            resource = OneToOneField(Resource)
            objects = AccessControlledManager()
            all_objects = UnscopedManager()
    
    Query:
        Policy.objects.all()  # Returns only accessible to current tenant
    """
    def get_queryset(self):
        from django.db.models import Q, Exists, OuterRef
        from access.models import AccessGrant
        
        tenant = get_current_tenant()
        if not tenant:
            raise TenantContextRequired(
                f"Cannot query {self.model.__name__} without tenant context."
            )
        
        qs = super().get_queryset()
        
        # Optimization: Subquery for grants
        grants = AccessGrant.objects.filter(
            resource_id=OuterRef('resource_id'),
            tenant=tenant,
            revoked_at__isnull=True,
        ).filter(
            Q(expires_at__isnull=True) | Q(expires_at__gt=timezone.now())
        )
        
        # Filter: Owner OR has grant
        qs = qs.filter(
            Q(resource__owner_tenant=tenant) | Q(Exists(grants))
        )
        
        # Exclude soft-deleted
        if hasattr(self.model, 'deleted_at'):
            qs = qs.filter(deleted_at__isnull=True)
        
        return qs
```

---

## 8. Network Domain (The Trust Layer)

### 8.1. Entity (with canonical_entity)

```python
"""
network/models/entity.py

Entity: The Golden Record.

Design:
- Global (not tenant-scoped)
- Canonical (holds verified public data)
- Multi-role (can have multiple profiles)
- Mergeable (via canonical_entity pointer)

Who Edits (Golden Record Sources):
- System (FMCSA sync, verified extraction)
- The Brain (AI Extraction & Enrichment) — see `core/brain/`
- Claimed owner (via EntityChangeProposal auto-apply)
- Others (via EntityChangeProposal review queue)
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class Entity(BaseModel, SoftDeleteMixin):
    """
    Business entity in the network.
    
    Merge Strategy:
    - On merge: Set canonical_entity to survivor, status='merged'
    - Reads resolve via resolve() method
    - Avoids rewriting all FKs across system
    """
    class VerificationStatus(models.TextChoices):
        UNVERIFIED = 'unverified', 'Unverified'
        PROVISIONAL = 'provisional', 'Provisional'  # User-created, unmatched
        VERIFIED = 'verified', 'Verified'            # Matched to govt data
        CLAIMED = 'claimed', 'Claimed'               # Owned by a tenant
        MERGED = 'merged', 'Merged'                  # Replaced by canonical_entity
    
    name = models.CharField(max_length=255, db_index=True)
    dba_name = models.CharField(max_length=255, blank=True)
    
    verification_status = models.CharField(
        max_length=20,
        choices=VerificationStatus.choices,
        default=VerificationStatus.UNVERIFIED,
        db_index=True
    )
    
    # Merge resolution: Points to survivor
    canonical_entity = models.ForeignKey(
        'self',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='merged_duplicates',
        help_text="If merged, points to the canonical entity"
    )
    
    # Claiming
    claimed_by = models.ForeignKey(
        'identity.Tenant',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='claimed_entities'
    )
    claimed_at = models.DateTimeField(null=True, blank=True)
    
    # Canonical contact (public)
    email = models.EmailField(blank=True)
    phone = models.CharField(max_length=50, blank=True)
    
    # Address (embedded for queryability)
    address_street = models.CharField(max_length=255, blank=True)
    address_city = models.CharField(max_length=100, blank=True)
    address_state = models.CharField(max_length=2, blank=True, db_index=True)
    address_zip = models.CharField(max_length=10, blank=True)
    
    class Meta:
        db_table = 'network_entity'
        indexes = [
            models.Index(fields=['name']),
            models.Index(fields=['verification_status']),
            models.Index(fields=['canonical_entity']),
        ]
    
    def resolve(self):
        """
        Resolve to canonical entity.
        
        Handles chains: A→B→C resolves to C.
        """
        if not self.canonical_entity_id:
            return self
        
        # Follow chain (max 10 hops to prevent infinite loop)
        current = self
        for _ in range(10):
            if not current.canonical_entity_id:
                return current
            current = current.canonical_entity
        
        # If we hit max hops, something is wrong
        raise ValueError(f"Entity {self.id} has circular canonical_entity chain")
    
    @classmethod
    def resolve_id(cls, entity_id):
        """
        Resolve entity ID to canonical ID.
        
        Usage:
            canonical_id = Entity.resolve_id(some_id)
            policies = Policy.objects.filter(insured_id=canonical_id)
        """
        # Recursive CTE for chain resolution
        from django.db import connection
        with connection.cursor() as cursor:
            cursor.execute("""
                WITH RECURSIVE entity_chain AS (
                    SELECT id, canonical_entity_id, 0 as depth
                    FROM network_entity
                    WHERE id = %s
                    
                    UNION ALL
                    
                    SELECT e.id, e.canonical_entity_id, ec.depth + 1
                    FROM network_entity e
                    JOIN entity_chain ec ON e.id = ec.canonical_entity_id
                    WHERE ec.depth < 10
                )
                SELECT id FROM entity_chain
                WHERE canonical_entity_id IS NULL
                LIMIT 1
            """, [entity_id])
            
            row = cursor.fetchone()
            return row[0] if row else entity_id
    
    @classmethod
    def resolve_many(cls, entity_ids):
        """
        Batch resolve entity IDs to canonical IDs.
        
        Returns: dict mapping original_id -> canonical_id
        
        Usage:
            ids = [id1, id2, id3]
            resolved = Entity.resolve_many(ids)
            # {id1: canonical_id1, id2: canonical_id2, ...}
        """
        # Recursive CTE for batch resolution
        from django.db import connection
        with connection.cursor() as cursor:
            cursor.execute("""
                WITH RECURSIVE entity_chain AS (
                    SELECT id as original_id, id, canonical_entity_id, 0 as depth
                    FROM network_entity
                    WHERE id = ANY(%s)
                    
                    UNION ALL
                    
                    SELECT ec.original_id, e.id, e.canonical_entity_id, ec.depth + 1
                    FROM network_entity e
                    JOIN entity_chain ec ON e.id = ec.canonical_entity_id
                    WHERE ec.depth < 10
                )
                SELECT original_id, id as canonical_id
                FROM entity_chain
                WHERE canonical_entity_id IS NULL
            """, [list(entity_ids)])
            
            return dict(cursor.fetchall())
```

### 8.2. EntityIdentifier (with Normalization)

```python
"""
network/models/identifier.py

EntityIdentifier: First-class, normalized, globally unique identifiers.

Normalization Rules:
- DOT/MC: Strip non-digits, remove leading zeros
- NAIC: Strip non-digits
- FEIN: Strip non-digits
- Others: Uppercase, strip whitespace/hyphens

This prevents "123456" vs "0123456" vs "123-456" becoming 3 carriers.
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class EntityIdentifier(BaseModel, SoftDeleteMixin):
    """Global identifiers for an Entity."""
    
    class IdType(models.TextChoices):
        DOT = 'dot', 'USDOT Number'
        MC = 'mc', 'MC Number'
        NAIC = 'naic', 'NAIC Code'
        FEIN = 'fein', 'Federal EIN'
        NPN = 'npn', 'National Producer Number'
        SCAC = 'scac', 'SCAC Code'
        CUSTOM = 'custom', 'Custom'
    
    entity = models.ForeignKey(
        'network.Entity',
        on_delete=models.CASCADE,
        related_name='identifiers'
    )
    
    id_type = models.CharField(max_length=20, choices=IdType.choices, db_index=True)
    
    # Raw value (for display)
    value = models.CharField(max_length=50)
    
    # Normalized value (for matching/uniqueness)
    normalized_value = models.CharField(max_length=50, db_index=True)
    
    is_primary = models.BooleanField(default=False)
    
    # Source tracking
    source = models.CharField(max_length=50)
    # 'fmcsa_sync', 'user_entry', 'document_extraction', 'api_import'
    confidence = models.FloatField(default=1.0)
    verified_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        db_table = 'network_entity_identifier'
        constraints = [
            # Global uniqueness on normalized value
            models.UniqueConstraint(
                fields=['id_type', 'normalized_value'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_active_identifier'
            ),
            # Only one primary per entity per type
            models.UniqueConstraint(
                fields=['entity', 'id_type'],
                condition=models.Q(is_primary=True, deleted_at__isnull=True),
                name='uniq_primary_per_type'
            ),
        ]
        indexes = [
            models.Index(fields=['id_type', 'normalized_value']),
            models.Index(fields=['entity', 'id_type']),
        ]
    
    def save(self, *args, **kwargs):
        # Auto-compute normalized value
        self.normalized_value = self.normalize(self.id_type, self.value)
        super().save(*args, **kwargs)
    
    @staticmethod
    def normalize(id_type: str, value: str) -> str:
        """Normalize identifier for matching."""
        if not value:
            return ''
        
        value = value.strip()
        
        if id_type in ('dot', 'mc'):
            # Keep only digits, strip leading zeros
            digits = ''.join(c for c in value if c.isdigit())
            return digits.lstrip('0') or '0'
        
        elif id_type == 'naic':
            # Keep only digits
            return ''.join(c for c in value if c.isdigit())
        
        elif id_type == 'fein':
            # Keep only digits
            digits = ''.join(c for c in value if c.isdigit())
            return digits
        
        else:
            # Default: uppercase, strip whitespace/hyphens
            return value.upper().replace(' ', '').replace('-', '')
    
    @classmethod
    def find_entity(cls, id_type: str, value: str):
        """
        Find entity by identifier (with normalization and merge resolution).
        
        Usage:
            entity = EntityIdentifier.find_entity('dot', '123456')
        """
        normalized = cls.normalize(id_type, value)
        identifier = cls.objects.filter(
            id_type=id_type,
            normalized_value=normalized,
            deleted_at__isnull=True,
        ).select_related('entity').first()
        
        if identifier:
            return identifier.entity.resolve()  # Resolve if merged
        return None
```

---

## 9. Insurance Domain (Contracts)

### 9.1. Policy (with Resource + Unique Constraint)

```python
"""
insurance/models/policy.py

Policy: Insurance contract between Entities.

Access Control:
- Owner: broker_tenant (the insurance_agency that brokered it)
- Sharing: Via AccessGrant to insurance_carrier, insured's claimed tenant, etc.

Uniqueness:
- Policy level: (insured, insurer, policy_number, policy_type)
- Term level: (policy, effective_date, revision)
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class Policy(BaseModel, SoftDeleteMixin):
    """
    Insurance contract.
    
    Snapshots:
    - insured_snapshot: Entity data at bind time
    - insurer_snapshot: Entity data at bind time
    - Immutable legal record
    """
    class PolicyType(models.TextChoices):
        PACKAGE = 'package', 'Package'
        AUTO = 'auto', 'Auto Liability'
        CARGO = 'cargo', 'Motor Truck Cargo'
        GL = 'gl', 'General Liability'
        WC = 'wc', 'Workers Comp'
        UMBRELLA = 'umbrella', 'Umbrella'
        PD = 'pd', 'Physical Damage'
    
    # ACL anchor (lazy creation via service layer)
    resource = models.OneToOneField(
        'access.Resource',
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name='policy'
    )
    
    # Parties (Network Entities)
    insured = models.ForeignKey(
        'network.Entity',
        on_delete=models.PROTECT,
        related_name='policies_as_insured'
    )
    insurer = models.ForeignKey(
        'network.Entity',
        on_delete=models.PROTECT,
        related_name='policies_as_insurer'
    )
    
    # Who brokered it (Resource owner)
    broker_tenant = models.ForeignKey(
        'identity.Tenant',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='brokered_policies'
    )
    
    policy_number = models.CharField(max_length=100, db_index=True)
    policy_type = models.CharField(max_length=20, choices=PolicyType.choices)
    
    # SNAPSHOTS (Immutable - captured at bind)
    insured_snapshot = models.JSONField(default=dict)
    # {"name": "ABC Trucking", "dot": "123456", "address": {...}}
    insurer_snapshot = models.JSONField(default=dict)
    # {"name": "Progressive", "naic": "12345"}
    
    class Meta:
        db_table = 'insurance_policy'
        constraints = [
            # Policy-level uniqueness
            models.UniqueConstraint(
                fields=['insured', 'insurer', 'policy_number', 'policy_type'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_active_policy'
            )
        ]
        indexes = [
            models.Index(fields=['insured', 'policy_type']),
            models.Index(fields=['insurer', 'policy_number']),
        ]


class PolicyTerm(BaseModel, SoftDeleteMixin):
    """
    A specific term/period of a Policy.
    
    Uniqueness includes revision for mid-term rewrites/endorsements.
    """
    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        ACTIVE = 'active', 'Active'
        EXPIRED = 'expired', 'Expired'
        CANCELLED = 'cancelled', 'Cancelled'
    
    # ACL anchor (inherits from policy but can have separate grants)
    resource = models.OneToOneField(
        'access.Resource',
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name='policy_term'
    )
    
    policy = models.ForeignKey(Policy, on_delete=models.CASCADE, related_name='terms')
    
    effective_date = models.DateField(db_index=True)
    expiration_date = models.DateField(db_index=True)
    
    # Revision for mid-term rewrites
    revision = models.IntegerField(default=1)
    
    # Term-specific policy number (may differ for renewals)
    term_policy_number = models.CharField(max_length=100)
    
    total_premium_cents = models.BigIntegerField(default=0)
    
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.PENDING, db_index=True)
    
    # Snapshot timing
    bound_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        db_table = 'insurance_policy_term'
        constraints = [
            models.UniqueConstraint(
                fields=['policy', 'effective_date', 'revision'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_active_policy_term'
            )
        ]
        indexes = [
            models.Index(fields=['effective_date', 'expiration_date']),
            models.Index(fields=['status', 'expiration_date']),
        ]
    
    def bind(self, user=None):
        """
        Bind the policy term (activate).
        Captures snapshots at this moment.
        """
        from django.utils import timezone
        self.bound_at = timezone.now()
        self.insured_snapshot = self.policy.insured.to_snapshot()
        self.insurer_snapshot = self.policy.insurer.to_snapshot()
        self.status = 'active'
        self.updated_by = user
        self.save()


class PolicyCoverage(BaseModel, SoftDeleteMixin):
    """Specific coverage line within a PolicyTerm."""
    
    class CoverageType(models.TextChoices):
        AUTO_LIABILITY = 'auto', 'Auto Liability'
        GENERAL_LIABILITY = 'gl', 'General Liability'
        CARGO = 'cargo', 'Motor Truck Cargo'
        PHYSICAL_DAMAGE = 'pd', 'Physical Damage'
        WORKERS_COMP = 'wc', 'Workers Compensation'
        UMBRELLA = 'umbrella', 'Umbrella/Excess'
        UNINSURED_MOTORIST = 'um', 'Uninsured Motorist'
        HIRED_AUTO = 'hired', 'Hired Auto'
        NON_OWNED = 'non_owned', 'Non-Owned Auto'
    
    policy_term = models.ForeignKey(PolicyTerm, on_delete=models.CASCADE, related_name='coverages')
    
    coverage_type = models.CharField(max_length=30, choices=CoverageType.choices)
    
    # Limits (in cents)
    limit_per_occurrence_cents = models.BigIntegerField(null=True, blank=True)
    limit_aggregate_cents = models.BigIntegerField(null=True, blank=True)
    combined_single_limit_cents = models.BigIntegerField(null=True, blank=True)
    
    # Deductibles
    deductible_cents = models.BigIntegerField(null=True, blank=True)
    
    # Special conditions
    special_conditions = models.JSONField(default=dict)
    # {"ice_cream_deductible_cents": 500000}
    
    class Meta:
        db_table = 'insurance_policy_coverage'
        constraints = [
            models.UniqueConstraint(
                fields=['policy_term', 'coverage_type'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_coverage_per_term'
            )
        ]
```

---

## 10. The Shadow Tenant Engine (Viral Growth)

### 10.1. How It Works

**Scenario:** Agency XYZ enters a lead for "Bob's Trucking" (not on platform).

**Step 1:** System checks for existing Entity by DOT.
- If none exists: Create `Entity(name="Bob's Trucking", dot="123456", status='provisional')`.

**Step 2:** Create Shadow Tenant.
```python
shadow_tenant = Tenant.objects.create(
    name="Bob's Trucking",
    slug=generate_slug("bobs-trucking"),
    tenant_type='motor_carrier',
    status='shadow',
    entity=entity,
    created_by_tenant=agency_xyz_tenant,
)
```

**Step 3:** Link Entity to Shadow Tenant.
```python
entity.claimed_by = shadow_tenant
entity.save()
```

**Step 4:** Grant Admin Access to Creator.
```python
# Create Resource for Entity
entity_resource = Resource.objects.create(
    resource_type='entity',
    owner_tenant=shadow_tenant,
)

# Grant admin access to Agency XYZ
AccessGrant.objects.create(
    resource=entity_resource,
    tenant=agency_xyz_tenant,
    permission='admin',
    source_type='shadow_creator',
    granted_by=creating_user,
    reason="Created shadow tenant",
)
```

**Step 5 (The Claim):** Agency sends invite. Bob clicks link.
```python
# Bob creates User account
user = User.objects.create(email='bob@bobstrucking.com', ...)

# System finds shadow_tenant by invitation token
shadow_tenant.status = 'active'
shadow_tenant.save()

# Add Bob as Owner
Membership.objects.create(
    user=user,
    tenant=shadow_tenant,
    role='owner',
)
```

**Result:** Bob logs in and sees all the data Agency XYZ entered (drivers, vehicles, policies).

### 10.2. Shadow Tenant Service

```python
"""
identity/services/shadow.py

Shadow Tenant creation and claiming logic.
"""
from identity.models import Tenant, Membership
from network.models import Entity
from access.models import Resource
from access.services.permissions import PermissionService


class ShadowTenantService:
    """Manages shadow tenant creation and claiming."""
    
    @classmethod
    def create_shadow_for_entity(cls, entity: Entity, creator_tenant, creator_user) -> Tenant:
        """
        Create shadow tenant for an unclaimed entity.
        
        Returns: (shadow_tenant, access_granted)
        """
        # Create shadow tenant
        shadow = Tenant.objects.create(
            name=entity.name,
            slug=cls._generate_shadow_slug(entity),
            tenant_type=cls._infer_type(entity),
            status='shadow',
            entity=entity,
            created_by_tenant=creator_tenant,
        )
        
        # Link entity to shadow
        entity.claimed_by = shadow
        entity.save(update_fields=['claimed_by', 'updated_at'])
        
        # Grant admin access to creator
        entity_resource = Resource.objects.create(
            resource_type='entity',
            owner_tenant=shadow,
        )
        
        PermissionService.grant_access(
            resource=entity_resource,
            tenant=creator_tenant,
            permission='admin',
            source_type='shadow_creator',
            granted_by=creator_user,
            reason="Created shadow tenant",
        )
        
        return shadow
    
    @classmethod
    def claim_shadow(cls, shadow_tenant: Tenant, claiming_user):
        """
        User claims a shadow tenant.
        
        Flow:
        1. Verify claiming_user has valid claim (email domain match, etc.)
        2. Activate shadow tenant
        3. Add claiming_user as owner
        4. Optionally revoke creator's admin access (configurable)
        """
        if shadow_tenant.status != 'shadow':
            raise ValueError("Can only claim shadow tenants")
        
        # Activate
        shadow_tenant.status = 'active'
        shadow_tenant.save(update_fields=['status', 'updated_at'])
        
        # Add claimer as owner
        Membership.objects.create(
            user=claiming_user,
            tenant=shadow_tenant,
            role='owner',
        )
        
        # Update entity status
        if shadow_tenant.entity:
            shadow_tenant.entity.verification_status = 'claimed'
            shadow_tenant.entity.claimed_at = timezone.now()
            shadow_tenant.entity.save(update_fields=['verification_status', 'claimed_at', 'updated_at'])
        
        return shadow_tenant
```

---

## 11. The Submission Workflow (Quote Request)

### 11.1. Submission Model

```python
"""
insurance/models/submission.py

Submission: Request for quote from a carrier.

This is a first-class workflow that triggers AccessGrant propagation.
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class Submission(BaseModel, SoftDeleteMixin):
    """
    A quote request sent to an insurance_carrier.
    
    Lifecycle:
    1. Agency creates submission
    2. System creates AccessGrant for carrier on policy + drivers + loss runs
    3. Carrier reviews and returns quote
    4. On quote accept: Bind
    5. On reject/expire: Revoke grants
    """
    class Status(models.TextChoices):
        DRAFT = 'draft', 'Draft'
        SUBMITTED = 'submitted', 'Submitted'
        QUOTED = 'quoted', 'Quoted'
        DECLINED = 'declined', 'Declined'
        ACCEPTED = 'accepted', 'Accepted'
        EXPIRED = 'expired', 'Expired'
    
    # Workflow owner
    agency_tenant = models.ForeignKey(
        'identity.Tenant',
        on_delete=models.CASCADE,
        related_name='submissions_as_agency'
    )
    
    # Target carrier
    carrier_tenant = models.ForeignKey(
        'identity.Tenant',
        on_delete=models.CASCADE,
        related_name='submissions_as_carrier'
    )
    
    # Subject
    policy = models.ForeignKey('insurance.Policy', on_delete=models.CASCADE, related_name='submissions')
    
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT, db_index=True)
    
    submitted_at = models.DateTimeField(null=True, blank=True)
    expires_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        db_table = 'insurance_submission'
    
    def submit(self, user):
        """
        Submit to carrier.
        
        Triggers:
        - Create AccessGrant for carrier_tenant on policy.resource
        - Create AccessGrant for carrier_tenant on related documents
        - Emit domain event
        """
        from django.utils import timezone
        from access.services.permissions import PermissionService
        from access.services.auto_grant import AutoGrantService
        
        self.status = 'submitted'
        self.submitted_at = timezone.now()
        self.expires_at = timezone.now() + timedelta(days=30)
        self.save()
        
        # Grant carrier access to policy
        PermissionService.grant_access(
            resource=self.policy.resource,
            tenant=self.carrier_tenant,
            permission='view',
            source_type='workflow',
            source_id=self.id,
            granted_by=user,
            reason=f"Submission {self.id}",
            expires_at=self.expires_at,
        )
        
        # Auto-grant related documents
        AutoGrantService.grant_for_submission(self)
```

### 11.2. AutoGrantService

```python
"""
access/services/auto_grant.py

Auto-Grant Service: Propagates grants based on relationships.

Triggered by:
- Connection created
- Submission submitted
- Document marked 'public_to_connections'
"""
from access.services.permissions import PermissionService
from documents.models import Document
from insurance.models import Submission


class AutoGrantService:
    """Manages automatic access grants based on relationships."""
    
    @classmethod
    def grant_for_submission(cls, submission: Submission):
        """
        Grant carrier access to documents related to policy.
        
        Rules:
        - Documents linked to policy: VIEW
        - Documents linked to insured entity: VIEW (if not PII)
        - Driver data: BLOCKED (PII protection)
        """
        from documents.models import DocumentPolicyLink
        
        # Find all documents linked to this policy
        doc_links = DocumentPolicyLink.objects.filter(
            policy=submission.policy
        ).select_related('document')
        
        for link in doc_links:
            document = link.document
            
            # Skip PII documents
            if document.data_classification == 'pii':
                continue
            
            # Grant view access
            if document.resource:
                PermissionService.grant_access(
                    resource=document.resource,
                    tenant=submission.carrier_tenant,
                    permission='view',
                    source_type='workflow',
                    source_id=submission.id,
                    granted_by=None,
                    reason=f"Submission {submission.id}",
                    expires_at=submission.expires_at,
                )
    
    @classmethod
    def grant_for_connection(cls, connection):
        """
        Grant access based on connection type.
        
        Example:
        - Freight broker monitors fleet
        - Auto-grant VIEW to all COI documents
        - BLOCK access to loss runs, driver lists
        """
        from network.models import Connection
        from documents.models import Document
        
        if connection.relationship_type == 'monitored_by':
            # Freight broker monitoring: Grant COI access only
            documents = Document.objects.filter(
                links__entity=connection.from_entity,
                document_type='coi',
                visibility__in=['public_to_entity', 'shared_to_connections'],
            )
            
            for doc in documents:
                if doc.resource:
                    PermissionService.grant_access(
                        resource=doc.resource,
                        tenant=connection.to_entity.claimed_by,  # Broker's tenant
                        permission='view',
                        source_type='connection',
                        source_id=connection.id,
                        reason=f"Monitoring relationship",
                    )
```

---

## 12. Documents Domain (Separate Link Tables)

### 12.1. Document

```python
"""
documents/models/document.py

Document: Classified, versioned, access-controlled trust artifact.
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class Document(BaseModel, SoftDeleteMixin):
    """
    A classified document.
    
    Visibility drives auto-grants:
    - private: Only owner_tenant
    - shared_to_connections: Auto-grant to connected tenants
    - public_to_entity: Visible to all tenants with entity relationship
    """
    class DocumentType(models.TextChoices):
        DEC_PAGE = 'dec_page', 'Declaration Page'
        COI = 'coi', 'Certificate of Insurance'
        LOSS_RUN = 'loss_run', 'Loss Run'
        DRIVER_LIST = 'driver_list', 'Driver List'
        VEHICLE_SCHEDULE = 'vehicle_schedule', 'Vehicle Schedule'
        LOA = 'loa', 'Letter of Authorization'
        MVR = 'mvr', 'Motor Vehicle Report'
        APPLICATION = 'application', 'Insurance Application'
        ENDORSEMENT = 'endorsement', 'Endorsement'
        OTHER = 'other', 'Other'
    
    class Visibility(models.TextChoices):
        PRIVATE = 'private', 'Private'
        SHARED_TO_CONNECTIONS = 'connections', 'Shared to Connections'
        PUBLIC_TO_ENTITY = 'entity_public', 'Public to Entity'
    
    class DataClassification(models.TextChoices):
        PUBLIC = 'public', 'Public'
        INTERNAL = 'internal', 'Internal'
        CONFIDENTIAL = 'confidential', 'Confidential'
        PII = 'pii', 'Contains PII'
    
    # ACL anchor
    resource = models.OneToOneField(
        'access.Resource',
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name='document'
    )
    
    media = models.ForeignKey(
        'documents.Media',
        on_delete=models.PROTECT,
        related_name='documents'
    )
    
    document_type = models.CharField(max_length=30, choices=DocumentType.choices, db_index=True)
    
    # Ownership
    source_tenant = models.ForeignKey(
        'identity.Tenant',
        null=True,
        on_delete=models.SET_NULL,
        related_name='owned_documents'
    )
    
    visibility = models.CharField(
        max_length=30,
        choices=Visibility.choices,
        default=Visibility.PRIVATE
    )
    
    data_classification = models.CharField(
        max_length=30,
        choices=DataClassification.choices,
        default=DataClassification.INTERNAL,
        db_index=True
    )
    
    # Versioning
    document_chain_id = models.UUIDField(db_index=True)  # Groups all versions
    version = models.IntegerField(default=1)
    is_current = models.BooleanField(default=True)
    supersedes = models.ForeignKey(
        'self',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='superseded_by'
    )
    
    # AI Extraction
    extracted_data = models.JSONField(default=dict)
    extraction_status = models.CharField(
        max_length=20,
        default='pending'
    )
    extraction_confidence = models.FloatField(null=True, blank=True)
    
    class Meta:
        db_table = 'documents_document'
        constraints = [
            # Only one current version per chain
            models.UniqueConstraint(
                fields=['document_chain_id'],
                condition=models.Q(is_current=True, deleted_at__isnull=True),
                name='uniq_current_version'
            )
        ]
```

### 12.2. Separate Link Tables

```python
"""
documents/models/links.py

Separate link tables per target type.

Why not one polymorphic table:
- Easier constraint management
- Clearer semantics
- No "exactly one FK" complexity
"""
from django.db import models
from common.models.base import BaseModel


class DocumentEntityLink(BaseModel):
    """Link document to entity."""
    class LinkType(models.TextChoices):
        INSURED = 'insured', 'Insured'
        INSURER = 'insurer', 'Insurer'
        CERTIFICATE_HOLDER = 'cert_holder', 'Certificate Holder'
        SUBJECT = 'subject', 'Subject'
    
    document = models.ForeignKey('documents.Document', on_delete=models.CASCADE, related_name='entity_links')
    entity = models.ForeignKey('network.Entity', on_delete=models.CASCADE, related_name='document_links')
    link_type = models.CharField(max_length=30, choices=LinkType.choices)
    
    class Meta:
        db_table = 'documents_entity_link'
        constraints = [
            models.UniqueConstraint(
                fields=['document', 'entity', 'link_type'],
                name='uniq_doc_entity_link'
            )
        ]


class DocumentPolicyLink(BaseModel):
    """Link document to policy."""
    class LinkType(models.TextChoices):
        EVIDENCE = 'evidence', 'Evidence'
        DEC_PAGE = 'dec_page', 'Declaration Page'
        LOSS_RUN = 'loss_run', 'Loss Run'
    
    document = models.ForeignKey('documents.Document', on_delete=models.CASCADE, related_name='policy_links')
    policy = models.ForeignKey('insurance.Policy', on_delete=models.CASCADE, related_name='document_links')
    link_type = models.CharField(max_length=30, choices=LinkType.choices)
    
    class Meta:
        db_table = 'documents_policy_link'
        constraints = [
            models.UniqueConstraint(
                fields=['document', 'policy', 'link_type'],
                name='uniq_doc_policy_link'
            )
        ]
```

---

## 13. Assets Domain (with PII Protection)

### 13.1. Driver (PII - Restricted)

```python
"""
assets/models/driver.py

Driver: CONTAINS PII.

Security:
- All reads logged to AuditLog
- Default: Private to entity owner only
- Explicit grant required for others
- Consider field-level encryption for DOB, license_number
"""
from django.db import models
from common.models.base import BaseModel, SoftDeleteMixin


class Driver(BaseModel, SoftDeleteMixin):
    """
    Driver employed by an Entity.
    
    ⚠️ PII: DOB, License Number
    
    Access Rules:
    - Only entity owner can view by default
    - Insurance carriers need explicit grant for underwriting
    - Freight brokers: BLOCKED (no legitimate need)
    """
    class Status(models.TextChoices):
        ACTIVE = 'active', 'Active'
        TERMINATED = 'terminated', 'Terminated'
        LEAVE = 'leave', 'On Leave'
    
    # ACL anchor
    resource = models.OneToOneField(
        'access.Resource',
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name='driver'
    )
    
    entity = models.ForeignKey(
        'network.Entity',
        on_delete=models.CASCADE,
        related_name='drivers'
    )
    
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    
    # PII Fields (consider encryption)
    license_number = models.CharField(max_length=50, db_index=True)
    license_state = models.CharField(max_length=2)
    license_expiration = models.DateField(null=True, blank=True)
    date_of_birth = models.DateField(null=True, blank=True)  # PII
    
    hire_date = models.DateField(null=True, blank=True)
    termination_date = models.DateField(null=True, blank=True)
    
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.ACTIVE, db_index=True)
    
    is_excluded = models.BooleanField(default=False)
    
    class Meta:
        db_table = 'assets_driver'
        constraints = [
            models.UniqueConstraint(
                fields=['entity', 'license_number', 'license_state'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_driver_license'
            )
        ]


class Vehicle(BaseModel, SoftDeleteMixin):
    """Vehicle owned by Entity."""
    class VehicleType(models.TextChoices):
        TRACTOR = 'tractor', 'Tractor'
        TRAILER = 'trailer', 'Trailer'
        BOX_TRUCK = 'box_truck', 'Box Truck'
        OTHER = 'other', 'Other'
    
    # ACL anchor
    resource = models.OneToOneField(
        'access.Resource',
        null=True,
        blank=True,
        on_delete=models.CASCADE,
        related_name='vehicle'
    )
    
    entity = models.ForeignKey('network.Entity', on_delete=models.CASCADE, related_name='vehicles')
    
    vin = models.CharField(max_length=17, db_index=True)
    vehicle_type = models.CharField(max_length=20, choices=VehicleType.choices)
    year = models.IntegerField(null=True, blank=True)
    make = models.CharField(max_length=50, blank=True)
    model = models.CharField(max_length=50, blank=True)
    value_cents = models.BigIntegerField(default=0)
    
    class Meta:
        db_table = 'assets_vehicle'
        constraints = [
            models.UniqueConstraint(
                fields=['entity', 'vin'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_vehicle_vin'
            )
        ]


class Commodity(BaseModel, SoftDeleteMixin):
    """Cargo types hauled by Entity."""
    entity = models.ForeignKey('network.Entity', on_delete=models.CASCADE, related_name='commodities')
    name = models.CharField(max_length=100)
    is_hazmat = models.BooleanField(default=False)
    percentage = models.IntegerField(default=0)
    
    class Meta:
        db_table = 'assets_commodity'
        constraints = [
            models.UniqueConstraint(
                fields=['entity', 'name'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_commodity'
            )
        ]
```

---

## 14. Critical Implementation Rules

### Security (P0 - Non-Negotiable)
1. **Every sensitive model has Resource** - Policy, Driver, Vehicle, Document
2. **ACL checked on every access** - Use `PermissionService.can_access()`
3. **Tenant context via contextvars** - NOT thread locals (async-safe)
4. **Background jobs pass tenant_id explicitly** - Use `TenantContext`
5. **User: NO soft delete** - Use `is_active=False`
6. **PII: Audit all reads** - Driver, MVR documents
7. **TenantScopedManager raises** - NOT `.none()` (fail loud)

### Data Integrity
8. **All unique constraints are partial** - `condition=Q(deleted_at__isnull=True)`
9. **Soft delete is opt-in** - `SoftDeleteMixin`, not on BaseModel
10. **EntityIdentifier: normalized_value** - Unique on normalized, not raw
11. **Entity.canonical_entity** - For merge resolution without FK rewrites
12. **PolicyTerm: includes revision** - For endorsements
13. **Policy: unique at policy level** - (insured, insurer, policy_number, policy_type)

### Architecture
14. **Shadow Tenants for growth** - Pre-create tenants for unclaimed entities
15. **AccessGrant has provenance** - `source_type` tracks why grant exists
16. **RelationshipEdge is materialized** - Updated via events, not computed
17. **Submission triggers auto-grants** - Workflow-based access
18. **Connection != Insurance** - `INSURED_BY` derived from Policy.terms
19. **Documents default PRIVATE** - Explicit sharing required
20. **Domain events via outbox** - Idempotency key, SKIP LOCKED processing

### Brain Integration (AI Orchestration)
21. **BrainTask uses identity.Tenant** - NOT legacy `core.models.Org`
22. **Executor wraps in TenantContext** - Required for TenantScopedManager
23. **Tools receive serialized context** - Prevents Jinja2 lazy-loading crashes
24. **PII redacted from logs** - Execution logs store `data_keys`, not values
25. **Token usage tracked per task** - For billing and rate limiting
26. **Temporal is fire-and-forget** - Brain doesn't wait for workflow completion
27. **trace_id for observability** - Passed to AI services and Temporal

---

## 15. Implementation Roadmap (Security-First)

### Phase 1: Security Foundation (Day 1-2) ⚠️ P0
**Goal:** ACL engine + async-safe tenant context

- [ ] Scaffold Django project structure
- [ ] Implement `common/models/base.py` (BaseModel, SoftDeleteMixin)
- [ ] Implement `common/context/tenant.py` (ContextVars, TenantContext)
- [ ] Implement `common/managers/scoped.py` (TenantScopedManager - raises)
- [ ] Implement `common/middleware/tenant.py`
- [ ] **Implement `access/` (Resource, AccessGrant, PermissionService)**
- [ ] **Implement `access/relationship_edge.py` (Materialized graph)**
- [ ] Test: Verify context in views AND Celery tasks

### Phase 2: Identity + Shadow Tenants (Day 2-3)
**Goal:** Auth + viral growth engine

- [ ] Implement `identity/user.py` (NO soft-delete)
- [ ] Implement `identity/tenant.py` (with Shadow status)
- [ ] Implement `identity/membership.py`
- [ ] Implement `identity/services/shadow.py` (Shadow creation + claiming)
- [ ] Test: Create shadow tenant, claim it, verify access transfer

### Phase 3: Network Core (Day 3-4)
**Goal:** Entity resolution + identifiers

- [ ] Implement `network/entity.py` (with canonical_entity, resolve())
- [ ] Implement `network/identifier.py` (with normalized_value)
- [ ] Implement `network/tenant_view.py` (TenantEntityView)
- [ ] Implement `network/change_proposal.py` (EntityChangeProposal)
- [ ] Test: Create entity, add identifiers, verify DOT lookup

### Phase 4: Documents (Day 4-5)
**Goal:** Trust artifacts with ACL

- [ ] Implement `documents/media.py`
- [ ] Implement `documents/document.py` (with visibility, versioning)
- [ ] Implement `documents/entity_link.py` (Separate link table)
- [ ] Implement `documents/policy_link.py` (Separate link table)
- [ ] Test: Upload doc, link to entity, verify ACL

### Phase 5: Insurance (Day 5-6)
**Goal:** Policy contracts + submission workflow

- [ ] Implement `insurance/policy.py` (Policy with Resource, unique constraint)
- [ ] Implement `insurance/policy_term.py` (with revision, bind())
- [ ] Implement `insurance/coverage.py` (PolicyCoverage)
- [ ] Implement `insurance/submission.py` (Submission with auto-grants)
- [ ] Implement `access/services/auto_grant.py` (AutoGrantService)
- [ ] Test: Create submission, verify carrier gets access

### Phase 6: Assets (Day 6-7)
**Goal:** Fleet data with PII protection

- [ ] Implement `assets/vehicle.py` (with Resource)
- [ ] Implement `assets/driver.py` (PII handling)
- [ ] Implement `assets/commodity.py`
- [ ] Implement `audit/services/audit_service.py` (log_read for Driver)
- [ ] Test: Verify driver access is restricted

### Phase 7: Network Extended (Day 7-8)
- [ ] Implement `network/profiles.py` (CarrierProfile, InsurerProfile)
- [ ] Implement `network/matching.py` (EntityMatch, EntityMerge)
- [ ] Implement `network/verification.py` (VerificationEvidence, ClaimRequest)
- [ ] Implement `network/connection.py` (Connection)

### Phase 8: Events & Intelligence (Day 8-9)
- [ ] Implement `common/events/outbox.py` (DomainEvent with idempotency)
- [ ] Set up outbox worker (SKIP LOCKED)
- [ ] Port `intelligence/` (Ingestion, Extraction, Enrichment)
- [ ] Wire extraction → Entity/Document with ACL

### Phase 8.5: Brain Integration (Day 9-10) ⚠️ CRITICAL
**Prerequisite:** Phase 1 (TenantContext) and Phase 2 (Tenant) MUST be complete.

- [ ] Scaffold `core/brain/` app structure
- [ ] Implement `BrainTask` model with `tenant = FK(identity.Tenant)`
- [ ] Implement `BrainService._execute_plan()` with TenantContext wrapper
- [ ] Implement `_build_tool_context()` with ORM serialization
- [ ] Implement `_sanitize_log_data()` for PII redaction
- [ ] Add `token_usage` and `cost_cents` fields to BrainTask
- [ ] Add `trace_id` propagation to AI services
- [ ] Test: Brain tools can query TenantScopedManager models

### Phase 8.6: Marketplace (Day 10)
**Goal:** Product layer for Brain capabilities

- [ ] Implement `core/marketplace/` app structure
- [ ] Implement `BrainCapability` and `TenantCapabilitySubscription`
- [ ] Implement `CapabilityConnection` for cross-tool orchestration
- [ ] Implement `core.evaluate_connections` tool
- [ ] Update Brain intent detector to respect subscriptions

### Phase 9: Agency OS (Day 10)
- [ ] Implement `apps/agency_os/deal.py`
- [ ] Implement `apps/agency_os/action_request.py`
- [ ] Connect Command Center UI
- [ ] Test: Full flow - lead → shadow tenant → deal → document → access

---

## 16. Day 1 Validation Checklist

Before writing any business logic:

- [ ] Create shadow tenant via service
- [ ] Verify creator gets admin access to shadow's resources
- [ ] Create entity, add DOT identifier, verify normalized lookup
- [ ] Upload document, link to entity, set visibility='shared_to_connections'
- [ ] Verify auto-grant triggers
- [ ] Test Celery task with `TenantContext`, verify it raises if no context
- [ ] Test async view with tenant context

**If these pass:** ACL engine is production-ready. Build features on top.

**If these fail:** Fix before proceeding.

---

## 17. Brain Integration Validation Checklist

Before deploying Brain to production:

- [ ] `BrainTask.tenant` is `FK(identity.Tenant)`, NOT legacy `Org`
- [ ] `_execute_plan()` wraps tool execution in `TenantContext(task.tenant)`
- [ ] `_build_tool_context()` serializes ORM objects (no lazy loading)
- [ ] `_sanitize_log_data()` redacts PII keys from execution logs
- [ ] `token_usage` and `cost_cents` are tracked per task
- [ ] `trace_id` is generated and passed to AI services
- [ ] Test: Domain tool using `TenantScopedManager` works inside Brain
- [ ] Test: Brain task with no tenant raises appropriate error
- [ ] Test: Temporal workflow dispatch is fire-and-forget

**Critical Test:**
```python
# This must work without raising TenantContextRequired
def test_brain_tool_with_tenant_scoped_manager():
    tenant = Tenant.objects.create(name="Test", status="active", ...)
    task = BrainTask.objects.create(tenant=tenant, ...)
    
    # Simulate Brain executor
    brain_service._execute_plan(task)
    
    # Domain tool should have been able to query Deal.objects (TenantScopedManager)
    assert task.status == BrainTask.Status.COMPLETED
```

**If these pass:** Brain is production-ready. Point webhooks at it, delete old code.

**If these fail:** Fix before cutover.

---

---

## 18. Legacy Model Migration Note

### `core.models.Org` → `identity.Tenant`

The legacy codebase (`mw-dj6/`) uses `core.models.Org` as the tenant model. The v4.2 architecture (`mw-dj6-v2/`) uses `identity.Tenant`.

**For Brain v1 (`core.brain/`):**
- All new code MUST use `identity.Tenant`
- DO NOT import `core.models.Org` in Brain code
- `BrainTask.tenant` → `FK(identity.Tenant)`
- `BrainWorkflowPlan.tenant` → `FK(identity.Tenant)`

**Migration Path:**
1. Build Brain on v4.2 identity model
2. During cutover, webhooks point at Brain
3. Brain uses new Tenant, not legacy Org
4. Legacy Org is only used by legacy code being deleted

---

**Architecture Status: Production-Ready**

