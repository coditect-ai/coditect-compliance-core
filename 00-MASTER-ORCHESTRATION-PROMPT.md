# CODITECT Compliance Module - Master Orchestration Prompt

**Module Name:** CODITECT-COMPLIANCE  
**Version:** 1.0.0  
**Classification:** Enterprise Plugin Module  
**Target Verticals:** Healthcare (HIPAA/FDA), Fintech (SOC2/PCI), AI Governance (EU AI Act/ISO 42001)

---

## Executive Context

You are CODITECT-CORE, an autonomous AI development platform. This master prompt orchestrates the complete build of the **CODITECT-COMPLIANCE** module - an enterprise-grade compliance and regulatory oversight management system that rivals Vanta, Drata, and Hyperproof while providing unique multi-agent orchestration capabilities.

### Strategic Positioning

CODITECT-COMPLIANCE differentiates through:
1. **Agentic Compliance Automation** - AI agents that autonomously monitor, assess, and remediate compliance gaps
2. **Control Graph Architecture** - Graph-based compliance knowledge representation enabling cross-framework mapping
3. **Regulated Industry Focus** - Deep vertical specialization for healthcare AI and fintech
4. **Autonomous Evidence Collection** - Integration-driven continuous compliance monitoring
5. **CODITECT Integration** - Native integration with autonomous code generation ensuring compliant-by-default software delivery

---

## Build Orchestration Sequence

Execute the following prompts in sequence. Each prompt generates artifacts that subsequent prompts depend on.

### Phase 1: Foundation Documents

| Sequence | Prompt File | Output Artifact | Dependencies |
|----------|------------|-----------------|--------------|
| 1.1 | `01-PRODUCT-DEFINITION-PROMPT.md` | Product Definition Document | None |
| 1.2 | `02-PRODUCT-REQUIREMENTS-PROMPT.md` | Product Requirements Document (PRD) | 1.1 |
| 1.3 | `03-SOFTWARE-DESIGN-DOCUMENT-PROMPT.md` | Software Design Document (SDD) | 1.1, 1.2 |
| 1.4 | `04-TECHNICAL-DESIGN-DOCUMENT-PROMPT.md` | Technical Design Document (TDD) | 1.2, 1.3 |

### Phase 2: Architecture Decisions

| Sequence | Prompt File | Output Artifact | Dependencies |
|----------|------------|-----------------|--------------|
| 2.1 | `05-ADR-CONTROL-GRAPH-PROMPT.md` | ADR-001: Control Graph Database | 1.3, 1.4 |
| 2.2 | `06-ADR-AGENT-ORCHESTRATION-PROMPT.md` | ADR-002: Multi-Agent Orchestration | 1.3, 1.4 |
| 2.3 | `07-ADR-EVIDENCE-COLLECTION-PROMPT.md` | ADR-003: Evidence Collection Pipeline | 1.3, 1.4 |
| 2.4 | `08-ADR-INTEGRATION-FRAMEWORK-PROMPT.md` | ADR-004: Integration Framework | 1.3, 1.4 |
| 2.5 | `09-ADR-MULTI-TENANCY-PROMPT.md` | ADR-005: Multi-Tenancy & Isolation | 1.3, 1.4 |

### Phase 3: Full Stack Component Build

| Sequence | Prompt File | Output Component | Dependencies |
|----------|------------|------------------|--------------|
| 3.1 | `10-BUILD-DOMAIN-MODELS-PROMPT.md` | Domain Models & Schemas | 2.1-2.5 |
| 3.2 | `11-BUILD-CONTROL-GRAPH-PROMPT.md` | Control Graph Service | 3.1 |
| 3.3 | `12-BUILD-EVIDENCE-ENGINE-PROMPT.md` | Evidence Collection Engine | 3.1, 3.2 |
| 3.4 | `13-BUILD-AGENT-FRAMEWORK-PROMPT.md` | Compliance Agent Framework | 3.1, 3.2 |
| 3.5 | `14-BUILD-INTEGRATION-CONNECTORS-PROMPT.md` | Integration Connectors | 3.1, 3.3 |
| 3.6 | `15-BUILD-API-LAYER-PROMPT.md` | REST/GraphQL API Layer | 3.1-3.5 |
| 3.7 | `16-BUILD-UI-COMPONENTS-PROMPT.md` | UI Components (React/Theia) | 3.6 |
| 3.8 | `17-BUILD-TRUST-CENTER-PROMPT.md` | Trust Center Portal | 3.6, 3.7 |

### Phase 4: Quality & Deployment

| Sequence | Prompt File | Output Artifact | Dependencies |
|----------|------------|-----------------|--------------|
| 4.1 | `18-BUILD-TEST-SUITE-PROMPT.md` | Test Suite (Unit/Integration/E2E) | 3.1-3.8 |
| 4.2 | `19-BUILD-DEPLOYMENT-PROMPT.md` | K8s/Docker Deployment Configs | 3.1-3.8 |
| 4.3 | `20-BUILD-DOCUMENTATION-PROMPT.md` | Technical Documentation | All |

---

## Core Domain Model Reference

The CODITECT-COMPLIANCE module operates on these core entities:

```yaml
entities:
  # Regulatory Foundation
  Regulation:
    description: "Legal/standard framework (SOC2, HIPAA, EU AI Act)"
    relationships: [requires → Control, contains → Article]
    
  Control:
    description: "Atomic compliance requirement"
    relationships: [implemented_by → Policy, verified_by → Test, overlaps → Control]
    
  Policy:
    description: "Organizational policy document"
    relationships: [implements → Control, applies_to → Asset]

  # Asset Management  
  Asset:
    description: "Cloud resource, application, dataset, AI model"
    relationships: [subject_to → Regulation, runs → Test, contains → Data]
    
  AIModel:
    description: "AI/ML model with governance requirements"
    relationships: [classified_as → RiskCategory, trained_on → Dataset, deployed_on → Asset]

  # Evidence & Monitoring
  Test:
    description: "Automated compliance check"
    relationships: [verifies → Control, runs_on → Asset, produces → Evidence]
    
  Evidence:
    description: "Proof of compliance (logs, configs, screenshots)"
    relationships: [supports → Control, collected_from → Asset]
    
  # Risk & Issues
  Risk:
    description: "Identified compliance/security risk"
    relationships: [mitigated_by → Control, impacts → Asset]
    
  Issue:
    description: "Compliance gap or finding"
    relationships: [relates_to → Control, assigned_to → User, remediated_by → Task]

  # Organizational
  Organization:
    description: "Tenant organization"
    relationships: [owns → Asset, subscribes_to → Framework]
    
  User:
    description: "Platform user with roles"
    relationships: [belongs_to → Organization, assigned → Task]
```

---

## Technology Stack Constraints

All implementations MUST use:

```yaml
backend:
  language: Python 3.12+
  framework: FastAPI
  database: FoundationDB (primary), Neo4j (control graph)
  queue: Redis Streams
  cache: Redis
  
frontend:
  framework: React 18+ with TypeScript
  ide_integration: Eclipse Theia
  state: Zustand
  ui_components: shadcn/ui + Tailwind
  
infrastructure:
  container: Docker
  orchestration: Kubernetes
  cloud: GCP (primary), AWS (secondary)
  
ai_agents:
  framework: Custom CODITECT Agent Framework
  llm: Claude API (claude-sonnet-4)
  orchestration: Event-driven with FoundationDB
```

---

## Agent Role Definitions

The compliance module deploys these specialized agents:

| Agent | Role | Primary Tools | Output |
|-------|------|---------------|--------|
| `RegulatoryIntelligenceAgent` | Monitor regulatory changes | web_search, doc_parser | Regulation updates, control mappings |
| `ControlPostureAgent` | Assess control health | integration_apis, telemetry | Posture scores, gap analysis |
| `EvidenceCollectionAgent` | Gather compliance evidence | cloud_apis, logs, configs | Evidence items, audit packages |
| `RemediationAgent` | Plan and execute fixes | ticketing_apis, automation | Remediation tasks, status updates |
| `AuditPreparationAgent` | Prepare audit materials | doc_generator, evidence_mapper | Audit packages, narratives |
| `VendorRiskAgent` | Assess third-party risk | vendor_apis, questionnaires | Vendor scores, risk assessments |

---

## Framework Coverage Requirements

MVP must support:

```yaml
tier_1_frameworks:  # Full automation
  - SOC 2 Type II (Trust Services Criteria)
  - ISO 27001:2022
  - HIPAA (Security & Privacy Rules)
  
tier_2_frameworks:  # Partial automation
  - GDPR
  - PCI DSS 4.0
  - NIST CSF 2.0
  
tier_3_frameworks:  # AI Governance (roadmap)
  - EU AI Act
  - ISO/IEC 42001
  - NIST AI RMF
  - FDA SaMD Guidance
```

---

## Integration Requirements

Must integrate with (MVP):

```yaml
cloud_providers:
  - AWS (IAM, Config, CloudTrail, GuardDuty)
  - GCP (IAM, Security Command Center, Audit Logs)
  - Azure (AD, Security Center, Monitor)

identity_providers:
  - Okta
  - Google Workspace
  - Azure AD / Entra ID

hris:
  - Rippling
  - BambooHR
  - Workday

ticketing:
  - Jira
  - Linear
  - ServiceNow

code_repos:
  - GitHub
  - GitLab

vulnerability_scanners:
  - Snyk
  - Dependabot
  - Qualys
```

---

## Success Criteria

The build is complete when:

1. **Functional Completeness**
   - [ ] Control graph stores 500+ controls across 6+ frameworks
   - [ ] Evidence collection runs hourly for 10+ integration types
   - [ ] 5+ compliance agents operate autonomously
   - [ ] Trust Center serves real-time posture data

2. **Quality Gates**
   - [ ] Test coverage ≥ 80%
   - [ ] API response time P99 < 500ms
   - [ ] Agent token efficiency < 1000 tokens/tool call
   - [ ] Zero critical security vulnerabilities

3. **Documentation**
   - [ ] Complete API documentation (OpenAPI 3.1)
   - [ ] Architecture Decision Records (5+)
   - [ ] User guides for each role
   - [ ] Integration guides for each connector

---

## Execution Instructions

For each prompt in the sequence:

1. **Read** the prompt file completely
2. **Generate** the specified artifact(s)
3. **Validate** against stated acceptance criteria
4. **Store** in appropriate project structure
5. **Update** dependency graph for next prompt

Use checkpoint after each phase completion. If errors occur:
- Log error with context
- Attempt retry with modified approach (max 3)
- Escalate to human review if unresolvable

---

## Project Structure

```
coditect-compliance/
├── docs/
│   ├── product/
│   │   ├── PRODUCT_DEFINITION.md
│   │   ├── PRD.md
│   │   └── ROADMAP.md
│   ├── architecture/
│   │   ├── SDD.md
│   │   ├── TDD.md
│   │   └── adrs/
│   │       ├── ADR-001-control-graph.md
│   │       ├── ADR-002-agent-orchestration.md
│   │       └── ...
│   └── api/
│       └── openapi.yaml
├── src/
│   ├── domain/
│   ├── services/
│   ├── agents/
│   ├── integrations/
│   ├── api/
│   └── ui/
├── tests/
├── deploy/
└── scripts/
```

---

**BEGIN EXECUTION WITH PROMPT 01-PRODUCT-DEFINITION-PROMPT.md**
