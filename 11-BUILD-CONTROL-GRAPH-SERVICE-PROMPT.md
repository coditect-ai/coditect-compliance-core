# Prompt 11: Component Build - Control Graph Service

## Context

You are a senior software engineer implementing the Control Graph Service for CODITECT-COMPLIANCE. This service manages compliance frameworks, controls, and their cross-framework mappings using Neo4j as the graph database.

## Output Specification

Generate complete, production-ready Python code for the Control Graph Service. Output should be 1,500-2,500 lines of code.

## Implementation Requirements

### Technology Stack
- Python 3.12+
- Neo4j Python Driver (async)
- FastAPI for service endpoints
- Pydantic for request/response models
- Redis for caching

### Architecture Pattern
- Repository pattern for data access
- Service layer for business logic
- Event emission for state changes

## Service Specifications

### 1. Repository Layer

```python
# File: src/services/control_graph/repository.py

"""
Neo4j repository for control graph operations.

Handles all graph database interactions with tenant isolation.
"""

from typing import List, Optional, Dict, Any, AsyncIterator
from neo4j import AsyncDriver, AsyncSession
from contextlib import asynccontextmanager

from domain.models.framework import Framework, Control, ControlMapping
from domain.models.common import TenantContext, get_tenant_context

class ControlGraphRepository:
    """
    Repository for control graph operations in Neo4j.
    
    All operations are scoped to the current tenant context.
    Uses parameterized queries to prevent injection.
    """
    
    def __init__(self, driver: AsyncDriver):
        self.driver = driver
        
    @asynccontextmanager
    async def _session(self) -> AsyncIterator[AsyncSession]:
        """Get a Neo4j session with tenant context."""
        async with self.driver.session() as session:
            yield session
            
    @property
    def organization_id(self) -> str:
        """Get current tenant's organization ID."""
        return get_tenant_context().organization_id
    
    # Framework Operations
    
    async def create_framework(self, framework: Framework) -> Framework:
        """
        Create a new framework node in the graph.
        
        Args:
            framework: Framework to create
            
        Returns:
            Created framework with any generated fields
            
        Raises:
            DuplicateFrameworkError: If framework code already exists
        """
        query = """
        CREATE (f:Framework {
            id: $id,
            organization_id: $org_id,
            code: $code,
            name: $name,
            version: $version,
            authority: $authority,
            status: $status,
            control_count: $control_count,
            is_custom: $is_custom,
            created_at: datetime()
        })
        RETURN f
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                id=framework.id,
                org_id=self.organization_id,
                code=framework.code,
                name=framework.name,
                version=framework.version,
                authority=framework.authority.value,
                status=framework.status.value,
                control_count=framework.control_count,
                is_custom=framework.is_custom
            )
            record = await result.single()
            return framework
            
    async def get_framework(self, framework_id: str) -> Optional[Framework]:
        """Get framework by ID, scoped to tenant."""
        query = """
        MATCH (f:Framework {id: $id, organization_id: $org_id})
        RETURN f
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                id=framework_id,
                org_id=self.organization_id
            )
            record = await result.single()
            if record:
                return self._node_to_framework(record["f"])
            return None
            
    async def list_frameworks(
        self,
        status: Optional[str] = None,
        limit: int = 100,
        offset: int = 0
    ) -> List[Framework]:
        """List all frameworks for tenant."""
        query = """
        MATCH (f:Framework {organization_id: $org_id})
        WHERE $status IS NULL OR f.status = $status
        RETURN f
        ORDER BY f.code
        SKIP $offset
        LIMIT $limit
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                org_id=self.organization_id,
                status=status,
                offset=offset,
                limit=limit
            )
            records = await result.fetch_all()
            return [self._node_to_framework(r["f"]) for r in records]
    
    # Control Operations
    
    async def create_control(self, control: Control) -> Control:
        """Create a control node and link to framework."""
        query = """
        MATCH (f:Framework {id: $framework_id, organization_id: $org_id})
        CREATE (c:Control {
            id: $id,
            organization_id: $org_id,
            framework_id: $framework_id,
            control_id: $control_id,
            title: $title,
            description: $description,
            category: $category,
            level: $level,
            severity: $severity,
            weight: $weight,
            implementation_status: $impl_status,
            created_at: datetime()
        })
        CREATE (f)-[:CONTAINS]->(c)
        WITH c, f
        OPTIONAL MATCH (parent:Control {id: $parent_id, organization_id: $org_id})
        FOREACH (_ IN CASE WHEN parent IS NOT NULL THEN [1] ELSE [] END |
            CREATE (parent)-[:HAS_CHILD]->(c)
        )
        SET f.control_count = f.control_count + 1
        RETURN c
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                id=control.id,
                org_id=self.organization_id,
                framework_id=control.framework_id,
                control_id=control.control_id,
                title=control.title,
                description=control.description,
                category=control.category.value,
                level=control.level,
                severity=control.severity,
                weight=control.weight,
                impl_status=control.implementation_status.value,
                parent_id=control.parent_id
            )
            await result.consume()
            return control
            
    async def get_control(self, control_id: str) -> Optional[Control]:
        """Get control by ID."""
        query = """
        MATCH (c:Control {id: $id, organization_id: $org_id})
        RETURN c
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                id=control_id,
                org_id=self.organization_id
            )
            record = await result.single()
            if record:
                return self._node_to_control(record["c"])
            return None
            
    async def get_framework_controls(
        self,
        framework_id: str,
        category: Optional[str] = None
    ) -> List[Control]:
        """Get all controls for a framework."""
        query = """
        MATCH (f:Framework {id: $framework_id, organization_id: $org_id})-[:CONTAINS]->(c:Control)
        WHERE $category IS NULL OR c.category = $category
        RETURN c
        ORDER BY c.control_id
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                framework_id=framework_id,
                org_id=self.organization_id,
                category=category
            )
            records = await result.fetch_all()
            return [self._node_to_control(r["c"]) for r in records]
    
    # Control Mapping Operations
    
    async def create_mapping(self, mapping: ControlMapping) -> ControlMapping:
        """Create a cross-framework control mapping."""
        query = """
        MATCH (source:Control {id: $source_id, organization_id: $org_id})
        MATCH (target:Control {id: $target_id, organization_id: $org_id})
        CREATE (source)-[m:MAPS_TO {
            id: $id,
            mapping_type: $mapping_type,
            confidence: $confidence,
            rationale: $rationale,
            created_at: datetime()
        }]->(target)
        RETURN m
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                id=mapping.id,
                org_id=self.organization_id,
                source_id=mapping.source_control_id,
                target_id=mapping.target_control_id,
                mapping_type=mapping.mapping_type.value,
                confidence=mapping.confidence,
                rationale=mapping.rationale
            )
            await result.consume()
            return mapping
            
    async def find_mapped_controls(
        self,
        control_id: str,
        max_depth: int = 2
    ) -> List[Dict[str, Any]]:
        """
        Find all controls mapped to/from a given control.
        
        Uses graph traversal to find transitive mappings.
        """
        query = """
        MATCH (source:Control {id: $control_id, organization_id: $org_id})
        CALL apoc.path.subgraphNodes(source, {
            relationshipFilter: "MAPS_TO",
            maxLevel: $max_depth
        }) YIELD node as mapped
        WHERE mapped <> source
        RETURN mapped, 
               [(source)-[r:MAPS_TO*1..2]->(mapped) | r] as paths
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                control_id=control_id,
                org_id=self.organization_id,
                max_depth=max_depth
            )
            records = await result.fetch_all()
            return [
                {
                    "control": self._node_to_control(r["mapped"]),
                    "path_length": len(r["paths"])
                }
                for r in records
            ]
    
    # Gap Analysis Queries
    
    async def find_controls_without_evidence(
        self,
        framework_id: str,
        days: int = 90
    ) -> List[Control]:
        """Find controls lacking recent evidence."""
        query = """
        MATCH (f:Framework {id: $framework_id, organization_id: $org_id})-[:CONTAINS]->(c:Control)
        WHERE NOT EXISTS {
            MATCH (c)-[:SATISFIED_BY]->(e:Evidence)
            WHERE e.collected_at > datetime() - duration({days: $days})
        }
        RETURN c
        ORDER BY c.severity DESC, c.control_id
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                framework_id=framework_id,
                org_id=self.organization_id,
                days=days
            )
            records = await result.fetch_all()
            return [self._node_to_control(r["c"]) for r in records]
            
    async def calculate_posture_score(
        self,
        framework_id: str
    ) -> Dict[str, Any]:
        """Calculate compliance posture score for framework."""
        query = """
        MATCH (f:Framework {id: $framework_id, organization_id: $org_id})-[:CONTAINS]->(c:Control)
        WITH c.category as category,
             c.weight as weight,
             c.last_test_result as result
        WITH category,
             sum(weight) as total_weight,
             sum(CASE WHEN result = 'pass' THEN weight ELSE 0 END) as passing_weight,
             count(*) as total_controls,
             sum(CASE WHEN result = 'pass' THEN 1 ELSE 0 END) as passing_controls
        RETURN category,
               total_controls,
               passing_controls,
               round(passing_weight / total_weight * 100, 2) as score
        ORDER BY category
        """
        
        async with self._session() as session:
            result = await session.run(
                query,
                framework_id=framework_id,
                org_id=self.organization_id
            )
            records = await result.fetch_all()
            
            categories = {r["category"]: r for r in records}
            overall_score = sum(r["score"] * r["total_weight"] for r in records) / sum(r["total_weight"] for r in records) if records else 0
            
            return {
                "framework_id": framework_id,
                "overall_score": round(overall_score, 2),
                "categories": categories,
                "calculated_at": datetime.utcnow().isoformat()
            }
    
    # Helper methods
    
    def _node_to_framework(self, node: Dict) -> Framework:
        """Convert Neo4j node to Framework model."""
        return Framework(
            id=node["id"],
            organization_id=node["organization_id"],
            code=node["code"],
            name=node["name"],
            version=node["version"],
            authority=FrameworkAuthority(node["authority"]),
            status=FrameworkStatus(node["status"]),
            control_count=node["control_count"],
            is_custom=node["is_custom"]
        )
        
    def _node_to_control(self, node: Dict) -> Control:
        """Convert Neo4j node to Control model."""
        return Control(
            id=node["id"],
            organization_id=node["organization_id"],
            framework_id=node["framework_id"],
            control_id=node["control_id"],
            title=node["title"],
            description=node.get("description", ""),
            category=ControlCategory(node["category"]),
            level=node["level"],
            severity=node["severity"],
            weight=node["weight"],
            implementation_status=ControlImplementationStatus(node["implementation_status"])
        )
```

### 2. Service Layer

```python
# File: src/services/control_graph/service.py

"""
Control Graph Service business logic.

Coordinates repository operations and emits domain events.
"""

class ControlGraphService:
    """
    Service for managing compliance frameworks and controls.
    
    Handles business logic, validation, and event emission.
    """
    
    def __init__(
        self,
        repository: ControlGraphRepository,
        cache: RedisCache,
        event_bus: EventBus,
        regulatory_parser: RegulatoryParser
    ):
        self.repo = repository
        self.cache = cache
        self.events = event_bus
        self.parser = regulatory_parser
        
    async def import_framework(
        self,
        framework_code: str,
        version: str,
        source: Optional[str] = None
    ) -> Framework:
        """
        Import a standard compliance framework.
        
        Parses framework definition and creates all controls.
        """
        # Load framework definition
        definition = await self.parser.load_framework(framework_code, version)
        
        # Create framework
        framework = Framework(
            code=framework_code,
            name=definition.name,
            version=version,
            authority=definition.authority,
            description=definition.description,
            organization_id=get_tenant_context().organization_id
        )
        
        framework = await self.repo.create_framework(framework)
        
        # Create controls
        for control_def in definition.controls:
            control = Control(
                framework_id=framework.id,
                control_id=control_def.id,
                title=control_def.title,
                description=control_def.description,
                category=control_def.category,
                parent_id=control_def.parent_id,
                level=control_def.level,
                organization_id=framework.organization_id
            )
            await self.repo.create_control(control)
            
        # Update control count
        framework.control_count = len(definition.controls)
        
        # Emit event
        await self.events.publish(FrameworkImportedEvent(
            framework_id=framework.id,
            code=framework_code,
            control_count=framework.control_count
        ))
        
        # Invalidate cache
        await self.cache.delete(f"frameworks:{framework.organization_id}")
        
        return framework
        
    async def create_control_mapping(
        self,
        source_control_id: str,
        target_control_id: str,
        mapping_type: MappingType,
        confidence: float,
        rationale: str
    ) -> ControlMapping:
        """Create a mapping between controls."""
        # Validate controls exist
        source = await self.repo.get_control(source_control_id)
        target = await self.repo.get_control(target_control_id)
        
        if not source or not target:
            raise ControlNotFoundError("Source or target control not found")
            
        # Create mapping
        mapping = ControlMapping(
            source_control_id=source_control_id,
            target_control_id=target_control_id,
            mapping_type=mapping_type,
            confidence=confidence,
            rationale=rationale
        )
        
        mapping = await self.repo.create_mapping(mapping)
        
        # Emit event
        await self.events.publish(ControlMappingCreatedEvent(
            mapping_id=mapping.id,
            source_control_id=source_control_id,
            target_control_id=target_control_id
        ))
        
        return mapping
        
    async def get_gap_analysis(
        self,
        framework_id: str,
        lookback_days: int = 90
    ) -> GapAnalysisResult:
        """
        Perform gap analysis for a framework.
        
        Identifies controls lacking evidence or failing tests.
        """
        # Check cache
        cache_key = f"gap_analysis:{framework_id}:{lookback_days}"
        cached = await self.cache.get(cache_key)
        if cached:
            return GapAnalysisResult.model_validate_json(cached)
            
        # Get controls without evidence
        gaps = await self.repo.find_controls_without_evidence(
            framework_id,
            lookback_days
        )
        
        # Get posture score
        posture = await self.repo.calculate_posture_score(framework_id)
        
        result = GapAnalysisResult(
            framework_id=framework_id,
            total_controls=posture.get("total_controls", 0),
            controls_with_gaps=len(gaps),
            gap_controls=gaps,
            posture_score=posture["overall_score"],
            category_scores=posture["categories"],
            analyzed_at=datetime.utcnow()
        )
        
        # Cache for 5 minutes
        await self.cache.set(cache_key, result.model_dump_json(), ttl=300)
        
        return result
```

### 3. API Layer

```python
# File: src/services/control_graph/api.py

"""
FastAPI endpoints for Control Graph Service.
"""

from fastapi import APIRouter, Depends, HTTPException, Query
from typing import List, Optional

router = APIRouter(prefix="/api/v1/control-graph", tags=["control-graph"])

@router.get("/frameworks", response_model=List[FrameworkResponse])
async def list_frameworks(
    status: Optional[str] = None,
    limit: int = Query(default=100, le=500),
    offset: int = Query(default=0, ge=0),
    service: ControlGraphService = Depends(get_control_graph_service)
) -> List[FrameworkResponse]:
    """List all compliance frameworks."""
    frameworks = await service.list_frameworks(status=status, limit=limit, offset=offset)
    return [FrameworkResponse.from_domain(f) for f in frameworks]

@router.post("/frameworks/import", response_model=FrameworkResponse)
async def import_framework(
    request: ImportFrameworkRequest,
    service: ControlGraphService = Depends(get_control_graph_service)
) -> FrameworkResponse:
    """Import a standard compliance framework."""
    framework = await service.import_framework(
        framework_code=request.framework_code,
        version=request.version
    )
    return FrameworkResponse.from_domain(framework)

@router.get("/frameworks/{framework_id}/gap-analysis", response_model=GapAnalysisResponse)
async def get_gap_analysis(
    framework_id: str,
    lookback_days: int = Query(default=90, ge=1, le=365),
    service: ControlGraphService = Depends(get_control_graph_service)
) -> GapAnalysisResponse:
    """Get gap analysis for a framework."""
    result = await service.get_gap_analysis(framework_id, lookback_days)
    return GapAnalysisResponse.from_domain(result)

@router.get("/controls/{control_id}/mappings", response_model=List[ControlMappingResponse])
async def get_control_mappings(
    control_id: str,
    max_depth: int = Query(default=2, ge=1, le=5),
    service: ControlGraphService = Depends(get_control_graph_service)
) -> List[ControlMappingResponse]:
    """Get all controls mapped to/from a control."""
    mappings = await service.get_mapped_controls(control_id, max_depth)
    return [ControlMappingResponse.from_domain(m) for m in mappings]
```

## File Structure

```
src/services/control_graph/
├── __init__.py
├── repository.py      # Neo4j data access
├── service.py         # Business logic
├── api.py             # FastAPI endpoints
├── models.py          # Request/response models
├── events.py          # Domain events
├── parsers/
│   ├── __init__.py
│   ├── base.py        # Base parser interface
│   ├── soc2.py        # SOC 2 framework parser
│   ├── iso27001.py    # ISO 27001 parser
│   └── hipaa.py       # HIPAA parser
└── tests/
    ├── __init__.py
    ├── test_repository.py
    ├── test_service.py
    └── test_api.py
```

## Acceptance Criteria

1. **CRUD Operations**: Complete framework and control management
2. **Graph Queries**: Cross-framework mapping traversal
3. **Gap Analysis**: Identify controls lacking evidence
4. **Posture Scoring**: Weighted score calculation
5. **Tenant Isolation**: All queries scoped to organization
6. **Caching**: Redis caching for expensive queries
7. **Events**: Domain events for state changes

## Token Budget

- Target: 18,000-25,000 tokens
- Priority: Repository queries and service logic

## Dependencies

- Input: ADR-001 (Control Graph architecture)
- Input: Domain models (Prompt 10)
- Output: Used by Evidence Engine, Agent Framework
