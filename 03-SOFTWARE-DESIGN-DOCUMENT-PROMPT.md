---
title: CODITECT-COMPLIANCE Software Design Document Prompt
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
summary: 'CODITECT-COMPLIANCE Software Design Document Prompt Prompt ID: 03-SOFTWARE-DESIGN-DOCUMENT
  Output: Dependencies: 01-PRODUCT-DEFINITION, 02-PRODUCT-REQUIREMENTS Estimated Tokens:
  30,000-40,000 Generate a comprehensive Software Design Document (SDD)...'
---
# CODITECT-COMPLIANCE Software Design Document Prompt

**Prompt ID:** 03-SOFTWARE-DESIGN-DOCUMENT  
**Output:** `docs/architecture/SDD.md`  
**Dependencies:** 01-PRODUCT-DEFINITION, 02-PRODUCT-REQUIREMENTS  
**Estimated Tokens:** 30,000-40,000

---

## Objective

Generate a comprehensive Software Design Document (SDD) that translates product requirements into a detailed system architecture. This document defines the high-level structure, component interactions, data flows, and design decisions enabling autonomous implementation.

---

## Instructions

You are CODITECT-CORE generating a Software Design Document. Reference the Product Definition and PRD for requirements context. Generate a complete architectural blueprint suitable for implementation by autonomous agents.

---

## Required Document Sections

### Section 1: Document Overview

Generate document metadata and purpose:

```yaml
document:
  title: "CODITECT-COMPLIANCE Software Design Document"
  version: "1.0.0"
  status: "Draft"
  classification: "Internal - Engineering"
  
purpose: |
  This SDD defines the software architecture for CODITECT-COMPLIANCE,
  providing sufficient detail for autonomous implementation by CODITECT-CORE.
  
scope: |
  - System architecture and component design
  - Data models and storage strategy
  - Integration patterns and APIs
  - Security architecture
  - Deployment architecture
  
audience:
  - CODITECT-CORE (primary - autonomous implementation)
  - Engineering team (review and oversight)
  - Security team (security review)
  - DevOps team (deployment planning)
```

---

### Section 2: System Context (C4 Level 1)

Generate a C4 Context diagram description:

```yaml
system_context:
  name: "CODITECT-COMPLIANCE"
  description: "Enterprise compliance and regulatory oversight management system"
  
  actors:
    compliance_manager:
      type: "Person"
      description: "Manages compliance program, configures frameworks, reviews posture"
      interactions:
        - "Configures compliance frameworks and controls"
        - "Reviews compliance posture and gaps"
        - "Manages audit workflows"
        
    security_engineer:
      type: "Person"
      description: "Configures integrations, manages evidence collection"
      interactions:
        - "Configures integration connections"
        - "Reviews security findings"
        - "Remediates compliance issues"
        
    auditor:
      type: "Person"
      description: "External auditor reviewing compliance evidence"
      interactions:
        - "Reviews evidence packages"
        - "Requests additional evidence"
        - "Documents findings"
        
    customer_prospect:
      type: "Person"
      description: "Potential customer viewing Trust Center"
      interactions:
        - "Views compliance posture"
        - "Downloads certifications"
        - "Submits security questionnaires"
        
  external_systems:
    cloud_providers:
      type: "External System"
      systems: ["AWS", "GCP", "Azure"]
      interactions: "Evidence collection via APIs"
      
    identity_providers:
      type: "External System"
      systems: ["Okta", "Google Workspace", "Azure AD"]
      interactions: "User/access data collection"
      
    code_platforms:
      type: "External System"
      systems: ["GitHub", "GitLab"]
      interactions: "Repository and security data"
      
    ticketing_systems:
      type: "External System"
      systems: ["Jira", "Linear", "ServiceNow"]
      interactions: "Issue creation and tracking"
      
    coditect_core:
      type: "Internal System"
      description: "CODITECT autonomous development platform"
      interactions: "Compliance-as-code integration"
      
    llm_provider:
      type: "External System"
      systems: ["Claude API"]
      interactions: "AI agent reasoning and generation"
```

Generate a text description of the context diagram with all relationships.

---

### Section 3: Container Architecture (C4 Level 2)

Generate detailed container descriptions:

```yaml
containers:
  # Frontend Containers
  web_application:
    name: "Compliance Web Application"
    technology: "React 18, TypeScript, Vite"
    description: "Primary user interface for compliance management"
    responsibilities:
      - Dashboard and posture visualization
      - Framework and control management
      - Evidence browsing and search
      - Integration configuration
      - Audit workflow management
      - Settings and user management
    interfaces:
      - "REST/GraphQL API Gateway"
      - "WebSocket for real-time updates"
    deployment: "CDN + Static hosting (GCP Cloud Storage)"
    
  trust_center_portal:
    name: "Trust Center Portal"
    technology: "Next.js, React, TypeScript"
    description: "Public-facing compliance posture portal"
    responsibilities:
      - Public compliance status display
      - Document sharing and NDA gating
      - Questionnaire submission
    interfaces:
      - "Trust Center API"
    deployment: "Vercel or GCP Cloud Run"
    
  theia_extension:
    name: "Compliance Theia Extension"
    technology: "TypeScript, Eclipse Theia SDK"
    description: "IDE integration for developer compliance views"
    responsibilities:
      - Inline compliance indicators
      - Control-to-code mapping
      - Evidence attachment from IDE
    interfaces:
      - "Compliance API Gateway"
    deployment: "Bundled with CODITECT IDE"
    
  # API Containers
  api_gateway:
    name: "API Gateway"
    technology: "Kong or Envoy"
    description: "Unified API entry point with authentication and routing"
    responsibilities:
      - Request authentication (JWT validation)
      - Rate limiting
      - Request routing to services
      - API versioning
      - Request/response logging
    deployment: "Kubernetes Ingress"
    
  graphql_api:
    name: "GraphQL API Service"
    technology: "Python, Strawberry GraphQL, FastAPI"
    description: "GraphQL API for complex queries"
    responsibilities:
      - Control graph queries
      - Evidence and posture queries
      - Subscription for real-time updates
    deployment: "Kubernetes Deployment"
    
  rest_api:
    name: "REST API Service"
    technology: "Python, FastAPI"
    description: "REST API for CRUD operations"
    responsibilities:
      - Resource CRUD operations
      - Webhook endpoints
      - Integration callbacks
    deployment: "Kubernetes Deployment"
    
  # Core Services
  control_graph_service:
    name: "Control Graph Service"
    technology: "Python, Neo4j Driver"
    description: "Manages compliance control graph"
    responsibilities:
      - Framework and control CRUD
      - Cross-framework mapping
      - Control relationship queries
      - Regulatory intelligence ingestion
    data_store: "Neo4j"
    deployment: "Kubernetes Deployment"
    
  evidence_service:
    name: "Evidence Service"
    technology: "Python, FastAPI"
    description: "Manages evidence collection and storage"
    responsibilities:
      - Evidence item CRUD
      - Evidence collection orchestration
      - Gap detection
      - Evidence packaging
    data_store: "FoundationDB + Object Storage (GCS)"
    deployment: "Kubernetes Deployment"
    
  monitoring_service:
    name: "Continuous Monitoring Service"
    technology: "Python, FastAPI"
    description: "Executes and manages compliance checks"
    responsibilities:
      - Check scheduling and execution
      - Posture score calculation
      - Alert generation
      - Check result storage
    data_store: "FoundationDB"
    deployment: "Kubernetes Deployment"
    
  integration_service:
    name: "Integration Service"
    technology: "Python, FastAPI"
    description: "Manages external system integrations"
    responsibilities:
      - Integration CRUD
      - Credential management
      - Connection health monitoring
      - Data normalization
    data_store: "FoundationDB + Secrets Manager"
    deployment: "Kubernetes Deployment"
    
  agent_orchestrator:
    name: "Agent Orchestrator Service"
    technology: "Python, FastAPI"
    description: "Orchestrates compliance AI agents"
    responsibilities:
      - Agent task queue management
      - Agent execution environment
      - Human-in-the-loop workflows
      - Agent metrics and logging
    data_store: "Redis Streams + FoundationDB"
    deployment: "Kubernetes Deployment"
    
  notification_service:
    name: "Notification Service"
    technology: "Python, FastAPI"
    description: "Handles all system notifications"
    responsibilities:
      - Email notifications
      - Slack/webhook integrations
      - In-app notification management
      - Notification preferences
    deployment: "Kubernetes Deployment"
    
  audit_service:
    name: "Audit Service"
    technology: "Python, FastAPI"
    description: "Manages audit workflows"
    responsibilities:
      - Audit lifecycle management
      - PBC list management
      - Auditor portal
      - Finding tracking
    data_store: "FoundationDB"
    deployment: "Kubernetes Deployment"
    
  # Data Stores
  foundationdb:
    name: "FoundationDB Cluster"
    technology: "FoundationDB 7.x"
    description: "Primary transactional data store"
    data_stored:
      - Organizations and users
      - Evidence items metadata
      - Check results
      - Integrations
      - Audits
      - Agent executions
    deployment: "Kubernetes StatefulSet or Managed Service"
    
  neo4j:
    name: "Neo4j Graph Database"
    technology: "Neo4j 5.x"
    description: "Control graph and relationship store"
    data_stored:
      - Frameworks and controls
      - Control mappings
      - Asset-control relationships
      - Risk-control relationships
    deployment: "Neo4j Aura or Self-managed"
    
  redis:
    name: "Redis Cluster"
    technology: "Redis 7.x"
    description: "Caching and message streaming"
    data_stored:
      - Session cache
      - API response cache
      - Agent task queue (Streams)
      - Real-time subscriptions (Pub/Sub)
    deployment: "Redis Enterprise or Self-managed"
    
  object_storage:
    name: "Object Storage"
    technology: "Google Cloud Storage"
    description: "Evidence file storage"
    data_stored:
      - Evidence files (configs, logs, screenshots)
      - Document uploads
      - Report exports
    deployment: "GCS multi-regional"
```

Generate a text description of container interactions and data flows.

---

### Section 4: Component Design (C4 Level 3)

For each major service, generate component breakdown:

#### 4.1 Control Graph Service Components

```yaml
control_graph_service:
  components:
    framework_manager:
      description: "Manages framework lifecycle"
      responsibilities:
        - Framework CRUD operations
        - Framework versioning
        - Framework import (OSCAL, CSV)
      interfaces:
        exposed:
          - "FrameworkAPI (REST)"
        consumed:
          - "Neo4j Graph API"
          
    control_manager:
      description: "Manages control definitions"
      responsibilities:
        - Control CRUD operations
        - Control tagging and categorization
        - Custom field management
      interfaces:
        exposed:
          - "ControlAPI (REST)"
        consumed:
          - "Neo4j Graph API"
          
    mapping_engine:
      description: "Manages cross-framework mappings"
      responsibilities:
        - Semantic similarity calculation
        - Mapping suggestion generation
        - Mapping approval workflow
      interfaces:
        exposed:
          - "MappingAPI (REST)"
        consumed:
          - "Neo4j Graph API"
          - "LLM API (for semantic analysis)"
          
    graph_query_engine:
      description: "Executes complex graph queries"
      responsibilities:
        - Control traversal queries
        - Gap analysis queries
        - Posture aggregation queries
      interfaces:
        exposed:
          - "GraphQL Resolvers"
        consumed:
          - "Neo4j Graph API"
          
    regulatory_parser:
      description: "Parses regulatory documents into controls"
      responsibilities:
        - Document parsing (PDF, HTML)
        - Control extraction
        - Update detection
      interfaces:
        consumed:
          - "LLM API"
          - "Neo4j Graph API"
```

#### 4.2 Evidence Service Components

```yaml
evidence_service:
  components:
    collection_orchestrator:
      description: "Orchestrates evidence collection jobs"
      responsibilities:
        - Job scheduling (cron-based)
        - Job execution coordination
        - Retry and error handling
      interfaces:
        exposed:
          - "CollectionJobAPI (REST)"
        consumed:
          - "Integration Service API"
          - "Redis Streams"
          
    evidence_processor:
      description: "Processes and stores evidence items"
      responsibilities:
        - Evidence normalization
        - Metadata extraction
        - Storage coordination
        - Deduplication
      interfaces:
        consumed:
          - "Object Storage API"
          - "FoundationDB"
          
    gap_detector:
      description: "Detects evidence gaps"
      responsibilities:
        - Freshness monitoring
        - Coverage analysis
        - Gap alerting
      interfaces:
        exposed:
          - "GapAnalysisAPI (REST)"
        consumed:
          - "FoundationDB"
          - "Notification Service"
          
    evidence_validator:
      description: "Validates evidence against criteria"
      responsibilities:
        - Rule-based validation
        - AI-assisted validation
        - Validation result storage
      interfaces:
        consumed:
          - "LLM API"
          - "FoundationDB"
          
    package_generator:
      description: "Generates audit evidence packages"
      responsibilities:
        - Package composition
        - PDF generation
        - ZIP packaging
      interfaces:
        exposed:
          - "PackageAPI (REST)"
        consumed:
          - "Object Storage API"
```

#### 4.3 Agent Orchestrator Components

```yaml
agent_orchestrator:
  components:
    task_queue_manager:
      description: "Manages agent task queue"
      responsibilities:
        - Task enqueue/dequeue
        - Priority management
        - Dead letter handling
      interfaces:
        exposed:
          - "TaskQueueAPI (REST)"
        consumed:
          - "Redis Streams"
          
    agent_executor:
      description: "Executes agent tasks"
      responsibilities:
        - Agent instantiation
        - Tool invocation coordination
        - Token budget management
        - Execution logging
      interfaces:
        consumed:
          - "LLM API"
          - "Tool APIs (various)"
          
    tool_registry:
      description: "Manages available agent tools"
      responsibilities:
        - Tool registration
        - Tool permission management
        - Tool invocation routing
      interfaces:
        exposed:
          - "ToolRegistryAPI (REST)"
          
    approval_workflow:
      description: "Manages human-in-the-loop approvals"
      responsibilities:
        - Approval request creation
        - Approval routing
        - Timeout and escalation
      interfaces:
        exposed:
          - "ApprovalAPI (REST)"
        consumed:
          - "Notification Service"
          
    checkpoint_manager:
      description: "Manages agent execution checkpoints"
      responsibilities:
        - Checkpoint creation
        - State recovery
        - Checkpoint cleanup
      interfaces:
        consumed:
          - "FoundationDB"
```

Generate similar component breakdowns for remaining services.

---

### Section 5: Data Architecture

#### 5.1 Data Model Overview

```yaml
data_domains:
  compliance_graph:
    store: "Neo4j"
    entities:
      - Framework
      - Control
      - ControlMapping
      - Policy
      - Procedure
    reasoning: "Graph database optimal for relationship-heavy compliance data"
    
  operational_data:
    store: "FoundationDB"
    entities:
      - Organization
      - User
      - Integration
      - EvidenceItem (metadata)
      - CheckResult
      - AgentExecution
      - Audit
      - Risk
    reasoning: "ACID transactions, horizontal scaling, strong consistency"
    
  file_storage:
    store: "Google Cloud Storage"
    entities:
      - Evidence files
      - Document uploads
      - Generated reports
    reasoning: "Cost-effective, highly durable object storage"
    
  cache_and_queue:
    store: "Redis"
    entities:
      - Session data
      - API response cache
      - Agent task queue
      - Real-time event streams
    reasoning: "Low-latency caching and pub/sub"
```

#### 5.2 Core Entity Schemas

Generate detailed schemas for all core entities. For example:

```yaml
Organization:
  description: "Tenant organization in the system"
  storage: "FoundationDB"
  key_space: "org/{org_id}"
  schema:
    id:
      type: "uuid"
      description: "Unique identifier"
      required: true
    name:
      type: "string"
      max_length: 256
      required: true
    slug:
      type: "string"
      pattern: "^[a-z0-9-]+$"
      max_length: 64
      unique: true
      required: true
    subscription_tier:
      type: "enum"
      values: ["essentials", "growth", "enterprise"]
      required: true
    settings:
      type: "json"
      description: "Organization-level settings"
    created_at:
      type: "timestamp"
      required: true
    updated_at:
      type: "timestamp"
      required: true
  indexes:
    - field: "slug"
      unique: true
  relationships:
    - has_many: "User"
    - has_many: "Integration"
    - has_many: "EvidenceItem"
    - subscribes_to_many: "Framework"

Control:
  description: "Compliance control definition"
  storage: "Neo4j"
  label: "Control"
  schema:
    id:
      type: "uuid"
      required: true
    framework_id:
      type: "uuid"
      required: true
    control_id:
      type: "string"
      max_length: 64
      description: "Framework-specific ID (e.g., CC6.1)"
      required: true
    name:
      type: "string"
      max_length: 256
      required: true
    description:
      type: "text"
      max_length: 4000
    objective:
      type: "text"
      max_length: 2000
    category:
      type: "enum"
      values: ["ACCESS_CONTROL", "CHANGE_MANAGEMENT", "LOGGING_MONITORING", ...]
      required: true
    implementation_guidance:
      type: "text"
    testing_procedure:
      type: "text"
    evidence_requirements:
      type: "json_array"
      item_schema:
        type: "string"
        description: "Evidence type required"
    tags:
      type: "string_array"
    custom_fields:
      type: "json"
  relationships:
    - belongs_to: "Framework" (edge: BELONGS_TO)
    - overlaps_with: "Control" (edge: OVERLAPS)
    - mapped_to: "Control" (edge: MAPS_TO, properties: [confidence, relationship_type])
    - implemented_by: "Policy" (edge: IMPLEMENTED_BY)
    - verified_by: "Test" (edge: VERIFIED_BY)

EvidenceItem:
  description: "Collected compliance evidence"
  storage: "FoundationDB (metadata) + GCS (files)"
  key_space: "org/{org_id}/evidence/{evidence_id}"
  schema:
    id:
      type: "uuid"
      required: true
    organization_id:
      type: "uuid"
      required: true
    type:
      type: "enum"
      values: ["CONFIG", "LOG", "SCREENSHOT", "DOCUMENT", "API_RESPONSE"]
      required: true
    source_integration_id:
      type: "uuid"
    collected_at:
      type: "timestamp"
      required: true
    content_hash:
      type: "string"
      description: "SHA-256 hash of content"
      required: true
    storage_location:
      type: "string"
      description: "GCS object path"
      required: true
    control_ids:
      type: "uuid_array"
      description: "Controls this evidence supports"
    metadata:
      type: "json"
      description: "Source-specific metadata"
    retention_until:
      type: "date"
    version:
      type: "integer"
      default: 1
  indexes:
    - fields: ["organization_id", "collected_at"]
    - fields: ["organization_id", "type"]
    - fields: ["control_ids"]
```

Generate complete schemas for all entities listed in the PRD.

---

### Section 6: API Design

#### 6.1 API Architecture

```yaml
api_architecture:
  style: "REST + GraphQL hybrid"
  versioning: "URL path versioning (v1, v2)"
  authentication: "JWT Bearer tokens (OAuth 2.0)"
  
  rest_api:
    base_url: "https://api.compliance.coditect.ai/v1"
    use_cases:
      - CRUD operations on resources
      - Webhook endpoints
      - File uploads
      - Simple queries
    standards:
      - JSON:API specification for resource responses
      - Problem Details (RFC 7807) for errors
      
  graphql_api:
    endpoint: "https://api.compliance.coditect.ai/graphql"
    use_cases:
      - Complex nested queries
      - Control graph traversal
      - Dashboard aggregations
      - Real-time subscriptions
    standards:
      - GraphQL specification
      - Relay pagination
```

#### 6.2 REST API Endpoints

Generate complete endpoint definitions:

```yaml
endpoints:
  # Frameworks
  frameworks:
    list:
      method: "GET"
      path: "/v1/frameworks"
      description: "List all frameworks"
      query_params:
        - name: "status"
          type: "string"
          enum: ["ACTIVE", "DEPRECATED", "DRAFT"]
        - name: "page"
          type: "integer"
        - name: "per_page"
          type: "integer"
      response:
        status: 200
        body: "FrameworkListResponse"
        
    get:
      method: "GET"
      path: "/v1/frameworks/{framework_id}"
      description: "Get framework details"
      path_params:
        - name: "framework_id"
          type: "uuid"
      response:
        status: 200
        body: "FrameworkResponse"
        
    create:
      method: "POST"
      path: "/v1/frameworks"
      description: "Create new framework"
      request_body: "FrameworkCreateRequest"
      response:
        status: 201
        body: "FrameworkResponse"
        
  # Controls
  controls:
    list:
      method: "GET"
      path: "/v1/frameworks/{framework_id}/controls"
      description: "List controls in framework"
      query_params:
        - name: "category"
          type: "string"
        - name: "search"
          type: "string"
      response:
        status: 200
        body: "ControlListResponse"
        
    get:
      method: "GET"
      path: "/v1/controls/{control_id}"
      description: "Get control details"
      response:
        status: 200
        body: "ControlResponse"
        
    get_mappings:
      method: "GET"
      path: "/v1/controls/{control_id}/mappings"
      description: "Get control mappings to other frameworks"
      response:
        status: 200
        body: "ControlMappingListResponse"
        
  # Evidence
  evidence:
    list:
      method: "GET"
      path: "/v1/evidence"
      description: "List evidence items"
      query_params:
        - name: "control_id"
          type: "uuid"
        - name: "type"
          type: "string"
        - name: "from_date"
          type: "datetime"
        - name: "to_date"
          type: "datetime"
      response:
        status: 200
        body: "EvidenceListResponse"
        
    get:
      method: "GET"
      path: "/v1/evidence/{evidence_id}"
      description: "Get evidence item details"
      response:
        status: 200
        body: "EvidenceResponse"
        
    download:
      method: "GET"
      path: "/v1/evidence/{evidence_id}/download"
      description: "Download evidence file"
      response:
        status: 200
        content_type: "application/octet-stream"
        
  # Integrations
  integrations:
    list:
      method: "GET"
      path: "/v1/integrations"
      response:
        body: "IntegrationListResponse"
        
    create:
      method: "POST"
      path: "/v1/integrations"
      request_body: "IntegrationCreateRequest"
      response:
        status: 201
        body: "IntegrationResponse"
        
    test:
      method: "POST"
      path: "/v1/integrations/{integration_id}/test"
      description: "Test integration connection"
      response:
        status: 200
        body: "IntegrationTestResponse"
        
    sync:
      method: "POST"
      path: "/v1/integrations/{integration_id}/sync"
      description: "Trigger manual sync"
      response:
        status: 202
        body: "SyncJobResponse"
```

#### 6.3 GraphQL Schema

Generate complete GraphQL schema:

```graphql
# Type definitions
type Framework {
  id: ID!
  name: String!
  shortName: String!
  version: String!
  status: FrameworkStatus!
  controls(
    category: ControlCategory
    search: String
    first: Int
    after: String
  ): ControlConnection!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Control {
  id: ID!
  framework: Framework!
  controlId: String!
  name: String!
  description: String
  objective: String
  category: ControlCategory!
  implementationGuidance: String
  testingProcedure: String
  evidenceRequirements: [String!]
  tags: [String!]
  
  # Relationships
  mappings: [ControlMapping!]!
  implementation(organizationId: ID!): ControlImplementation
  evidence(
    organizationId: ID!
    first: Int
    after: String
  ): EvidenceConnection!
  checks: [ControlCheck!]!
}

type ControlMapping {
  id: ID!
  sourceControl: Control!
  targetControl: Control!
  relationshipType: MappingRelationship!
  confidenceScore: Int!
  status: MappingStatus!
}

type EvidenceItem {
  id: ID!
  type: EvidenceType!
  sourceIntegration: Integration
  collectedAt: DateTime!
  controls: [Control!]!
  metadata: JSON
  downloadUrl: String!
}

type CompliancePosture {
  organizationId: ID!
  overallScore: Float!
  frameworkScores: [FrameworkScore!]!
  categoryScores: [CategoryScore!]!
  trend: PostureTrend!
  lastUpdated: DateTime!
}

# Queries
type Query {
  # Frameworks
  frameworks(status: FrameworkStatus): [Framework!]!
  framework(id: ID!): Framework
  
  # Controls
  control(id: ID!): Control
  searchControls(
    query: String!
    frameworks: [ID!]
    categories: [ControlCategory!]
    first: Int
    after: String
  ): ControlConnection!
  
  # Evidence
  evidence(
    controlId: ID
    type: EvidenceType
    fromDate: DateTime
    toDate: DateTime
    first: Int
    after: String
  ): EvidenceConnection!
  
  # Posture
  compliancePosture(frameworkId: ID): CompliancePosture!
  
  # Gap Analysis
  evidenceGaps(
    frameworkId: ID
    freshnessThreshold: Int
  ): [EvidenceGap!]!
  
  # Agent Activity
  agentExecutions(
    agentType: AgentType
    status: ExecutionStatus
    first: Int
    after: String
  ): AgentExecutionConnection!
}

# Mutations
type Mutation {
  # Frameworks
  createFramework(input: CreateFrameworkInput!): Framework!
  updateFramework(id: ID!, input: UpdateFrameworkInput!): Framework!
  
  # Controls
  updateControlImplementation(
    controlId: ID!
    input: UpdateImplementationInput!
  ): ControlImplementation!
  
  # Integrations
  createIntegration(input: CreateIntegrationInput!): Integration!
  triggerSync(integrationId: ID!): SyncJob!
  
  # Agent Tasks
  createAgentTask(input: CreateAgentTaskInput!): AgentTask!
  approveAgentAction(taskId: ID!, approved: Boolean!): AgentTask!
}

# Subscriptions
type Subscription {
  postureUpdated(organizationId: ID!): CompliancePosture!
  checkResultCreated(organizationId: ID!): CheckResult!
  alertCreated(organizationId: ID!): Alert!
}
```

---

### Section 7: Security Architecture

```yaml
security_architecture:
  authentication:
    primary: "OAuth 2.0 / OIDC"
    providers:
      - "Internal (email/password + MFA)"
      - "Google Workspace SSO"
      - "Okta SSO"
      - "Azure AD SSO"
    token_format: "JWT"
    token_lifetime:
      access_token: "15 minutes"
      refresh_token: "7 days"
    mfa:
      methods: ["TOTP", "WebAuthn"]
      enforcement: "Required for all users"
      
  authorization:
    model: "Role-Based Access Control (RBAC)"
    roles:
      owner:
        description: "Organization owner"
        permissions: ["*"]
      admin:
        description: "Administrator"
        permissions:
          - "users:*"
          - "integrations:*"
          - "frameworks:*"
          - "evidence:*"
          - "audits:*"
          - "settings:*"
      compliance_manager:
        description: "Compliance team member"
        permissions:
          - "frameworks:read"
          - "controls:*"
          - "evidence:*"
          - "audits:*"
          - "risks:*"
      security_engineer:
        description: "Security team member"
        permissions:
          - "integrations:*"
          - "evidence:read"
          - "checks:*"
          - "alerts:*"
      viewer:
        description: "Read-only access"
        permissions:
          - "*:read"
      auditor:
        description: "External auditor"
        permissions:
          - "evidence:read (scoped to audit)"
          - "controls:read"
          - "audits:read (scoped)"
          
  data_protection:
    encryption_at_rest:
      method: "AES-256-GCM"
      key_management: "Google Cloud KMS"
      scope: "All data stores"
    encryption_in_transit:
      method: "TLS 1.3"
      scope: "All connections"
    secrets_management:
      provider: "Google Secret Manager"
      rotation: "90 days"
      
  tenant_isolation:
    model: "Logical multi-tenancy"
    enforcement:
      - "Organization ID in all queries"
      - "Row-level security in FoundationDB"
      - "Neo4j label-based isolation"
    audit_logging:
      - "All data access logged"
      - "Cross-tenant access attempts blocked and alerted"
      
  api_security:
    rate_limiting:
      default: "1000 requests/minute"
      burst: "100 requests/second"
    input_validation:
      - "JSON Schema validation"
      - "SQL/NoSQL injection prevention"
      - "XSS prevention"
    cors:
      allowed_origins: ["*.coditect.ai", "localhost:*"]
```

---

### Section 8: Integration Patterns

```yaml
integration_architecture:
  connector_pattern:
    description: "Standardized integration connector pattern"
    components:
      connector_interface:
        methods:
          - "authenticate(): Credentials"
          - "test_connection(): ConnectionStatus"
          - "list_resources(): List[Resource]"
          - "collect_evidence(resource): EvidenceItem"
          - "execute_action(action): ActionResult"
      credential_adapter:
        supported:
          - "OAuth2 (authorization code, client credentials)"
          - "API Key"
          - "AWS IAM Role"
          - "Service Account"
      rate_limiter:
        strategy: "Token bucket"
        configuration: "Per-integration type"
      retry_handler:
        strategy: "Exponential backoff"
        max_retries: 3
        
  evidence_collection_flow:
    steps:
      1_schedule:
        trigger: "Cron or event"
        action: "Create collection job"
      2_dispatch:
        action: "Place job on Redis queue"
      3_execute:
        action: "Worker picks up job"
        substeps:
          - "Load connector for integration type"
          - "Authenticate with cached credentials"
          - "Query for resources"
          - "Collect evidence for each resource"
          - "Store evidence in GCS"
          - "Update evidence metadata in FoundationDB"
      4_validate:
        action: "Validate evidence against criteria"
      5_notify:
        action: "Send notifications if gaps or failures"
        
  webhook_handling:
    inbound:
      endpoint: "/v1/webhooks/{integration_type}"
      security:
        - "Signature validation (HMAC)"
        - "IP allowlisting (optional)"
        - "Replay protection (timestamp + nonce)"
      processing:
        - "Async via message queue"
        - "Idempotency via event ID"
    outbound:
      supported_events:
        - "posture.changed"
        - "evidence.collected"
        - "check.failed"
        - "alert.created"
      delivery:
        - "At-least-once via retry queue"
        - "Exponential backoff on failure"
        - "Dead letter after 72 hours"
```

---

### Section 9: Deployment Architecture

```yaml
deployment_architecture:
  platform: "Google Kubernetes Engine (GKE)"
  
  environments:
    development:
      cluster: "coditect-compliance-dev"
      region: "us-central1"
      replicas: 1
      resources: "Minimal"
      
    staging:
      cluster: "coditect-compliance-staging"
      region: "us-central1"
      replicas: 2
      resources: "Production-like"
      
    production:
      cluster: "coditect-compliance-prod"
      regions: ["us-central1", "europe-west1"]
      replicas: "3+ with HPA"
      resources: "Full production"
      
  kubernetes_resources:
    namespaces:
      - "compliance-api"
      - "compliance-workers"
      - "compliance-agents"
      - "compliance-data"
      
    deployments:
      api_services:
        - name: "graphql-api"
          replicas: 3
          resources:
            requests: {cpu: "500m", memory: "512Mi"}
            limits: {cpu: "2000m", memory: "2Gi"}
        - name: "rest-api"
          replicas: 3
          resources:
            requests: {cpu: "500m", memory: "512Mi"}
            limits: {cpu: "2000m", memory: "2Gi"}
            
      workers:
        - name: "evidence-collector"
          replicas: 5
          resources:
            requests: {cpu: "250m", memory: "256Mi"}
            limits: {cpu: "1000m", memory: "1Gi"}
        - name: "check-executor"
          replicas: 5
          resources:
            requests: {cpu: "250m", memory: "256Mi"}
            limits: {cpu: "1000m", memory: "1Gi"}
            
      agents:
        - name: "agent-executor"
          replicas: 3
          resources:
            requests: {cpu: "500m", memory: "1Gi"}
            limits: {cpu: "2000m", memory: "4Gi"}
            
    statefulsets:
      - name: "redis"
        replicas: 3
        storage: "10Gi SSD"
        
    services:
      - name: "api-gateway"
        type: "LoadBalancer"
        annotations:
          - "cloud.google.com/load-balancer-type: External"
          
  infrastructure_as_code:
    tool: "Terraform"
    modules:
      - "gke-cluster"
      - "foundationdb-cluster"
      - "neo4j-aura"
      - "redis-cluster"
      - "cloud-storage"
      - "cloud-kms"
      - "secret-manager"
      - "cloud-armor"
      - "cloud-cdn"
```

---

### Section 10: Observability

```yaml
observability:
  logging:
    platform: "Google Cloud Logging"
    format: "Structured JSON"
    levels: ["DEBUG", "INFO", "WARN", "ERROR"]
    retention: "30 days (hot), 1 year (cold)"
    required_fields:
      - "timestamp"
      - "level"
      - "service"
      - "trace_id"
      - "organization_id"
      - "user_id"
      - "message"
      
  metrics:
    platform: "Google Cloud Monitoring + Prometheus"
    types:
      application:
        - "api_request_duration_seconds"
        - "api_request_total"
        - "evidence_collection_duration_seconds"
        - "check_execution_duration_seconds"
        - "agent_token_usage_total"
      business:
        - "compliance_posture_score"
        - "evidence_gaps_total"
        - "active_integrations"
        - "agent_tasks_completed"
        
  tracing:
    platform: "Google Cloud Trace + OpenTelemetry"
    sampling: "10% in production, 100% in staging"
    propagation: "W3C Trace Context"
    
  alerting:
    platform: "Google Cloud Monitoring + PagerDuty"
    policies:
      critical:
        - "API error rate > 5%"
        - "API latency P99 > 5s"
        - "Database connection failures"
      warning:
        - "Evidence collection failures > 10%"
        - "Agent execution timeouts > 5%"
```

---

## Output Format

Generate the document as a single markdown file with:
- C4 diagram descriptions (text-based, renderable as diagrams)
- YAML code blocks for structured specifications
- Complete data model schemas
- Full API specifications
- Clear section navigation

---

## Acceptance Criteria

The generated document must:
1. Be 10,000-15,000 words
2. Include C4 diagrams for context, container, and component levels
3. Include complete data models for all core entities
4. Include complete API specifications (REST + GraphQL)
5. Include security architecture details
6. Include deployment architecture
7. Be sufficient for autonomous implementation

---

## Execution

Generate the complete Software Design Document now.
