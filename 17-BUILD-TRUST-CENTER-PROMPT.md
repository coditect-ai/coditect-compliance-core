---
title: 'Prompt 17: Component Build - Trust Center'
type: reference
component_type: reference
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
- automation
- backend
- compliance
summary: You are a senior software engineer implementing the Trust Center for CODITECT-COMPLIANCE.
  This is a public-facing portal where customers share their...
moe_confidence: 0.950
moe_classified: 2025-12-31
---
# Prompt 17: Component Build - Trust Center

## Context

You are a senior software engineer implementing the Trust Center for CODITECT-COMPLIANCE. This is a public-facing portal where customers share their compliance posture with prospects, auditors, and partners.

## Output Specification

Generate complete, production-ready code for the Trust Center. Output should be 1,500-2,000 lines of code (TypeScript/React + Python backend).

## Implementation Requirements

### Technology Stack
- Next.js 14 for frontend (SSR for SEO)
- FastAPI backend endpoints
- Public/private document sharing
- Questionnaire automation

## Component Specifications

### 1. Trust Center Portal

```tsx
// File: trust-center/src/app/[slug]/page.tsx

import { Metadata } from 'next';

interface TrustCenterPageProps {
  params: { slug: string };
}

export async function generateMetadata({ params }: TrustCenterPageProps): Promise<Metadata> {
  const org = await getOrganization(params.slug);
  return {
    title: `${org.name} Trust Center`,
    description: `Security and compliance information for ${org.name}`
  };
}

export default async function TrustCenterPage({ params }: TrustCenterPageProps) {
  const org = await getOrganization(params.slug);
  const certifications = await getCertifications(org.id);
  const documents = await getPublicDocuments(org.id);

  return (
    <div className="min-h-screen bg-gradient-to-b from-slate-50 to-white">
      <TrustCenterHeader org={org} />
      
      <main className="container mx-auto px-4 py-12">
        {/* Certifications */}
        <section className="mb-12">
          <h2 className="text-2xl font-bold mb-6">Certifications & Compliance</h2>
          <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
            {certifications.map((cert) => (
              <CertificationCard key={cert.id} certification={cert} />
            ))}
          </div>
        </section>
        
        {/* Security Documents */}
        <section className="mb-12">
          <h2 className="text-2xl font-bold mb-6">Security Documents</h2>
          <DocumentLibrary documents={documents} />
        </section>
        
        {/* Request Access / Questionnaire */}
        <section>
          <h2 className="text-2xl font-bold mb-6">Security Review</h2>
          <RequestAccessForm orgId={org.id} />
        </section>
      </main>
    </div>
  );
}

function CertificationCard({ certification }) {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-center gap-3">
          <CertificationBadge type={certification.type} />
          <CardTitle>{certification.name}</CardTitle>
        </div>
      </CardHeader>
      <CardContent>
        <p className="text-sm text-muted-foreground mb-4">
          {certification.description}
        </p>
        <div className="flex justify-between items-center">
          <Badge variant={certification.status === 'active' ? 'default' : 'secondary'}>
            {certification.status}
          </Badge>
          <span className="text-xs text-muted-foreground">
            Valid until {formatDate(certification.validUntil)}
          </span>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 2. Document Sharing

```tsx
// File: trust-center/src/components/DocumentLibrary.tsx

interface Document {
  id: string;
  title: string;
  type: 'policy' | 'report' | 'certification';
  accessLevel: 'public' | 'nda' | 'request';
  lastUpdated: string;
}

export function DocumentLibrary({ documents }: { documents: Document[] }) {
  return (
    <div className="space-y-4">
      {documents.map((doc) => (
        <DocumentRow key={doc.id} document={doc} />
      ))}
    </div>
  );
}

function DocumentRow({ document }: { document: Document }) {
  const handleAccess = async () => {
    if (document.accessLevel === 'public') {
      window.open(`/api/documents/${document.id}/download`);
    } else if (document.accessLevel === 'nda') {
      setShowNDAModal(true);
    } else {
      setShowRequestModal(true);
    }
  };

  return (
    <div className="flex items-center justify-between p-4 border rounded-lg">
      <div className="flex items-center gap-4">
        <DocumentIcon type={document.type} />
        <div>
          <h3 className="font-medium">{document.title}</h3>
          <p className="text-sm text-muted-foreground">
            Updated {formatRelativeTime(document.lastUpdated)}
          </p>
        </div>
      </div>
      
      <Button 
        variant={document.accessLevel === 'public' ? 'default' : 'outline'}
        onClick={handleAccess}
      >
        {document.accessLevel === 'public' && 'Download'}
        {document.accessLevel === 'nda' && 'Sign NDA to Access'}
        {document.accessLevel === 'request' && 'Request Access'}
      </Button>
    </div>
  );
}
```

### 3. Questionnaire Automation

```python
# File: src/services/trust_center/questionnaire.py

class QuestionnaireService:
    """Automate security questionnaire responses."""
    
    def __init__(
        self,
        evidence_repo: EvidenceRepository,
        agent_service: AgentService,
        template_store: TemplateStore
    ):
        self.evidence = evidence_repo
        self.agents = agent_service
        self.templates = template_store
        
    async def process_questionnaire(
        self,
        organization_id: str,
        questionnaire: UploadedQuestionnaire
    ) -> QuestionnaireResponse:
        """
        Process an uploaded security questionnaire.
        
        Steps:
        1. Parse questionnaire format (PDF, XLSX, DOCX)
        2. Extract questions
        3. Match questions to controls/evidence
        4. Generate draft responses
        5. Return for human review
        """
        # Parse questionnaire
        questions = await self._parse_questionnaire(questionnaire)
        
        # Match to evidence
        responses = []
        for question in questions:
            evidence = await self._find_matching_evidence(
                organization_id,
                question
            )
            
            draft_response = await self._generate_response(
                question,
                evidence
            )
            
            responses.append(QuestionResponse(
                question_id=question.id,
                question_text=question.text,
                draft_response=draft_response,
                confidence=draft_response.confidence,
                supporting_evidence=evidence,
                needs_review=draft_response.confidence < 0.8
            ))
            
        return QuestionnaireResponse(
            id=str(uuid4()),
            organization_id=organization_id,
            questions=len(questions),
            auto_answered=sum(1 for r in responses if not r.needs_review),
            responses=responses
        )
        
    async def _generate_response(
        self,
        question: Question,
        evidence: List[EvidenceItem]
    ) -> DraftResponse:
        """Use AI to generate response from evidence."""
        # Call agent to synthesize response
        result = await self.agents.execute(
            agent_type="questionnaire",
            task=AgentTask(
                type="generate_response",
                context={
                    "question": question.text,
                    "category": question.category,
                    "evidence": [e.to_dict() for e in evidence]
                }
            )
        )
        
        return DraftResponse(
            text=result.output["response"],
            confidence=result.output["confidence"],
            citations=[e.id for e in evidence[:5]]
        )
```

### 4. Access Request Management

```python
# File: src/services/trust_center/access.py

class AccessRequestService:
    """Manage document access requests."""
    
    async def request_access(
        self,
        organization_id: str,
        requester: AccessRequester,
        documents: List[str],
        purpose: str
    ) -> AccessRequest:
        """Create document access request."""
        request = AccessRequest(
            id=str(uuid4()),
            organization_id=organization_id,
            requester_name=requester.name,
            requester_email=requester.email,
            requester_company=requester.company,
            documents=documents,
            purpose=purpose,
            status="pending",
            created_at=datetime.utcnow()
        )
        
        await self.repo.save(request)
        
        # Notify organization admins
        await self.notifications.send(
            template="access_request",
            recipients=await self._get_admins(organization_id),
            data={"request": request}
        )
        
        return request
        
    async def approve_access(
        self,
        request_id: str,
        approver_id: str,
        expiry_days: int = 30
    ) -> AccessGrant:
        """Approve access request."""
        request = await self.repo.get(request_id)
        
        grant = AccessGrant(
            id=str(uuid4()),
            request_id=request_id,
            documents=request.documents,
            expires_at=datetime.utcnow() + timedelta(days=expiry_days),
            access_token=secrets.token_urlsafe(32)
        )
        
        await self.grant_repo.save(grant)
        
        # Send access link
        await self.notifications.send(
            template="access_granted",
            recipients=[request.requester_email],
            data={"grant": grant}
        )
        
        return grant
```

## File Structure

```
trust-center/
├── src/
│   ├── app/
│   │   ├── [slug]/
│   │   │   └── page.tsx        # Main trust center page
│   │   ├── layout.tsx
│   │   └── globals.css
│   ├── components/
│   │   ├── DocumentLibrary.tsx
│   │   ├── CertificationCard.tsx
│   │   ├── RequestAccessForm.tsx
│   │   └── NDAModal.tsx
│   └── lib/
│       └── api.ts

src/services/trust_center/
├── __init__.py
├── service.py               # Main trust center service
├── questionnaire.py         # Questionnaire automation
├── access.py                # Access management
├── documents.py             # Document management
└── api.py                   # FastAPI endpoints
```

## Acceptance Criteria

1. **Public Portal**: SEO-optimized trust center pages
2. **Certifications**: Display compliance badges
3. **Documents**: Public/NDA/request access levels
4. **Questionnaire**: AI-powered auto-responses
5. **Access Requests**: Approval workflow
6. **Customization**: Organization branding

## Token Budget

- Target: 15,000-20,000 tokens

## Dependencies

- Input: PRD Trust Center requirements
- Input: Evidence Engine (Prompt 12)
- Output: Customer-facing portal
