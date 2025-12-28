---
title: CODITECT-COMPLIANCE Technical Design Document Prompt
type: reference
version: 1.0.0
created: '2025-12-27'
updated: '2025-12-27'
status: deprecated
tags:
- ai-ml
- authentication
- security
- testing
- api
- architecture
- automation
- backend
summary: 'CODITECT-COMPLIANCE Technical Design Document Prompt Prompt ID: 04-TECHNICAL-DESIGN-DOCUMENT
  Output: Dependencies: 02-PRODUCT-REQUIREMENTS, 03-SOFTWARE-DESIGN-DOCUMENT Estimated
  Tokens: 35,000-45,000 Generate a comprehensive Technical Design...'
---
# CODITECT-COMPLIANCE Technical Design Document Prompt

**Prompt ID:** 04-TECHNICAL-DESIGN-DOCUMENT  
**Output:** `docs/architecture/TDD.md`  
**Dependencies:** 02-PRODUCT-REQUIREMENTS, 03-SOFTWARE-DESIGN-DOCUMENT  
**Estimated Tokens:** 35,000-45,000

---

## Objective

Generate a comprehensive Technical Design Document (TDD) that provides implementation-level detail for each component. This document bridges the SDD architecture to actual code, specifying algorithms, data structures, interfaces, and implementation patterns.

---

## Instructions

You are CODITECT-CORE generating a Technical Design Document. Reference the SDD for architectural context. Generate implementation-ready specifications with code examples, algorithm definitions, and detailed interface contracts.

---

## Required Document Sections

### Section 1: Document Overview

```yaml
document:
  title: "CODITECT-COMPLIANCE Technical Design Document"
  version: "1.0.0"
  status: "Draft"
  
purpose: |
  This TDD provides implementation-level specifications for CODITECT-COMPLIANCE,
  enabling direct code generation by CODITECT-CORE. Each section includes
  algorithms, data structures, interface contracts, and code examples.
  
implementation_standards:
  language_primary: "Python 3.12+"
  language_secondary: "TypeScript 5.x"
  style_guide: "PEP 8 / ESLint + Prettier"
  type_hints: "Required (Python: strict mypy, TS: strict)"
  documentation: "Google-style docstrings"
  testing: "pytest / Vitest"
```

---

### Section 2: Core Domain Models

Generate complete Python dataclass/Pydantic models:

#### 2.1 Framework & Control Models

```python
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime, date
from enum import Enum
from typing import Optional, List
from uuid import UUID, uuid4
from pydantic import BaseModel, Field, ConfigDict

class FrameworkStatus(str, Enum):
    ACTIVE = "ACTIVE"
    DEPRECATED = "DEPRECATED"
    DRAFT = "DRAFT"

class ControlCategory(str, Enum):
    ACCESS_CONTROL = "ACCESS_CONTROL"
    AI_OPERATIONS = "AI_OPERATIONS"
    CHANGE_MANAGEMENT = "CHANGE_MANAGEMENT"
    LOGGING_MONITORING = "LOGGING_MONITORING"
    DATA_PROTECTION = "DATA_PROTECTION"
    RISK_MANAGEMENT = "RISK_MANAGEMENT"
    INCIDENT_RESPONSE = "INCIDENT_RESPONSE"
    VENDOR_MANAGEMENT = "VENDOR_MANAGEMENT"
    OTHER = "OTHER"

class MappingRelationship(str, Enum):
    EQUIVALENT = "EQUIVALENT"
    OVERLAPS = "OVERLAPS"
    PARTIALLY_SATISFIES = "PARTIALLY_SATISFIES"

class MappingStatus(str, Enum):
    SUGGESTED = "SUGGESTED"
    APPROVED = "APPROVED"
    REJECTED = "REJECTED"

class ImplementationStatus(str, Enum):
    NOT_STARTED = "NOT_STARTED"
    IN_PROGRESS = "IN_PROGRESS"
    IMPLEMENTED = "IMPLEMENTED"
    NOT_APPLICABLE = "NOT_APPLICABLE"


@dataclass(frozen=True)
class FrameworkId:
    """Value object for framework identification."""
    value: UUID
    
    @classmethod
    def generate(cls) -> FrameworkId:
        return cls(uuid4())
    
    @classmethod
    def from_string(cls, s: str) -> FrameworkId:
        return cls(UUID(s))


class Framework(BaseModel):
    """
    Compliance framework definition (e.g., SOC 2, HIPAA, EU AI Act).
    
    Stored in Neo4j as a node with label :Framework.
    """
    model_config = ConfigDict(frozen=True)
    
    id: UUID = Field(default_factory=uuid4)
    name: str = Field(..., max_length=256)
    short_name: str = Field(..., max_length=32, pattern=r"^[A-Z0-9_]+$")
    version: str = Field(..., pattern=r"^\d+\.\d+(\.\d+)?$")
    effective_date: date
    source_url: Optional[str] = None
    description: Optional[str] = Field(None, max_length=4000)
    status: FrameworkStatus = FrameworkStatus.ACTIVE
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    
    def to_neo4j_properties(self) -> dict:
        """Convert to Neo4j node properties."""
        return {
            "id": str(self.id),
            "name": self.name,
            "short_name": self.short_name,
            "version": self.version,
            "effective_date": self.effective_date.isoformat(),
            "source_url": self.source_url,
            "description": self.description,
            "status": self.status.value,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat(),
        }


class Control(BaseModel):
    """
    Individual compliance control within a framework.
    
    Stored in Neo4j as a node with label :Control.
    Connected to Framework via :BELONGS_TO relationship.
    """
    model_config = ConfigDict(frozen=True)
    
    id: UUID = Field(default_factory=uuid4)
    framework_id: UUID
    control_id: str = Field(..., max_length=64, description="Framework-specific ID")
    name: str = Field(..., max_length=256)
    description: Optional[str] = Field(None, max_length=4000)
    objective: Optional[str] = Field(None, max_length=2000)
    category: ControlCategory
    implementation_guidance: Optional[str] = None
    testing_procedure: Optional[str] = None
    evidence_requirements: List[str] = Field(default_factory=list)
    tags: List[str] = Field(default_factory=list)
    custom_fields: dict = Field(default_factory=dict)
    
    @property
    def qualified_id(self) -> str:
        """Return framework-qualified control ID."""
        return f"{self.framework_id}:{self.control_id}"


class ControlMapping(BaseModel):
    """
    Cross-framework control mapping relationship.
    
    Stored in Neo4j as a :MAPS_TO relationship between Control nodes.
    """
    model_config = ConfigDict(frozen=True)
    
    id: UUID = Field(default_factory=uuid4)
    source_control_id: UUID
    target_control_id: UUID
    relationship_type: MappingRelationship
    confidence_score: int = Field(..., ge=0, le=100)
    status: MappingStatus = MappingStatus.SUGGESTED
    approved_by: Optional[UUID] = None
    notes: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)


class ControlImplementation(BaseModel):
    """
    Organization-specific control implementation status.
    
    Stored in FoundationDB with key: org/{org_id}/control_impl/{control_id}
    """
    id: UUID = Field(default_factory=uuid4)
    organization_id: UUID
    control_id: UUID
    status: ImplementationStatus = ImplementationStatus.NOT_STARTED
    owner_id: Optional[UUID] = None
    implementation_date: Optional[date] = None
    review_date: Optional[date] = None
    notes: Optional[str] = None
    artifacts: List[str] = Field(default_factory=list)
    last_evidence_date: Optional[datetime] = None
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

Generate similar complete model definitions for:
- Evidence models (EvidenceItem, EvidenceGap, EvidencePackage)
- Integration models (Integration, IntegrationCredential, CollectionJob)
- Monitoring models (ControlCheck, CheckResult, PostureScore, Alert)
- Agent models (AgentTask, AgentExecution, ApprovalRequest)
- Organization models (Organization, User, Role, Permission)
- Audit models (Audit, Finding, RemediationTask)
- Risk models (Risk, RiskAssessment, Mitigation)

---

### Section 3: Repository Layer

#### 3.1 Repository Interface Pattern

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar, Optional, List
from uuid import UUID

T = TypeVar("T")

class Repository(ABC, Generic[T]):
    """
    Base repository interface following Repository pattern.
    All concrete repositories must implement these methods.
    """
    
    @abstractmethod
    async def get(self, id: UUID) -> Optional[T]:
        """Retrieve entity by ID."""
        ...
    
    @abstractmethod
    async def get_many(self, ids: List[UUID]) -> List[T]:
        """Retrieve multiple entities by IDs."""
        ...
    
    @abstractmethod
    async def save(self, entity: T) -> T:
        """Create or update entity."""
        ...
    
    @abstractmethod
    async def delete(self, id: UUID) -> bool:
        """Delete entity by ID. Returns True if deleted."""
        ...
    
    @abstractmethod
    async def exists(self, id: UUID) -> bool:
        """Check if entity exists."""
        ...


class FoundationDBRepository(Repository[T], Generic[T]):
    """
    Base FoundationDB repository implementation.
    
    Uses FoundationDB's tuple layer for key construction:
    Key format: (tenant_id, entity_type, entity_id)
    """
    
    def __init__(
        self,
        db: fdb.Database,
        entity_type: str,
        model_class: type[T],
    ):
        self.db = db
        self.entity_type = entity_type
        self.model_class = model_class
        self.key_space = fdb.directory.create_or_open(db, (entity_type,))
    
    def _make_key(self, tenant_id: UUID, entity_id: UUID) -> bytes:
        """Construct key tuple for entity."""
        return self.key_space.pack((str(tenant_id), str(entity_id)))
    
    async def get(self, id: UUID, tenant_id: UUID) -> Optional[T]:
        """Retrieve entity from FoundationDB."""
        @fdb.transactional
        def _get(tr):
            key = self._make_key(tenant_id, id)
            value = tr.get(key)
            if value is None:
                return None
            return self.model_class.model_validate_json(value)
        
        return await asyncio.to_thread(_get, self.db)
    
    async def save(self, entity: T, tenant_id: UUID) -> T:
        """Save entity to FoundationDB."""
        @fdb.transactional
        def _save(tr):
            key = self._make_key(tenant_id, entity.id)
            value = entity.model_dump_json()
            tr.set(key, value.encode())
            return entity
        
        return await asyncio.to_thread(_save, self.db)
    
    async def list_by_tenant(
        self,
        tenant_id: UUID,
        limit: int = 100,
        cursor: Optional[str] = None,
    ) -> tuple[List[T], Optional[str]]:
        """List entities for a tenant with cursor pagination."""
        @fdb.transactional
        def _list(tr):
            prefix = self.key_space.pack((str(tenant_id),))
            start = prefix if cursor is None else cursor.encode()
            
            results = []
            next_cursor = None
            
            for key, value in tr.get_range(start, fdb.strinc(prefix), limit=limit + 1):
                if len(results) < limit:
                    results.append(self.model_class.model_validate_json(value))
                else:
                    next_cursor = key.decode()
            
            return results, next_cursor
        
        return await asyncio.to_thread(_list, self.db)
```

#### 3.2 Neo4j Graph Repository

```python
from neo4j import AsyncDriver, AsyncSession
from typing import List, Optional, Tuple

class ControlGraphRepository:
    """
    Repository for compliance control graph operations in Neo4j.
    
    Graph structure:
    - (:Framework)-[:CONTAINS]->(:Control)
    - (:Control)-[:MAPS_TO {confidence: int, relationship: str}]->(:Control)
    - (:Control)-[:VERIFIED_BY]->(:Check)
    - (:Control)-[:IMPLEMENTED_BY]->(:Policy)
    """
    
    def __init__(self, driver: AsyncDriver):
        self.driver = driver
    
    async def create_framework(self, framework: Framework) -> Framework:
        """Create framework node in graph."""
        query = """
        CREATE (f:Framework $props)
        RETURN f
        """
        async with self.driver.session() as session:
            result = await session.run(
                query,
                props=framework.to_neo4j_properties()
            )
            record = await result.single()
            return framework
    
    async def create_control(self, control: Control) -> Control:
        """Create control node and link to framework."""
        query = """
        MATCH (f:Framework {id: $framework_id})
        CREATE (c:Control $props)
        CREATE (f)-[:CONTAINS]->(c)
        RETURN c
        """
        async with self.driver.session() as session:
            result = await session.run(
                query,
                framework_id=str(control.framework_id),
                props={
                    "id": str(control.id),
                    "control_id": control.control_id,
                    "name": control.name,
                    "description": control.description,
                    "category": control.category.value,
                    # ... other properties
                }
            )
            return control
    
    async def create_mapping(self, mapping: ControlMapping) -> ControlMapping:
        """Create mapping relationship between controls."""
        query = """
        MATCH (source:Control {id: $source_id})
        MATCH (target:Control {id: $target_id})
        CREATE (source)-[r:MAPS_TO {
            id: $mapping_id,
            relationship_type: $relationship_type,
            confidence_score: $confidence,
            status: $status
        }]->(target)
        RETURN r
        """
        async with self.driver.session() as session:
            await session.run(
                query,
                source_id=str(mapping.source_control_id),
                target_id=str(mapping.target_control_id),
                mapping_id=str(mapping.id),
                relationship_type=mapping.relationship_type.value,
                confidence=mapping.confidence_score,
                status=mapping.status.value,
            )
            return mapping
    
    async def find_mapped_controls(
        self,
        control_id: UUID,
        min_confidence: int = 70,
    ) -> List[Tuple[Control, ControlMapping]]:
        """Find controls mapped to given control with minimum confidence."""
        query = """
        MATCH (source:Control {id: $control_id})-[m:MAPS_TO]->(target:Control)
        WHERE m.confidence_score >= $min_confidence
          AND m.status = 'APPROVED'
        RETURN target, m
        ORDER BY m.confidence_score DESC
        """
        async with self.driver.session() as session:
            result = await session.run(
                query,
                control_id=str(control_id),
                min_confidence=min_confidence,
            )
            mappings = []
            async for record in result:
                target = Control(**record["target"])
                mapping = ControlMapping(**record["m"])
                mappings.append((target, mapping))
            return mappings
    
    async def find_controls_satisfying_frameworks(
        self,
        framework_ids: List[UUID],
    ) -> List[Control]:
        """
        Find controls that satisfy requirements across multiple frameworks.
        
        Uses graph traversal to find controls with MAPS_TO relationships
        covering all specified frameworks.
        """
        query = """
        MATCH (c:Control)
        WHERE c.framework_id IN $framework_ids
           OR EXISTS {
               MATCH (c)-[:MAPS_TO*1..2]-(mapped:Control)
               WHERE mapped.framework_id IN $framework_ids
           }
        WITH c, 
             [fw IN $framework_ids | 
              CASE WHEN c.framework_id = fw 
                   OR EXISTS((c)-[:MAPS_TO*1..2]-(:Control {framework_id: fw}))
              THEN 1 ELSE 0 END
             ] AS coverage
        WHERE reduce(sum = 0, x IN coverage | sum + x) = size($framework_ids)
        RETURN DISTINCT c
        """
        async with self.driver.session() as session:
            result = await session.run(
                query,
                framework_ids=[str(fid) for fid in framework_ids],
            )
            controls = []
            async for record in result:
                controls.append(Control(**record["c"]))
            return controls
    
    async def calculate_gap_analysis(
        self,
        organization_id: UUID,
        framework_id: UUID,
    ) -> dict:
        """
        Calculate compliance gap analysis for organization against framework.
        
        Returns control counts by implementation status and evidence freshness.
        """
        query = """
        MATCH (f:Framework {id: $framework_id})-[:CONTAINS]->(c:Control)
        OPTIONAL MATCH (c)-[:HAS_IMPLEMENTATION {org_id: $org_id}]->(impl)
        OPTIONAL MATCH (c)-[:SUPPORTED_BY {org_id: $org_id}]->(ev:Evidence)
        WITH c, impl, 
             CASE WHEN ev IS NULL THEN 'MISSING'
                  WHEN ev.collected_at < datetime() - duration('P30D') THEN 'STALE'
                  ELSE 'CURRENT' END AS evidence_status
        RETURN 
            count(c) AS total_controls,
            sum(CASE WHEN impl.status = 'IMPLEMENTED' THEN 1 ELSE 0 END) AS implemented,
            sum(CASE WHEN impl.status = 'IN_PROGRESS' THEN 1 ELSE 0 END) AS in_progress,
            sum(CASE WHEN impl.status = 'NOT_STARTED' OR impl IS NULL THEN 1 ELSE 0 END) AS not_started,
            sum(CASE WHEN evidence_status = 'CURRENT' THEN 1 ELSE 0 END) AS current_evidence,
            sum(CASE WHEN evidence_status = 'STALE' THEN 1 ELSE 0 END) AS stale_evidence,
            sum(CASE WHEN evidence_status = 'MISSING' THEN 1 ELSE 0 END) AS missing_evidence
        """
        async with self.driver.session() as session:
            result = await session.run(
                query,
                framework_id=str(framework_id),
                org_id=str(organization_id),
            )
            record = await result.single()
            return dict(record)
```

---

### Section 4: Service Layer

#### 4.1 Evidence Collection Service

```python
from dataclasses import dataclass
from typing import Protocol, AsyncIterator
import hashlib

class IntegrationConnector(Protocol):
    """Protocol for integration connectors."""
    
    async def authenticate(self) -> None:
        """Authenticate with the external system."""
        ...
    
    async def test_connection(self) -> bool:
        """Test if connection is healthy."""
        ...
    
    async def collect_evidence(
        self,
        resource_types: List[str],
    ) -> AsyncIterator[EvidenceItem]:
        """Collect evidence items from the system."""
        ...


@dataclass
class CollectionContext:
    """Context for evidence collection job."""
    organization_id: UUID
    integration_id: UUID
    job_id: UUID
    connector: IntegrationConnector
    started_at: datetime


class EvidenceCollectionService:
    """
    Service for orchestrating evidence collection from integrations.
    
    Responsibilities:
    - Schedule and execute collection jobs
    - Store collected evidence
    - Detect and report gaps
    - Handle collection failures with retry
    """
    
    def __init__(
        self,
        evidence_repo: FoundationDBRepository[EvidenceItem],
        job_repo: FoundationDBRepository[CollectionJob],
        storage: ObjectStorage,
        connector_factory: ConnectorFactory,
        gap_detector: GapDetector,
        notification_service: NotificationService,
    ):
        self.evidence_repo = evidence_repo
        self.job_repo = job_repo
        self.storage = storage
        self.connector_factory = connector_factory
        self.gap_detector = gap_detector
        self.notification_service = notification_service
    
    async def execute_collection_job(
        self,
        job: CollectionJob,
    ) -> CollectionJobResult:
        """
        Execute evidence collection job.
        
        Algorithm:
        1. Create connector for integration type
        2. Authenticate with stored credentials
        3. Iterate through resource types
        4. For each resource, collect and store evidence
        5. Calculate content hash for deduplication
        6. Update job status and metrics
        7. Run gap detection
        8. Send notifications if gaps found
        """
        context = CollectionContext(
            organization_id=job.organization_id,
            integration_id=job.integration_id,
            job_id=job.id,
            connector=await self.connector_factory.create(job.integration_id),
            started_at=datetime.utcnow(),
        )
        
        collected_count = 0
        error_count = 0
        
        try:
            await context.connector.authenticate()
            
            async for evidence_item in context.connector.collect_evidence(
                resource_types=job.resource_types
            ):
                try:
                    stored_item = await self._store_evidence(context, evidence_item)
                    collected_count += 1
                except Exception as e:
                    logger.error(f"Failed to store evidence: {e}")
                    error_count += 1
            
            # Run gap detection
            gaps = await self.gap_detector.detect_gaps(
                organization_id=context.organization_id,
                framework_ids=job.framework_ids,
            )
            
            if gaps:
                await self.notification_service.send_gap_alert(
                    organization_id=context.organization_id,
                    gaps=gaps,
                )
            
            return CollectionJobResult(
                job_id=job.id,
                status=JobStatus.COMPLETED,
                collected_count=collected_count,
                error_count=error_count,
                gaps_detected=len(gaps),
                duration_seconds=(datetime.utcnow() - context.started_at).total_seconds(),
            )
            
        except Exception as e:
            logger.exception(f"Collection job {job.id} failed: {e}")
            return CollectionJobResult(
                job_id=job.id,
                status=JobStatus.FAILED,
                error_message=str(e),
            )
    
    async def _store_evidence(
        self,
        context: CollectionContext,
        evidence: EvidenceItem,
    ) -> EvidenceItem:
        """
        Store evidence item with deduplication.
        
        Algorithm:
        1. Calculate content hash
        2. Check for existing evidence with same hash
        3. If duplicate, skip storage (idempotent)
        4. Store content in object storage
        5. Save metadata to FoundationDB
        """
        # Calculate content hash
        content_bytes = evidence.content if isinstance(evidence.content, bytes) else evidence.content.encode()
        content_hash = hashlib.sha256(content_bytes).hexdigest()
        
        # Check for duplicate
        existing = await self.evidence_repo.find_by_hash(
            tenant_id=context.organization_id,
            content_hash=content_hash,
        )
        
        if existing:
            logger.debug(f"Duplicate evidence detected: {content_hash}")
            return existing
        
        # Store content in object storage
        storage_path = f"evidence/{context.organization_id}/{evidence.id}/{evidence.type.value}"
        await self.storage.upload(storage_path, content_bytes)
        
        # Create evidence record
        evidence_record = EvidenceItem(
            id=evidence.id,
            organization_id=context.organization_id,
            type=evidence.type,
            source_integration_id=context.integration_id,
            collected_at=datetime.utcnow(),
            content_hash=content_hash,
            storage_location=storage_path,
            control_ids=evidence.control_ids,
            metadata=evidence.metadata,
        )
        
        await self.evidence_repo.save(evidence_record, context.organization_id)
        return evidence_record
```

#### 4.2 Posture Scoring Service

```python
from dataclasses import dataclass
from typing import Dict, List
from decimal import Decimal

@dataclass
class ScoreWeight:
    """Weight configuration for posture scoring."""
    critical: int = 10
    high: int = 5
    medium: int = 2
    low: int = 1
    info: int = 0


class PostureScoringService:
    """
    Service for calculating compliance posture scores.
    
    Scoring Algorithm:
    - Score = Σ(check_weight × check_result) / Σ(check_weight)
    - Weight based on check severity
    - Result: 1 for pass, 0 for fail
    - Skipped/error checks excluded from calculation
    """
    
    def __init__(
        self,
        check_repo: FoundationDBRepository[CheckResult],
        control_repo: ControlGraphRepository,
        weights: ScoreWeight = ScoreWeight(),
    ):
        self.check_repo = check_repo
        self.control_repo = control_repo
        self.weights = weights
    
    async def calculate_posture(
        self,
        organization_id: UUID,
        framework_id: Optional[UUID] = None,
    ) -> PostureScore:
        """
        Calculate overall compliance posture score.
        
        Returns score 0-100 with breakdown by category.
        """
        # Get latest check results
        check_results = await self.check_repo.get_latest_results(
            tenant_id=organization_id,
            framework_id=framework_id,
        )
        
        # Calculate weighted score
        total_weight = Decimal(0)
        weighted_sum = Decimal(0)
        category_scores: Dict[ControlCategory, Dict] = {}
        
        for result in check_results:
            if result.status in (CheckStatus.SKIPPED, CheckStatus.ERROR):
                continue
            
            weight = self._get_weight(result.severity)
            total_weight += weight
            
            if result.status == CheckStatus.PASS:
                weighted_sum += weight
            
            # Track by category
            category = result.control_category
            if category not in category_scores:
                category_scores[category] = {"weight": 0, "sum": 0}
            category_scores[category]["weight"] += weight
            if result.status == CheckStatus.PASS:
                category_scores[category]["sum"] += weight
        
        overall_score = (
            float(weighted_sum / total_weight * 100)
            if total_weight > 0
            else 0.0
        )
        
        # Calculate category breakdowns
        category_breakdown = {
            cat: float(data["sum"] / data["weight"] * 100)
            if data["weight"] > 0 else 0.0
            for cat, data in category_scores.items()
        }
        
        # Calculate trend (compare to 30 days ago)
        historical_score = await self._get_historical_score(
            organization_id, framework_id, days_ago=30
        )
        trend = overall_score - historical_score if historical_score else 0.0
        
        return PostureScore(
            organization_id=organization_id,
            framework_id=framework_id,
            overall_score=round(overall_score, 2),
            category_scores=category_breakdown,
            trend_30d=round(trend, 2),
            total_checks=len(check_results),
            passing_checks=sum(1 for r in check_results if r.status == CheckStatus.PASS),
            failing_checks=sum(1 for r in check_results if r.status == CheckStatus.FAIL),
            calculated_at=datetime.utcnow(),
        )
    
    def _get_weight(self, severity: CheckSeverity) -> Decimal:
        """Get weight for check severity."""
        weight_map = {
            CheckSeverity.CRITICAL: self.weights.critical,
            CheckSeverity.HIGH: self.weights.high,
            CheckSeverity.MEDIUM: self.weights.medium,
            CheckSeverity.LOW: self.weights.low,
            CheckSeverity.INFO: self.weights.info,
        }
        return Decimal(weight_map.get(severity, 1))
```

---

### Section 5: Agent Framework Implementation

#### 5.1 Agent Base Classes

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional, Callable, Awaitable
from enum import Enum
import anthropic

class AgentType(str, Enum):
    REGULATORY_INTELLIGENCE = "regulatory_intelligence"
    EVIDENCE_COLLECTION = "evidence_collection"
    CONTROL_POSTURE = "control_posture"
    REMEDIATION = "remediation"
    AUDIT_PREPARATION = "audit_preparation"
    VENDOR_RISK = "vendor_risk"


@dataclass
class AgentTool:
    """Tool definition for agent use."""
    name: str
    description: str
    input_schema: Dict[str, Any]
    handler: Callable[..., Awaitable[Any]]
    requires_approval: bool = False


@dataclass
class AgentConfig:
    """Configuration for agent execution."""
    agent_type: AgentType
    max_tokens: int = 4096
    max_tool_calls: int = 20
    timeout_seconds: int = 300
    temperature: float = 0.7
    model: str = "claude-sonnet-4-20250514"


@dataclass 
class AgentContext:
    """Execution context passed to agent."""
    organization_id: UUID
    user_id: UUID
    task_id: UUID
    input_params: Dict[str, Any]
    tools: List[AgentTool]
    config: AgentConfig


@dataclass
class AgentResult:
    """Result from agent execution."""
    task_id: UUID
    status: str  # completed, failed, requires_approval
    output: Dict[str, Any]
    tool_calls: List[Dict[str, Any]]
    token_usage: int
    duration_seconds: float
    error_message: Optional[str] = None
    approval_request: Optional[Dict[str, Any]] = None


class ComplianceAgent(ABC):
    """
    Base class for compliance agents.
    
    Implements the agent loop pattern:
    1. Receive task with context
    2. Build system prompt with tool definitions
    3. Execute conversation loop with Claude
    4. Handle tool calls with guardrails
    5. Return structured result
    """
    
    def __init__(
        self,
        client: anthropic.AsyncAnthropic,
        tool_registry: "ToolRegistry",
        checkpoint_manager: "CheckpointManager",
    ):
        self.client = client
        self.tool_registry = tool_registry
        self.checkpoint_manager = checkpoint_manager
        self._circuit_breaker = CircuitBreaker(
            failure_threshold=3,
            recovery_timeout=60.0,
        )
    
    @property
    @abstractmethod
    def agent_type(self) -> AgentType:
        """Return agent type identifier."""
        ...
    
    @property
    @abstractmethod
    def system_prompt(self) -> str:
        """Return agent system prompt."""
        ...
    
    @abstractmethod
    def get_tools(self, context: AgentContext) -> List[AgentTool]:
        """Return tools available to this agent."""
        ...
    
    async def execute(self, context: AgentContext) -> AgentResult:
        """
        Execute agent task.
        
        Algorithm:
        1. Build tool definitions from registry
        2. Initialize conversation with system prompt
        3. Enter agent loop:
           a. Call Claude with current messages
           b. If tool_use, execute tool with guardrails
           c. If text response, check for completion
           d. Handle approval requirements
        4. Return structured result
        """
        start_time = datetime.utcnow()
        tool_calls = []
        token_usage = 0
        
        # Build Claude tool definitions
        tools = self.get_tools(context)
        claude_tools = [
            {
                "name": tool.name,
                "description": tool.description,
                "input_schema": tool.input_schema,
            }
            for tool in tools
        ]
        
        # Initialize messages
        messages = [
            {
                "role": "user",
                "content": self._build_task_prompt(context),
            }
        ]
        
        try:
            # Agent loop
            for iteration in range(context.config.max_tool_calls):
                # Call Claude
                response = await self._circuit_breaker.call(
                    self.client.messages.create,
                    model=context.config.model,
                    max_tokens=context.config.max_tokens,
                    system=self.system_prompt,
                    tools=claude_tools,
                    messages=messages,
                )
                
                token_usage += response.usage.input_tokens + response.usage.output_tokens
                
                # Check for stop conditions
                if response.stop_reason == "end_turn":
                    # Extract final response
                    final_text = self._extract_text(response.content)
                    return AgentResult(
                        task_id=context.task_id,
                        status="completed",
                        output={"response": final_text},
                        tool_calls=tool_calls,
                        token_usage=token_usage,
                        duration_seconds=(datetime.utcnow() - start_time).total_seconds(),
                    )
                
                # Handle tool use
                if response.stop_reason == "tool_use":
                    tool_use_block = self._extract_tool_use(response.content)
                    tool = self._find_tool(tools, tool_use_block.name)
                    
                    # Check if approval required
                    if tool.requires_approval:
                        return AgentResult(
                            task_id=context.task_id,
                            status="requires_approval",
                            output={},
                            tool_calls=tool_calls,
                            token_usage=token_usage,
                            duration_seconds=(datetime.utcnow() - start_time).total_seconds(),
                            approval_request={
                                "tool_name": tool.name,
                                "tool_input": tool_use_block.input,
                                "iteration": iteration,
                            },
                        )
                    
                    # Execute tool
                    tool_result = await self._execute_tool(
                        tool, tool_use_block.input, context
                    )
                    tool_calls.append({
                        "tool": tool.name,
                        "input": tool_use_block.input,
                        "result": tool_result,
                    })
                    
                    # Add to messages for next iteration
                    messages.append({"role": "assistant", "content": response.content})
                    messages.append({
                        "role": "user",
                        "content": [
                            {
                                "type": "tool_result",
                                "tool_use_id": tool_use_block.id,
                                "content": str(tool_result),
                            }
                        ],
                    })
                    
                    # Checkpoint
                    await self.checkpoint_manager.save(
                        context.task_id,
                        {
                            "iteration": iteration,
                            "messages": messages,
                            "tool_calls": tool_calls,
                        },
                    )
            
            # Max iterations reached
            return AgentResult(
                task_id=context.task_id,
                status="completed",
                output={"response": "Max iterations reached"},
                tool_calls=tool_calls,
                token_usage=token_usage,
                duration_seconds=(datetime.utcnow() - start_time).total_seconds(),
            )
            
        except Exception as e:
            logger.exception(f"Agent execution failed: {e}")
            return AgentResult(
                task_id=context.task_id,
                status="failed",
                output={},
                tool_calls=tool_calls,
                token_usage=token_usage,
                duration_seconds=(datetime.utcnow() - start_time).total_seconds(),
                error_message=str(e),
            )
```

#### 5.2 Concrete Agent: Evidence Collection Agent

```python
class EvidenceCollectionAgent(ComplianceAgent):
    """
    Agent for automated evidence collection and validation.
    
    Capabilities:
    - Query integration APIs for evidence
    - Validate evidence against control requirements
    - Detect and report evidence gaps
    - Suggest remediation for missing evidence
    """
    
    @property
    def agent_type(self) -> AgentType:
        return AgentType.EVIDENCE_COLLECTION
    
    @property
    def system_prompt(self) -> str:
        return """You are a compliance evidence collection agent for CODITECT-COMPLIANCE.

Your responsibilities:
1. Collect evidence from integrated systems to verify control compliance
2. Validate collected evidence against control requirements
3. Detect gaps where evidence is missing or stale
4. Suggest remediation actions for gaps

Available tools allow you to:
- Query integrations for configuration and log data
- Store collected evidence with control mappings
- Check evidence freshness and coverage
- Create remediation tasks for gaps

Guidelines:
- Collect evidence systematically by control category
- Validate evidence matches expected criteria before storing
- Report all gaps found with specific control references
- Suggest concrete remediation steps

Output your findings as structured JSON at the end."""
    
    def get_tools(self, context: AgentContext) -> List[AgentTool]:
        return [
            AgentTool(
                name="query_integration",
                description="Query an integration for evidence. Specify the integration type and resource path.",
                input_schema={
                    "type": "object",
                    "properties": {
                        "integration_id": {"type": "string", "format": "uuid"},
                        "resource_type": {"type": "string"},
                        "query_params": {"type": "object"},
                    },
                    "required": ["integration_id", "resource_type"],
                },
                handler=self._query_integration,
            ),
            AgentTool(
                name="store_evidence",
                description="Store collected evidence with control mappings.",
                input_schema={
                    "type": "object",
                    "properties": {
                        "evidence_type": {
                            "type": "string",
                            "enum": ["CONFIG", "LOG", "SCREENSHOT", "DOCUMENT"],
                        },
                        "content": {"type": "string"},
                        "control_ids": {
                            "type": "array",
                            "items": {"type": "string", "format": "uuid"},
                        },
                        "metadata": {"type": "object"},
                    },
                    "required": ["evidence_type", "content", "control_ids"],
                },
                handler=self._store_evidence,
            ),
            AgentTool(
                name="check_evidence_coverage",
                description="Check evidence coverage for a framework or set of controls.",
                input_schema={
                    "type": "object",
                    "properties": {
                        "framework_id": {"type": "string", "format": "uuid"},
                        "control_ids": {
                            "type": "array",
                            "items": {"type": "string", "format": "uuid"},
                        },
                        "freshness_days": {"type": "integer", "default": 30},
                    },
                },
                handler=self._check_coverage,
            ),
            AgentTool(
                name="create_gap_task",
                description="Create a remediation task for an evidence gap.",
                input_schema={
                    "type": "object",
                    "properties": {
                        "control_id": {"type": "string", "format": "uuid"},
                        "gap_type": {
                            "type": "string",
                            "enum": ["MISSING", "STALE", "INVALID"],
                        },
                        "description": {"type": "string"},
                        "suggested_action": {"type": "string"},
                    },
                    "required": ["control_id", "gap_type", "description"],
                },
                handler=self._create_gap_task,
                requires_approval=False,
            ),
        ]
    
    async def _query_integration(
        self,
        integration_id: str,
        resource_type: str,
        query_params: Optional[Dict] = None,
        context: AgentContext = None,
    ) -> Dict[str, Any]:
        """Query integration for evidence data."""
        connector = await self.connector_factory.create(UUID(integration_id))
        result = await connector.query(resource_type, query_params or {})
        return {"status": "success", "data": result}
    
    async def _store_evidence(
        self,
        evidence_type: str,
        content: str,
        control_ids: List[str],
        metadata: Optional[Dict] = None,
        context: AgentContext = None,
    ) -> Dict[str, Any]:
        """Store evidence item."""
        evidence = EvidenceItem(
            organization_id=context.organization_id,
            type=EvidenceType(evidence_type),
            content=content,
            control_ids=[UUID(cid) for cid in control_ids],
            metadata=metadata or {},
        )
        stored = await self.evidence_service.store(evidence)
        return {"status": "success", "evidence_id": str(stored.id)}
```

---

### Section 6: Integration Connector Implementations

#### 6.1 AWS Connector

```python
import boto3
from botocore.config import Config
from typing import AsyncIterator, Dict, Any, List

class AWSConnector(IntegrationConnector):
    """
    AWS integration connector for evidence collection.
    
    Supports:
    - IAM users, roles, policies
    - S3 bucket configurations
    - CloudTrail logs
    - GuardDuty findings
    - Security Hub findings
    - Config compliance
    """
    
    def __init__(
        self,
        credentials: AWSCredentials,
        regions: List[str],
    ):
        self.credentials = credentials
        self.regions = regions
        self._clients: Dict[str, Any] = {}
    
    async def authenticate(self) -> None:
        """Authenticate using AssumeRole or access keys."""
        if self.credentials.role_arn:
            sts = boto3.client(
                "sts",
                aws_access_key_id=self.credentials.access_key_id,
                aws_secret_access_key=self.credentials.secret_access_key,
            )
            assumed = sts.assume_role(
                RoleArn=self.credentials.role_arn,
                RoleSessionName="coditect-compliance",
                ExternalId=self.credentials.external_id,
            )
            self._session_credentials = assumed["Credentials"]
        else:
            self._session_credentials = {
                "AccessKeyId": self.credentials.access_key_id,
                "SecretAccessKey": self.credentials.secret_access_key,
            }
    
    async def test_connection(self) -> bool:
        """Test AWS connection by calling STS GetCallerIdentity."""
        try:
            sts = self._get_client("sts")
            sts.get_caller_identity()
            return True
        except Exception:
            return False
    
    async def collect_evidence(
        self,
        resource_types: List[str],
    ) -> AsyncIterator[EvidenceItem]:
        """Collect evidence from AWS resources."""
        for resource_type in resource_types:
            if resource_type == "iam_users":
                async for item in self._collect_iam_users():
                    yield item
            elif resource_type == "iam_mfa_status":
                async for item in self._collect_iam_mfa():
                    yield item
            elif resource_type == "s3_buckets":
                async for item in self._collect_s3_configs():
                    yield item
            elif resource_type == "cloudtrail_logs":
                async for item in self._collect_cloudtrail():
                    yield item
            elif resource_type == "guardduty_findings":
                async for item in self._collect_guardduty():
                    yield item
            elif resource_type == "securityhub_findings":
                async for item in self._collect_securityhub():
                    yield item
    
    async def _collect_iam_users(self) -> AsyncIterator[EvidenceItem]:
        """Collect IAM user configurations."""
        iam = self._get_client("iam")
        paginator = iam.get_paginator("list_users")
        
        for page in paginator.paginate():
            for user in page["Users"]:
                user_detail = iam.get_user(UserName=user["UserName"])
                mfa_devices = iam.list_mfa_devices(UserName=user["UserName"])
                access_keys = iam.list_access_keys(UserName=user["UserName"])
                
                yield EvidenceItem(
                    type=EvidenceType.CONFIG,
                    content=json.dumps({
                        "user": user_detail["User"],
                        "mfa_devices": mfa_devices["MFADevices"],
                        "access_keys": access_keys["AccessKeyMetadata"],
                    }),
                    control_ids=[],  # Mapped during storage
                    metadata={
                        "aws_resource_type": "iam_user",
                        "aws_account_id": self._account_id,
                        "user_name": user["UserName"],
                    },
                )
    
    async def _collect_iam_mfa(self) -> AsyncIterator[EvidenceItem]:
        """Collect IAM MFA enforcement status."""
        iam = self._get_client("iam")
        
        # Get credential report
        iam.generate_credential_report()
        report = iam.get_credential_report()
        
        yield EvidenceItem(
            type=EvidenceType.CONFIG,
            content=report["Content"].decode("utf-8"),
            control_ids=[],
            metadata={
                "aws_resource_type": "credential_report",
                "aws_account_id": self._account_id,
                "report_format": report["ReportFormat"],
            },
        )
    
    async def _collect_s3_configs(self) -> AsyncIterator[EvidenceItem]:
        """Collect S3 bucket configurations."""
        s3 = self._get_client("s3")
        buckets = s3.list_buckets()["Buckets"]
        
        for bucket in buckets:
            bucket_name = bucket["Name"]
            
            # Collect all relevant configurations
            config = {
                "name": bucket_name,
                "creation_date": bucket["CreationDate"].isoformat(),
            }
            
            try:
                config["encryption"] = s3.get_bucket_encryption(Bucket=bucket_name)
            except s3.exceptions.ClientError:
                config["encryption"] = None
            
            try:
                config["versioning"] = s3.get_bucket_versioning(Bucket=bucket_name)
            except s3.exceptions.ClientError:
                config["versioning"] = None
            
            try:
                config["public_access_block"] = s3.get_public_access_block(
                    Bucket=bucket_name
                )["PublicAccessBlockConfiguration"]
            except s3.exceptions.ClientError:
                config["public_access_block"] = None
            
            try:
                config["logging"] = s3.get_bucket_logging(Bucket=bucket_name)
            except s3.exceptions.ClientError:
                config["logging"] = None
            
            yield EvidenceItem(
                type=EvidenceType.CONFIG,
                content=json.dumps(config, default=str),
                control_ids=[],
                metadata={
                    "aws_resource_type": "s3_bucket",
                    "aws_account_id": self._account_id,
                    "bucket_name": bucket_name,
                },
            )
```

Generate similar connector implementations for:
- GCP Connector
- Okta Connector  
- GitHub Connector
- Jira Connector

---

### Section 7: API Implementation

#### 7.1 FastAPI Router Example

```python
from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from typing import List, Optional
from uuid import UUID

router = APIRouter(prefix="/v1/frameworks", tags=["frameworks"])
security = HTTPBearer()


class FrameworkResponse(BaseModel):
    """API response model for Framework."""
    id: UUID
    name: str
    short_name: str
    version: str
    status: FrameworkStatus
    effective_date: date
    source_url: Optional[str]
    description: Optional[str]
    control_count: int
    created_at: datetime
    updated_at: datetime


class FrameworkListResponse(BaseModel):
    """Paginated list response."""
    data: List[FrameworkResponse]
    pagination: PaginationMeta


class CreateFrameworkRequest(BaseModel):
    """Request model for creating framework."""
    name: str = Field(..., max_length=256)
    short_name: str = Field(..., max_length=32, pattern=r"^[A-Z0-9_]+$")
    version: str = Field(..., pattern=r"^\d+\.\d+(\.\d+)?$")
    effective_date: date
    source_url: Optional[str] = None
    description: Optional[str] = Field(None, max_length=4000)


@router.get("", response_model=FrameworkListResponse)
async def list_frameworks(
    status: Optional[FrameworkStatus] = None,
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    auth: HTTPAuthorizationCredentials = Depends(security),
    framework_service: FrameworkService = Depends(get_framework_service),
    current_user: User = Depends(get_current_user),
) -> FrameworkListResponse:
    """
    List all frameworks accessible to the organization.
    
    Filters:
    - status: Filter by framework status
    
    Pagination:
    - page: Page number (1-indexed)
    - per_page: Items per page (max 100)
    """
    result = await framework_service.list_frameworks(
        organization_id=current_user.organization_id,
        status=status,
        page=page,
        per_page=per_page,
    )
    
    return FrameworkListResponse(
        data=[FrameworkResponse(**f.model_dump()) for f in result.items],
        pagination=PaginationMeta(
            page=page,
            per_page=per_page,
            total=result.total,
            total_pages=result.total_pages,
        ),
    )


@router.get("/{framework_id}", response_model=FrameworkResponse)
async def get_framework(
    framework_id: UUID,
    auth: HTTPAuthorizationCredentials = Depends(security),
    framework_service: FrameworkService = Depends(get_framework_service),
    current_user: User = Depends(get_current_user),
) -> FrameworkResponse:
    """Get framework by ID."""
    framework = await framework_service.get_framework(
        organization_id=current_user.organization_id,
        framework_id=framework_id,
    )
    
    if not framework:
        raise HTTPException(status_code=404, detail="Framework not found")
    
    return FrameworkResponse(**framework.model_dump())


@router.post("", response_model=FrameworkResponse, status_code=201)
async def create_framework(
    request: CreateFrameworkRequest,
    auth: HTTPAuthorizationCredentials = Depends(security),
    framework_service: FrameworkService = Depends(get_framework_service),
    current_user: User = Depends(get_current_user),
) -> FrameworkResponse:
    """
    Create a new framework.
    
    Requires admin role.
    """
    if not current_user.has_permission("frameworks:create"):
        raise HTTPException(status_code=403, detail="Insufficient permissions")
    
    framework = await framework_service.create_framework(
        organization_id=current_user.organization_id,
        created_by=current_user.id,
        **request.model_dump(),
    )
    
    return FrameworkResponse(**framework.model_dump())
```

---

### Section 8: Check Definitions

Generate check definition schema and examples:

```yaml
check_definition_schema:
  description: "Schema for defining automated compliance checks"
  properties:
    id:
      type: "string"
      pattern: "^chk-[a-z0-9-]+$"
    name:
      type: "string"
      max_length: 256
    description:
      type: "string"
    severity:
      type: "enum"
      values: ["CRITICAL", "HIGH", "MEDIUM", "LOW", "INFO"]
    control_ids:
      type: "array"
      items:
        type: "uuid"
    integration_type:
      type: "string"
      description: "Integration type this check runs against"
    resource_type:
      type: "string"
      description: "Resource type within the integration"
    check_logic:
      type: "object"
      description: "Logic definition for the check"

check_examples:
  aws_mfa_enforcement:
    id: "chk-aws-iam-mfa-enforcement"
    name: "IAM MFA Enforcement"
    description: "Verify MFA is enabled for all IAM users with console access"
    severity: "CRITICAL"
    control_ids: ["CC6.1", "A.9.4.2"]  # SOC2, ISO27001 mappings
    integration_type: "aws"
    resource_type: "credential_report"
    check_logic:
      type: "csv_query"
      query: |
        SELECT user, mfa_active, password_enabled
        FROM credential_report
        WHERE password_enabled = 'true' AND mfa_active = 'false'
      pass_condition: "result_count == 0"
      failure_message: "Users with console access but no MFA: {result}"
      
  aws_s3_encryption:
    id: "chk-aws-s3-encryption"
    name: "S3 Bucket Encryption"
    description: "Verify all S3 buckets have server-side encryption enabled"
    severity: "HIGH"
    control_ids: ["CC6.7", "A.10.1.1"]
    integration_type: "aws"
    resource_type: "s3_bucket"
    check_logic:
      type: "json_query"
      query: |
        $.encryption.ServerSideEncryptionConfiguration
      pass_condition: "result != null"
      failure_message: "Bucket {metadata.bucket_name} lacks encryption"
      
  github_branch_protection:
    id: "chk-github-branch-protection"
    name: "GitHub Branch Protection"
    description: "Verify main/master branches have protection rules enabled"
    severity: "HIGH"
    control_ids: ["CC8.1", "A.14.2.2"]
    integration_type: "github"
    resource_type: "repository"
    check_logic:
      type: "json_query"
      query: |
        $.branch_protection_rules[?(@.pattern == 'main' || @.pattern == 'master')]
      pass_condition: |
        result != null &&
        result.required_pull_request_reviews != null &&
        result.required_status_checks != null
      failure_message: "Repository {metadata.repo_name} lacks branch protection"
```

---

## Output Format

Generate the document as a single markdown file with:
- Complete Python code examples
- Type definitions and schemas
- Algorithm descriptions with pseudocode
- Interface contracts
- Integration implementations

---

## Acceptance Criteria

The generated document must:
1. Be 15,000-25,000 words
2. Include complete, runnable Python code examples
3. Define all domain models with Pydantic
4. Include repository patterns for all data stores
5. Include service layer implementations
6. Include agent framework with concrete agent
7. Include at least one complete integration connector
8. Include API route implementations
9. Include check definition schema and examples
10. Be directly usable for code generation

---

## Execution

Generate the complete Technical Design Document now.
