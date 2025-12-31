---
title: 'PROMPT 05: Architecture Decision Record - Control Graph Data Model'
type: adr
component_type: adr
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
summary: You are a Principal Architect at CODITECT, tasked with documenting the critical
  architecture decision for the Control Graph data model in the...
moe_confidence: 0.950
moe_classified: 2025-12-31
---
# PROMPT 05: Architecture Decision Record - Control Graph Data Model

## Prompt Metadata
```yaml
prompt_id: CODITECT-COMPLIANCE-ADR-001
prompt_type: architecture_decision_record
output_artifact: ADR-001-control-graph-data-model.md
estimated_tokens: 8,000-12,000
dependencies:
  - 01-PRODUCT-DEFINITION-PROMPT.md (completed)
  - 02-PRODUCT-REQUIREMENTS-PROMPT.md (completed)
  - 03-SOFTWARE-DESIGN-DOCUMENT-PROMPT.md (completed)
```

---

## System Context

You are a Principal Architect at CODITECT, tasked with documenting the critical architecture decision for the Control Graph data model in the CODITECT-COMPLIANCE module. This ADR will guide all downstream implementation of the compliance control relationships, framework mappings, and regulatory requirement tracking.

---

## Input Documents Required

Before generating this ADR, ensure you have access to:
1. Product Requirements Document (PRD) - specifically FR-CG-* requirements
2. Software Design Document (SDD) - Section 4 (Component Design) and Section 5 (Data Architecture)
3. Vanta compliance features analysis document

---

## Output Specification

Generate a comprehensive Architecture Decision Record following this exact structure:

### Document Structure

```markdown
# ADR-001: Control Graph Data Model Architecture

## Status
[Proposed | Accepted | Deprecated | Superseded]

## Date
[YYYY-MM-DD]

## Decision Makers
- [Role]: [Responsibility in decision]

## Context

### Business Context
[Describe the business problem driving this decision]

### Technical Context  
[Describe the technical landscape and constraints]

### Current State
[If applicable, describe what exists today]

## Decision Drivers

### Primary Drivers
1. [Driver with priority and rationale]
2. [Driver with priority and rationale]
...

### Constraints
1. [Hard constraint that cannot be violated]
2. [Soft constraint that should be respected]
...

## Considered Options

### Option 1: [Name]
[Detailed description]

**Pros:**
- [Advantage]

**Cons:**
- [Disadvantage]

**Estimated Effort:** [T-shirt size]
**Risk Level:** [Low/Medium/High]

### Option 2: [Name]
[Repeat structure]

### Option 3: [Name]
[Repeat structure]

## Decision

### Selected Option
[State the chosen option]

### Rationale
[Explain why this option was selected]

### Trade-offs Accepted
[Document what we're giving up]

## Consequences

### Positive Consequences
1. [Benefit]

### Negative Consequences
1. [Drawback and mitigation]

### Neutral Consequences
1. [Observation]

## Implementation Guidelines

### Phase 1: [Name]
[Implementation steps]

### Phase 2: [Name]
[Implementation steps]

## Validation Criteria

### Success Metrics
1. [Measurable outcome]

### Acceptance Criteria
1. [Testable criterion]

## Related Decisions
- [Link to related ADRs]

## References
- [External references]
```

---

## Content Requirements

### Section: Context

Generate comprehensive context covering:

**Business Context Requirements:**
- Compliance frameworks have hierarchical, overlapping structures
- Single control can satisfy multiple framework requirements
- Organizations need to map their implementations to framework controls
- Auditors require evidence linked to specific controls
- Cross-framework efficiency is a key differentiator (implement once, satisfy many)

**Technical Context Requirements:**
- Graph relationships are central to the domain
- Query patterns include: ancestor/descendant traversal, shortest path, subgraph extraction
- Write patterns: bulk framework import, incremental control updates
- Read-heavy workload (95% reads, 5% writes)
- Multi-tenant isolation required

**Quantitative Context:**
```yaml
expected_scale:
  frameworks: 50+ (30+ at launch)
  controls_per_framework: 50-500
  total_controls: 5,000-15,000
  control_mappings: 50,000-200,000 (many-to-many)
  organizations: 1,000+
  implementations_per_org: 500-5,000

query_patterns:
  framework_hierarchy_traversal: 40%
  cross_framework_mapping: 25%
  implementation_status_lookup: 20%
  gap_analysis: 10%
  audit_evidence_linking: 5%

performance_requirements:
  hierarchy_traversal: <100ms P95
  cross_framework_query: <500ms P95
  gap_analysis_full: <5s P95
  bulk_import: <30s for 500 controls
```

---

### Section: Decision Drivers

Generate decision drivers including:

**Primary Drivers (in priority order):**

1. **Relationship Query Performance**
   - Graph traversals must be efficient
   - Multi-hop queries (control → mapping → control → framework) common
   - Priority: CRITICAL

2. **Schema Flexibility**
   - New frameworks added frequently
   - Control structures vary by framework
   - Custom attributes per framework
   - Priority: HIGH

3. **Cross-Framework Mapping Efficiency**
   - Many-to-many relationships with metadata
   - Mapping confidence scores
   - Bidirectional traversal
   - Priority: HIGH

4. **Multi-Tenant Data Isolation**
   - Organization-specific implementations
   - Shared framework definitions
   - Tenant data never crosses boundaries
   - Priority: CRITICAL

5. **Audit Trail Requirements**
   - All changes must be tracked
   - Point-in-time queries for audits
   - Evidence chain integrity
   - Priority: HIGH

6. **Integration with CODITECT-CORE**
   - Must work with FoundationDB operational store
   - Event sourcing compatibility
   - Consistent transaction boundaries
   - Priority: MEDIUM

**Constraints:**

1. **HARD: Multi-tenant isolation** - Tenant data must never leak
2. **HARD: ACID for implementations** - Organization data changes must be transactional
3. **HARD: Sub-second read latency** - Common queries under 500ms
4. **SOFT: Minimize operational complexity** - Prefer managed services
5. **SOFT: Cost efficiency** - Scale costs linearly with usage

---

### Section: Considered Options

Generate detailed analysis for these options:

**Option 1: Neo4j (Dedicated Graph Database)**

```yaml
description: |
  Use Neo4j as dedicated graph database for all control graph data.
  Frameworks, controls, and mappings stored as nodes and relationships.
  Organization implementations stored as separate subgraphs.

architecture:
  primary_store: Neo4j
  query_language: Cypher
  deployment: Neo4j AuraDB (managed) or self-hosted
  
data_model:
  nodes:
    - Framework (id, name, version, category, status)
    - Control (id, framework_id, code, title, description, attributes)
    - ControlMapping (id, confidence, rationale, created_at)
    - Implementation (id, org_id, control_id, status, evidence_refs)
  relationships:
    - (Framework)-[:CONTAINS]->(Control)
    - (Control)-[:PARENT_OF]->(Control)
    - (Control)-[:MAPS_TO {confidence, rationale}]->(Control)
    - (Implementation)-[:IMPLEMENTS]->(Control)

query_examples:
  hierarchy: |
    MATCH (f:Framework {id: $framework_id})-[:CONTAINS]->(c:Control)
    OPTIONAL MATCH (c)-[:PARENT_OF*]->(child:Control)
    RETURN c, collect(child) as children
    
  cross_mapping: |
    MATCH (c1:Control {id: $control_id})-[:MAPS_TO*1..2]-(c2:Control)
    WHERE c2.framework_id <> c1.framework_id
    RETURN c2, relationships(path) as mappings
    
  gap_analysis: |
    MATCH (f:Framework {id: $framework_id})-[:CONTAINS]->(c:Control)
    OPTIONAL MATCH (i:Implementation {org_id: $org_id})-[:IMPLEMENTS]->(c)
    RETURN c, i IS NOT NULL as implemented

pros:
  - Native graph query performance (index-free adjacency)
  - Expressive Cypher query language
  - Built-in graph algorithms (shortest path, community detection)
  - Schema flexibility (add properties without migration)
  - Visualization tools for debugging

cons:
  - Additional operational complexity (another database)
  - Transaction boundaries don't align with FoundationDB
  - Multi-tenancy requires careful modeling
  - Cost scales with data size
  - Learning curve for Cypher

effort: Medium
risk: Medium
```

**Option 2: FoundationDB with Graph Layer**

```yaml
description: |
  Implement graph semantics on top of FoundationDB using custom layer.
  Leverage FoundationDB's ordered key-value model for adjacency lists.
  Single database for all CODITECT-COMPLIANCE data.

architecture:
  primary_store: FoundationDB
  query_language: Custom Python API
  deployment: Existing CODITECT-CORE FoundationDB cluster
  
data_model:
  key_patterns:
    # Node storage
    node: "graph/{tenant}/node/{type}/{id}"
    node_props: "graph/{tenant}/node/{type}/{id}/props"
    
    # Edge storage (adjacency lists)
    outgoing: "graph/{tenant}/edge/{from_type}/{from_id}/out/{rel_type}/{to_id}"
    incoming: "graph/{tenant}/edge/{to_type}/{to_id}/in/{rel_type}/{from_id}"
    
    # Indexes
    by_framework: "graph/{tenant}/idx/framework/{framework_id}/{control_id}"
    by_status: "graph/{tenant}/idx/status/{status}/{type}/{id}"
    
  traversal_implementation: |
    async def get_children(node_id: str, depth: int = 1) -> List[Node]:
        prefix = f"graph/{tenant}/edge/control/{node_id}/out/PARENT_OF/"
        children = []
        async for key, value in db.get_range_startswith(prefix):
            child_id = key.split('/')[-1]
            child = await get_node(child_id)
            children.append(child)
            if depth > 1:
                children.extend(await get_children(child_id, depth - 1))
        return children

pros:
  - Single database (operational simplicity)
  - ACID transactions across all data
  - Native multi-tenancy via key prefixes
  - Consistent with CODITECT-CORE patterns
  - No additional infrastructure

cons:
  - Complex traversal implementation
  - No built-in graph algorithms
  - Performance degrades with hop count
  - More code to maintain
  - No visualization tools

effort: High
risk: High
```

**Option 3: Hybrid (Neo4j for Graph + FoundationDB for Operations)**

```yaml
description: |
  Use Neo4j for framework/control graph (shared reference data).
  Use FoundationDB for organization-specific implementations.
  Synchronize via event-driven updates.

architecture:
  graph_store: Neo4j (frameworks, controls, mappings)
  operational_store: FoundationDB (implementations, evidence, audit)
  sync_mechanism: Redis Streams events
  
data_distribution:
  neo4j:
    - Framework definitions (shared)
    - Control definitions (shared)
    - Control mappings (shared)
    - Control hierarchy (shared)
    
  foundationdb:
    - Organization profiles
    - Implementation records
    - Evidence items
    - Check results
    - Audit logs
    
  sync_patterns:
    # On implementation change
    - Event: implementation.updated
    - Action: Update Neo4j implementation node for queries
    - Consistency: Eventual (< 1s)
    
    # On framework import
    - Event: framework.imported
    - Action: Bulk load to Neo4j
    - Consistency: Strong (wait for completion)

query_routing:
  neo4j_queries:
    - Framework hierarchy traversal
    - Cross-framework mapping discovery
    - Gap analysis (control coverage)
    - Graph visualizations
    
  foundationdb_queries:
    - Implementation CRUD
    - Evidence retrieval
    - Check result history
    - Audit trail

pros:
  - Best tool for each job
  - Graph queries remain fast
  - Transactional integrity for org data
  - Clear separation of concerns
  - Scalable independently

cons:
  - Two databases to operate
  - Sync complexity
  - Eventual consistency for some queries
  - More complex deployment
  - Cross-store queries require orchestration

effort: Medium-High
risk: Medium
```

---

### Section: Decision

Generate decision section with:

**Selected Option: Option 3 - Hybrid (Neo4j for Graph + FoundationDB for Operations)**

**Rationale:**

1. **Query Pattern Alignment**
   - Graph traversals (40% of queries) need native graph performance
   - Operational queries (CRUD, history) fit key-value model
   - Neither database alone optimally serves both patterns

2. **Data Ownership Clarity**
   - Shared reference data (frameworks) in Neo4j makes sense
   - Organization-specific data in FoundationDB aligns with CODITECT-CORE
   - Clear boundaries reduce confusion

3. **Performance Optimization**
   - Neo4j handles complex traversals in <100ms
   - FoundationDB handles transactional writes with ACID
   - Neither compromises on its strength

4. **Operational Trade-off Acceptance**
   - Additional database is worth the query performance
   - Event-driven sync is well-understood pattern
   - Managed Neo4j (AuraDB) reduces operational burden

**Trade-offs Accepted:**
- Eventual consistency between stores (mitigated by <1s sync)
- Additional operational complexity (mitigated by managed services)
- Cross-store queries require orchestration (mitigated by clear routing)

---

### Section: Implementation Guidelines

Generate implementation phases:

**Phase 1: Neo4j Foundation (Week 1-2)**

```python
# Neo4j Schema Setup
CREATE CONSTRAINT framework_id ON (f:Framework) ASSERT f.id IS UNIQUE;
CREATE CONSTRAINT control_id ON (c:Control) ASSERT c.id IS UNIQUE;
CREATE INDEX control_framework FOR (c:Control) ON (c.framework_id);
CREATE INDEX control_status FOR (c:Control) ON (c.status);

# Python Driver Setup
from neo4j import AsyncGraphDatabase

class ControlGraphRepository:
    def __init__(self, uri: str, auth: tuple):
        self.driver = AsyncGraphDatabase.driver(uri, auth=auth)
    
    async def create_framework(self, framework: Framework) -> str:
        async with self.driver.session() as session:
            result = await session.run("""
                CREATE (f:Framework {
                    id: $id,
                    name: $name,
                    version: $version,
                    category: $category,
                    status: $status,
                    created_at: datetime()
                })
                RETURN f.id
            """, framework.dict())
            return result.single()['f.id']
```

**Phase 2: FoundationDB Integration (Week 2-3)**

```python
# Implementation Repository
class ImplementationRepository:
    def __init__(self, db: FoundationDB):
        self.db = db
        
    async def create_implementation(
        self, 
        org_id: str, 
        implementation: Implementation
    ) -> str:
        key = f"compliance/{org_id}/impl/{implementation.control_id}"
        
        @fdb.transactional
        async def do_create(tr):
            # Store implementation
            tr[key] = implementation.to_bytes()
            
            # Publish event for Neo4j sync
            event = ImplementationCreatedEvent(
                org_id=org_id,
                implementation=implementation
            )
            await self.event_publisher.publish(event)
            
            return implementation.id
            
        return await do_create(self.db)
```

**Phase 3: Sync Mechanism (Week 3-4)**

```python
# Event Handler for Neo4j Sync
class Neo4jSyncHandler:
    async def handle_implementation_created(
        self, 
        event: ImplementationCreatedEvent
    ):
        async with self.neo4j.session() as session:
            await session.run("""
                MATCH (c:Control {id: $control_id})
                MERGE (i:Implementation {
                    id: $impl_id,
                    org_id: $org_id
                })
                SET i.status = $status,
                    i.updated_at = datetime()
                MERGE (i)-[:IMPLEMENTS]->(c)
            """, {
                'control_id': event.implementation.control_id,
                'impl_id': event.implementation.id,
                'org_id': event.org_id,
                'status': event.implementation.status
            })
```

**Phase 4: Query Router (Week 4-5)**

```python
# Unified Query Interface
class ControlGraphService:
    def __init__(
        self,
        neo4j_repo: ControlGraphRepository,
        fdb_repo: ImplementationRepository
    ):
        self.neo4j = neo4j_repo
        self.fdb = fdb_repo
    
    async def get_gap_analysis(
        self, 
        org_id: str, 
        framework_id: str
    ) -> GapAnalysis:
        # Get required controls from Neo4j
        controls = await self.neo4j.get_framework_controls(framework_id)
        
        # Get implementations from FoundationDB
        implementations = await self.fdb.get_implementations(
            org_id, 
            [c.id for c in controls]
        )
        
        # Calculate gaps
        impl_map = {i.control_id: i for i in implementations}
        gaps = []
        for control in controls:
            if control.id not in impl_map:
                gaps.append(GapItem(
                    control=control,
                    status='missing'
                ))
            elif impl_map[control.id].status != 'compliant':
                gaps.append(GapItem(
                    control=control,
                    status='incomplete',
                    implementation=impl_map[control.id]
                ))
        
        return GapAnalysis(
            framework_id=framework_id,
            total_controls=len(controls),
            implemented=len(implementations),
            gaps=gaps
        )
```

---

### Section: Validation Criteria

**Success Metrics:**

1. **Graph Query Performance**
   - Hierarchy traversal: <100ms P95 for 5-level depth
   - Cross-framework mapping: <200ms P95 for 2-hop queries
   - Gap analysis: <2s P95 for 500-control framework

2. **Sync Latency**
   - Implementation changes reflected in Neo4j: <1s P95
   - Framework imports fully synced: <30s for 500 controls

3. **Data Consistency**
   - Zero cross-tenant data leaks (verified by audit)
   - 100% eventual consistency within SLA

**Acceptance Criteria:**

1. [ ] Neo4j cluster deployed and accessible
2. [ ] All framework definitions loadable via import
3. [ ] Control hierarchy queries return correct results
4. [ ] Cross-framework mappings queryable bidirectionally
5. [ ] Implementation sync events processed reliably
6. [ ] Gap analysis returns accurate results
7. [ ] Multi-tenant isolation verified by security audit
8. [ ] Performance benchmarks meet targets

---

### Section: Related Decisions

- **ADR-002**: Agent Orchestration Architecture (agents query control graph)
- **ADR-003**: Evidence Collection Pipeline (evidence links to controls)
- **ADR-005**: Multi-Tenancy Strategy (isolation patterns)

---

## Output Format Requirements

1. Generate complete ADR in Markdown format
2. Include all code examples with proper syntax highlighting
3. Include Mermaid diagrams for architecture visualization
4. Use tables for comparison matrices
5. Include YAML/JSON for configuration examples
6. Total length: 4,000-6,000 words

---

## Quality Criteria

The generated ADR must:
- [ ] Clearly state the decision and rationale
- [ ] Document all considered alternatives fairly
- [ ] Include concrete implementation guidance
- [ ] Provide measurable success criteria
- [ ] Reference related architectural decisions
- [ ] Be understandable by both technical and non-technical stakeholders
- [ ] Include diagrams for complex concepts
- [ ] Provide enough detail for implementation to begin

---

## Execution Instructions

1. Read the SDD Section 5 (Data Architecture) for context
2. Generate the complete ADR following the structure above
3. Ensure all code examples are syntactically correct
4. Include at least 2 Mermaid diagrams (data model, sync flow)
5. Output as a single Markdown file
6. Self-validate against quality criteria before completion
