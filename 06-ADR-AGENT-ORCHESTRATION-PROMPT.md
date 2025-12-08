# Prompt 06: Architecture Decision Record - Agent Orchestration Framework

## Context

You are a principal architect designing the autonomous agent orchestration system for CODITECT-COMPLIANCE. This ADR establishes how AI agents are coordinated, tasked, monitored, and controlled within the compliance automation platform.

## Output Specification

Generate a comprehensive Architecture Decision Record (ADR) following the standard ADR format. The document should be 4,000-6,000 words (12,000-18,000 tokens).

## Document Structure

### ADR-002: Agent Orchestration Framework

```markdown
# ADR-002: Agent Orchestration Framework for Compliance Automation

## Status
Proposed | Accepted | Deprecated | Superseded

## Date
[Current Date]

## Decision Makers
- [Role: Chief Architect]
- [Role: AI/ML Platform Lead]
- [Role: Security Architect]

## Context

[Describe the agent orchestration requirements]

### Problem Statement
CODITECT-COMPLIANCE requires autonomous AI agents to:
- Collect evidence from 15+ integrated systems
- Detect compliance gaps and recommend remediation
- Prepare audit documentation packages
- Respond to security questionnaires
- Monitor regulatory changes and update controls
- Assess vendor risk profiles

These agents must operate with:
- **Predictability**: Consistent, auditable behavior
- **Safety**: Human oversight for high-risk actions
- **Efficiency**: Optimal token usage and parallelization
- **Reliability**: Fault tolerance and recovery
- **Compliance**: Actions must be logged for audit trails

### Requirements Analysis

#### Agent Types Required

| Agent | Autonomy Level | Approval Required | Tools |
|-------|----------------|-------------------|-------|
| RegulatoryIntelligence | High | No | web_search, document_parse |
| ControlPosture | High | No | graph_query, run_checks |
| EvidenceCollection | Medium | No | integration_api, store_evidence |
| Remediation | Low | Yes (changes) | create_ticket, update_config |
| AuditPreparation | Medium | Yes (external) | package_evidence, generate_report |
| VendorRisk | Medium | No | web_search, questionnaire_send |

#### Coordination Patterns Needed
1. **Sequential**: Framework import → Control mapping → Gap analysis
2. **Parallel**: Multi-integration evidence collection
3. **Hierarchical**: Lead agent delegates to specialized sub-agents
4. **Reactive**: Event-triggered remediation workflows
5. **Scheduled**: Daily posture checks, weekly reports

### Technical Constraints
- Claude API as LLM provider (claude-sonnet-4-20250514)
- Token budget awareness (15x multiplier for multi-agent)
- CODITECT-CORE event system integration
- FoundationDB for state persistence
- Redis Streams for task queue
- Kubernetes for agent execution environment

## Decision Drivers

1. **Token Economics**: Minimize LLM costs while maintaining quality
2. **Auditability**: Every agent action must be traceable
3. **Safety**: Human-in-the-loop for risky operations
4. **Reliability**: Handle failures without data loss
5. **Scalability**: Support 1000s of concurrent agent tasks
6. **Latency**: Time-sensitive operations (e.g., real-time questionnaire)

## Options Considered

### Option 1: Monolithic Agent with Tool Switching

**Description**: Single agent instance handles all compliance tasks, switching between tool sets based on task type.

**Pros**:
- Simple architecture
- Lower token overhead (single system prompt)
- Easier state management

**Cons**:
- Context window pollution from diverse tools
- No parallelization
- Single point of failure
- Harder to tune for specific tasks

**Implementation Example**:
```python
class MonolithicComplianceAgent:
    def __init__(self, claude_client: ClaudeClient):
        self.claude = claude_client
        self.tools = self._register_all_tools()
        
    async def execute(self, task: ComplianceTask) -> TaskResult:
        # One agent handles everything
        messages = [{"role": "user", "content": task.to_prompt()}]
        
        while True:
            response = await self.claude.messages.create(
                model="claude-sonnet-4-20250514",
                messages=messages,
                tools=self.tools,  # All 50+ tools loaded
                max_tokens=4096
            )
            # Process response...
```

### Option 2: Specialized Agent Pool with Router

**Description**: Pool of specialized agents, each optimized for specific tasks. Router dispatches tasks to appropriate agent.

**Pros**:
- Optimized prompts per agent type
- Natural parallelization
- Independent scaling
- Failure isolation

**Cons**:
- Router adds latency
- Cross-agent coordination complexity
- More infrastructure to manage

**Implementation Example**:
```python
class AgentRouter:
    def __init__(self):
        self.agents = {
            AgentType.EVIDENCE: EvidenceCollectionAgent(),
            AgentType.POSTURE: PostureScoringAgent(),
            AgentType.REMEDIATION: RemediationAgent(),
            AgentType.AUDIT: AuditPreparationAgent(),
        }
        
    async def route(self, task: ComplianceTask) -> AgentType:
        """Determine optimal agent for task."""
        # Rule-based routing with ML fallback
        if task.type in self.routing_rules:
            return self.routing_rules[task.type]
        return await self.ml_router.classify(task)
```

### Option 3: Hierarchical Multi-Agent with Orchestrator

**Description**: Lead orchestrator agent coordinates specialized sub-agents. Orchestrator handles planning; sub-agents handle execution.

**Pros**:
- Complex workflow support
- Dynamic task decomposition
- Optimal for multi-step compliance processes
- Clear separation of concerns

**Cons**:
- Higher token cost (orchestrator overhead)
- Increased latency for simple tasks
- Complex error propagation

**Implementation Example**:
```python
class HierarchicalOrchestrator:
    """Lead agent that coordinates specialized sub-agents."""
    
    def __init__(self):
        self.sub_agents = {
            "researcher": RegulatoryIntelligenceAgent(),
            "collector": EvidenceCollectionAgent(),
            "analyzer": PostureAnalysisAgent(),
            "remediator": RemediationAgent(),
        }
        
    async def execute_workflow(
        self,
        workflow: ComplianceWorkflow
    ) -> WorkflowResult:
        # Orchestrator plans the workflow
        plan = await self._create_execution_plan(workflow)
        
        results = {}
        for step in plan.steps:
            if step.parallel_tasks:
                # Fan out to sub-agents
                step_results = await asyncio.gather(*[
                    self._delegate_to_agent(task)
                    for task in step.parallel_tasks
                ])
            else:
                step_results = await self._delegate_to_agent(step.task)
            
            results[step.id] = step_results
            
            # Check if workflow should continue
            if not self._should_continue(results):
                break
                
        return self._synthesize_results(results)
```

### Option 4: Event-Driven Agent Mesh

**Description**: Agents subscribe to event streams and react to relevant events. No central orchestrator; coordination through events.

**Pros**:
- Highly decoupled
- Natural scalability
- Resilient to individual failures
- Real-time responsiveness

**Cons**:
- Harder to reason about workflows
- Eventual consistency challenges
- Complex debugging
- Event storm risk

**Implementation Example**:
```python
class EventDrivenAgent:
    """Agent that reacts to events from Redis Streams."""
    
    def __init__(self, agent_type: AgentType, event_bus: EventBus):
        self.agent_type = agent_type
        self.event_bus = event_bus
        self.subscriptions = self._get_subscriptions()
        
    async def start(self):
        """Subscribe to relevant event streams."""
        for event_type in self.subscriptions:
            await self.event_bus.subscribe(
                event_type,
                self.handle_event
            )
            
    async def handle_event(self, event: DomainEvent):
        """React to event by executing appropriate action."""
        action = self._determine_action(event)
        result = await self._execute_action(action)
        
        # Publish result as new event
        await self.event_bus.publish(
            AgentActionCompletedEvent(
                agent_type=self.agent_type,
                action=action,
                result=result
            )
        )
```

## Decision

**Chosen Option**: Hybrid of Option 2 (Specialized Agent Pool) + Option 3 (Hierarchical for Complex Workflows) + Option 4 (Event-Driven Triggers)

### Rationale

1. **Specialized agents** provide optimal performance for distinct compliance tasks
2. **Hierarchical orchestration** handles complex multi-step workflows (audits, framework adoptions)
3. **Event-driven triggers** enable real-time response to compliance events
4. **Router** provides flexibility to evolve routing logic over time

## Detailed Design

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Task Ingestion Layer                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  API        │  │  Scheduler  │  │  Event      │          │
│  │  Requests   │  │  (Cron)     │  │  Stream     │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
└─────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                      Task Queue (Redis Streams)              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ compliance.tasks.{priority}.{org_id}                 │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Agent Router                            │
│  ┌────────────────┐  ┌────────────────┐                     │
│  │ Routing Rules  │  │ Load Balancer  │                     │
│  └────────────────┘  └────────────────┘                     │
└────────────┬────────────────┬────────────────┬──────────────┘
             │                │                │
             ▼                ▼                ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│  Evidence      │ │  Posture       │ │  Workflow      │
│  Agents (N)    │ │  Agents (N)    │ │  Orchestrators │
└────────────────┘ └────────────────┘ └────────────────┘
             │                │                │
             ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                    Tool Execution Layer                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │Integration│ │ Graph    │ │ Check    │ │ Ticket   │       │
│  │ API      │ │ Query    │ │ Runner   │ │ System   │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Agent Base Class

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Dict, List, Any, Optional, Callable
from enum import Enum
import asyncio
import time

class ApprovalLevel(Enum):
    NONE = "none"                    # Fully autonomous
    NOTIFY = "notify"                # Execute, notify human
    REVIEW = "review"                # Execute, human reviews
    APPROVE = "approve"              # Human approves before execution

@dataclass
class AgentConfig:
    """Configuration for agent behavior."""
    agent_type: str
    model: str = "claude-sonnet-4-20250514"
    max_tokens: int = 4096
    max_tool_calls: int = 20
    timeout_seconds: int = 300
    approval_level: ApprovalLevel = ApprovalLevel.NONE
    token_budget: int = 50_000
    checkpoint_interval: int = 5  # Tool calls between checkpoints

@dataclass
class AgentCheckpoint:
    """Checkpoint for agent state recovery."""
    task_id: str
    agent_type: str
    messages: List[Dict]
    tool_results: Dict[str, Any]
    token_usage: int
    timestamp: float
    step: int

@dataclass
class ToolDefinition:
    """Tool available to agent."""
    name: str
    description: str
    input_schema: Dict
    handler: Callable
    requires_approval: bool = False
    timeout: int = 30

class CircuitBreaker:
    """Prevent cascading failures."""
    
    def __init__(
        self,
        failure_threshold: int = 3,
        recovery_timeout: float = 60.0
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.last_failure: Optional[float] = None
        self.state = "closed"
        
    def record_failure(self):
        self.failures += 1
        self.last_failure = time.time()
        if self.failures >= self.failure_threshold:
            self.state = "open"
            
    def record_success(self):
        self.failures = 0
        self.state = "closed"
        
    def can_execute(self) -> bool:
        if self.state == "closed":
            return True
        if self.state == "open":
            if time.time() - self.last_failure > self.recovery_timeout:
                self.state = "half_open"
                return True
            return False
        return True  # half_open allows one attempt


class ComplianceAgent(ABC):
    """Base class for all compliance agents."""
    
    def __init__(
        self,
        config: AgentConfig,
        claude_client: ClaudeClient,
        checkpoint_store: CheckpointStore,
        approval_service: ApprovalService,
        metrics: MetricsCollector
    ):
        self.config = config
        self.claude = claude_client
        self.checkpoints = checkpoint_store
        self.approvals = approval_service
        self.metrics = metrics
        self.circuit_breaker = CircuitBreaker()
        self.tools = self._register_tools()
        
    @abstractmethod
    def _register_tools(self) -> List[ToolDefinition]:
        """Register tools available to this agent type."""
        pass
        
    @abstractmethod
    def _build_system_prompt(self, task: AgentTask) -> str:
        """Build agent-specific system prompt."""
        pass
        
    async def execute(self, task: AgentTask) -> AgentResult:
        """
        Main agent execution loop with safety controls.
        """
        # Check circuit breaker
        if not self.circuit_breaker.can_execute():
            raise CircuitBreakerOpenError(
                f"Agent {self.config.agent_type} circuit breaker open"
            )
            
        # Check for existing checkpoint
        checkpoint = await self.checkpoints.get_latest(task.id)
        if checkpoint:
            messages = checkpoint.messages
            token_usage = checkpoint.token_usage
            step = checkpoint.step
        else:
            messages = [
                {"role": "user", "content": task.to_prompt()}
            ]
            token_usage = 0
            step = 0
            
        start_time = time.time()
        
        try:
            while step < self.config.max_tool_calls:
                # Token budget check
                if token_usage >= self.config.token_budget:
                    return AgentResult(
                        task_id=task.id,
                        status=ResultStatus.TOKEN_LIMIT,
                        output=self._extract_partial_result(messages),
                        token_usage=token_usage
                    )
                    
                # Timeout check
                if time.time() - start_time > self.config.timeout_seconds:
                    return AgentResult(
                        task_id=task.id,
                        status=ResultStatus.TIMEOUT,
                        output=self._extract_partial_result(messages),
                        token_usage=token_usage
                    )
                    
                # Call LLM
                response = await self.claude.messages.create(
                    model=self.config.model,
                    system=self._build_system_prompt(task),
                    messages=messages,
                    tools=[t.to_api_format() for t in self.tools],
                    max_tokens=self.config.max_tokens
                )
                
                token_usage += response.usage.input_tokens + response.usage.output_tokens
                
                # Check for completion
                if response.stop_reason == "end_turn":
                    self.circuit_breaker.record_success()
                    return AgentResult(
                        task_id=task.id,
                        status=ResultStatus.SUCCESS,
                        output=self._extract_output(response),
                        token_usage=token_usage
                    )
                    
                # Process tool calls
                if response.stop_reason == "tool_use":
                    tool_results = await self._execute_tools(
                        response.content,
                        task
                    )
                    
                    messages.append({
                        "role": "assistant",
                        "content": response.content
                    })
                    messages.append({
                        "role": "user",
                        "content": tool_results
                    })
                    
                step += 1
                
                # Periodic checkpoint
                if step % self.config.checkpoint_interval == 0:
                    await self._save_checkpoint(
                        task, messages, token_usage, step
                    )
                    
            # Max iterations reached
            return AgentResult(
                task_id=task.id,
                status=ResultStatus.MAX_ITERATIONS,
                output=self._extract_partial_result(messages),
                token_usage=token_usage
            )
            
        except Exception as e:
            self.circuit_breaker.record_failure()
            await self._save_checkpoint(task, messages, token_usage, step)
            raise AgentExecutionError(str(e)) from e
            
    async def _execute_tools(
        self,
        content: List[ContentBlock],
        task: AgentTask
    ) -> List[ToolResultBlock]:
        """Execute tool calls with approval checks."""
        results = []
        
        for block in content:
            if block.type != "tool_use":
                continue
                
            tool = self._get_tool(block.name)
            if not tool:
                results.append(ToolResultBlock(
                    tool_use_id=block.id,
                    content=f"Unknown tool: {block.name}"
                ))
                continue
                
            # Check approval requirement
            if tool.requires_approval:
                approval = await self.approvals.request_approval(
                    task_id=task.id,
                    agent_type=self.config.agent_type,
                    tool_name=tool.name,
                    tool_input=block.input,
                    timeout=300  # 5 min approval timeout
                )
                
                if not approval.granted:
                    results.append(ToolResultBlock(
                        tool_use_id=block.id,
                        content=f"Tool execution denied: {approval.reason}"
                    ))
                    continue
                    
            # Execute tool
            try:
                result = await asyncio.wait_for(
                    tool.handler(block.input, task.context),
                    timeout=tool.timeout
                )
                results.append(ToolResultBlock(
                    tool_use_id=block.id,
                    content=result
                ))
            except asyncio.TimeoutError:
                results.append(ToolResultBlock(
                    tool_use_id=block.id,
                    content=f"Tool timeout after {tool.timeout}s",
                    is_error=True
                ))
            except Exception as e:
                results.append(ToolResultBlock(
                    tool_use_id=block.id,
                    content=f"Tool error: {str(e)}",
                    is_error=True
                ))
                
        return results
```

### Specialized Agent: Evidence Collection

```python
class EvidenceCollectionAgent(ComplianceAgent):
    """Agent specialized for collecting compliance evidence."""
    
    def _register_tools(self) -> List[ToolDefinition]:
        return [
            ToolDefinition(
                name="query_integration",
                description="Query an integrated system for data",
                input_schema={
                    "type": "object",
                    "properties": {
                        "integration_id": {"type": "string"},
                        "query_type": {"type": "string"},
                        "parameters": {"type": "object"}
                    },
                    "required": ["integration_id", "query_type"]
                },
                handler=self._handle_query_integration
            ),
            ToolDefinition(
                name="store_evidence",
                description="Store collected evidence item",
                input_schema={
                    "type": "object",
                    "properties": {
                        "evidence_type": {"type": "string"},
                        "content": {"type": "string"},
                        "control_ids": {"type": "array", "items": {"type": "string"}},
                        "metadata": {"type": "object"}
                    },
                    "required": ["evidence_type", "content", "control_ids"]
                },
                handler=self._handle_store_evidence
            ),
            ToolDefinition(
                name="check_coverage",
                description="Check which controls lack recent evidence",
                input_schema={
                    "type": "object",
                    "properties": {
                        "framework_id": {"type": "string"},
                        "lookback_days": {"type": "integer", "default": 90}
                    },
                    "required": ["framework_id"]
                },
                handler=self._handle_check_coverage
            ),
            ToolDefinition(
                name="create_gap_task",
                description="Create task for missing evidence",
                input_schema={
                    "type": "object",
                    "properties": {
                        "control_id": {"type": "string"},
                        "gap_type": {"type": "string"},
                        "priority": {"type": "string"}
                    },
                    "required": ["control_id", "gap_type"]
                },
                handler=self._handle_create_gap_task
            )
        ]
        
    def _build_system_prompt(self, task: AgentTask) -> str:
        return f"""You are an Evidence Collection Agent for CODITECT-COMPLIANCE.

Your mission: Collect compliance evidence from integrated systems to satisfy control requirements.

Current Task Context:
- Organization: {task.context.organization_id}
- Target Framework: {task.context.get('framework_id', 'all')}
- Integration Scope: {task.context.get('integrations', 'all enabled')}

Evidence Collection Protocol:
1. Use check_coverage to identify controls needing evidence
2. For each gap, use query_integration to fetch relevant data
3. Validate collected data against control requirements
4. Use store_evidence to persist validated evidence
5. Create gap_task for any controls you cannot satisfy

Quality Standards:
- Evidence must be dated within 90 days
- Include metadata: source, timestamp, collector
- Link evidence to specific control IDs
- Note any partial coverage

Token Efficiency:
- Batch related queries when possible
- Stop after collecting evidence for {task.context.get('max_controls', 50)} controls
- Report progress every 10 controls"""
```

### Workflow Orchestrator

```python
class ComplianceWorkflowOrchestrator:
    """Orchestrates multi-step compliance workflows."""
    
    def __init__(
        self,
        agent_pool: AgentPool,
        task_queue: TaskQueue,
        event_bus: EventBus
    ):
        self.agents = agent_pool
        self.queue = task_queue
        self.events = event_bus
        
    async def execute_audit_preparation(
        self,
        audit_request: AuditRequest
    ) -> AuditPackage:
        """
        Orchestrate full audit preparation workflow.
        
        Steps:
        1. Gap analysis (PostureAgent)
        2. Evidence collection (EvidenceAgent, parallelized)
        3. Remediation planning (RemediationAgent)
        4. Package generation (AuditAgent)
        """
        workflow_id = str(uuid.uuid4())
        
        # Step 1: Gap Analysis
        gap_result = await self._execute_step(
            workflow_id=workflow_id,
            step="gap_analysis",
            agent_type=AgentType.POSTURE,
            task=AgentTask(
                type="gap_analysis",
                context={
                    "framework_id": audit_request.framework_id,
                    "organization_id": audit_request.organization_id,
                    "audit_date": audit_request.target_date
                }
            )
        )
        
        # Step 2: Parallel Evidence Collection
        evidence_tasks = [
            AgentTask(
                type="evidence_collection",
                context={
                    "control_ids": batch,
                    "organization_id": audit_request.organization_id
                }
            )
            for batch in self._batch_controls(gap_result.gaps, batch_size=20)
        ]
        
        evidence_results = await asyncio.gather(*[
            self._execute_step(
                workflow_id=workflow_id,
                step=f"evidence_collection_{i}",
                agent_type=AgentType.EVIDENCE,
                task=task
            )
            for i, task in enumerate(evidence_tasks)
        ])
        
        # Step 3: Remediation Planning (if gaps remain)
        remaining_gaps = self._identify_remaining_gaps(
            gap_result.gaps,
            evidence_results
        )
        
        if remaining_gaps:
            remediation_result = await self._execute_step(
                workflow_id=workflow_id,
                step="remediation_planning",
                agent_type=AgentType.REMEDIATION,
                task=AgentTask(
                    type="remediation_plan",
                    context={
                        "gaps": remaining_gaps,
                        "organization_id": audit_request.organization_id,
                        "timeline": audit_request.target_date
                    }
                ),
                requires_approval=True  # Human reviews remediation plan
            )
        
        # Step 4: Package Generation
        package_result = await self._execute_step(
            workflow_id=workflow_id,
            step="package_generation",
            agent_type=AgentType.AUDIT,
            task=AgentTask(
                type="audit_package",
                context={
                    "framework_id": audit_request.framework_id,
                    "evidence_results": evidence_results,
                    "remediation_plan": remediation_result if remaining_gaps else None,
                    "organization_id": audit_request.organization_id
                }
            ),
            requires_approval=True  # Human reviews before auditor access
        )
        
        return AuditPackage(
            workflow_id=workflow_id,
            framework_id=audit_request.framework_id,
            evidence_items=self._aggregate_evidence(evidence_results),
            gap_analysis=gap_result,
            remediation_plan=remediation_result if remaining_gaps else None,
            generated_at=datetime.utcnow()
        )
```

### Task Queue Design

```python
class RedisTaskQueue:
    """Task queue implementation using Redis Streams."""
    
    STREAM_KEY = "compliance:tasks:{priority}:{org_id}"
    CONSUMER_GROUP = "agent-workers"
    
    def __init__(self, redis: Redis):
        self.redis = redis
        
    async def enqueue(
        self,
        task: AgentTask,
        priority: TaskPriority = TaskPriority.NORMAL
    ) -> str:
        """Add task to queue."""
        stream_key = self.STREAM_KEY.format(
            priority=priority.value,
            org_id=task.context.organization_id
        )
        
        task_id = await self.redis.xadd(
            stream_key,
            {
                "task_id": task.id,
                "task_type": task.type,
                "payload": json.dumps(task.to_dict()),
                "created_at": datetime.utcnow().isoformat()
            }
        )
        
        return task_id
        
    async def dequeue(
        self,
        agent_type: AgentType,
        count: int = 1,
        block_ms: int = 5000
    ) -> List[AgentTask]:
        """Claim tasks from queue."""
        # Read from multiple priority streams
        streams = {
            self.STREAM_KEY.format(priority="high", org_id="*"): ">",
            self.STREAM_KEY.format(priority="normal", org_id="*"): ">",
        }
        
        messages = await self.redis.xreadgroup(
            groupname=self.CONSUMER_GROUP,
            consumername=f"{agent_type.value}-{uuid.uuid4().hex[:8]}",
            streams=streams,
            count=count,
            block=block_ms
        )
        
        return [
            AgentTask.from_dict(json.loads(msg["payload"]))
            for stream_msgs in messages
            for msg in stream_msgs[1]
        ]
        
    async def acknowledge(self, task_id: str, stream_key: str):
        """Mark task as completed."""
        await self.redis.xack(stream_key, self.CONSUMER_GROUP, task_id)
        
    async def dead_letter(self, task: AgentTask, error: str):
        """Move failed task to dead letter queue."""
        await self.redis.xadd(
            "compliance:tasks:dead_letter",
            {
                "task_id": task.id,
                "error": error,
                "failed_at": datetime.utcnow().isoformat(),
                "payload": json.dumps(task.to_dict())
            }
        )
```

### Human-in-the-Loop Approval

```python
class ApprovalService:
    """Manage human approvals for sensitive agent actions."""
    
    def __init__(
        self,
        notification_service: NotificationService,
        approval_store: ApprovalStore,
        timeout_handler: TimeoutHandler
    ):
        self.notifications = notification_service
        self.store = approval_store
        self.timeout = timeout_handler
        
    async def request_approval(
        self,
        task_id: str,
        agent_type: str,
        tool_name: str,
        tool_input: Dict,
        timeout: int = 300
    ) -> ApprovalDecision:
        """
        Request human approval for agent action.
        """
        approval_id = str(uuid.uuid4())
        
        # Create approval request
        request = ApprovalRequest(
            id=approval_id,
            task_id=task_id,
            agent_type=agent_type,
            tool_name=tool_name,
            tool_input=tool_input,
            requested_at=datetime.utcnow(),
            expires_at=datetime.utcnow() + timedelta(seconds=timeout),
            status=ApprovalStatus.PENDING
        )
        
        await self.store.save(request)
        
        # Notify approvers
        await self.notifications.send_approval_request(
            request,
            channel=NotificationChannel.SLACK,
            urgency="high" if "remediation" in tool_name else "normal"
        )
        
        # Wait for decision
        try:
            decision = await asyncio.wait_for(
                self._wait_for_decision(approval_id),
                timeout=timeout
            )
            return decision
        except asyncio.TimeoutError:
            # Check auto-approval policy
            if self._can_auto_approve(request):
                return ApprovalDecision(
                    granted=True,
                    reason="Auto-approved: low-risk action",
                    approver="system"
                )
            return ApprovalDecision(
                granted=False,
                reason="Approval timeout",
                approver="system"
            )
```

## Consequences

### Positive
1. **Optimal Token Usage**: Specialized agents have focused system prompts
2. **Scalability**: Pool-based architecture scales horizontally
3. **Reliability**: Circuit breakers and checkpoints prevent cascading failures
4. **Auditability**: Every action logged with approval chain
5. **Safety**: Human-in-the-loop for high-risk actions
6. **Flexibility**: Easy to add new agent types

### Negative
1. **Complexity**: Multiple components to monitor and maintain
2. **Latency**: Routing adds ~50ms per task dispatch
3. **Cost**: Redis Streams infrastructure cost
4. **Debugging**: Distributed tracing required

### Mitigations
- Implement comprehensive observability (OpenTelemetry)
- Create agent debugging dashboard
- Automated health checks and alerting

## Implementation Plan

### Phase 1: Core Framework (Week 1-2)
- Implement ComplianceAgent base class
- Set up Redis Streams task queue
- Create EvidenceCollectionAgent
- Unit tests for agent loop

### Phase 2: Orchestration (Week 3-4)
- Implement WorkflowOrchestrator
- Add PostureAgent and RemediationAgent
- Integration tests for multi-agent workflows

### Phase 3: Safety Controls (Week 5)
- Implement ApprovalService
- Add circuit breakers
- Checkpoint and recovery testing

### Phase 4: Production Hardening (Week 6)
- Load testing with 1000 concurrent tasks
- Chaos engineering (agent failures)
- Documentation and runbooks

## Validation Criteria

1. **Performance**: Agent response time < 30s for 90% of tasks
2. **Reliability**: 99.9% task completion rate (excluding human-blocked)
3. **Token Efficiency**: < 20,000 tokens average per workflow
4. **Recovery**: 100% recovery from checkpoint after failure
5. **Approval SLA**: 95% of approvals processed within timeout

## References

- [Anthropic Claude API Documentation](https://docs.anthropic.com)
- [Redis Streams Patterns](https://redis.io/docs/data-types/streams/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
```

## Acceptance Criteria

1. **Architecture Coverage**: All agent types documented with tools
2. **Safety Mechanisms**: Circuit breaker, checkpoint, approval patterns
3. **Code Quality**: Production-ready Python with type hints
4. **Scalability Design**: Queue-based architecture for horizontal scaling
5. **Observability**: Metrics and tracing hooks defined

## Token Budget

- Target: 15,000-22,000 tokens
- Priority: Detailed design and implementation sections

## Dependencies

- Input: PRD agent framework requirements (FR-AG-*)
- Input: SDD agent orchestrator container
- Output: Feeds into component build prompts for Agent Framework

## Integration Points

This ADR establishes patterns used by:
- Evidence Collection ADR (agent-driven collection)
- All component build prompts (agent integration patterns)
- Monitoring and alerting configuration
