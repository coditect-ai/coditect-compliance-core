# Prompt 12: Component Build - Evidence Collection Engine

## Context

You are a senior software engineer implementing the Evidence Collection Engine for CODITECT-COMPLIANCE. This component automates the collection, validation, and storage of compliance evidence from integrated systems.

## Output Specification

Generate complete, production-ready Python code for the Evidence Collection Engine. Output should be 2,000-3,000 lines of code.

## Implementation Requirements

### Technology Stack
- Python 3.12+
- Redis Streams for message queue
- FoundationDB for metadata storage
- GCS for blob storage
- Async processing with asyncio

### Architecture Pattern
- Pipeline pattern for evidence processing
- Event-driven collection triggers
- Deduplication via content hashing

## Component Specifications

### 1. Evidence Collector

```python
# File: src/services/evidence/collector.py

"""
Evidence collection orchestrator.

Coordinates evidence collection from multiple integrations.
"""

from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import List, AsyncIterator, Optional, Dict, Any
from enum import Enum
import asyncio
import hashlib

from domain.models.evidence import (
    RawEvidenceItem, 
    ProcessedEvidenceItem,
    EvidenceType,
    EvidenceClassification,
    EvidenceStatus
)
from domain.models.integration import Integration

class CollectionTrigger(Enum):
    """What triggered evidence collection."""
    SCHEDULED = "scheduled"      # Cron-based collection
    WEBHOOK = "webhook"          # Push from integration
    MANUAL = "manual"            # User-initiated
    AGENT = "agent"              # AI agent request
    GAP_DETECTION = "gap"        # Gap analysis triggered

@dataclass
class CollectionJob:
    """A collection job to be executed."""
    id: str
    organization_id: str
    integration_id: str
    trigger: CollectionTrigger
    control_ids: Optional[List[str]] = None  # Specific controls to collect for
    created_at: datetime = None
    
    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.utcnow()

class EvidenceCollector:
    """
    Orchestrates evidence collection from integrations.
    
    Responsibilities:
    - Schedule and execute collection jobs
    - Coordinate with integration connectors
    - Route collected evidence to processing pipeline
    """
    
    def __init__(
        self,
        connector_registry: ConnectorRegistry,
        job_queue: RedisJobQueue,
        pipeline: EvidenceProcessingPipeline,
        metrics: MetricsCollector
    ):
        self.connectors = connector_registry
        self.queue = job_queue
        self.pipeline = pipeline
        self.metrics = metrics
        
    async def schedule_collection(
        self,
        integration_id: str,
        trigger: CollectionTrigger,
        control_ids: Optional[List[str]] = None,
        priority: str = "normal"
    ) -> str:
        """
        Schedule an evidence collection job.
        
        Args:
            integration_id: Integration to collect from
            trigger: What triggered this collection
            control_ids: Optional specific controls to target
            priority: Job priority (high, normal, low)
            
        Returns:
            Job ID for tracking
        """
        ctx = get_tenant_context()
        
        job = CollectionJob(
            id=str(uuid4()),
            organization_id=ctx.organization_id,
            integration_id=integration_id,
            trigger=trigger,
            control_ids=control_ids
        )
        
        await self.queue.enqueue(job, priority=priority)
        
        self.metrics.increment(
            "evidence_collection_jobs_scheduled",
            labels={"trigger": trigger.value, "priority": priority}
        )
        
        return job.id
        
    async def execute_collection(self, job: CollectionJob) -> CollectionResult:
        """
        Execute a collection job.
        
        Steps:
        1. Get integration connector
        2. Fetch credentials
        3. Collect evidence items
        4. Send to processing pipeline
        5. Record results
        """
        start_time = datetime.utcnow()
        items_collected = 0
        errors = []
        
        try:
            # Get connector for integration type
            integration = await self.integration_repo.get(job.integration_id)
            connector = self.connectors.get(integration.integration_type)
            
            if not connector:
                raise ConnectorNotFoundError(
                    f"No connector for {integration.integration_type}"
                )
            
            # Get credentials
            credentials = await self.credential_vault.get(
                integration.credential_id
            )
            
            # Collect evidence
            async for raw_item in connector.collect_evidence(
                credentials=credentials,
                config=CollectionConfig(
                    control_ids=job.control_ids,
                    since=self._get_last_collection_time(job.integration_id)
                )
            ):
                try:
                    # Send to processing pipeline
                    await self.pipeline.process(raw_item)
                    items_collected += 1
                    
                except ProcessingError as e:
                    errors.append({
                        "item": raw_item.id,
                        "error": str(e)
                    })
                    
            # Record success
            result = CollectionResult(
                job_id=job.id,
                status="completed",
                items_collected=items_collected,
                errors=errors,
                duration_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )
            
        except Exception as e:
            result = CollectionResult(
                job_id=job.id,
                status="failed",
                items_collected=items_collected,
                errors=[{"error": str(e)}],
                duration_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )
            
        # Emit metrics
        self.metrics.observe(
            "evidence_collection_duration_ms",
            result.duration_ms,
            labels={"status": result.status}
        )
        
        return result
```

### 2. Processing Pipeline

```python
# File: src/services/evidence/pipeline.py

"""
Evidence processing pipeline.

Transforms raw evidence through normalization, validation, 
deduplication, and storage.
"""

class EvidenceProcessingPipeline:
    """
    Multi-stage evidence processing pipeline.
    
    Stages:
    1. Normalize - Convert to standard format
    2. Deduplicate - Check for existing evidence
    3. Validate - Verify against control requirements
    4. Enrich - Add control mappings
    5. Classify - Determine storage tier
    6. Store - Persist to appropriate storage
    7. Index - Update graph relationships
    """
    
    def __init__(
        self,
        normalizer: EvidenceNormalizer,
        deduplicator: EvidenceDeduplicator,
        validator: EvidenceValidator,
        enricher: ControlEnricher,
        classifier: EvidenceClassifier,
        storage: EvidenceStorage,
        graph_sync: GraphSynchronizer,
        event_bus: EventBus
    ):
        self.normalizer = normalizer
        self.deduplicator = deduplicator
        self.validator = validator
        self.enricher = enricher
        self.classifier = classifier
        self.storage = storage
        self.graph_sync = graph_sync
        self.events = event_bus
        
    async def process(self, raw: RawEvidenceItem) -> ProcessedEvidenceItem:
        """Process a single evidence item through the pipeline."""
        
        # Stage 1: Normalize
        normalized = await self.normalizer.normalize(raw)
        
        # Stage 2: Deduplicate
        existing = await self.deduplicator.find_duplicate(normalized)
        if existing:
            return await self._handle_duplicate(existing, normalized)
            
        # Stage 3: Validate
        validation = await self.validator.validate(normalized)
        normalized.status = (
            EvidenceStatus.VALIDATED if validation.is_valid 
            else EvidenceStatus.REJECTED
        )
        normalized.validation_result = validation.to_dict()
        
        # Stage 4: Enrich with control mappings
        enriched = await self.enricher.enrich(normalized)
        
        # Stage 5: Classify for storage routing
        enriched.classification = await self.classifier.classify(enriched)
        
        # Stage 6: Store
        stored = await self.storage.store(enriched)
        
        # Stage 7: Update graph
        await self.graph_sync.sync_evidence(stored)
        
        # Emit event
        await self.events.publish(EvidenceCollectedEvent(
            evidence_id=stored.id,
            organization_id=stored.organization_id,
            control_ids=stored.control_ids,
            classification=stored.classification.value
        ))
        
        return stored
        
    async def _handle_duplicate(
        self,
        existing: ProcessedEvidenceItem,
        new: ProcessedEvidenceItem
    ) -> ProcessedEvidenceItem:
        """Handle duplicate evidence detection."""
        # Update last seen timestamp
        existing.metadata["last_seen"] = datetime.utcnow().isoformat()
        existing.metadata["duplicate_count"] = existing.metadata.get("duplicate_count", 0) + 1
        
        await self.storage.update_metadata(existing)
        
        return existing

class EvidenceNormalizer:
    """Normalize raw evidence to standard format."""
    
    async def normalize(self, raw: RawEvidenceItem) -> ProcessedEvidenceItem:
        """Convert raw evidence to processed format."""
        
        # Compute content hash for deduplication
        content_hash = self._compute_hash(raw.raw_data)
        
        # Determine evidence type from raw data
        evidence_type = self._infer_type(raw)
        
        # Extract title and description
        title, description = self._extract_metadata(raw)
        
        return ProcessedEvidenceItem(
            id=str(uuid4()),
            organization_id=raw.organization_id,
            evidence_type=evidence_type,
            classification=EvidenceClassification.STANDARD,  # Default, updated by classifier
            integration_id=raw.integration_id,
            integration_type=raw.integration_type,
            collector=raw.collector,
            title=title,
            description=description,
            content_hash=content_hash,
            storage_uri="",  # Set during storage
            collected_at=raw.collected_at,
            effective_date=raw.collected_at,
            expires_at=self._calculate_expiry(evidence_type),
            control_ids=[],  # Set during enrichment
            framework_ids=[],
            status=EvidenceStatus.PENDING,
            metadata={"raw_data": raw.raw_data}
        )
        
    def _compute_hash(self, data: Dict[str, Any]) -> str:
        """Compute SHA-256 hash of evidence content."""
        normalized = json.dumps(data, sort_keys=True, default=str)
        return hashlib.sha256(normalized.encode()).hexdigest()
        
    def _calculate_expiry(self, evidence_type: EvidenceType) -> Optional[datetime]:
        """Calculate evidence expiry based on type."""
        expiry_days = {
            EvidenceType.CONFIGURATION: 30,
            EvidenceType.SCREENSHOT: 90,
            EvidenceType.LOG: 7,
            EvidenceType.REPORT: 365,
            EvidenceType.POLICY_DOCUMENT: None,  # No expiry
            EvidenceType.CERTIFICATION: 365,
        }
        
        days = expiry_days.get(evidence_type, 90)
        if days:
            return datetime.utcnow() + timedelta(days=days)
        return None

class EvidenceDeduplicator:
    """Detect and handle duplicate evidence."""
    
    def __init__(self, evidence_repo: EvidenceRepository):
        self.repo = evidence_repo
        
    async def find_duplicate(
        self,
        evidence: ProcessedEvidenceItem
    ) -> Optional[ProcessedEvidenceItem]:
        """Find existing evidence with same content hash."""
        return await self.repo.find_by_hash(
            organization_id=evidence.organization_id,
            integration_id=evidence.integration_id,
            content_hash=evidence.content_hash,
            since=datetime.utcnow() - timedelta(days=90)
        )

class ControlEnricher:
    """Enrich evidence with control mappings."""
    
    def __init__(
        self,
        control_graph: ControlGraphRepository,
        mapping_rules: MappingRuleEngine
    ):
        self.graph = control_graph
        self.rules = mapping_rules
        
    async def enrich(
        self,
        evidence: ProcessedEvidenceItem
    ) -> ProcessedEvidenceItem:
        """Add control mappings based on evidence type and content."""
        
        # Get applicable controls based on integration and evidence type
        controls = await self.rules.find_matching_controls(
            integration_type=evidence.integration_type,
            evidence_type=evidence.evidence_type,
            evidence_content=evidence.metadata.get("raw_data", {})
        )
        
        evidence.control_ids = [c.id for c in controls]
        evidence.framework_ids = list(set(c.framework_id for c in controls))
        
        return evidence
```

### 3. Storage Layer

```python
# File: src/services/evidence/storage.py

"""
Evidence storage implementation.

Handles storage routing based on classification.
"""

class EvidenceStorage:
    """
    Multi-tier evidence storage.
    
    Routes evidence to appropriate storage based on classification:
    - AUDIT_CRITICAL: Event store (FoundationDB) for immutable trail
    - STANDARD: GCS for content, FoundationDB for metadata
    - TRANSIENT: Redis cache with TTL
    """
    
    def __init__(
        self,
        fdb_client: FoundationDBClient,
        gcs_client: storage.Client,
        redis: Redis,
        event_store: EventStore
    ):
        self.fdb = fdb_client
        self.gcs = gcs_client
        self.redis = redis
        self.event_store = event_store
        
    async def store(
        self,
        evidence: ProcessedEvidenceItem
    ) -> ProcessedEvidenceItem:
        """Store evidence based on classification."""
        
        if evidence.classification == EvidenceClassification.AUDIT_CRITICAL:
            return await self._store_audit_critical(evidence)
        elif evidence.classification == EvidenceClassification.TRANSIENT:
            return await self._store_transient(evidence)
        else:
            return await self._store_standard(evidence)
            
    async def _store_standard(
        self,
        evidence: ProcessedEvidenceItem
    ) -> ProcessedEvidenceItem:
        """Store standard evidence: content to GCS, metadata to FDB."""
        
        # Store content in GCS
        bucket_name = f"coditect-evidence-{get_data_residency()}"
        blob_path = self._generate_blob_path(evidence)
        
        bucket = self.gcs.bucket(bucket_name)
        blob = bucket.blob(blob_path)
        
        content = json.dumps(evidence.metadata.get("raw_data", {}))
        blob.upload_from_string(
            content,
            content_type="application/json"
        )
        
        evidence.storage_uri = f"gs://{bucket_name}/{blob_path}"
        
        # Remove raw data from metadata before storing in FDB
        evidence.metadata.pop("raw_data", None)
        
        # Store metadata in FoundationDB
        key = f"/{evidence.organization_id}/evidence/{evidence.id}".encode()
        await self.fdb.set(key, evidence.model_dump_json().encode())
        
        return evidence
        
    async def _store_audit_critical(
        self,
        evidence: ProcessedEvidenceItem
    ) -> ProcessedEvidenceItem:
        """Store audit-critical evidence with event sourcing."""
        
        # Create immutable event
        event = EvidenceEvent(
            event_id=str(uuid4()),
            event_type="collected",
            evidence_id=evidence.id,
            organization_id=evidence.organization_id,
            timestamp=datetime.utcnow(),
            actor=evidence.collector,
            payload=evidence.model_dump(),
            previous_event_id=None
        )
        
        await self.event_store.append(event)
        evidence.storage_uri = f"events://{evidence.id}"
        
        return evidence
        
    async def _store_transient(
        self,
        evidence: ProcessedEvidenceItem
    ) -> ProcessedEvidenceItem:
        """Store transient evidence in Redis cache."""
        
        key = f"evidence:transient:{evidence.id}"
        await self.redis.setex(
            key,
            86400,  # 24 hour TTL
            evidence.model_dump_json()
        )
        
        evidence.storage_uri = f"cache://{evidence.id}"
        return evidence
```

### 4. Gap Detection

```python
# File: src/services/evidence/gap_detector.py

"""
Evidence gap detection.

Identifies controls lacking sufficient evidence.
"""

class EvidenceGapDetector:
    """Detect and report evidence gaps."""
    
    async def detect_gaps(
        self,
        framework_id: str,
        lookback_days: int = 90
    ) -> List[EvidenceGap]:
        """
        Identify controls with evidence gaps.
        
        Gap types:
        - NO_EVIDENCE: No evidence exists
        - EXPIRED_EVIDENCE: Only expired evidence
        - PARTIAL_COVERAGE: Incomplete evidence
        - FAILED_VALIDATION: Evidence rejected
        """
        gaps = []
        cutoff = datetime.utcnow() - timedelta(days=lookback_days)
        
        # Get all controls for framework
        controls = await self.graph_repo.get_framework_controls(framework_id)
        
        for control in controls:
            gap = await self._check_control_gap(control, cutoff)
            if gap:
                gaps.append(gap)
                
        return sorted(gaps, key=lambda g: (g.severity_rank, g.control_id))
        
    async def _check_control_gap(
        self,
        control: Control,
        cutoff: datetime
    ) -> Optional[EvidenceGap]:
        """Check if a control has an evidence gap."""
        
        evidence = await self.evidence_repo.find_for_control(
            control_id=control.id,
            since=cutoff,
            status=EvidenceStatus.VALIDATED
        )
        
        if not evidence:
            return EvidenceGap(
                control_id=control.id,
                control_name=control.title,
                gap_type=GapType.NO_EVIDENCE,
                severity=control.severity,
                last_evidence_date=None,
                recommended_action=self._recommend_action(control, GapType.NO_EVIDENCE)
            )
            
        # Check for expired evidence
        latest = max(evidence, key=lambda e: e.collected_at)
        if latest.expires_at and latest.expires_at < datetime.utcnow():
            return EvidenceGap(
                control_id=control.id,
                control_name=control.title,
                gap_type=GapType.EXPIRED_EVIDENCE,
                severity=control.severity,
                last_evidence_date=latest.collected_at,
                recommended_action=self._recommend_action(control, GapType.EXPIRED_EVIDENCE)
            )
            
        return None
```

## File Structure

```
src/services/evidence/
├── __init__.py
├── collector.py           # Collection orchestration
├── pipeline.py            # Processing pipeline
├── storage.py             # Multi-tier storage
├── gap_detector.py        # Gap detection
├── normalizer.py          # Evidence normalization
├── validator.py           # Evidence validation
├── classifier.py          # Storage classification
├── enricher.py            # Control mapping enrichment
├── deduplicator.py        # Duplicate detection
├── scheduler.py           # Collection scheduling
├── api.py                 # FastAPI endpoints
├── models.py              # Request/response models
├── events.py              # Domain events
└── tests/
    ├── test_collector.py
    ├── test_pipeline.py
    └── test_gap_detector.py
```

## Acceptance Criteria

1. **Collection**: Support scheduled, webhook, and manual triggers
2. **Pipeline**: All 7 processing stages implemented
3. **Deduplication**: Content-hash based duplicate detection
4. **Storage Routing**: Classification-based storage selection
5. **Gap Detection**: Identify all gap types
6. **Tenant Isolation**: All evidence scoped to organization
7. **Events**: Emit events for collection and gap detection

## Token Budget

- Target: 20,000-28,000 tokens

## Dependencies

- Input: ADR-003 (Evidence Collection architecture)
- Input: Domain models (Prompt 10)
- Input: Integration connectors (Prompt 14)
- Output: Used by Agent Framework, API Layer
