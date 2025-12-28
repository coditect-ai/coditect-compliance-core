---
title: 'Prompt 16: Component Build - UI Components'
type: reference
version: 1.0.0
created: '2025-12-27'
updated: '2025-12-27'
status: active
tags:
- authentication
- deployment
- testing
- api
- architecture
- automation
- compliance
- configuration
summary: You are a senior frontend engineer implementing the React UI components for
  CODITECT-COMPLIANCE. These components provide the compliance dashboard,...
---
# Prompt 16: Component Build - UI Components

## Context

You are a senior frontend engineer implementing the React UI components for CODITECT-COMPLIANCE. These components provide the compliance dashboard, control management, and evidence visualization interfaces.

## Output Specification

Generate complete, production-ready TypeScript/React code for the UI components. Output should be 2,000-3,000 lines of code.

## Implementation Requirements

### Technology Stack
- React 18+
- TypeScript 5.x
- Zustand for state management
- TanStack Query for data fetching
- shadcn/ui components
- Tailwind CSS
- Recharts for visualizations

## Component Specifications

### 1. Compliance Dashboard

```tsx
// File: src/components/dashboard/ComplianceDashboard.tsx

import { useQuery } from '@tanstack/react-query';
import { PostureScoreCard } from './PostureScoreCard';
import { FrameworkOverview } from './FrameworkOverview';
import { GapsSummary } from './GapsSummary';
import { RecentActivity } from './RecentActivity';

export function ComplianceDashboard() {
  const { data: posture, isLoading } = useQuery({
    queryKey: ['posture-summary'],
    queryFn: () => api.getPostureSummary()
  });

  if (isLoading) return <DashboardSkeleton />;

  return (
    <div className="grid grid-cols-12 gap-6 p-6">
      {/* Posture Score Overview */}
      <div className="col-span-12 lg:col-span-4">
        <PostureScoreCard 
          score={posture.overallScore}
          trend={posture.trend}
          lastUpdated={posture.calculatedAt}
        />
      </div>
      
      {/* Framework Cards */}
      <div className="col-span-12 lg:col-span-8">
        <FrameworkOverview frameworks={posture.frameworks} />
      </div>
      
      {/* Gaps Summary */}
      <div className="col-span-12 lg:col-span-6">
        <GapsSummary gaps={posture.gaps} />
      </div>
      
      {/* Recent Activity */}
      <div className="col-span-12 lg:col-span-6">
        <RecentActivity />
      </div>
    </div>
  );
}

// File: src/components/dashboard/PostureScoreCard.tsx

interface PostureScoreCardProps {
  score: number;
  trend: 'up' | 'down' | 'stable';
  lastUpdated: string;
}

export function PostureScoreCard({ score, trend, lastUpdated }: PostureScoreCardProps) {
  const getScoreColor = (score: number) => {
    if (score >= 80) return 'text-green-600';
    if (score >= 60) return 'text-yellow-600';
    return 'text-red-600';
  };

  return (
    <Card>
      <CardHeader>
        <CardTitle>Compliance Posture</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="flex items-center justify-center">
          <div className={`text-6xl font-bold ${getScoreColor(score)}`}>
            {score}%
          </div>
          <TrendIndicator trend={trend} />
        </div>
        <p className="text-sm text-muted-foreground text-center mt-2">
          Last updated: {formatRelativeTime(lastUpdated)}
        </p>
      </CardContent>
    </Card>
  );
}
```

### 2. Control Graph Visualization

```tsx
// File: src/components/controls/ControlGraph.tsx

import { useCallback, useMemo } from 'react';
import ReactFlow, { 
  Node, 
  Edge, 
  Controls, 
  Background,
  useNodesState,
  useEdgesState
} from 'reactflow';

interface ControlGraphProps {
  frameworkId: string;
  onControlSelect: (controlId: string) => void;
}

export function ControlGraph({ frameworkId, onControlSelect }: ControlGraphProps) {
  const { data: graphData } = useQuery({
    queryKey: ['control-graph', frameworkId],
    queryFn: () => api.getControlGraph(frameworkId)
  });

  const { nodes, edges } = useMemo(() => {
    if (!graphData) return { nodes: [], edges: [] };
    
    return {
      nodes: graphData.controls.map(control => ({
        id: control.id,
        position: control.position,
        data: { 
          label: control.controlId,
          status: control.implementationStatus,
          category: control.category
        },
        type: 'controlNode'
      })),
      edges: graphData.mappings.map(mapping => ({
        id: mapping.id,
        source: mapping.sourceId,
        target: mapping.targetId,
        type: 'smoothstep',
        animated: mapping.type === 'equivalent'
      }))
    };
  }, [graphData]);

  return (
    <div className="h-[600px] border rounded-lg">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        nodeTypes={{ controlNode: ControlNode }}
        onNodeClick={(_, node) => onControlSelect(node.id)}
        fitView
      >
        <Controls />
        <Background />
      </ReactFlow>
    </div>
  );
}

function ControlNode({ data }: { data: any }) {
  const statusColors = {
    implemented: 'bg-green-100 border-green-500',
    in_progress: 'bg-yellow-100 border-yellow-500',
    not_started: 'bg-gray-100 border-gray-500'
  };

  return (
    <div className={`px-4 py-2 rounded border-2 ${statusColors[data.status]}`}>
      <div className="font-medium">{data.label}</div>
      <div className="text-xs text-muted-foreground">{data.category}</div>
    </div>
  );
}
```

### 3. Evidence Timeline

```tsx
// File: src/components/evidence/EvidenceTimeline.tsx

interface EvidenceTimelineProps {
  controlId: string;
}

export function EvidenceTimeline({ controlId }: EvidenceTimelineProps) {
  const { data: evidence } = useQuery({
    queryKey: ['evidence-timeline', controlId],
    queryFn: () => api.getEvidenceForControl(controlId)
  });

  return (
    <div className="space-y-4">
      <h3 className="text-lg font-semibold">Evidence Timeline</h3>
      
      <div className="relative border-l-2 border-gray-200 pl-6 space-y-6">
        {evidence?.map((item) => (
          <div key={item.id} className="relative">
            <div className="absolute -left-8 w-4 h-4 rounded-full bg-primary" />
            
            <Card>
              <CardHeader className="pb-2">
                <div className="flex justify-between items-start">
                  <CardTitle className="text-sm">{item.title}</CardTitle>
                  <Badge variant={item.status === 'validated' ? 'default' : 'secondary'}>
                    {item.status}
                  </Badge>
                </div>
                <CardDescription>
                  {formatDate(item.collectedAt)} via {item.integrationName}
                </CardDescription>
              </CardHeader>
              <CardContent>
                <p className="text-sm text-muted-foreground">{item.description}</p>
                <Button variant="link" size="sm" className="p-0 mt-2">
                  View Details →
                </Button>
              </CardContent>
            </Card>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 4. Integration Management

```tsx
// File: src/components/integrations/IntegrationList.tsx

export function IntegrationList() {
  const { data: integrations } = useQuery({
    queryKey: ['integrations'],
    queryFn: api.listIntegrations
  });

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h2 className="text-2xl font-bold">Integrations</h2>
        <Button onClick={() => setShowAddDialog(true)}>
          <PlusIcon className="mr-2 h-4 w-4" />
          Add Integration
        </Button>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {integrations?.map((integration) => (
          <IntegrationCard
            key={integration.id}
            integration={integration}
            onConfigure={() => handleConfigure(integration)}
            onTest={() => handleTest(integration)}
          />
        ))}
      </div>
    </div>
  );
}

function IntegrationCard({ integration, onConfigure, onTest }) {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-center gap-3">
          <IntegrationIcon type={integration.type} />
          <div>
            <CardTitle>{integration.name}</CardTitle>
            <CardDescription>{integration.type}</CardDescription>
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <div className="flex items-center gap-2 mb-4">
          <HealthIndicator status={integration.health} />
          <span className="text-sm">{integration.health}</span>
        </div>
        <div className="flex gap-2">
          <Button variant="outline" size="sm" onClick={onTest}>
            Test Connection
          </Button>
          <Button variant="outline" size="sm" onClick={onConfigure}>
            Configure
          </Button>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 5. State Management

```tsx
// File: src/stores/complianceStore.ts

import { create } from 'zustand';

interface ComplianceState {
  selectedFrameworkId: string | null;
  selectedControlId: string | null;
  filters: {
    status: string[];
    category: string[];
    severity: string[];
  };
  setSelectedFramework: (id: string | null) => void;
  setSelectedControl: (id: string | null) => void;
  setFilters: (filters: Partial<ComplianceState['filters']>) => void;
}

export const useComplianceStore = create<ComplianceState>((set) => ({
  selectedFrameworkId: null,
  selectedControlId: null,
  filters: {
    status: [],
    category: [],
    severity: []
  },
  setSelectedFramework: (id) => set({ selectedFrameworkId: id }),
  setSelectedControl: (id) => set({ selectedControlId: id }),
  setFilters: (filters) => set((state) => ({
    filters: { ...state.filters, ...filters }
  }))
}));
```

## File Structure

```
src/
├── components/
│   ├── dashboard/
│   │   ├── ComplianceDashboard.tsx
│   │   ├── PostureScoreCard.tsx
│   │   ├── FrameworkOverview.tsx
│   │   └── GapsSummary.tsx
│   ├── controls/
│   │   ├── ControlGraph.tsx
│   │   ├── ControlList.tsx
│   │   └── ControlDetail.tsx
│   ├── evidence/
│   │   ├── EvidenceTimeline.tsx
│   │   └── EvidenceUpload.tsx
│   ├── integrations/
│   │   ├── IntegrationList.tsx
│   │   └── IntegrationConfig.tsx
│   └── ui/               # shadcn components
├── stores/
│   └── complianceStore.ts
├── hooks/
│   └── useCompliance.ts
├── api/
│   └── client.ts
└── lib/
    └── utils.ts
```

## Acceptance Criteria

1. **Dashboard**: Posture overview with charts
2. **Control Graph**: Interactive visualization
3. **Evidence Timeline**: Chronological evidence view
4. **Integrations**: CRUD for integrations
5. **State Management**: Zustand store
6. **Responsive**: Mobile-friendly design

## Token Budget

- Target: 18,000-25,000 tokens

## Dependencies

- Input: SDD UI requirements
- Input: API Layer (Prompt 15)
- Output: Deployed as web application
