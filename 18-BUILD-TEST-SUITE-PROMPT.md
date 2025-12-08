# Prompt 18: Component Build - Test Suite

## Context

You are a senior QA engineer implementing the comprehensive test suite for CODITECT-COMPLIANCE. This includes unit tests, integration tests, end-to-end tests, and performance tests.

## Output Specification

Generate complete test suite code covering all components. Output should be 2,000-3,000 lines of test code.

## Implementation Requirements

### Technology Stack
- pytest for Python tests
- pytest-asyncio for async tests
- Vitest for TypeScript tests
- Playwright for E2E tests
- Locust for load testing

## Test Specifications

### 1. Unit Tests - Domain Models

```python
# File: tests/unit/test_domain_models.py

import pytest
from datetime import datetime
from domain.models.framework import Framework, Control, ControlCategory
from domain.models.evidence import EvidenceItem, EvidenceType

class TestFramework:
    def test_create_framework(self):
        framework = Framework(
            code="SOC2",
            name="SOC 2 Type II",
            version="2017",
            authority="aicpa",
            organization_id="org_123"
        )
        assert framework.code == "SOC2"
        assert framework.full_code == "SOC2:2017"
        
    def test_framework_code_validation(self):
        with pytest.raises(ValueError):
            Framework(code="soc2", ...)  # Must be uppercase
            
    def test_framework_to_neo4j(self):
        framework = Framework(...)
        props = framework.to_neo4j_properties()
        assert "id" in props
        assert "organization_id" in props

class TestControl:
    def test_create_control(self):
        control = Control(
            framework_id="frm_123",
            control_id="CC6.1",
            title="Logical Access Controls",
            category=ControlCategory.ACCESS_CONTROL,
            organization_id="org_123"
        )
        assert control.control_id == "CC6.1"
        
    def test_control_hierarchy(self):
        parent = Control(control_id="CC6", level=0, ...)
        child = Control(control_id="CC6.1", parent_id=parent.id, level=1, ...)
        assert child.parent_id == parent.id
        assert child.level == parent.level + 1

class TestEvidence:
    def test_compute_hash(self):
        evidence = EvidenceItem(
            evidence_type=EvidenceType.CONFIGURATION,
            raw_data={"key": "value"},
            ...
        )
        hash1 = evidence.compute_hash()
        hash2 = evidence.compute_hash()
        assert hash1 == hash2  # Deterministic
        
    def test_expiry_calculation(self):
        evidence = EvidenceItem(
            evidence_type=EvidenceType.LOG,
            ...
        )
        assert evidence.expires_at is not None
        assert evidence.expires_at > datetime.utcnow()
```

### 2. Unit Tests - Services

```python
# File: tests/unit/test_control_graph_service.py

import pytest
from unittest.mock import AsyncMock, MagicMock

class TestControlGraphService:
    @pytest.fixture
    def service(self):
        repo = AsyncMock()
        cache = AsyncMock()
        events = AsyncMock()
        return ControlGraphService(repo, cache, events)
        
    @pytest.mark.asyncio
    async def test_import_framework(self, service):
        service.parser.load_framework = AsyncMock(return_value=mock_definition)
        
        result = await service.import_framework("SOC2", "2017")
        
        assert result.code == "SOC2"
        service.repo.create_framework.assert_called_once()
        service.events.publish.assert_called_once()
        
    @pytest.mark.asyncio
    async def test_gap_analysis_caching(self, service):
        service.cache.get = AsyncMock(return_value=None)
        service.repo.find_controls_without_evidence = AsyncMock(return_value=[])
        
        await service.get_gap_analysis("frm_123")
        
        service.cache.set.assert_called_once()
```

### 3. Integration Tests

```python
# File: tests/integration/test_evidence_pipeline.py

import pytest
from testcontainers.redis import RedisContainer
from testcontainers.neo4j import Neo4jContainer

class TestEvidencePipeline:
    @pytest.fixture(scope="class")
    async def pipeline(self):
        with RedisContainer() as redis, Neo4jContainer() as neo4j:
            # Setup real pipeline with test containers
            yield create_pipeline(redis.get_connection_url(), neo4j.get_connection_url())
            
    @pytest.mark.asyncio
    async def test_full_pipeline_flow(self, pipeline):
        raw_evidence = RawEvidenceItem(
            integration_type="aws",
            raw_data={"resource_type": "iam_user", "user": "test"},
            organization_id="org_test"
        )
        
        result = await pipeline.process(raw_evidence)
        
        assert result.status == EvidenceStatus.VALIDATED
        assert result.storage_uri is not None
        assert len(result.control_ids) > 0

class TestAgentExecution:
    @pytest.mark.asyncio
    async def test_evidence_agent_collection(self, agent_service):
        task = AgentTask(
            type="evidence_collection",
            context={"framework_id": "frm_test", "integration_id": "int_test"}
        )
        
        result = await agent_service.execute(task)
        
        assert result.status == "success"
        assert result.token_usage < 50000
```

### 4. E2E Tests

```typescript
// File: tests/e2e/compliance-dashboard.spec.ts

import { test, expect } from '@playwright/test';

test.describe('Compliance Dashboard', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'password');
    await page.click('button[type="submit"]');
    await page.waitForURL('/dashboard');
  });

  test('displays posture score', async ({ page }) => {
    await expect(page.locator('[data-testid="posture-score"]')).toBeVisible();
    const score = await page.locator('[data-testid="posture-score"]').textContent();
    expect(parseInt(score)).toBeGreaterThanOrEqual(0);
    expect(parseInt(score)).toBeLessThanOrEqual(100);
  });

  test('shows framework list', async ({ page }) => {
    await expect(page.locator('[data-testid="framework-card"]')).toHaveCount.greaterThan(0);
  });

  test('navigates to control graph', async ({ page }) => {
    await page.click('[data-testid="framework-card"]:first-child');
    await expect(page.locator('[data-testid="control-graph"]')).toBeVisible();
  });
});

test.describe('Evidence Collection', () => {
  test('triggers collection manually', async ({ page }) => {
    await page.goto('/integrations');
    await page.click('[data-testid="integration-aws"] button:has-text("Collect")');
    
    await expect(page.locator('[data-testid="collection-status"]')).toContainText('In Progress');
    
    // Wait for completion (with timeout)
    await expect(page.locator('[data-testid="collection-status"]'))
      .toContainText('Completed', { timeout: 60000 });
  });
});
```

### 5. Performance Tests

```python
# File: tests/performance/locustfile.py

from locust import HttpUser, task, between

class ComplianceAPIUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        # Login and get token
        response = self.client.post("/api/v1/auth/login", json={
            "email": "loadtest@example.com",
            "password": "password"
        })
        self.token = response.json()["token"]
        self.headers = {"Authorization": f"Bearer {self.token}"}
        
    @task(10)
    def get_frameworks(self):
        self.client.get("/api/v1/frameworks", headers=self.headers)
        
    @task(5)
    def get_posture_score(self):
        self.client.get("/api/v1/frameworks/frm_123/posture", headers=self.headers)
        
    @task(3)
    def get_gap_analysis(self):
        self.client.get("/api/v1/frameworks/frm_123/gap-analysis", headers=self.headers)
        
    @task(1)
    def trigger_collection(self):
        self.client.post("/api/v1/evidence/collect", 
            json={"integration_id": "int_123"},
            headers=self.headers
        )

class AgentLoadTest(HttpUser):
    wait_time = between(5, 15)
    
    @task
    def execute_agent_task(self):
        self.client.post("/api/v1/agents/tasks", 
            json={
                "agent_type": "evidence",
                "context": {"framework_id": "frm_123"}
            },
            headers=self.headers
        )
```

### 6. Test Fixtures

```python
# File: tests/conftest.py

import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def mock_tenant_context():
    ctx = TenantContext(
        organization_id="org_test",
        user_id="user_test",
        roles=["admin"],
        permissions=["*"]
    )
    set_tenant_context(ctx)
    yield ctx

@pytest.fixture
def sample_framework():
    return Framework(
        id="frm_test",
        code="SOC2",
        name="SOC 2 Type II",
        version="2017",
        authority="aicpa",
        organization_id="org_test"
    )

@pytest.fixture
def sample_control(sample_framework):
    return Control(
        id="ctrl_test",
        framework_id=sample_framework.id,
        control_id="CC6.1",
        title="Logical Access Controls",
        category=ControlCategory.ACCESS_CONTROL,
        organization_id="org_test"
    )

@pytest.fixture
def mock_claude_client():
    client = AsyncMock()
    client.messages.create = AsyncMock(return_value=MockResponse(
        stop_reason="end_turn",
        content=[{"type": "text", "text": "Task completed"}],
        usage={"input_tokens": 100, "output_tokens": 50}
    ))
    return client
```

## File Structure

```
tests/
├── conftest.py                 # Shared fixtures
├── unit/
│   ├── test_domain_models.py
│   ├── test_control_graph_service.py
│   ├── test_evidence_service.py
│   └── test_agent_framework.py
├── integration/
│   ├── test_evidence_pipeline.py
│   ├── test_agent_execution.py
│   └── test_integration_connectors.py
├── e2e/
│   ├── compliance-dashboard.spec.ts
│   ├── evidence-collection.spec.ts
│   └── trust-center.spec.ts
├── performance/
│   ├── locustfile.py
│   └── scenarios/
└── fixtures/
    ├── frameworks.json
    ├── controls.json
    └── evidence.json
```

## Acceptance Criteria

1. **Unit Tests**: 80%+ code coverage
2. **Integration Tests**: All service interactions
3. **E2E Tests**: Critical user journeys
4. **Performance**: Load test baselines
5. **CI Integration**: GitHub Actions workflow

## Token Budget

- Target: 18,000-25,000 tokens

## Dependencies

- Input: All component implementations
- Output: CI/CD pipeline integration
