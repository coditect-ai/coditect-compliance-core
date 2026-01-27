---
title: 'Prompt 09: Architecture Decision Record - Multi-Tenancy Architecture'
type: reference
version: 1.0.0
created: '2025-12-27'
updated: '2025-12-27'
status: deprecated
tags:
- ai-ml
- authentication
- deployment
- security
- testing
- api
- architecture
- automation
summary: You are a principal architect designing the multi-tenancy model for CODITECT-COMPLIANCE.
  This ADR establishes how customer organizations are...
---
# Prompt 09: Architecture Decision Record - Multi-Tenancy Architecture

## Context

You are a principal architect designing the multi-tenancy model for CODITECT-COMPLIANCE. This ADR establishes how customer organizations are isolated, how data is partitioned, and how the platform scales across thousands of tenants.

## Output Specification

Generate a comprehensive Architecture Decision Record (ADR) following the standard ADR format. The document should be 3,000-4,500 words (9,000-14,000 tokens).

## Document Structure

### ADR-005: Multi-Tenancy Architecture

```markdown
# ADR-005: Multi-Tenancy Architecture and Data Isolation

## Status
Proposed | Accepted | Deprecated | Superseded

## Date
[Current Date]

## Decision Makers
- [Role: Chief Architect]
- [Role: Security Architect]
- [Role: Platform Engineering Lead]

## Context

### Problem Statement

CODITECT-COMPLIANCE must support thousands of customer organizations with:
- **Data Isolation**: Customer A cannot access Customer B's data
- **Performance Isolation**: One customer's load cannot impact others
- **Compliance Isolation**: Each customer has independent compliance postures
- **Administrative Isolation**: Customer admins manage only their organization

### Scale Requirements

| Metric | Year 1 | Year 3 | Year 5 |
|--------|--------|--------|--------|
| Organizations | 100 | 1,000 | 10,000 |
| Users per Org | 10-50 | 10-500 | 10-1000 |
| Controls per Org | 500-2000 | 500-5000 | 500-10000 |
| Evidence Items | 10K-100K | 100K-1M | 1M-10M |
| Integrations | 5-15 | 15-50 | 50-100 |

### Compliance Requirements

- SOC 2 Type II: Logical access controls, data segregation
- HIPAA: PHI isolation for healthcare customers
- GDPR: Data residency and right to erasure
- Enterprise: Single-tenant deployment option

### Technical Constraints

- FoundationDB as primary database (supports keyspace isolation)
- Neo4j for graph data (supports multi-database or label isolation)
- GCS for blob storage (bucket-level or prefix isolation)
- Kubernetes for compute (namespace isolation)
- Single codebase, multi-tenant deployment

## Decision Drivers

1. **Security**: Cryptographic guarantees of data isolation
2. **Cost Efficiency**: Shared infrastructure where safe
3. **Operational Simplicity**: Manageable at scale
4. **Performance**: Predictable latency regardless of neighbor load
5. **Compliance**: Meet regulatory isolation requirements
6. **Scalability**: Support 10,000+ organizations

## Options Considered

### Option 1: Single Database with Logical Isolation

**Description**: All tenants in one database, isolated by organization_id column.

```
┌─────────────────────────────────────────┐
│           Shared Database               │
│  ┌──────────────────────────────────┐  │
│  │ controls                          │  │
│  │ - organization_id (index)         │  │
│  │ - control_id                      │  │
│  │ - ...                             │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**Pros**:
- Simplest implementation
- Lowest infrastructure cost
- Easy schema migrations

**Cons**:
- No isolation for noisy neighbors
- Cross-tenant bugs possible
- Single compliance boundary
- Difficult to provide single-tenant option

### Option 2: Database-per-Tenant

**Description**: Each tenant gets dedicated database instances.

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Org A DB    │  │ Org B DB    │  │ Org C DB    │
└─────────────┘  └─────────────┘  └─────────────┘
```

**Pros**:
- Complete isolation
- Per-tenant performance
- Easy data residency
- Clean deletion

**Cons**:
- Expensive at scale (10,000 databases)
- Complex connection management
- Schema migration complexity
- High operational overhead

### Option 3: Hybrid - Shared Infrastructure with Isolation Boundaries

**Description**: Shared databases with strong isolation guarantees via keyspace partitioning, encryption, and access controls.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Shared Infrastructure                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                  FoundationDB Cluster                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │ Keyspace:   │  │ Keyspace:   │  │ Keyspace:   │       │  │
│  │  │ org_a/...   │  │ org_b/...   │  │ org_c/...   │       │  │
│  │  │ (encrypted) │  │ (encrypted) │  │ (encrypted) │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Neo4j Cluster                          │  │
│  │  Multi-database OR Label-based isolation                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    GCS Buckets                             │  │
│  │  gs://coditect-evidence/{org_id}/...                       │  │
│  │  (per-org encryption keys)                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Pros**:
- Cost-efficient shared infrastructure
- Strong isolation via keyspace + encryption
- Scalable to 10,000+ tenants
- Optional dedicated infrastructure for enterprise

**Cons**:
- More complex access control implementation
- Requires discipline in query construction
- Mixed compliance boundary

### Option 4: Cell-Based Architecture

**Description**: Tenants grouped into "cells" with dedicated infrastructure per cell.

```
┌─────────────────────────┐  ┌─────────────────────────┐
│        Cell 1           │  │        Cell 2           │
│  ┌──────────────────┐   │  │  ┌──────────────────┐   │
│  │ Orgs 1-100       │   │  │  │ Orgs 101-200     │   │
│  │ Dedicated DB     │   │  │  │ Dedicated DB     │   │
│  │ Dedicated Cache  │   │  │  │ Dedicated Cache  │   │
│  └──────────────────┘   │  │  └──────────────────┘   │
└─────────────────────────┘  └─────────────────────────┘
```

**Pros**:
- Blast radius limited to cell
- Easier capacity planning
- Natural compliance boundaries
- Supports data residency

**Cons**:
- More complex routing
- Cross-cell operations difficult
- Cell sizing challenges

## Decision

**Chosen Option**: Hybrid of Option 3 (Shared + Isolation) + Option 4 (Cell-Based for Enterprise)

### Rationale

1. **Default**: Shared infrastructure with keyspace isolation for cost efficiency
2. **Enterprise Tier**: Cell-based deployment for customers requiring dedicated infrastructure
3. **Compliance**: Per-org encryption keys for data-at-rest
4. **Data Residency**: Regional cells for GDPR/data sovereignty requirements

## Detailed Design

### Tenant Context Model

```python
from dataclasses import dataclass
from typing import Optional, List
from enum import Enum
from contextvars import ContextVar

class TenantTier(Enum):
    STARTER = "starter"       # Shared infrastructure
    GROWTH = "growth"         # Shared with priority
    ENTERPRISE = "enterprise" # Dedicated cell

class DataResidency(Enum):
    US = "us"
    EU = "eu"
    APAC = "apac"
    
@dataclass
class Organization:
    """Organization (tenant) entity."""
    id: str
    name: str
    slug: str
    tier: TenantTier
    data_residency: DataResidency
    
    # Infrastructure assignment
    cell_id: Optional[str]  # For enterprise tier
    encryption_key_id: str
    
    # Limits
    max_users: int
    max_integrations: int
    max_frameworks: int
    
    # Metadata
    created_at: datetime
    settings: Dict[str, Any]

@dataclass
class TenantContext:
    """
    Context for current request/operation.
    Propagated through all service calls.
    """
    organization_id: str
    user_id: str
    roles: List[str]
    permissions: List[str]
    tier: TenantTier
    cell_id: Optional[str]
    encryption_key_id: str
    
    def has_permission(self, permission: str) -> bool:
        return permission in self.permissions

# Context variable for request-scoped tenant context
_tenant_context: ContextVar[Optional[TenantContext]] = ContextVar(
    'tenant_context',
    default=None
)

def get_tenant_context() -> TenantContext:
    """Get current tenant context."""
    ctx = _tenant_context.get()
    if ctx is None:
        raise TenantContextError("No tenant context available")
    return ctx

def set_tenant_context(ctx: TenantContext) -> None:
    """Set tenant context for current request."""
    _tenant_context.set(ctx)
```

### Data Isolation Layers

```python
class TenantIsolationMiddleware:
    """
    Middleware to establish tenant context from JWT.
    """
    
    async def __call__(
        self,
        request: Request,
        call_next: Callable
    ) -> Response:
        # Extract tenant from JWT
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        claims = await self.verify_token(token)
        
        # Build tenant context
        org = await self.org_repo.get(claims["org_id"])
        user = await self.user_repo.get(claims["sub"])
        
        context = TenantContext(
            organization_id=org.id,
            user_id=user.id,
            roles=user.roles,
            permissions=self._expand_permissions(user.roles),
            tier=org.tier,
            cell_id=org.cell_id,
            encryption_key_id=org.encryption_key_id
        )
        
        # Set context for request
        set_tenant_context(context)
        
        try:
            response = await call_next(request)
            return response
        finally:
            # Clear context
            _tenant_context.set(None)
```

### FoundationDB Keyspace Design

```python
class TenantKeyspace:
    """
    FoundationDB keyspace design for multi-tenancy.
    
    Key structure:
    /{org_id}/{entity_type}/{entity_id}
    
    Example:
    /org_abc123/controls/ctrl_xyz789
    /org_abc123/evidence/evid_123456
    /org_abc123/frameworks/frm_soc2v2
    """
    
    @staticmethod
    def org_prefix(organization_id: str) -> bytes:
        """Get keyspace prefix for organization."""
        return f"/{organization_id}/".encode()
        
    @staticmethod
    def entity_key(
        organization_id: str,
        entity_type: str,
        entity_id: str
    ) -> bytes:
        """Build full key for entity."""
        return f"/{organization_id}/{entity_type}/{entity_id}".encode()
        
    @staticmethod
    def entity_range(
        organization_id: str,
        entity_type: str
    ) -> tuple[bytes, bytes]:
        """Get key range for all entities of type in org."""
        prefix = f"/{organization_id}/{entity_type}/".encode()
        return (prefix, prefix + b'\xff')

class TenantAwareRepository:
    """
    Base repository that enforces tenant isolation.
    """
    
    def __init__(self, fdb_client: FoundationDBClient):
        self.fdb = fdb_client
        
    @property
    def organization_id(self) -> str:
        """Get organization_id from tenant context."""
        return get_tenant_context().organization_id
        
    async def get(self, entity_id: str) -> Optional[Entity]:
        """
        Get entity by ID, scoped to current tenant.
        """
        key = TenantKeyspace.entity_key(
            self.organization_id,
            self.entity_type,
            entity_id
        )
        
        data = await self.fdb.get(key)
        if data is None:
            return None
            
        # Decrypt if needed
        decrypted = await self._decrypt(data)
        return self._deserialize(decrypted)
        
    async def list(
        self,
        limit: int = 100,
        cursor: Optional[str] = None
    ) -> tuple[List[Entity], Optional[str]]:
        """
        List entities for current tenant.
        """
        start, end = TenantKeyspace.entity_range(
            self.organization_id,
            self.entity_type
        )
        
        if cursor:
            start = base64.b64decode(cursor)
            
        results = await self.fdb.get_range(start, end, limit=limit + 1)
        
        entities = [self._deserialize(r.value) for r in results[:limit]]
        next_cursor = None
        if len(results) > limit:
            next_cursor = base64.b64encode(results[limit].key).decode()
            
        return entities, next_cursor
        
    async def save(self, entity: Entity) -> None:
        """
        Save entity, automatically scoped to tenant.
        """
        # Ensure entity belongs to current tenant
        if entity.organization_id != self.organization_id:
            raise TenantIsolationError(
                "Cannot save entity for different organization"
            )
            
        key = TenantKeyspace.entity_key(
            self.organization_id,
            self.entity_type,
            entity.id
        )
        
        data = self._serialize(entity)
        encrypted = await self._encrypt(data)
        
        await self.fdb.set(key, encrypted)
        
    async def _encrypt(self, data: bytes) -> bytes:
        """Encrypt data with tenant's key."""
        ctx = get_tenant_context()
        key = await self.key_vault.get_key(ctx.encryption_key_id)
        return self.cipher.encrypt(data, key)
        
    async def _decrypt(self, data: bytes) -> bytes:
        """Decrypt data with tenant's key."""
        ctx = get_tenant_context()
        key = await self.key_vault.get_key(ctx.encryption_key_id)
        return self.cipher.decrypt(data, key)
```

### Neo4j Multi-Tenancy

```python
class TenantAwareGraphRepository:
    """
    Neo4j repository with tenant isolation.
    
    Isolation strategy:
    - All nodes have organization_id property
    - All queries filter by organization_id
    - Query builder prevents cross-tenant access
    """
    
    def __init__(self, driver: neo4j.Driver):
        self.driver = driver
        
    @property
    def organization_id(self) -> str:
        return get_tenant_context().organization_id
        
    async def query(
        self,
        cypher: str,
        parameters: Dict[str, Any]
    ) -> List[Dict]:
        """
        Execute Cypher query with tenant isolation.
        
        Automatically injects organization_id filter.
        """
        # Validate query doesn't bypass isolation
        self._validate_query(cypher)
        
        # Inject tenant parameter
        parameters["_organization_id"] = self.organization_id
        
        # Wrap query with tenant filter
        isolated_query = self._wrap_with_tenant_filter(cypher)
        
        async with self.driver.session() as session:
            result = await session.run(isolated_query, parameters)
            return [record.data() for record in await result.fetch_all()]
            
    def _validate_query(self, cypher: str) -> None:
        """
        Validate query doesn't attempt cross-tenant access.
        
        Raises exception if query:
        - Doesn't use parameterized organization_id
        - Uses UNION across different orgs
        - Bypasses tenant filter
        """
        # Simple validation - production would use query parser
        if "organization_id:" in cypher.lower():
            raise TenantIsolationError(
                "Direct organization_id literals not allowed"
            )
            
    def _wrap_with_tenant_filter(self, cypher: str) -> str:
        """
        Wrap Cypher query to enforce tenant isolation.
        
        Adds WHERE clause to all MATCH patterns.
        """
        # This is simplified - production uses proper AST manipulation
        return cypher.replace(
            "MATCH (",
            "MATCH (n {organization_id: $_organization_id})--(n2) WHERE n"
        )
```

### GCS Blob Isolation

```python
class TenantBlobStorage:
    """
    GCS storage with tenant isolation.
    
    Structure:
    gs://coditect-evidence-{region}/{org_id}/{evidence_type}/{year}/{month}/{file}
    
    Security:
    - Per-org encryption keys (CMEK)
    - Signed URLs with short expiry
    - Audit logging for all access
    """
    
    BUCKET_TEMPLATE = "coditect-evidence-{region}"
    
    def __init__(
        self,
        gcs_client: storage.Client,
        key_manager: KeyManager
    ):
        self.gcs = gcs_client
        self.keys = key_manager
        
    def _get_bucket_name(self, region: str) -> str:
        return self.BUCKET_TEMPLATE.format(region=region)
        
    def _get_blob_path(
        self,
        organization_id: str,
        evidence_type: str,
        filename: str
    ) -> str:
        now = datetime.utcnow()
        return f"{organization_id}/{evidence_type}/{now.year}/{now.month:02d}/{filename}"
        
    async def upload(
        self,
        evidence_type: str,
        content: bytes,
        filename: str,
        content_type: str
    ) -> str:
        """Upload blob for current tenant."""
        ctx = get_tenant_context()
        org = await self.org_repo.get(ctx.organization_id)
        
        bucket_name = self._get_bucket_name(org.data_residency.value)
        blob_path = self._get_blob_path(
            ctx.organization_id,
            evidence_type,
            filename
        )
        
        bucket = self.gcs.bucket(bucket_name)
        blob = bucket.blob(blob_path)
        
        # Set customer-managed encryption key
        blob.upload_from_string(
            content,
            content_type=content_type,
            encryption_key=await self.keys.get_key(org.encryption_key_id)
        )
        
        return f"gs://{bucket_name}/{blob_path}"
        
    async def get_signed_url(
        self,
        blob_uri: str,
        expiry_minutes: int = 15
    ) -> str:
        """Generate signed URL for blob access."""
        ctx = get_tenant_context()
        
        # Parse URI
        bucket_name, blob_path = self._parse_uri(blob_uri)
        
        # Verify blob belongs to current tenant
        if not blob_path.startswith(f"{ctx.organization_id}/"):
            raise TenantIsolationError(
                "Cannot access blob from different organization"
            )
            
        bucket = self.gcs.bucket(bucket_name)
        blob = bucket.blob(blob_path)
        
        return blob.generate_signed_url(
            version="v4",
            expiration=timedelta(minutes=expiry_minutes),
            method="GET"
        )
```

### Resource Limits

```python
@dataclass
class TenantLimits:
    """Resource limits per tenant tier."""
    max_users: int
    max_integrations: int
    max_frameworks: int
    max_controls: int
    max_evidence_items: int
    max_storage_gb: int
    api_rate_limit: int  # requests per minute
    agent_task_limit: int  # concurrent agent tasks

TIER_LIMITS = {
    TenantTier.STARTER: TenantLimits(
        max_users=10,
        max_integrations=5,
        max_frameworks=3,
        max_controls=1000,
        max_evidence_items=50000,
        max_storage_gb=10,
        api_rate_limit=100,
        agent_task_limit=2
    ),
    TenantTier.GROWTH: TenantLimits(
        max_users=100,
        max_integrations=20,
        max_frameworks=10,
        max_controls=5000,
        max_evidence_items=500000,
        max_storage_gb=100,
        api_rate_limit=500,
        agent_task_limit=10
    ),
    TenantTier.ENTERPRISE: TenantLimits(
        max_users=1000,
        max_integrations=100,
        max_frameworks=50,
        max_controls=20000,
        max_evidence_items=5000000,
        max_storage_gb=1000,
        api_rate_limit=2000,
        agent_task_limit=50
    )
}

class TenantLimitEnforcer:
    """Enforce resource limits per tenant."""
    
    async def check_limit(
        self,
        resource_type: str,
        increment: int = 1
    ) -> bool:
        """Check if operation would exceed limit."""
        ctx = get_tenant_context()
        limits = TIER_LIMITS[ctx.tier]
        
        current = await self._get_current_usage(
            ctx.organization_id,
            resource_type
        )
        max_allowed = getattr(limits, f"max_{resource_type}")
        
        return current + increment <= max_allowed
        
    async def enforce_limit(
        self,
        resource_type: str,
        increment: int = 1
    ) -> None:
        """Raise exception if limit exceeded."""
        if not await self.check_limit(resource_type, increment):
            ctx = get_tenant_context()
            raise TenantLimitExceeded(
                f"Organization {ctx.organization_id} has reached "
                f"{resource_type} limit for {ctx.tier.value} tier"
            )
```

### Tenant Deletion

```python
class TenantDeletionService:
    """
    Handle complete tenant data deletion for GDPR compliance.
    """
    
    async def delete_tenant(
        self,
        organization_id: str,
        requester: str,
        reason: str
    ) -> DeletionReceipt:
        """
        Delete all data for a tenant.
        
        Steps:
        1. Verify authorization
        2. Create deletion audit record
        3. Delete FoundationDB keyspace
        4. Delete Neo4j nodes/relationships
        5. Delete GCS blobs
        6. Revoke encryption keys
        7. Return receipt
        """
        # Verify authorization
        await self._verify_deletion_authorization(organization_id, requester)
        
        # Create audit record
        deletion_id = await self._create_deletion_record(
            organization_id, requester, reason
        )
        
        try:
            # Delete FoundationDB data
            await self._delete_fdb_keyspace(organization_id)
            
            # Delete Neo4j data
            await self._delete_graph_data(organization_id)
            
            # Delete GCS blobs
            await self._delete_blob_storage(organization_id)
            
            # Revoke encryption keys
            await self._revoke_encryption_keys(organization_id)
            
            # Mark deletion complete
            receipt = await self._complete_deletion(deletion_id)
            
            return receipt
            
        except Exception as e:
            # Mark deletion failed, trigger manual review
            await self._fail_deletion(deletion_id, str(e))
            raise
            
    async def _delete_fdb_keyspace(self, organization_id: str) -> None:
        """Delete all FoundationDB keys for organization."""
        prefix = TenantKeyspace.org_prefix(organization_id)
        await self.fdb.clear_range_startswith(prefix)
        
    async def _delete_graph_data(self, organization_id: str) -> None:
        """Delete all Neo4j nodes for organization."""
        async with self.neo4j.session() as session:
            await session.run(
                """
                MATCH (n {organization_id: $org_id})
                DETACH DELETE n
                """,
                org_id=organization_id
            )
            
    async def _delete_blob_storage(self, organization_id: str) -> None:
        """Delete all GCS blobs for organization."""
        for region in DataResidency:
            bucket_name = f"coditect-evidence-{region.value}"
            bucket = self.gcs.bucket(bucket_name)
            
            # List and delete all blobs with org prefix
            blobs = bucket.list_blobs(prefix=f"{organization_id}/")
            for blob in blobs:
                blob.delete()
```

## Consequences

### Positive
1. **Strong Isolation**: Per-tenant encryption + keyspace isolation
2. **Cost Efficiency**: Shared infrastructure for most tenants
3. **Compliance**: Supports GDPR deletion, data residency
4. **Scalability**: Handles 10,000+ tenants
5. **Flexibility**: Enterprise tier gets dedicated infrastructure

### Negative
1. **Complexity**: Multiple isolation mechanisms to maintain
2. **Performance**: Per-tenant encryption adds latency
3. **Debugging**: Harder to diagnose cross-tenant issues

### Mitigations
- Comprehensive tenant context logging
- Performance benchmarks per operation type
- Automated isolation testing in CI/CD

## Implementation Plan

### Phase 1: Core Model (Week 1)
- TenantContext implementation
- Middleware for context propagation
- FoundationDB keyspace design

### Phase 2: Data Layer (Week 2-3)
- TenantAwareRepository base class
- Neo4j isolation wrapper
- GCS tenant storage

### Phase 3: Limits & Quotas (Week 4)
- Resource limit enforcement
- Quota monitoring
- Alerting for limit approaching

### Phase 4: Deletion (Week 5)
- Tenant deletion workflow
- Audit trail
- Verification tests

## Validation Criteria

1. **Isolation**: Zero cross-tenant data access possible
2. **Deletion**: Complete data removal within 24 hours
3. **Performance**: < 5% latency impact from encryption
4. **Audit**: 100% of data access logged
5. **Limits**: Enforcement prevents resource exhaustion

## References

- [Multi-Tenant SaaS Architecture](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/multi-tenant-architecture.html)
- [FoundationDB Keyspace Design](https://apple.github.io/foundationdb/data-modeling.html)
- [GDPR Right to Erasure](https://gdpr.eu/right-to-erasure-erase/)
```

## Acceptance Criteria

1. **Isolation Model**: Complete design for all data layers
2. **Context Propagation**: Request-scoped tenant context
3. **Encryption**: Per-tenant key management
4. **Deletion**: GDPR-compliant tenant removal
5. **Limits**: Resource quota enforcement

## Token Budget

- Target: 10,000-16,000 tokens
- Priority: Data isolation and deletion sections

## Dependencies

- Input: SDD multi-tenant requirements
- Input: ADR-001 (Control Graph) for graph isolation
- Output: Feeds into all component build prompts

## Integration Points

This ADR establishes patterns used by:
- All repository implementations
- API authentication middleware
- Agent task execution context
- Evidence storage service
