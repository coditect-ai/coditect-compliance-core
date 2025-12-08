# Prompt 07: Architecture Decision Record - Evidence Collection Pipeline

## Context

You are a principal architect designing the automated evidence collection pipeline for CODITECT-COMPLIANCE. This ADR establishes how compliance evidence is collected, validated, stored, and linked to controls across multiple integrated systems.

## Output Specification

Generate a comprehensive Architecture Decision Record (ADR) following the standard ADR format. The document should be 3,500-5,000 words (10,000-15,000 tokens).

## Document Structure

### ADR-003: Evidence Collection Pipeline Architecture

```markdown
# ADR-003: Evidence Collection Pipeline Architecture

## Status
Proposed | Accepted | Deprecated | Superseded

## Date
[Current Date]

## Decision Makers
- [Role: Chief Architect]
- [Role: Integration Platform Lead]
- [Role: Compliance Domain Expert]

## Context

### Problem Statement

CODITECT-COMPLIANCE must automate evidence collection across 15+ integrated systems to achieve:
- **80-90% automation rate** for compliance evidence gathering
- **Continuous monitoring** with near-real-time posture updates
- **Audit-ready packages** with traceable evidence chains
- **Gap detection** for controls lacking sufficient evidence

### Evidence Types and Sources

| Evidence Type | Source Systems | Collection Method | Frequency |
|---------------|----------------|-------------------|-----------|
| Identity & Access | Okta, Azure AD, AWS IAM | API polling | Daily |
| Code Security | GitHub, GitLab | Webhooks + API | Real-time |
| Infrastructure Config | AWS, GCP, Azure | API + Config snapshots | Hourly |
| Vulnerability Scans | Snyk, Qualys | Webhook callbacks | On-scan |
| HR/Personnel | BambooHR, Workday | API polling | Daily |
| Security Training | KnowBe4, Curricula | API polling | Weekly |
| Ticket/Issues | Jira, Linear | Webhooks | Real-time |
| Logs/Monitoring | Datadog, Splunk | Log forwarding | Streaming |

### Collection Challenges

1. **Volume**: 1M+ evidence items per large enterprise
2. **Variety**: Unstructured (screenshots) to structured (JSON configs)
3. **Velocity**: Some evidence changes hourly (configs), some annually (policies)
4. **Verification**: Evidence must be validated against control requirements
5. **Versioning**: Historical evidence needed for audit periods
6. **Deduplication**: Avoid storing redundant evidence
7. **Retention**: Different retention periods per framework (1-7 years)

### Technical Constraints

- Integration connectors follow standardized interface
- Evidence stored in GCS with metadata in FoundationDB
- Neo4j maintains evidence-to-control relationships
- Must support both push (webhooks) and pull (polling) collection
- Multi-tenant with strict data isolation

## Decision Drivers

1. **Reliability**: No evidence loss, even during system failures
2. **Scalability**: Handle enterprise-scale evidence volumes
3. **Latency**: Near-real-time for continuous monitoring use cases
4. **Storage Efficiency**: Minimize redundant storage
5. **Query Performance**: Fast retrieval for audits and dashboards
6. **Compliance**: Evidence chain must be tamper-evident

## Options Considered

### Option 1: Synchronous Collection Pipeline

**Description**: Evidence collected synchronously during integration API calls, processed inline, stored immediately.

```
Integration API → Connector → Validator → Storage
                    ↓
              [Blocking Wait]
```

**Pros**:
- Simple architecture
- Immediate consistency
- Easy error handling

**Cons**:
- Poor scalability under load
- Slow for large evidence batches
- Integration timeouts block collection
- No retry mechanism

### Option 2: Asynchronous Queue-Based Pipeline

**Description**: Evidence collection produces messages to queue; separate workers process, validate, and store.

```
Integration → Connector → Queue → Worker Pool → Storage
                              ↓
                        [Async Processing]
```

**Pros**:
- Scalable worker pool
- Built-in retry with DLQ
- Decoupled components
- Handles burst traffic

**Cons**:
- Eventual consistency
- Queue infrastructure cost
- More complex monitoring

### Option 3: Event-Sourced Pipeline with CQRS

**Description**: All evidence events stored immutably; read models built from event stream.

```
Integration → Event Store → Projector → Read Model
                   ↓
            [Immutable Log]
```

**Pros**:
- Complete audit trail
- Temporal queries ("state at time X")
- Replay capability
- Tamper-evident

**Cons**:
- Storage growth
- Projection complexity
- Higher latency for reads
- Team learning curve

### Option 4: Hybrid - Queue + Event Sourcing for Audit Trail

**Description**: Queue-based processing with event sourcing for audit-critical evidence only.

```
Integration → Connector → Message Queue → Worker
                              ↓              ↓
                      [Priority Routing]     ↓
                              ↓              ↓
                    High-Value Evidence → Event Store
                              ↓              ↓
                    Standard Evidence → Direct Storage
```

## Decision

**Chosen Option**: Option 4 - Hybrid Queue + Event Sourcing

### Rationale

1. **Queue-based** provides scalability and reliability for all evidence
2. **Event sourcing** for audit-critical evidence ensures tamper-evident trails
3. **Priority routing** optimizes storage costs while maintaining compliance
4. **Flexibility** to evolve retention policies per evidence type

## Detailed Design

### Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Evidence Collection Pipeline                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Webhook    │     │   Polling    │     │   Agent      │
│   Receiver   │     │   Scheduler  │     │   Collected  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Integration Connector Layer                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ AWS         │ │ Okta        │ │ GitHub      │ │ Jira        │ ...       │
│  │ Connector   │ │ Connector   │ │ Connector   │ │ Connector   │           │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘           │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Message Queue (Redis Streams)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ evidence.raw.{integration_type}                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
           ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
           │   Worker 1    │ │   Worker 2    │ │   Worker N    │
           │ (Validation)  │ │ (Validation)  │ │ (Validation)  │
           └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
                   │                 │                 │
                   ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Evidence Processing Pipeline                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │ Normalize  │→ │ Dedupe     │→ │ Validate   │→ │ Enrich     │            │
│  │            │  │ (SHA-256)  │  │            │  │ (Controls) │            │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │        Classification Router      │
                    └─────────────────┬─────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            ▼                         ▼                         ▼
   ┌────────────────┐       ┌────────────────┐       ┌────────────────┐
   │ Audit-Critical │       │ Standard       │       │ Transient      │
   │ (Event Store)  │       │ (Direct Store) │       │ (Cache Only)   │
   └────────┬───────┘       └────────┬───────┘       └────────┬───────┘
            │                        │                        │
            ▼                        ▼                        ▼
   ┌────────────────┐       ┌────────────────┐       ┌────────────────┐
   │ FoundationDB   │       │ GCS + FDB      │       │ Redis Cache    │
   │ Event Log      │       │ Metadata       │       │ (TTL: 24h)     │
   └────────────────┘       └────────────────┘       └────────────────┘
            │                        │                        │
            └────────────────────────┼────────────────────────┘
                                     ▼
                        ┌────────────────────────┐
                        │ Neo4j Graph Update     │
                        │ (Evidence → Controls)  │
                        └────────────────────────┘
```

### Data Models

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Dict, List, Optional, Any
import hashlib

class EvidenceType(Enum):
    CONFIGURATION = "configuration"
    SCREENSHOT = "screenshot"
    LOG = "log"
    REPORT = "report"
    POLICY_DOCUMENT = "policy_document"
    CERTIFICATION = "certification"
    TRAINING_RECORD = "training_record"
    ACCESS_REVIEW = "access_review"
    VULNERABILITY_SCAN = "vulnerability_scan"
    PENETRATION_TEST = "penetration_test"

class EvidenceClassification(Enum):
    AUDIT_CRITICAL = "audit_critical"   # Event-sourced, 7-year retention
    STANDARD = "standard"               # Direct storage, framework-based retention
    TRANSIENT = "transient"             # Cache-only, 24h TTL

class EvidenceStatus(Enum):
    PENDING = "pending"
    VALIDATED = "validated"
    REJECTED = "rejected"
    EXPIRED = "expired"
    SUPERSEDED = "superseded"

@dataclass
class RawEvidenceItem:
    """Raw evidence from integration before processing."""
    integration_id: str
    integration_type: str
    raw_data: Dict[str, Any]
    collected_at: datetime
    organization_id: str
    collector: str  # "system", "agent:{agent_id}", "manual:{user_id}"
    
    def compute_hash(self) -> str:
        """Compute content hash for deduplication."""
        content = json.dumps(self.raw_data, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()

@dataclass
class ProcessedEvidenceItem:
    """Evidence after processing pipeline."""
    id: str
    organization_id: str
    evidence_type: EvidenceType
    classification: EvidenceClassification
    
    # Source tracking
    integration_id: str
    integration_type: str
    collector: str
    
    # Content
    title: str
    description: str
    content_hash: str
    storage_uri: str  # gs://bucket/path or fdb://key
    
    # Temporal
    collected_at: datetime
    effective_date: datetime
    expires_at: Optional[datetime]
    
    # Control mapping
    control_ids: List[str]
    framework_ids: List[str]
    
    # Validation
    status: EvidenceStatus
    validation_result: Optional[Dict]
    
    # Metadata
    metadata: Dict[str, Any] = field(default_factory=dict)
    tags: List[str] = field(default_factory=list)
    
    # Audit trail
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)
    version: int = 1

@dataclass
class EvidenceEvent:
    """Immutable event for audit-critical evidence."""
    event_id: str
    event_type: str  # "collected", "validated", "linked", "expired"
    evidence_id: str
    organization_id: str
    timestamp: datetime
    actor: str
    payload: Dict[str, Any]
    previous_event_id: Optional[str]  # Chain for integrity
    
    def compute_integrity_hash(self) -> str:
        """Compute hash including previous event for tamper evidence."""
        content = f"{self.event_id}:{self.previous_event_id}:{json.dumps(self.payload)}"
        return hashlib.sha256(content.encode()).hexdigest()
```

### Connector Interface

```python
from abc import ABC, abstractmethod
from typing import AsyncIterator

class IntegrationConnector(ABC):
    """Base class for all integration connectors."""
    
    @property
    @abstractmethod
    def integration_type(self) -> str:
        """Unique identifier for this integration type."""
        pass
        
    @property
    @abstractmethod
    def supported_evidence_types(self) -> List[EvidenceType]:
        """Evidence types this connector can collect."""
        pass
        
    @abstractmethod
    async def test_connection(
        self,
        credentials: IntegrationCredentials
    ) -> ConnectionTestResult:
        """Verify connectivity and permissions."""
        pass
        
    @abstractmethod
    async def collect_evidence(
        self,
        credentials: IntegrationCredentials,
        config: CollectionConfig
    ) -> AsyncIterator[RawEvidenceItem]:
        """
        Collect evidence from the integrated system.
        
        Yields RawEvidenceItem instances as they are collected.
        Must handle pagination internally.
        """
        pass
        
    @abstractmethod
    async def handle_webhook(
        self,
        payload: Dict[str, Any],
        headers: Dict[str, str]
    ) -> List[RawEvidenceItem]:
        """Process incoming webhook payload."""
        pass

class AWSConnector(IntegrationConnector):
    """AWS integration connector."""
    
    integration_type = "aws"
    supported_evidence_types = [
        EvidenceType.CONFIGURATION,
        EvidenceType.LOG,
        EvidenceType.VULNERABILITY_SCAN
    ]
    
    async def collect_evidence(
        self,
        credentials: IntegrationCredentials,
        config: CollectionConfig
    ) -> AsyncIterator[RawEvidenceItem]:
        """Collect AWS evidence."""
        session = await self._assume_role(credentials)
        
        # IAM Users and MFA
        async for user in self._list_iam_users(session):
            yield RawEvidenceItem(
                integration_id=credentials.integration_id,
                integration_type=self.integration_type,
                raw_data={
                    "resource_type": "iam_user",
                    "user": user,
                    "mfa_devices": await self._get_mfa_devices(session, user)
                },
                collected_at=datetime.utcnow(),
                organization_id=credentials.organization_id,
                collector="system"
            )
            
        # S3 Bucket Configurations
        async for bucket in self._list_s3_buckets(session):
            encryption = await self._get_bucket_encryption(session, bucket)
            public_access = await self._get_bucket_public_access(session, bucket)
            
            yield RawEvidenceItem(
                integration_id=credentials.integration_id,
                integration_type=self.integration_type,
                raw_data={
                    "resource_type": "s3_bucket",
                    "bucket_name": bucket,
                    "encryption": encryption,
                    "public_access_block": public_access
                },
                collected_at=datetime.utcnow(),
                organization_id=credentials.organization_id,
                collector="system"
            )
        
        # Continue for other AWS resources...
```

### Processing Pipeline

```python
class EvidenceProcessor:
    """Core evidence processing pipeline."""
    
    def __init__(
        self,
        normalizer: EvidenceNormalizer,
        deduplicator: EvidenceDeduplicator,
        validator: EvidenceValidator,
        enricher: ControlEnricher,
        classifier: EvidenceClassifier,
        storage: EvidenceStorage,
        event_store: EventStore,
        graph_sync: GraphSynchronizer
    ):
        self.normalizer = normalizer
        self.deduplicator = deduplicator
        self.validator = validator
        self.enricher = enricher
        self.classifier = classifier
        self.storage = storage
        self.event_store = event_store
        self.graph_sync = graph_sync
        
    async def process(
        self,
        raw_evidence: RawEvidenceItem
    ) -> ProcessedEvidenceItem:
        """
        Process raw evidence through the pipeline.
        
        Steps:
        1. Normalize to standard format
        2. Deduplicate against existing evidence
        3. Validate against schema and control requirements
        4. Enrich with control mappings
        5. Classify for storage routing
        6. Store in appropriate backend
        7. Sync to graph database
        """
        # Step 1: Normalize
        normalized = await self.normalizer.normalize(raw_evidence)
        
        # Step 2: Deduplicate
        existing = await self.deduplicator.find_duplicate(normalized)
        if existing:
            # Update existing instead of creating new
            return await self._handle_duplicate(existing, normalized)
            
        # Step 3: Validate
        validation_result = await self.validator.validate(normalized)
        if not validation_result.is_valid:
            normalized.status = EvidenceStatus.REJECTED
            normalized.validation_result = validation_result.to_dict()
            # Still store rejected evidence for audit trail
            
        # Step 4: Enrich with control mappings
        enriched = await self.enricher.enrich(normalized)
        
        # Step 5: Classify
        classification = await self.classifier.classify(enriched)
        enriched.classification = classification
        
        # Step 6: Store based on classification
        stored = await self._store_by_classification(enriched)
        
        # Step 7: Sync to graph
        await self.graph_sync.sync_evidence(stored)
        
        return stored
        
    async def _store_by_classification(
        self,
        evidence: ProcessedEvidenceItem
    ) -> ProcessedEvidenceItem:
        """Route to appropriate storage based on classification."""
        
        if evidence.classification == EvidenceClassification.AUDIT_CRITICAL:
            # Event-source for full audit trail
            event = EvidenceEvent(
                event_id=str(uuid.uuid4()),
                event_type="collected",
                evidence_id=evidence.id,
                organization_id=evidence.organization_id,
                timestamp=datetime.utcnow(),
                actor=evidence.collector,
                payload=evidence.to_dict(),
                previous_event_id=None
            )
            await self.event_store.append(event)
            evidence.storage_uri = f"fdb://events/{evidence.id}"
            
        elif evidence.classification == EvidenceClassification.STANDARD:
            # Standard storage: content to GCS, metadata to FDB
            gcs_uri = await self.storage.store_content(evidence)
            evidence.storage_uri = gcs_uri
            await self.storage.store_metadata(evidence)
            
        else:  # TRANSIENT
            # Cache only with TTL
            await self.storage.cache_evidence(evidence, ttl=86400)
            evidence.storage_uri = f"cache://{evidence.id}"
            
        return evidence

class EvidenceClassifier:
    """Classify evidence for storage routing."""
    
    # Evidence types that require event sourcing
    AUDIT_CRITICAL_TYPES = {
        EvidenceType.POLICY_DOCUMENT,
        EvidenceType.CERTIFICATION,
        EvidenceType.PENETRATION_TEST,
        EvidenceType.ACCESS_REVIEW
    }
    
    # Evidence types that can be cached only
    TRANSIENT_TYPES = {
        EvidenceType.LOG,  # If real-time metrics only
    }
    
    # Control categories requiring audit trail
    AUDIT_CRITICAL_CONTROL_CATEGORIES = {
        "access_control",
        "incident_response",
        "change_management",
        "vendor_management"
    }
    
    async def classify(
        self,
        evidence: ProcessedEvidenceItem
    ) -> EvidenceClassification:
        """Determine storage classification."""
        
        # Type-based classification
        if evidence.evidence_type in self.AUDIT_CRITICAL_TYPES:
            return EvidenceClassification.AUDIT_CRITICAL
            
        # Check if linked to audit-critical controls
        for control_id in evidence.control_ids:
            control = await self.control_repo.get(control_id)
            if control.category in self.AUDIT_CRITICAL_CONTROL_CATEGORIES:
                return EvidenceClassification.AUDIT_CRITICAL
                
        # Transient for specific real-time use cases
        if evidence.evidence_type in self.TRANSIENT_TYPES:
            if not evidence.control_ids:  # No control linkage
                return EvidenceClassification.TRANSIENT
                
        return EvidenceClassification.STANDARD
```

### Deduplication Strategy

```python
class EvidenceDeduplicator:
    """Deduplicate evidence based on content hash."""
    
    def __init__(self, evidence_repo: EvidenceRepository):
        self.repo = evidence_repo
        
    async def find_duplicate(
        self,
        evidence: ProcessedEvidenceItem
    ) -> Optional[ProcessedEvidenceItem]:
        """
        Find existing evidence with same content hash.
        
        Deduplication scope:
        - Same organization
        - Same integration
        - Same content hash
        - Within lookback window (default 90 days)
        """
        return await self.repo.find_by_hash(
            organization_id=evidence.organization_id,
            integration_id=evidence.integration_id,
            content_hash=evidence.content_hash,
            since=datetime.utcnow() - timedelta(days=90)
        )
        
    async def compute_hash(
        self,
        evidence: ProcessedEvidenceItem
    ) -> str:
        """
        Compute content hash for deduplication.
        
        Hash includes:
        - Evidence type
        - Core content (normalized)
        - Integration source
        
        Does NOT include:
        - Timestamps
        - Metadata
        - Control mappings (may change)
        """
        hash_input = {
            "type": evidence.evidence_type.value,
            "integration_type": evidence.integration_type,
            "content": self._normalize_content(evidence.raw_content)
        }
        
        return hashlib.sha256(
            json.dumps(hash_input, sort_keys=True).encode()
        ).hexdigest()
```

### Gap Detection

```python
class EvidenceGapDetector:
    """Detect controls lacking sufficient evidence."""
    
    def __init__(
        self,
        graph_repo: ControlGraphRepository,
        evidence_repo: EvidenceRepository
    ):
        self.graph = graph_repo
        self.evidence = evidence_repo
        
    async def detect_gaps(
        self,
        organization_id: str,
        framework_id: str,
        lookback_days: int = 90
    ) -> List[EvidenceGap]:
        """
        Identify controls with evidence gaps.
        
        A gap exists when:
        1. Control has no evidence within lookback period
        2. Control has only expired evidence
        3. Control has rejected/invalid evidence only
        4. Evidence coverage is below threshold (e.g., < 80%)
        """
        gaps = []
        
        # Get all controls for framework
        controls = await self.graph.get_framework_controls(
            organization_id=organization_id,
            framework_id=framework_id
        )
        
        cutoff = datetime.utcnow() - timedelta(days=lookback_days)
        
        for control in controls:
            # Get evidence for control
            evidence = await self.evidence.find_for_control(
                organization_id=organization_id,
                control_id=control.id,
                since=cutoff,
                status=EvidenceStatus.VALIDATED
            )
            
            if not evidence:
                gaps.append(EvidenceGap(
                    control_id=control.id,
                    control_name=control.title,
                    gap_type=GapType.NO_EVIDENCE,
                    severity=self._calculate_severity(control),
                    last_evidence_date=None,
                    recommended_action=self._recommend_action(control, GapType.NO_EVIDENCE)
                ))
                continue
                
            # Check for expired evidence
            latest = max(evidence, key=lambda e: e.collected_at)
            if latest.expires_at and latest.expires_at < datetime.utcnow():
                gaps.append(EvidenceGap(
                    control_id=control.id,
                    control_name=control.title,
                    gap_type=GapType.EXPIRED_EVIDENCE,
                    severity=self._calculate_severity(control),
                    last_evidence_date=latest.collected_at,
                    recommended_action=self._recommend_action(control, GapType.EXPIRED_EVIDENCE)
                ))
                continue
                
            # Check coverage threshold
            coverage = await self._calculate_coverage(control, evidence)
            if coverage < 0.8:
                gaps.append(EvidenceGap(
                    control_id=control.id,
                    control_name=control.title,
                    gap_type=GapType.PARTIAL_COVERAGE,
                    severity=self._calculate_severity(control),
                    last_evidence_date=latest.collected_at,
                    coverage_percent=coverage * 100,
                    recommended_action=self._recommend_action(control, GapType.PARTIAL_COVERAGE)
                ))
                
        return gaps
```

## Consequences

### Positive
1. **Reliability**: Queue-based processing handles failures gracefully
2. **Scalability**: Horizontal scaling of processing workers
3. **Auditability**: Event sourcing for critical evidence provides tamper-evident trail
4. **Efficiency**: Deduplication reduces storage costs
5. **Flexibility**: Classification allows cost-optimized storage routing

### Negative
1. **Complexity**: Multiple storage backends to manage
2. **Eventual Consistency**: Gap detection may be slightly stale
3. **Storage Costs**: Event store grows unbounded for audit-critical evidence

### Mitigations
- Implement comprehensive monitoring for pipeline health
- Use materialized views for frequently-accessed gap reports
- Implement event log compaction for evidence that's been superseded

## Implementation Plan

### Phase 1: Core Pipeline (Week 1-2)
- Implement base IntegrationConnector interface
- Set up Redis Streams for message queue
- Create processing worker pool
- Unit tests for pipeline stages

### Phase 2: Storage Layer (Week 3)
- Implement GCS storage for evidence content
- FoundationDB schema for metadata
- Event store for audit-critical evidence
- Neo4j sync for evidence-control relationships

### Phase 3: Connectors (Week 4-5)
- AWS Connector (IAM, S3, CloudTrail)
- Okta Connector (Users, Groups, MFA)
- GitHub Connector (Repos, Branch Protection)

### Phase 4: Gap Detection (Week 6)
- Implement gap detection queries
- Create gap notification workflows
- Dashboard integration

## Validation Criteria

1. **Throughput**: Process 10,000 evidence items/hour
2. **Latency**: 95th percentile processing time < 30 seconds
3. **Deduplication**: 99.9% accuracy in duplicate detection
4. **Gap Detection**: Identify 100% of controls lacking evidence
5. **Reliability**: Zero evidence loss under failure conditions

## References

- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
- [GCS Best Practices](https://cloud.google.com/storage/docs/best-practices)
- [Redis Streams Documentation](https://redis.io/docs/data-types/streams/)
```

## Acceptance Criteria

1. **Pipeline Coverage**: All stages documented with code examples
2. **Data Models**: Complete schemas for raw, processed, and events
3. **Classification Logic**: Clear rules for storage routing
4. **Gap Detection**: Algorithm for identifying evidence gaps
5. **Integration Pattern**: Connector interface with example implementation

## Token Budget

- Target: 12,000-18,000 tokens
- Priority: Pipeline architecture and data model sections

## Dependencies

- Input: PRD evidence requirements (FR-EV-*)
- Input: SDD evidence service container
- Input: ADR-001 (Control Graph) for evidence-control relationships
- Output: Feeds into component build prompts for Evidence Engine

## Integration Points

This ADR establishes patterns used by:
- Agent Orchestration ADR (EvidenceCollectionAgent tools)
- Integration Framework ADR (connector interface)
- Component build prompts for Evidence Engine and Integration Connectors
