---
title: 'Prompt 15: Component Build - API Layer'
type: reference
version: 1.0.0
created: '2025-12-27'
updated: '2025-12-27'
status: active
tags:
- ai-ml
- authentication
- security
- testing
- api
- architecture
- automation
- backend
summary: You are a senior software engineer implementing the API Layer for CODITECT-COMPLIANCE.
  This component provides REST and GraphQL APIs for all...
---
# Prompt 15: Component Build - API Layer

## Context

You are a senior software engineer implementing the API Layer for CODITECT-COMPLIANCE. This component provides REST and GraphQL APIs for all compliance operations.

## Output Specification

Generate complete, production-ready Python code for the API layer. Output should be 1,500-2,500 lines of code.

## Implementation Requirements

### Technology Stack
- Python 3.12+
- FastAPI for REST
- Strawberry for GraphQL
- Pydantic for validation
- OAuth 2.0 / JWT authentication

## Component Specifications

### 1. FastAPI Application

```python
# File: src/api/main.py

from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(
    title="CODITECT-COMPLIANCE API",
    version="1.0.0",
    description="Compliance automation platform API"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Mount routers
app.include_router(frameworks_router, prefix="/api/v1/frameworks", tags=["frameworks"])
app.include_router(controls_router, prefix="/api/v1/controls", tags=["controls"])
app.include_router(evidence_router, prefix="/api/v1/evidence", tags=["evidence"])
app.include_router(integrations_router, prefix="/api/v1/integrations", tags=["integrations"])
app.include_router(agents_router, prefix="/api/v1/agents", tags=["agents"])
app.include_router(audits_router, prefix="/api/v1/audits", tags=["audits"])
```

### 2. Authentication Middleware

```python
# File: src/api/auth.py

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["RS256"])
        user = await user_repo.get(payload["sub"])
        set_tenant_context(TenantContext(
            organization_id=user.organization_id,
            user_id=user.id,
            roles=user.roles,
            permissions=expand_permissions(user.roles)
        ))
        return user
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_permission(permission: str):
    async def checker(user: User = Depends(get_current_user)):
        if permission not in get_tenant_context().permissions:
            raise HTTPException(status_code=403, detail="Permission denied")
        return user
    return checker
```

### 3. Framework Endpoints

```python
# File: src/api/routes/frameworks.py

router = APIRouter()

@router.get("/", response_model=PaginatedResponse[FrameworkResponse])
async def list_frameworks(
    status: Optional[str] = None,
    page: int = Query(default=1, ge=1),
    per_page: int = Query(default=20, le=100),
    user: User = Depends(get_current_user),
    service: ControlGraphService = Depends()
):
    """List all compliance frameworks."""
    frameworks, total = await service.list_frameworks(
        status=status,
        offset=(page - 1) * per_page,
        limit=per_page
    )
    return PaginatedResponse(
        items=[FrameworkResponse.from_domain(f) for f in frameworks],
        total=total,
        page=page,
        per_page=per_page
    )

@router.post("/import", response_model=FrameworkResponse)
async def import_framework(
    request: ImportFrameworkRequest,
    user: User = Depends(require_permission("frameworks:write")),
    service: ControlGraphService = Depends()
):
    """Import a standard compliance framework."""
    framework = await service.import_framework(request.code, request.version)
    return FrameworkResponse.from_domain(framework)

@router.get("/{framework_id}/posture", response_model=PostureScoreResponse)
async def get_posture_score(
    framework_id: str,
    user: User = Depends(get_current_user),
    service: ControlGraphService = Depends()
):
    """Get compliance posture score for a framework."""
    posture = await service.calculate_posture_score(framework_id)
    return PostureScoreResponse.from_domain(posture)
```

### 4. GraphQL Schema

```python
# File: src/api/graphql/schema.py

import strawberry
from strawberry.fastapi import GraphQLRouter

@strawberry.type
class Framework:
    id: str
    code: str
    name: str
    version: str
    control_count: int
    posture_score: float

@strawberry.type
class Control:
    id: str
    control_id: str
    title: str
    category: str
    implementation_status: str
    evidence_count: int

@strawberry.type
class Query:
    @strawberry.field
    async def frameworks(self, info) -> List[Framework]:
        service = info.context["control_graph_service"]
        return await service.list_frameworks()
    
    @strawberry.field
    async def framework(self, info, id: str) -> Optional[Framework]:
        service = info.context["control_graph_service"]
        return await service.get_framework(id)
    
    @strawberry.field
    async def gap_analysis(self, info, framework_id: str) -> GapAnalysis:
        service = info.context["control_graph_service"]
        return await service.get_gap_analysis(framework_id)

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def import_framework(self, info, code: str, version: str) -> Framework:
        service = info.context["control_graph_service"]
        return await service.import_framework(code, version)
    
    @strawberry.mutation
    async def trigger_evidence_collection(
        self, info, integration_id: str
    ) -> CollectionJob:
        service = info.context["evidence_service"]
        return await service.trigger_collection(integration_id)

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def posture_updates(self, info, framework_id: str) -> AsyncIterator[PostureUpdate]:
        async for update in info.context["posture_stream"].subscribe(framework_id):
            yield update

schema = strawberry.Schema(query=Query, mutation=Mutation, subscription=Subscription)
graphql_router = GraphQLRouter(schema)
```

### 5. Request/Response Models

```python
# File: src/api/models.py

from pydantic import BaseModel, Field
from typing import List, Optional, Generic, TypeVar

T = TypeVar('T')

class PaginatedResponse(BaseModel, Generic[T]):
    items: List[T]
    total: int
    page: int
    per_page: int
    
class FrameworkResponse(BaseModel):
    id: str
    code: str
    name: str
    version: str
    authority: str
    control_count: int
    status: str
    posture_score: Optional[float] = None
    
    @classmethod
    def from_domain(cls, framework: Framework) -> "FrameworkResponse":
        return cls(**framework.model_dump())

class ImportFrameworkRequest(BaseModel):
    code: str = Field(..., pattern="^[A-Z0-9_-]+$")
    version: str
    
class TriggerCollectionRequest(BaseModel):
    integration_id: str
    control_ids: Optional[List[str]] = None
    priority: str = Field(default="normal", pattern="^(high|normal|low)$")
```

## File Structure

```
src/api/
├── __init__.py
├── main.py                 # FastAPI app
├── auth.py                 # Authentication
├── dependencies.py         # Dependency injection
├── middleware.py           # Custom middleware
├── models.py               # Pydantic models
├── routes/
│   ├── __init__.py
│   ├── frameworks.py
│   ├── controls.py
│   ├── evidence.py
│   ├── integrations.py
│   ├── agents.py
│   └── audits.py
├── graphql/
│   ├── __init__.py
│   ├── schema.py
│   ├── types.py
│   └── resolvers.py
└── tests/
```

## Acceptance Criteria

1. **REST API**: Complete CRUD for all resources
2. **GraphQL**: Query, Mutation, Subscription support
3. **Authentication**: JWT with RBAC
4. **Pagination**: Cursor-based pagination
5. **Validation**: Pydantic request validation
6. **Documentation**: OpenAPI auto-generation

## Token Budget

- Target: 15,000-22,000 tokens

## Dependencies

- Input: SDD API design
- Input: All service components
- Output: Consumed by UI, external clients
