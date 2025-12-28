---
title: 'Prompt 08: Architecture Decision Record - Integration Framework'
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
summary: You are a principal architect designing the integration framework for CODITECT-COMPLIANCE.
  This ADR establishes how external systems are connected,...
---
# Prompt 08: Architecture Decision Record - Integration Framework

## Context

You are a principal architect designing the integration framework for CODITECT-COMPLIANCE. This ADR establishes how external systems are connected, credentials managed, and data synchronized across 15+ integration types.

## Output Specification

Generate a comprehensive Architecture Decision Record (ADR) following the standard ADR format. The document should be 3,000-4,500 words (9,000-14,000 tokens).

## Document Structure

### ADR-004: Integration Framework Architecture

```markdown
# ADR-004: Integration Framework Architecture

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

CODITECT-COMPLIANCE must integrate with 15+ external systems at launch, expanding to 300+ integrations over time:

**Tier 1 (Launch)**: AWS, GCP, Azure, Okta, GitHub, Jira, Slack
**Tier 2 (Growth)**: GitLab, Linear, Azure AD, Google Workspace, Datadog
**Tier 3 (Enterprise)**: ServiceNow, Workday, SAP, Salesforce, custom APIs

### Integration Challenges

| Challenge | Description | Impact |
|-----------|-------------|--------|
| **Credential Security** | Store OAuth tokens, API keys, IAM roles | Data breach risk |
| **Rate Limiting** | Each provider has different limits | Service disruption |
| **Schema Variance** | No standard across providers | Complex normalization |
| **Authentication Diversity** | OAuth 2.0, API keys, SAML, IAM roles | Implementation complexity |
| **Webhook Reliability** | Providers may retry, deliver out of order | Duplicate/missing events |
| **Versioning** | APIs evolve, breaking changes | Maintenance burden |
| **Multi-tenancy** | Each org has own credentials | Isolation requirements |

### Integration Patterns Required

1. **Pull (Polling)**: Scheduled data fetches (IAM users, configs)
2. **Push (Webhooks)**: Real-time event notifications (code pushes, tickets)
3. **Bidirectional**: Query + write back (create tickets, update configs)
4. **Streaming**: Continuous log forwarding (SIEM, monitoring)

### Technical Constraints

- Credentials encrypted at rest (AES-256)
- Network isolation between tenants
- Support for customer-managed credentials (BYOC)
- Integration health monitoring required
- Must handle provider outages gracefully

## Decision Drivers

1. **Security**: Zero credential exposure in logs/errors
2. **Reliability**: Graceful degradation during outages
3. **Extensibility**: New integrations without core changes
4. **Performance**: Efficient polling without overwhelming providers
5. **Observability**: Monitor integration health and usage
6. **Developer Experience**: Easy to add new connectors

## Options Considered

### Option 1: Monolithic Integration Service

**Description**: Single service handles all integrations with provider-specific logic.

**Pros**:
- Simple deployment
- Shared infrastructure code
- Easy debugging

**Cons**:
- Single point of failure
- Blast radius of bugs affects all integrations
- Difficult to scale hot integrations independently

### Option 2: Plugin-Based Connector Architecture

**Description**: Core integration service with dynamically-loaded connector plugins.

**Pros**:
- Modular development
- Independent connector testing
- Shared core infrastructure

**Cons**:
- Plugin interface design critical
- Dynamic loading complexity
- Still single deployment unit

### Option 3: Microservice per Integration Type

**Description**: Each integration type deployed as separate microservice.

**Pros**:
- Independent scaling
- Failure isolation
- Team ownership clarity

**Cons**:
- Infrastructure overhead (15+ services)
- Cross-cutting concerns duplication
- Complex orchestration

### Option 4: Hybrid - Core Service + Connector Pool

**Description**: Core service handles orchestration; connector pool executes provider-specific logic.

```
┌─────────────────────┐
│ Integration Service │ ← Orchestration, scheduling, credentials
└─────────┬───────────┘
          │
          ▼
┌─────────────────────────────────────────────┐
│           Connector Worker Pool              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │AWS      │ │Okta     │ │GitHub   │  ...  │
│  │Workers  │ │Workers  │ │Workers  │       │
│  └─────────┘ └─────────┘ └─────────┘       │
└─────────────────────────────────────────────┘
```

**Pros**:
- Centralized orchestration
- Scalable worker pool
- Failure isolation per provider
- Shared infrastructure

**Cons**:
- Worker pool management complexity
- Need robust job distribution

## Decision

**Chosen Option**: Option 4 - Hybrid Core Service + Connector Pool

### Rationale

1. **Centralized orchestration** simplifies credential management and scheduling
2. **Worker pool** allows scaling by provider demand
3. **Connector isolation** prevents one provider's issues from affecting others
4. **Shared infrastructure** reduces duplication

## Detailed Design

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Integration Framework                                 │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                          API Layer                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │
│  │ Integration     │  │ Credential      │  │ Webhook         │               │
│  │ Management API  │  │ Management API  │  │ Receiver API    │               │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘               │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Integration Service (Core)                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │
│  │ Scheduler       │  │ Credential      │  │ Health          │               │
│  │ (Polling Jobs)  │  │ Vault           │  │ Monitor         │               │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐               │
│  │ Rate Limiter    │  │ Circuit         │  │ Metrics         │               │
│  │                 │  │ Breaker         │  │ Collector       │               │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘               │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Job Queue (Redis Streams)                             │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │ integrations.jobs.{provider_type}.{priority}                        │      │
│  └────────────────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
          ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
          │ AWS Worker  │  │ Okta Worker │  │ GitHub      │
          │ Pool (N)    │  │ Pool (N)    │  │ Worker Pool │
          └─────────────┘  └─────────────┘  └─────────────┘
                    │               │               │
                    ▼               ▼               ▼
          ┌─────────────────────────────────────────────────┐
          │              External Providers                  │
          │  ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐    │
          │  │ AWS   │  │ Okta  │  │ GitHub│  │ Jira  │    │
          │  └───────┘  └───────┘  └───────┘  └───────┘    │
          └─────────────────────────────────────────────────┘
```

### Connector Interface

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Dict, Any, AsyncIterator, Optional
from enum import Enum
from datetime import datetime

class AuthType(Enum):
    OAUTH2 = "oauth2"
    API_KEY = "api_key"
    IAM_ROLE = "iam_role"
    SAML = "saml"
    BASIC = "basic"
    CUSTOM = "custom"

class IntegrationCapability(Enum):
    EVIDENCE_COLLECTION = "evidence_collection"
    WEBHOOK_RECEIVER = "webhook_receiver"
    BIDIRECTIONAL = "bidirectional"
    STREAMING = "streaming"

@dataclass
class ConnectorMetadata:
    """Metadata about a connector implementation."""
    provider_type: str                    # "aws", "okta", "github"
    display_name: str                     # "Amazon Web Services"
    description: str
    version: str
    auth_types: List[AuthType]
    capabilities: List[IntegrationCapability]
    required_scopes: List[str]            # OAuth scopes or IAM permissions
    rate_limits: Dict[str, int]           # {"requests_per_second": 10}
    webhook_events: List[str]             # Events this connector handles
    documentation_url: str

@dataclass
class IntegrationCredentials:
    """Encrypted credentials for an integration."""
    id: str
    organization_id: str
    provider_type: str
    auth_type: AuthType
    
    # Encrypted credential data (decrypted at runtime)
    encrypted_data: bytes
    encryption_key_id: str
    
    # OAuth-specific
    access_token: Optional[str] = None
    refresh_token: Optional[str] = None
    token_expires_at: Optional[datetime] = None
    
    # Metadata
    created_at: datetime
    last_used_at: Optional[datetime] = None
    last_rotated_at: Optional[datetime] = None

@dataclass
class ConnectionTestResult:
    """Result of testing integration connectivity."""
    success: bool
    provider_type: str
    latency_ms: int
    permissions_verified: List[str]
    permissions_missing: List[str]
    error_message: Optional[str] = None
    error_code: Optional[str] = None

class BaseConnector(ABC):
    """Base class for all integration connectors."""
    
    @property
    @abstractmethod
    def metadata(self) -> ConnectorMetadata:
        """Return connector metadata."""
        pass
    
    @abstractmethod
    async def test_connection(
        self,
        credentials: IntegrationCredentials
    ) -> ConnectionTestResult:
        """
        Test connectivity and permissions.
        
        Should verify:
        1. Credentials are valid
        2. Required permissions are granted
        3. API is reachable
        """
        pass
    
    @abstractmethod
    async def collect_evidence(
        self,
        credentials: IntegrationCredentials,
        config: CollectionConfig
    ) -> AsyncIterator[RawEvidenceItem]:
        """
        Collect evidence from the provider.
        
        Must:
        - Handle pagination internally
        - Respect rate limits
        - Yield items as they're collected
        - Handle partial failures gracefully
        """
        pass
    
    async def handle_webhook(
        self,
        payload: Dict[str, Any],
        headers: Dict[str, str],
        credentials: IntegrationCredentials
    ) -> List[RawEvidenceItem]:
        """
        Process incoming webhook.
        
        Default implementation raises NotImplementedError.
        Override in connectors that support webhooks.
        """
        raise NotImplementedError(
            f"{self.metadata.provider_type} does not support webhooks"
        )
    
    async def execute_action(
        self,
        action: str,
        parameters: Dict[str, Any],
        credentials: IntegrationCredentials
    ) -> ActionResult:
        """
        Execute an action on the provider (e.g., create ticket).
        
        Default implementation raises NotImplementedError.
        Override in connectors that support bidirectional operations.
        """
        raise NotImplementedError(
            f"{self.metadata.provider_type} does not support actions"
        )
```

### Credential Management

```python
from cryptography.fernet import Fernet
from google.cloud import secretmanager
import json

class CredentialVault:
    """
    Secure credential storage and retrieval.
    
    Architecture:
    - Master encryption key stored in GCP Secret Manager
    - Credentials encrypted with per-organization keys
    - Keys rotated automatically every 90 days
    - Audit log for all credential access
    """
    
    def __init__(
        self,
        secret_manager: secretmanager.SecretManagerServiceClient,
        fdb_client: FoundationDBClient,
        audit_logger: AuditLogger
    ):
        self.secrets = secret_manager
        self.fdb = fdb_client
        self.audit = audit_logger
        
    async def store_credentials(
        self,
        organization_id: str,
        provider_type: str,
        credentials: Dict[str, Any],
        auth_type: AuthType
    ) -> IntegrationCredentials:
        """
        Store encrypted credentials.
        
        Steps:
        1. Get or create organization encryption key
        2. Encrypt credential data
        3. Store encrypted blob in FoundationDB
        4. Log credential creation
        """
        # Get organization's encryption key
        key_id, key = await self._get_or_create_org_key(organization_id)
        
        # Encrypt credentials
        fernet = Fernet(key)
        encrypted_data = fernet.encrypt(
            json.dumps(credentials).encode()
        )
        
        # Create credential record
        cred = IntegrationCredentials(
            id=str(uuid.uuid4()),
            organization_id=organization_id,
            provider_type=provider_type,
            auth_type=auth_type,
            encrypted_data=encrypted_data,
            encryption_key_id=key_id,
            created_at=datetime.utcnow()
        )
        
        # Store in FoundationDB
        await self._store_credential_record(cred)
        
        # Audit log
        await self.audit.log(
            event_type="credential_created",
            organization_id=organization_id,
            details={
                "credential_id": cred.id,
                "provider_type": provider_type,
                "auth_type": auth_type.value
            }
        )
        
        return cred
        
    async def retrieve_credentials(
        self,
        credential_id: str,
        requester: str
    ) -> Dict[str, Any]:
        """
        Retrieve and decrypt credentials.
        
        Access is logged for audit trail.
        """
        # Fetch encrypted record
        cred = await self._get_credential_record(credential_id)
        
        # Get decryption key
        key = await self._get_org_key(
            cred.organization_id,
            cred.encryption_key_id
        )
        
        # Decrypt
        fernet = Fernet(key)
        decrypted = fernet.decrypt(cred.encrypted_data)
        credentials = json.loads(decrypted)
        
        # Audit log
        await self.audit.log(
            event_type="credential_accessed",
            organization_id=cred.organization_id,
            details={
                "credential_id": credential_id,
                "provider_type": cred.provider_type,
                "requester": requester
            }
        )
        
        # Update last used
        await self._update_last_used(credential_id)
        
        return credentials
        
    async def rotate_credentials(
        self,
        credential_id: str
    ) -> IntegrationCredentials:
        """
        Rotate credentials (re-encrypt with new key).
        """
        # Implementation for key rotation
        pass
        
    async def _get_or_create_org_key(
        self,
        organization_id: str
    ) -> tuple[str, bytes]:
        """Get or create organization encryption key from Secret Manager."""
        secret_name = f"projects/coditect/secrets/org-{organization_id}-key"
        
        try:
            # Try to get existing key
            response = self.secrets.access_secret_version(
                request={"name": f"{secret_name}/versions/latest"}
            )
            return response.name.split("/")[-1], response.payload.data
        except Exception:
            # Create new key
            key = Fernet.generate_key()
            
            # Store in Secret Manager
            self.secrets.create_secret(
                request={
                    "parent": "projects/coditect",
                    "secret_id": f"org-{organization_id}-key",
                    "secret": {"replication": {"automatic": {}}}
                }
            )
            
            version = self.secrets.add_secret_version(
                request={
                    "parent": secret_name,
                    "payload": {"data": key}
                }
            )
            
            return version.name.split("/")[-1], key
```

### Rate Limiting

```python
from dataclasses import dataclass
from typing import Dict
import asyncio
import time

@dataclass
class RateLimitConfig:
    """Rate limit configuration per provider."""
    requests_per_second: float
    requests_per_minute: int
    requests_per_hour: int
    burst_size: int  # Max concurrent requests

class TokenBucketRateLimiter:
    """
    Token bucket rate limiter with Redis backing.
    
    Supports:
    - Per-organization limits
    - Per-provider limits
    - Burst allowance
    - Distributed coordination via Redis
    """
    
    def __init__(
        self,
        redis: Redis,
        config: Dict[str, RateLimitConfig]
    ):
        self.redis = redis
        self.config = config
        
    async def acquire(
        self,
        organization_id: str,
        provider_type: str,
        weight: int = 1
    ) -> bool:
        """
        Acquire rate limit tokens.
        
        Returns True if request can proceed, False if rate limited.
        """
        config = self.config.get(provider_type, self._default_config())
        key = f"ratelimit:{provider_type}:{organization_id}"
        
        # Lua script for atomic token bucket
        script = """
        local key = KEYS[1]
        local rate = tonumber(ARGV[1])
        local capacity = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])
        
        local data = redis.call('HMGET', key, 'tokens', 'last_update')
        local tokens = tonumber(data[1]) or capacity
        local last_update = tonumber(data[2]) or now
        
        -- Add tokens based on time elapsed
        local elapsed = now - last_update
        tokens = math.min(capacity, tokens + elapsed * rate)
        
        -- Check if we have enough tokens
        if tokens >= requested then
            tokens = tokens - requested
            redis.call('HMSET', key, 'tokens', tokens, 'last_update', now)
            redis.call('EXPIRE', key, 3600)
            return 1
        else
            return 0
        end
        """
        
        result = await self.redis.eval(
            script,
            keys=[key],
            args=[
                config.requests_per_second,
                config.burst_size,
                time.time(),
                weight
            ]
        )
        
        return bool(result)
        
    async def wait_for_token(
        self,
        organization_id: str,
        provider_type: str,
        timeout: float = 30.0
    ) -> bool:
        """Wait until rate limit token is available."""
        start = time.time()
        
        while time.time() - start < timeout:
            if await self.acquire(organization_id, provider_type):
                return True
            await asyncio.sleep(0.1)
            
        return False

class AdaptiveRateLimiter:
    """
    Adaptive rate limiter that adjusts based on provider responses.
    
    Learns optimal rate from:
    - 429 responses (reduce rate)
    - Successful responses (increase rate)
    - Response latency (adjust for congestion)
    """
    
    def __init__(self, base_limiter: TokenBucketRateLimiter):
        self.base = base_limiter
        self.rate_adjustments: Dict[str, float] = {}
        
    async def record_response(
        self,
        provider_type: str,
        status_code: int,
        latency_ms: int
    ):
        """Adjust rate based on response."""
        key = f"adjustment:{provider_type}"
        current = self.rate_adjustments.get(key, 1.0)
        
        if status_code == 429:
            # Rate limited - reduce by 50%
            self.rate_adjustments[key] = current * 0.5
        elif status_code >= 500:
            # Server error - reduce by 25%
            self.rate_adjustments[key] = current * 0.75
        elif status_code == 200 and latency_ms < 500:
            # Success and fast - slowly increase
            self.rate_adjustments[key] = min(1.0, current * 1.1)
```

### Health Monitoring

```python
@dataclass
class IntegrationHealth:
    """Health status for an integration."""
    integration_id: str
    provider_type: str
    organization_id: str
    
    status: str  # "healthy", "degraded", "unhealthy"
    last_check: datetime
    last_success: Optional[datetime]
    last_failure: Optional[datetime]
    
    # Metrics
    success_rate_1h: float
    avg_latency_ms: int
    error_count_1h: int
    
    # Recent errors
    recent_errors: List[Dict[str, Any]]

class IntegrationHealthMonitor:
    """Monitor and report integration health."""
    
    def __init__(
        self,
        metrics: MetricsCollector,
        alerting: AlertingService
    ):
        self.metrics = metrics
        self.alerting = alerting
        
    async def record_request(
        self,
        integration_id: str,
        provider_type: str,
        organization_id: str,
        success: bool,
        latency_ms: int,
        error: Optional[str] = None
    ):
        """Record integration request for health tracking."""
        labels = {
            "integration_id": integration_id,
            "provider_type": provider_type,
            "organization_id": organization_id
        }
        
        # Record metrics
        self.metrics.increment(
            "integration_requests_total",
            labels=labels,
            value=1
        )
        
        self.metrics.observe(
            "integration_latency_ms",
            labels=labels,
            value=latency_ms
        )
        
        if not success:
            self.metrics.increment(
                "integration_errors_total",
                labels={**labels, "error": error or "unknown"}
            )
            
            # Check if alerting threshold reached
            await self._check_alert_threshold(
                integration_id,
                provider_type,
                organization_id
            )
            
    async def get_health(
        self,
        integration_id: str
    ) -> IntegrationHealth:
        """Get current health status for integration."""
        # Query metrics for health calculation
        pass
        
    async def _check_alert_threshold(
        self,
        integration_id: str,
        provider_type: str,
        organization_id: str
    ):
        """Check if error rate exceeds alert threshold."""
        error_rate = await self._get_error_rate_1h(integration_id)
        
        if error_rate > 0.5:  # 50% error rate
            await self.alerting.send_alert(
                severity="critical",
                title=f"Integration {provider_type} critically degraded",
                message=f"Error rate: {error_rate:.1%}",
                organization_id=organization_id,
                integration_id=integration_id
            )
        elif error_rate > 0.1:  # 10% error rate
            await self.alerting.send_alert(
                severity="warning",
                title=f"Integration {provider_type} degraded",
                message=f"Error rate: {error_rate:.1%}",
                organization_id=organization_id,
                integration_id=integration_id
            )
```

### Webhook Handling

```python
from fastapi import Request, HTTPException
from typing import Callable, Dict
import hmac
import hashlib

class WebhookRouter:
    """Route incoming webhooks to appropriate handlers."""
    
    def __init__(self):
        self.handlers: Dict[str, Callable] = {}
        self.signature_validators: Dict[str, Callable] = {}
        
    def register_handler(
        self,
        provider_type: str,
        handler: Callable,
        signature_validator: Optional[Callable] = None
    ):
        """Register webhook handler for provider."""
        self.handlers[provider_type] = handler
        if signature_validator:
            self.signature_validators[provider_type] = signature_validator
            
    async def handle_webhook(
        self,
        provider_type: str,
        request: Request
    ) -> Dict[str, Any]:
        """
        Handle incoming webhook.
        
        Steps:
        1. Validate signature (provider-specific)
        2. Parse payload
        3. Route to handler
        4. Return acknowledgment
        """
        if provider_type not in self.handlers:
            raise HTTPException(404, f"Unknown provider: {provider_type}")
            
        body = await request.body()
        headers = dict(request.headers)
        
        # Validate signature
        if provider_type in self.signature_validators:
            validator = self.signature_validators[provider_type]
            if not await validator(body, headers):
                raise HTTPException(401, "Invalid webhook signature")
                
        # Parse and handle
        payload = json.loads(body)
        handler = self.handlers[provider_type]
        
        return await handler(payload, headers)

# Provider-specific signature validation
async def validate_github_signature(body: bytes, headers: Dict) -> bool:
    """Validate GitHub webhook signature."""
    signature = headers.get("x-hub-signature-256", "")
    if not signature.startswith("sha256="):
        return False
        
    secret = await get_github_webhook_secret()
    expected = hmac.new(
        secret.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(f"sha256={expected}", signature)

async def validate_okta_signature(body: bytes, headers: Dict) -> bool:
    """Validate Okta webhook signature."""
    # Okta uses different verification method
    pass
```

## Consequences

### Positive
1. **Security**: Credentials encrypted with per-org keys in Secret Manager
2. **Reliability**: Circuit breakers and health monitoring prevent cascading failures
3. **Scalability**: Worker pool scales with demand per provider
4. **Extensibility**: New connectors follow standard interface
5. **Observability**: Comprehensive metrics and health tracking

### Negative
1. **Complexity**: Multiple components to coordinate
2. **Latency**: Rate limiting adds potential delays
3. **Cost**: Secret Manager and Redis infrastructure costs

### Mitigations
- Implement comprehensive integration dashboard
- Pre-warm rate limit tokens for scheduled jobs
- Cache credentials in memory with short TTL

## Implementation Plan

### Phase 1: Core Framework (Week 1-2)
- Implement BaseConnector interface
- Set up CredentialVault with Secret Manager
- Create rate limiter infrastructure

### Phase 2: First Connectors (Week 3-4)
- AWS Connector
- Okta Connector
- GitHub Connector

### Phase 3: Webhook Support (Week 5)
- Webhook router implementation
- Signature validation per provider
- Idempotency handling

### Phase 4: Health Monitoring (Week 6)
- Metrics collection
- Health dashboard
- Alerting integration

## Validation Criteria

1. **Security**: Zero credential exposure in logs/errors
2. **Reliability**: 99.9% webhook delivery success
3. **Performance**: < 100ms rate limit check latency
4. **Coverage**: All Tier 1 integrations functional
5. **Monitoring**: Real-time health status for all integrations

## References

- [OAuth 2.0 Security Best Practices](https://oauth.net/2/security-best-practices/)
- [GCP Secret Manager](https://cloud.google.com/secret-manager)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)
```

## Acceptance Criteria

1. **Interface Design**: Complete connector interface with all methods
2. **Credential Security**: Encryption and vault patterns documented
3. **Rate Limiting**: Token bucket implementation with adaptive learning
4. **Webhook Handling**: Signature validation per provider type
5. **Health Monitoring**: Metrics and alerting integration

## Token Budget

- Target: 10,000-16,000 tokens
- Priority: Connector interface and credential management sections

## Dependencies

- Input: PRD integration requirements (FR-IN-*)
- Input: SDD integration service container
- Output: Feeds into component build prompts for Integration Connectors

## Integration Points

This ADR establishes patterns used by:
- Evidence Collection ADR (connector data flow)
- All connector build prompts
- Agent tools for integration queries
