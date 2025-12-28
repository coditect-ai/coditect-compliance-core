---
title: 'Prompt 13: Component Build - Agent Framework'
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
summary: You are a senior software engineer implementing the AI Agent Framework for
  CODITECT-COMPLIANCE. This component provides the infrastructure for...
---
# Prompt 13: Component Build - Agent Framework

## Context

You are a senior software engineer implementing the AI Agent Framework for CODITECT-COMPLIANCE. This component provides the infrastructure for autonomous AI agents that perform compliance tasks.

## Output Specification

Generate complete, production-ready Python code for the Agent Framework. Output should be 2,500-3,500 lines of code.

## Implementation Requirements

### Technology Stack
- Python 3.12+
- Anthropic Claude API (claude-sonnet-4-20250514)
- Redis Streams for task queue
- FoundationDB for checkpoints
- Circuit breaker pattern for reliability

## Component Specifications

### 1. Base Agent Class

```python
# File: src/agents/base.py

"""
Base agent implementation with safety controls.
"""

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import List, Dict, Any, Optional, Callable
from enum import Enum
import asyncio
import time

class ApprovalLevel(Enum):
    NONE = "none"
    NOTIFY = "notify"
    REVIEW = "review"
    APPROVE = "approve"

@dataclass
class AgentConfig:
    agent_type: str
    model: str = "claude-sonnet-4-20250514"
    max_tokens: int = 4096
    max_tool_calls: int = 20
    timeout_seconds: int = 300
    approval_level: ApprovalLevel = ApprovalLevel.NONE
    token_budget: int = 50_000
    checkpoint_interval: int = 5

@dataclass  
class ToolDefinition:
    name: str
    description: str
    input_schema: Dict[str, Any]
    handler: Callable
    requires_approval: bool = False
    timeout: int = 30

class CircuitBreaker:
    """Prevent cascading failures."""
    
    def __init__(self, failure_threshold: int = 3, recovery_timeout: float = 60.0):
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
        return True

class ComplianceAgent(ABC):
    """
    Base class for all compliance agents.
    
    Provides:
    - Conversation loop with Claude API
    - Tool execution with approval checks
    - Checkpointing for recovery
    - Circuit breaker for failure handling
    - Token budget enforcement
    """
    
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
        """Register tools available to this agent."""
        pass
        
    @abstractmethod
    def _build_system_prompt(self, task: AgentTask) -> str:
        """Build agent-specific system prompt."""
        pass
        
    async def execute(self, task: AgentTask) -> AgentResult:
        """Main agent execution loop."""
        
        if not self.circuit_breaker.can_execute():
            raise CircuitBreakerOpenError(f"Agent {self.config.agent_type} unavailable")
            
        # Resume from checkpoint if exists
        checkpoint = await self.checkpoints.get_latest(task.id)
        messages = checkpoint.messages if checkpoint else [{"role": "user", "content": task.to_prompt()}]
        token_usage = checkpoint.token_usage if checkpoint else 0
        step = checkpoint.step if checkpoint else 0
        
        start_time = time.time()
        
        try:
            while step < self.config.max_tool_calls:
                # Check budgets
                if token_usage >= self.config.token_budget:
                    return AgentResult(task_id=task.id, status="token_limit", token_usage=token_usage)
                    
                if time.time() - start_time > self.config.timeout_seconds:
                    return AgentResult(task_id=task.id, status="timeout", token_usage=token_usage)
                
                # Call Claude
                response = await self.claude.messages.create(
                    model=self.config.model,
                    system=self._build_system_prompt(task),
                    messages=messages,
                    tools=[self._tool_to_api(t) for t in self.tools],
                    max_tokens=self.config.max_tokens
                )
                
                token_usage += response.usage.input_tokens + response.usage.output_tokens
                
                if response.stop_reason == "end_turn":
                    self.circuit_breaker.record_success()
                    return AgentResult(
                        task_id=task.id,
                        status="success",
                        output=self._extract_output(response),
                        token_usage=token_usage
                    )
                    
                if response.stop_reason == "tool_use":
                    tool_results = await self._execute_tools(response.content, task)
                    messages.append({"role": "assistant", "content": response.content})
                    messages.append({"role": "user", "content": tool_results})
                    
                step += 1
                
                if step % self.config.checkpoint_interval == 0:
                    await self._save_checkpoint(task, messages, token_usage, step)
                    
            return AgentResult(task_id=task.id, status="max_iterations", token_usage=token_usage)
            
        except Exception as e:
            self.circuit_breaker.record_failure()
            await self._save_checkpoint(task, messages, token_usage, step)
            raise
```

### 2. Specialized Agents

```python
# File: src/agents/evidence_agent.py

class EvidenceCollectionAgent(ComplianceAgent):
    """Agent for collecting compliance evidence."""
    
    def _register_tools(self) -> List[ToolDefinition]:
        return [
            ToolDefinition(
                name="query_integration",
                description="Query an integrated system for data",
                input_schema={...},
                handler=self._query_integration
            ),
            ToolDefinition(
                name="store_evidence",
                description="Store collected evidence",
                input_schema={...},
                handler=self._store_evidence
            ),
            ToolDefinition(
                name="check_coverage",
                description="Check evidence coverage for controls",
                input_schema={...},
                handler=self._check_coverage
            ),
            ToolDefinition(
                name="create_gap_task",
                description="Create task for missing evidence",
                input_schema={...},
                handler=self._create_gap_task
            )
        ]
        
    def _build_system_prompt(self, task: AgentTask) -> str:
        return f"""You are an Evidence Collection Agent for CODITECT-COMPLIANCE.

Your mission: Collect compliance evidence from integrated systems.

Organization: {task.context.organization_id}
Target Framework: {task.context.get('framework_id', 'all')}

Protocol:
1. Check coverage to identify gaps
2. Query integrations for evidence
3. Store validated evidence
4. Create tasks for unfillable gaps

Budget: Maximum {self.config.max_tool_calls} tool calls."""

# File: src/agents/posture_agent.py

class PostureAnalysisAgent(ComplianceAgent):
    """Agent for analyzing compliance posture."""
    
    def _register_tools(self) -> List[ToolDefinition]:
        return [
            ToolDefinition(name="query_control_graph", ...),
            ToolDefinition(name="run_compliance_checks", ...),
            ToolDefinition(name="calculate_posture_score", ...),
            ToolDefinition(name="identify_risks", ...)
        ]

# File: src/agents/remediation_agent.py

class RemediationAgent(ComplianceAgent):
    """Agent for planning and executing remediation."""
    
    def _register_tools(self) -> List[ToolDefinition]:
        return [
            ToolDefinition(name="create_ticket", requires_approval=True, ...),
            ToolDefinition(name="update_configuration", requires_approval=True, ...),
            ToolDefinition(name="generate_remediation_plan", ...),
            ToolDefinition(name="estimate_effort", ...)
        ]
```

### 3. Task Queue

```python
# File: src/agents/queue.py

class AgentTaskQueue:
    """Redis Streams-based task queue for agents."""
    
    STREAM_KEY = "agents:tasks:{agent_type}:{priority}"
    
    async def enqueue(self, task: AgentTask, priority: str = "normal") -> str:
        stream_key = self.STREAM_KEY.format(
            agent_type=task.agent_type,
            priority=priority
        )
        return await self.redis.xadd(stream_key, task.to_dict())
        
    async def dequeue(self, agent_type: str, count: int = 1) -> List[AgentTask]:
        streams = {
            self.STREAM_KEY.format(agent_type=agent_type, priority="high"): ">",
            self.STREAM_KEY.format(agent_type=agent_type, priority="normal"): ">"
        }
        messages = await self.redis.xreadgroup(
            groupname="agent-workers",
            consumername=f"{agent_type}-{uuid4().hex[:8]}",
            streams=streams,
            count=count,
            block=5000
        )
        return [AgentTask.from_dict(m) for m in messages]
```

### 4. Workflow Orchestrator

```python
# File: src/agents/orchestrator.py

class WorkflowOrchestrator:
    """Orchestrate multi-agent compliance workflows."""
    
    async def execute_audit_preparation(self, request: AuditRequest) -> AuditPackage:
        """
        Orchestrate audit preparation workflow.
        
        Steps:
        1. Gap analysis (PostureAgent)
        2. Evidence collection (EvidenceAgent, parallel)
        3. Remediation planning (RemediationAgent)
        4. Package generation (AuditAgent)
        """
        workflow_id = str(uuid4())
        
        # Step 1: Gap Analysis
        gap_result = await self._execute_agent(
            AgentType.POSTURE,
            AgentTask(type="gap_analysis", context={"framework_id": request.framework_id})
        )
        
        # Step 2: Parallel Evidence Collection
        evidence_tasks = [
            AgentTask(type="evidence_collection", context={"control_ids": batch})
            for batch in self._batch_controls(gap_result.gaps, 20)
        ]
        evidence_results = await asyncio.gather(*[
            self._execute_agent(AgentType.EVIDENCE, task)
            for task in evidence_tasks
        ])
        
        # Step 3: Remediation (if gaps remain)
        remaining_gaps = self._identify_remaining_gaps(gap_result.gaps, evidence_results)
        if remaining_gaps:
            await self._execute_agent(
                AgentType.REMEDIATION,
                AgentTask(type="remediation_plan", context={"gaps": remaining_gaps}),
                requires_approval=True
            )
        
        # Step 4: Package Generation
        return await self._execute_agent(
            AgentType.AUDIT,
            AgentTask(type="audit_package", context={"evidence": evidence_results}),
            requires_approval=True
        )
```

### 5. Human-in-the-Loop Approval

```python
# File: src/agents/approval.py

class ApprovalService:
    """Manage human approvals for agent actions."""
    
    async def request_approval(
        self,
        task_id: str,
        agent_type: str,
        tool_name: str,
        tool_input: Dict,
        timeout: int = 300
    ) -> ApprovalDecision:
        """Request and wait for human approval."""
        
        approval_id = str(uuid4())
        request = ApprovalRequest(
            id=approval_id,
            task_id=task_id,
            agent_type=agent_type,
            tool_name=tool_name,
            tool_input=tool_input,
            expires_at=datetime.utcnow() + timedelta(seconds=timeout)
        )
        
        await self.store.save(request)
        await self.notifications.send_approval_request(request)
        
        try:
            return await asyncio.wait_for(
                self._wait_for_decision(approval_id),
                timeout=timeout
            )
        except asyncio.TimeoutError:
            return ApprovalDecision(granted=False, reason="Timeout")
```

## File Structure

```
src/agents/
├── __init__.py
├── base.py                 # Base agent class
├── evidence_agent.py       # Evidence collection agent
├── posture_agent.py        # Posture analysis agent
├── remediation_agent.py    # Remediation agent
├── audit_agent.py          # Audit preparation agent
├── regulatory_agent.py     # Regulatory intelligence agent
├── vendor_agent.py         # Vendor risk agent
├── queue.py                # Task queue
├── orchestrator.py         # Workflow orchestration
├── approval.py             # Human-in-the-loop
├── checkpoints.py          # Checkpoint management
├── metrics.py              # Agent metrics
└── tests/
```

## Acceptance Criteria

1. **Base Agent**: Complete execution loop with safety controls
2. **Specialized Agents**: 6 agent types with tools
3. **Task Queue**: Redis Streams implementation
4. **Orchestration**: Multi-agent workflow support
5. **Approval**: Human-in-the-loop for risky actions
6. **Checkpointing**: Recovery from failures
7. **Circuit Breaker**: Prevent cascading failures

## Token Budget

- Target: 22,000-30,000 tokens

## Dependencies

- Input: ADR-002 (Agent Orchestration)
- Input: Domain models (Prompt 10)
- Output: Used by API Layer, scheduled jobs
