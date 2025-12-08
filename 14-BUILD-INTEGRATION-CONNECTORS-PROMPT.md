# Prompt 14: Component Build - Integration Connectors

## Context

You are a senior software engineer implementing Integration Connectors for CODITECT-COMPLIANCE. These connectors interface with external systems (AWS, Okta, GitHub, etc.) to collect compliance evidence.

## Output Specification

Generate complete, production-ready Python code for the Integration Connector framework and Tier 1 connectors. Output should be 2,000-3,000 lines of code.

## Implementation Requirements

### Technology Stack
- Python 3.12+
- aiohttp for async HTTP
- boto3 for AWS
- Provider-specific SDKs
- Rate limiting with token bucket

## Component Specifications

### 1. Connector Base Interface

```python
# File: src/integrations/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Dict, Any, AsyncIterator, Optional
from enum import Enum

class AuthType(Enum):
    OAUTH2 = "oauth2"
    API_KEY = "api_key"
    IAM_ROLE = "iam_role"

@dataclass
class ConnectorMetadata:
    provider_type: str
    display_name: str
    description: str
    version: str
    auth_types: List[AuthType]
    required_scopes: List[str]
    rate_limits: Dict[str, int]

class BaseConnector(ABC):
    """Base interface for all integration connectors."""
    
    @property
    @abstractmethod
    def metadata(self) -> ConnectorMetadata:
        pass
    
    @abstractmethod
    async def test_connection(self, credentials: IntegrationCredentials) -> ConnectionTestResult:
        pass
    
    @abstractmethod
    async def collect_evidence(
        self,
        credentials: IntegrationCredentials,
        config: CollectionConfig
    ) -> AsyncIterator[RawEvidenceItem]:
        pass
```

### 2. AWS Connector

```python
# File: src/integrations/connectors/aws.py

class AWSConnector(BaseConnector):
    """AWS integration connector."""
    
    metadata = ConnectorMetadata(
        provider_type="aws",
        display_name="Amazon Web Services",
        description="Collect IAM, S3, CloudTrail evidence",
        version="1.0.0",
        auth_types=[AuthType.IAM_ROLE],
        required_scopes=["iam:*", "s3:Get*", "cloudtrail:*"],
        rate_limits={"requests_per_second": 10}
    )
    
    async def collect_evidence(self, credentials, config) -> AsyncIterator[RawEvidenceItem]:
        session = await self._assume_role(credentials)
        
        # IAM Users with MFA status
        async for user in self._collect_iam_users(session):
            yield RawEvidenceItem(
                integration_type="aws",
                raw_data={"resource_type": "iam_user", "user": user},
                collected_at=datetime.utcnow()
            )
        
        # S3 Bucket configurations
        async for bucket in self._collect_s3_buckets(session):
            yield RawEvidenceItem(
                integration_type="aws",
                raw_data={"resource_type": "s3_bucket", **bucket},
                collected_at=datetime.utcnow()
            )
            
        # CloudTrail status
        async for trail in self._collect_cloudtrail(session):
            yield RawEvidenceItem(
                integration_type="aws",
                raw_data={"resource_type": "cloudtrail", **trail},
                collected_at=datetime.utcnow()
            )
```

### 3. Okta Connector

```python
# File: src/integrations/connectors/okta.py

class OktaConnector(BaseConnector):
    """Okta integration connector."""
    
    metadata = ConnectorMetadata(
        provider_type="okta",
        display_name="Okta",
        description="Collect identity and access data",
        version="1.0.0",
        auth_types=[AuthType.API_KEY],
        required_scopes=["okta.users.read", "okta.groups.read"],
        rate_limits={"requests_per_second": 50}
    )
    
    async def collect_evidence(self, credentials, config) -> AsyncIterator[RawEvidenceItem]:
        async for user in self._list_users(credentials):
            mfa_factors = await self._get_mfa_factors(credentials, user["id"])
            yield RawEvidenceItem(
                integration_type="okta",
                raw_data={"resource_type": "user", "user": user, "mfa": mfa_factors},
                collected_at=datetime.utcnow()
            )
```

### 4. GitHub Connector

```python
# File: src/integrations/connectors/github.py

class GitHubConnector(BaseConnector):
    """GitHub integration connector."""
    
    async def collect_evidence(self, credentials, config) -> AsyncIterator[RawEvidenceItem]:
        async for repo in self._list_repositories(credentials):
            protection = await self._get_branch_protection(credentials, repo)
            yield RawEvidenceItem(
                integration_type="github",
                raw_data={"resource_type": "repository", "repo": repo, "protection": protection},
                collected_at=datetime.utcnow()
            )
    
    async def handle_webhook(self, payload: Dict, headers: Dict) -> List[RawEvidenceItem]:
        """Handle push and PR webhooks."""
        event_type = headers.get("X-GitHub-Event")
        if event_type == "push":
            return [RawEvidenceItem(
                integration_type="github",
                raw_data={"resource_type": "push", **payload},
                collected_at=datetime.utcnow()
            )]
        return []
```

### 5. Jira Connector

```python
# File: src/integrations/connectors/jira.py

class JiraConnector(BaseConnector):
    """Jira integration connector."""
    
    async def collect_evidence(self, credentials, config) -> AsyncIterator[RawEvidenceItem]:
        # Security-related tickets
        async for issue in self._search_issues(credentials, "type = Bug AND labels = security"):
            yield RawEvidenceItem(
                integration_type="jira",
                raw_data={"resource_type": "issue", "issue": issue},
                collected_at=datetime.utcnow()
            )
    
    async def execute_action(self, action: str, params: Dict, credentials) -> ActionResult:
        """Create tickets for remediation."""
        if action == "create_ticket":
            issue = await self._create_issue(credentials, params)
            return ActionResult(success=True, data={"issue_key": issue["key"]})
```

### 6. Rate Limiter

```python
# File: src/integrations/rate_limiter.py

class TokenBucketRateLimiter:
    """Rate limiter for API calls."""
    
    async def acquire(self, provider: str, weight: int = 1) -> bool:
        key = f"ratelimit:{provider}:{get_tenant_context().organization_id}"
        # Token bucket implementation with Redis
        ...
        
    async def wait_for_token(self, provider: str, timeout: float = 30.0) -> bool:
        while timeout > 0:
            if await self.acquire(provider):
                return True
            await asyncio.sleep(0.1)
            timeout -= 0.1
        return False
```

### 7. Credential Vault

```python
# File: src/integrations/credentials.py

class CredentialVault:
    """Secure credential storage with GCP Secret Manager."""
    
    async def store(self, org_id: str, provider: str, credentials: Dict) -> str:
        key = await self._get_org_encryption_key(org_id)
        encrypted = self._encrypt(credentials, key)
        credential_id = str(uuid4())
        await self.fdb.set(f"credentials/{credential_id}", encrypted)
        return credential_id
        
    async def retrieve(self, credential_id: str) -> Dict:
        encrypted = await self.fdb.get(f"credentials/{credential_id}")
        key = await self._get_org_encryption_key(get_tenant_context().organization_id)
        return self._decrypt(encrypted, key)
```

## File Structure

```
src/integrations/
├── __init__.py
├── base.py                    # Base connector interface
├── registry.py                # Connector registry
├── rate_limiter.py            # Rate limiting
├── credentials.py             # Credential vault
├── webhooks.py                # Webhook router
├── health.py                  # Health monitoring
├── connectors/
│   ├── __init__.py
│   ├── aws.py                 # AWS connector
│   ├── gcp.py                 # GCP connector
│   ├── azure.py               # Azure connector
│   ├── okta.py                # Okta connector
│   ├── github.py              # GitHub connector
│   ├── gitlab.py              # GitLab connector
│   ├── jira.py                # Jira connector
│   └── slack.py               # Slack connector
└── tests/
```

## Acceptance Criteria

1. **Base Interface**: Complete connector contract
2. **Tier 1 Connectors**: AWS, Okta, GitHub, Jira implemented
3. **Rate Limiting**: Token bucket per provider
4. **Credential Security**: Encrypted storage
5. **Webhook Support**: GitHub, Jira webhooks
6. **Health Monitoring**: Connection testing

## Token Budget

- Target: 18,000-25,000 tokens

## Dependencies

- Input: ADR-004 (Integration Framework)
- Output: Used by Evidence Engine, Agent tools
