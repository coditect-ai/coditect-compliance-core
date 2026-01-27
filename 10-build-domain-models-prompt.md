---
title: 'Prompt 10: Component Build - Core Domain Models'
type: reference
component_type: reference
version: 1.0.0
created: '2025-12-27'
updated: '2025-12-27'
status: deprecated
tags:
- ai-ml
- authentication
- security
- testing
- architecture
- automation
- backend
- cloud
summary: You are a senior software engineer implementing the core domain models for
  CODITECT-COMPLIANCE. These models form the foundation of the entire system...
moe_confidence: 0.950
moe_classified: 2025-12-31
---
# Prompt 10: Component Build - Core Domain Models

## Context

You are a senior software engineer implementing the core domain models for CODITECT-COMPLIANCE. These models form the foundation of the entire system and must be production-quality with comprehensive validation, serialization, and database mapping support.

## Output Specification

Generate complete, production-ready Python code for all domain models. Output should be 2,000-3,000 lines of code with full type hints, validation, and documentation.

## Implementation Requirements

### Technology Stack
- Python 3.12+
- Pydantic v2 for validation
- dataclasses for internal models
- FoundationDB-compatible serialization
- Neo4j-compatible graph conversion

### Code Quality Standards
- 100% type hint coverage
- Comprehensive docstrings (Google style)
- Pydantic validators for business rules
- Factory methods for common construction patterns
- Immutable where appropriate (frozen dataclasses)

## Domain Model Specifications

### 1. Organization & User Models

```python
# File: src/domain/models/organization.py

"""
Organization and user domain models.

These models represent the multi-tenant structure of the compliance platform.
"""

from datetime import datetime
from enum import Enum
from typing import List, Optional, Dict, Any
from pydantic import BaseModel, Field, field_validator, model_validator
from uuid import UUID, uuid4

class TenantTier(str, Enum):
    """Subscription tier determining features and limits."""
    STARTER = "starter"
    GROWTH = "growth"
    ENTERPRISE = "enterprise"

class DataResidency(str, Enum):
    """Data residency region for compliance."""
    US = "us"
    EU = "eu"
    APAC = "apac"

class OrganizationSettings(BaseModel):
    """Organization-level settings."""
    default_framework_id: Optional[str] = None
    notification_preferences: Dict[str, bool] = Field(default_factory=dict)
    audit_retention_days: int = Field(default=365, ge=90, le=2555)
    require_mfa: bool = True
    allowed_ip_ranges: List[str] = Field(default_factory=list)
    sso_config: Optional[Dict[str, Any]] = None

class Organization(BaseModel):
    """
    Organization (tenant) entity.
    
    Represents a customer organization in the multi-tenant platform.
    All compliance data is scoped to an organization.
    
    Attributes:
        id: Unique identifier (UUID format)
        name: Display name of the organization
        slug: URL-friendly identifier (unique)
        tier: Subscription tier
        data_residency: Region for data storage
        encryption_key_id: Reference to org-specific encryption key
        settings: Organization-level settings
        created_at: Timestamp of creation
        updated_at: Timestamp of last update
    
    Example:
        >>> org = Organization(
        ...     name="Acme Corp",
        ...     slug="acme-corp",
        ...     tier=TenantTier.GROWTH
        ... )
    """
    id: str = Field(default_factory=lambda: str(uuid4()))
    name: str = Field(..., min_length=1, max_length=255)
    slug: str = Field(..., min_length=1, max_length=63, pattern=r'^[a-z0-9-]+$')
    tier: TenantTier = TenantTier.STARTER
    data_residency: DataResidency = DataResidency.US
    encryption_key_id: Optional[str] = None
    cell_id: Optional[str] = None  # For enterprise tier
    settings: OrganizationSettings = Field(default_factory=OrganizationSettings)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    
    @field_validator('slug')
    @classmethod
    def validate_slug(cls, v: str) -> str:
        """Ensure slug is lowercase and valid."""
        if v != v.lower():
            raise ValueError('Slug must be lowercase')
        if v.startswith('-') or v.endswith('-'):
            raise ValueError('Slug cannot start or end with hyphen')
        return v
    
    @model_validator(mode='after')
    def validate_enterprise_cell(self) -> 'Organization':
        """Enterprise tier should have cell assignment."""
        if self.tier == TenantTier.ENTERPRISE and not self.cell_id:
            # Warning, not error - cell assigned during provisioning
            pass
        return self
    
    def to_fdb_key(self) -> bytes:
        """Generate FoundationDB key for this organization."""
        return f"/orgs/{self.id}".encode()
    
    def to_fdb_value(self) -> bytes:
        """Serialize for FoundationDB storage."""
        return self.model_dump_json().encode()
    
    @classmethod
    def from_fdb_value(cls, data: bytes) -> 'Organization':
        """Deserialize from FoundationDB."""
        return cls.model_validate_json(data)

# Continue with User, Role, Permission models...
```

### 2. Compliance Framework Models

```python
# File: src/domain/models/framework.py

"""
Compliance framework domain models.

Models for representing compliance frameworks (SOC 2, ISO 27001, etc.)
and their control structures.
"""

from datetime import date, datetime
from enum import Enum
from typing import List, Optional, Dict, Any, Set
from pydantic import BaseModel, Field, field_validator, computed_field
from uuid import uuid4

class FrameworkAuthority(str, Enum):
    """Standards body or authority."""
    AICPA = "aicpa"          # SOC 2
    ISO = "iso"              # ISO 27001, ISO 42001
    NIST = "nist"            # NIST CSF, NIST AI RMF
    HHS = "hhs"              # HIPAA
    GDPR = "gdpr"            # GDPR
    PCI_SSC = "pci_ssc"      # PCI DSS
    EU = "eu"                # EU AI Act
    FDA = "fda"              # FDA 21 CFR Part 11

class FrameworkStatus(str, Enum):
    """Framework lifecycle status."""
    DRAFT = "draft"
    ACTIVE = "active"
    DEPRECATED = "deprecated"
    SUPERSEDED = "superseded"

class Framework(BaseModel):
    """
    Compliance framework entity.
    
    Represents a compliance standard or regulation that an organization
    may need to comply with.
    
    Attributes:
        id: Unique identifier
        organization_id: Owning organization (for custom frameworks)
        code: Short code (e.g., "SOC2", "ISO27001")
        name: Full name of the framework
        version: Version string (e.g., "2017", "Type II")
        authority: Standards body
        description: Description of the framework
        effective_date: When the framework version became effective
        sunset_date: When the framework version will be deprecated
        control_count: Number of controls in this framework
        status: Lifecycle status
        is_custom: Whether this is a custom/organization-specific framework
        parent_framework_id: For custom frameworks based on standards
        metadata: Additional framework-specific data
    
    Example:
        >>> soc2 = Framework(
        ...     code="SOC2",
        ...     name="SOC 2 Type II",
        ...     version="2017",
        ...     authority=FrameworkAuthority.AICPA,
        ...     description="Service Organization Control 2"
        ... )
    """
    id: str = Field(default_factory=lambda: str(uuid4()))
    organization_id: str
    code: str = Field(..., min_length=1, max_length=50, pattern=r'^[A-Z0-9_-]+$')
    name: str = Field(..., min_length=1, max_length=255)
    version: str = Field(..., min_length=1, max_length=50)
    authority: FrameworkAuthority
    description: str = Field(default="")
    effective_date: Optional[date] = None
    sunset_date: Optional[date] = None
    control_count: int = Field(default=0, ge=0)
    status: FrameworkStatus = FrameworkStatus.ACTIVE
    is_custom: bool = False
    parent_framework_id: Optional[str] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    
    @computed_field
    @property
    def full_code(self) -> str:
        """Full code including version."""
        return f"{self.code}:{self.version}"
    
    @field_validator('code')
    @classmethod
    def validate_code_uppercase(cls, v: str) -> str:
        """Ensure code is uppercase."""
        return v.upper()
    
    def to_neo4j_properties(self) -> Dict[str, Any]:
        """Convert to Neo4j node properties."""
        return {
            "id": self.id,
            "organization_id": self.organization_id,
            "code": self.code,
            "name": self.name,
            "version": self.version,
            "authority": self.authority.value,
            "status": self.status.value,
            "control_count": self.control_count,
            "is_custom": self.is_custom
        }

class ControlCategory(str, Enum):
    """High-level control categories."""
    ACCESS_CONTROL = "access_control"
    ASSET_MANAGEMENT = "asset_management"
    AUDIT_LOGGING = "audit_logging"
    CHANGE_MANAGEMENT = "change_management"
    CONFIGURATION_MANAGEMENT = "configuration_management"
    CRYPTOGRAPHY = "cryptography"
    DATA_PROTECTION = "data_protection"
    INCIDENT_RESPONSE = "incident_response"
    NETWORK_SECURITY = "network_security"
    PERSONNEL_SECURITY = "personnel_security"
    PHYSICAL_SECURITY = "physical_security"
    RISK_MANAGEMENT = "risk_management"
    SECURITY_GOVERNANCE = "security_governance"
    SYSTEM_DEVELOPMENT = "system_development"
    THIRD_PARTY = "third_party"
    BUSINESS_CONTINUITY = "business_continuity"
    PRIVACY = "privacy"
    AI_GOVERNANCE = "ai_governance"

class ControlImplementationStatus(str, Enum):
    """Implementation status for a control."""
    NOT_STARTED = "not_started"
    IN_PROGRESS = "in_progress"
    IMPLEMENTED = "implemented"
    NOT_APPLICABLE = "not_applicable"
    COMPENSATING_CONTROL = "compensating_control"

class ControlTestResult(str, Enum):
    """Result of testing a control."""
    PASS = "pass"
    FAIL = "fail"
    PARTIAL = "partial"
    NOT_TESTED = "not_tested"
    ERROR = "error"

class Control(BaseModel):
    """
    Compliance control entity.
    
    Represents a specific control requirement within a framework.
    Controls can be hierarchical (parent/child relationships).
    
    Attributes:
        id: Unique identifier
        organization_id: Owning organization
        framework_id: Parent framework
        control_id: Framework-specific control identifier (e.g., "CC6.1")
        title: Short title
        description: Full control description
        category: Control category
        parent_id: Parent control for hierarchical structures
        level: Hierarchy depth (0 = top-level)
        guidance: Implementation guidance
        evidence_requirements: What evidence satisfies this control
        test_procedures: How to test this control
        implementation_status: Current implementation status
        last_test_result: Most recent test result
        last_tested_at: When last tested
        owner_id: User responsible for this control
        metadata: Additional control-specific data
    
    Example:
        >>> control = Control(
        ...     framework_id="frm_123",
        ...     control_id="CC6.1",
        ...     title="Logical and Physical Access Controls",
        ...     category=ControlCategory.ACCESS_CONTROL
        ... )
    """
    id: str = Field(default_factory=lambda: str(uuid4()))
    organization_id: str
    framework_id: str
    control_id: str = Field(..., min_length=1, max_length=50)
    title: str = Field(..., min_length=1, max_length=500)
    description: str = Field(default="")
    category: ControlCategory
    parent_id: Optional[str] = None
    level: int = Field(default=0, ge=0, le=10)
    guidance: str = Field(default="")
    evidence_requirements: List[str] = Field(default_factory=list)
    test_procedures: List[str] = Field(default_factory=list)
    implementation_status: ControlImplementationStatus = ControlImplementationStatus.NOT_STARTED
    last_test_result: ControlTestResult = ControlTestResult.NOT_TESTED
    last_tested_at: Optional[datetime] = None
    owner_id: Optional[str] = None
    severity: str = Field(default="medium", pattern=r'^(critical|high|medium|low)$')
    weight: float = Field(default=1.0, ge=0.0, le=10.0)
    metadata: Dict[str, Any] = Field(default_factory=dict)
    tags: List[str] = Field(default_factory=list)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    
    @computed_field
    @property
    def full_control_id(self) -> str:
        """Full control identifier including framework."""
        return f"{self.framework_id}:{self.control_id}"
    
    def to_neo4j_properties(self) -> Dict[str, Any]:
        """Convert to Neo4j node properties."""
        return {
            "id": self.id,
            "organization_id": self.organization_id,
            "framework_id": self.framework_id,
            "control_id": self.control_id,
            "title": self.title,
            "category": self.category.value,
            "level": self.level,
            "implementation_status": self.implementation_status.value,
            "last_test_result": self.last_test_result.value,
            "severity": self.severity,
            "weight": self.weight
        }

# Continue with ControlMapping model...
```

### 3. Evidence Models

```python
# File: src/domain/models/evidence.py

"""
Evidence domain models.

Models for representing compliance evidence collected from integrations.
"""

# Include EvidenceType, EvidenceClassification, EvidenceStatus enums
# Include RawEvidenceItem, ProcessedEvidenceItem, EvidenceEvent models
# Include EvidenceGap, GapType models
```

### 4. Integration Models

```python
# File: src/domain/models/integration.py

"""
Integration domain models.

Models for representing external system integrations.
"""

# Include IntegrationType, AuthType, IntegrationStatus enums
# Include Integration, IntegrationCredentials, IntegrationHealth models
# Include ConnectionTestResult model
```

### 5. Agent Task Models

```python
# File: src/domain/models/agent.py

"""
Agent task domain models.

Models for representing AI agent tasks and results.
"""

# Include AgentType, TaskStatus, TaskPriority enums
# Include AgentTask, AgentResult, AgentCheckpoint models
# Include ApprovalRequest, ApprovalDecision models
```

### 6. Risk Models

```python
# File: src/domain/models/risk.py

"""
Risk management domain models.

Models for representing risks, issues, and remediation.
"""

# Include RiskSeverity, RiskStatus, RiskCategory enums
# Include Risk, Issue, RemediationTask models
```

### 7. Audit Models

```python
# File: src/domain/models/audit.py

"""
Audit domain models.

Models for representing audit activities and packages.
"""

# Include AuditType, AuditStatus enums
# Include Audit, AuditPackage, AuditorAccess models
```

### 8. Check Definition Models

```python
# File: src/domain/models/check.py

"""
Compliance check domain models.

Models for representing automated compliance checks.
"""

from datetime import datetime
from enum import Enum
from typing import List, Optional, Dict, Any, Union
from pydantic import BaseModel, Field, field_validator
from uuid import uuid4

class CheckSeverity(str, Enum):
    """Severity of check failure."""
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"
    INFO = "info"

class CheckCategory(str, Enum):
    """Category of automated check."""
    IDENTITY = "identity"
    DATA_PROTECTION = "data_protection"
    NETWORK = "network"
    LOGGING = "logging"
    CONFIGURATION = "configuration"

class CheckResultStatus(str, Enum):
    """Result status of a check execution."""
    PASS = "pass"
    FAIL = "fail"
    ERROR = "error"
    SKIPPED = "skipped"

class CheckQueryType(str, Enum):
    """Type of query used in check logic."""
    CSV = "csv"      # Query CSV-formatted data
    JSON = "json"    # Query JSON data with JSONPath
    REGEX = "regex"  # Pattern matching
    CUSTOM = "custom" # Custom evaluation function

class CheckQuery(BaseModel):
    """Query definition for check logic."""
    query_type: CheckQueryType
    query: str  # CSV column query, JSONPath, or regex pattern
    
class PassCondition(BaseModel):
    """Condition that must be met for check to pass."""
    field: str
    operator: str  # "equals", "contains", "greater_than", "is_empty", "matches"
    value: Any
    
    def evaluate(self, actual_value: Any) -> bool:
        """Evaluate condition against actual value."""
        if self.operator == "equals":
            return actual_value == self.value
        elif self.operator == "not_equals":
            return actual_value != self.value
        elif self.operator == "contains":
            return self.value in str(actual_value)
        elif self.operator == "greater_than":
            return actual_value > self.value
        elif self.operator == "less_than":
            return actual_value < self.value
        elif self.operator == "is_empty":
            return not actual_value
        elif self.operator == "is_not_empty":
            return bool(actual_value)
        elif self.operator == "matches":
            import re
            return bool(re.match(self.value, str(actual_value)))
        elif self.operator == "all_true":
            return all(actual_value) if isinstance(actual_value, list) else bool(actual_value)
        else:
            raise ValueError(f"Unknown operator: {self.operator}")

class CheckDefinition(BaseModel):
    """
    Automated compliance check definition.
    
    Defines how to collect data and evaluate compliance for a specific
    control requirement.
    
    Attributes:
        id: Unique identifier
        organization_id: Owning organization (for custom checks)
        name: Human-readable name
        description: What this check verifies
        integration_type: Which integration provides data
        severity: Importance of check failure
        category: Type of check
        control_ids: Controls this check satisfies
        check_query: How to extract relevant data
        pass_conditions: Conditions for check to pass
        remediation_guidance: How to fix failures
        is_custom: Organization-specific check
        enabled: Whether check is active
    
    Example:
        >>> check = CheckDefinition(
        ...     name="aws_mfa_enforcement",
        ...     description="Verify all IAM users have MFA enabled",
        ...     integration_type="aws",
        ...     severity=CheckSeverity.HIGH,
        ...     category=CheckCategory.IDENTITY,
        ...     control_ids=["soc2_cc6.1", "iso27001_a.9.4.2"]
        ... )
    """
    id: str = Field(default_factory=lambda: str(uuid4()))
    organization_id: Optional[str] = None  # None for built-in checks
    name: str = Field(..., min_length=1, max_length=100, pattern=r'^[a-z0-9_]+$')
    description: str = Field(...)
    integration_type: str
    severity: CheckSeverity
    category: CheckCategory
    control_ids: List[str] = Field(default_factory=list)
    check_query: CheckQuery
    pass_conditions: List[PassCondition] = Field(default_factory=list)
    remediation_guidance: str = Field(default="")
    is_custom: bool = False
    enabled: bool = True
    schedule_cron: Optional[str] = None  # Cron expression for scheduled checks
    timeout_seconds: int = Field(default=300, ge=30, le=3600)
    metadata: Dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    
    @field_validator('name')
    @classmethod
    def validate_name_format(cls, v: str) -> str:
        """Ensure name is snake_case."""
        return v.lower()

class CheckResult(BaseModel):
    """
    Result of executing a compliance check.
    
    Attributes:
        id: Unique identifier
        organization_id: Organization that was checked
        check_definition_id: Which check was executed
        status: Pass/fail/error status
        resource_type: Type of resource checked
        resource_id: Specific resource identifier
        resource_name: Human-readable resource name
        details: Detailed findings
        evidence_id: Link to evidence item
        executed_at: When check was run
        duration_ms: Execution time
    """
    id: str = Field(default_factory=lambda: str(uuid4()))
    organization_id: str
    check_definition_id: str
    status: CheckResultStatus
    resource_type: str
    resource_id: str
    resource_name: str
    details: Dict[str, Any] = Field(default_factory=dict)
    evidence_id: Optional[str] = None
    executed_at: datetime = Field(default_factory=datetime.utcnow)
    duration_ms: int = Field(default=0, ge=0)
```

## File Structure

Generate the following files:

```
src/domain/
├── __init__.py
├── models/
│   ├── __init__.py
│   ├── organization.py      # Organization, User, Role, Permission
│   ├── framework.py         # Framework, Control, ControlMapping
│   ├── evidence.py          # Evidence types and lifecycle
│   ├── integration.py       # Integration configuration
│   ├── agent.py             # Agent tasks and results
│   ├── risk.py              # Risk and issue tracking
│   ├── audit.py             # Audit packages and access
│   ├── check.py             # Check definitions and results
│   └── common.py            # Shared types and utilities
├── events/
│   ├── __init__.py
│   └── domain_events.py     # Domain event definitions
└── exceptions/
    ├── __init__.py
    └── domain_exceptions.py # Domain-specific exceptions
```

## Acceptance Criteria

1. **Completeness**: All 15+ core entities implemented
2. **Validation**: Pydantic validators for all business rules
3. **Type Safety**: 100% type hint coverage
4. **Serialization**: FoundationDB and Neo4j conversion methods
5. **Documentation**: Google-style docstrings with examples
6. **Immutability**: Frozen models where appropriate
7. **Testing**: Include example usage in docstrings

## Token Budget

- Target: 25,000-35,000 tokens
- Priority: Framework, Control, Evidence, and Check models

## Dependencies

- Input: TDD domain model specifications
- Input: ADR-001 (Control Graph) for graph properties
- Input: ADR-005 (Multi-Tenancy) for organization model
- Output: Used by all service and repository implementations

## Quality Gates

Before considering complete:
- [ ] All models pass mypy strict type checking
- [ ] All Pydantic validators have test cases
- [ ] JSON serialization round-trips correctly
- [ ] Neo4j property conversion is complete
- [ ] Docstrings include usage examples
