---
title: CLAUDE.md - CODITECT Compliance & Regulatory Framework
type: workflow
component_type: workflow
version: 1.0.0
created: '2025-12-27'
updated: '2025-12-27'
status: active
tags:
- ai-ml
- deployment
- security
- testing
- architecture
- automation
- backend
- compliance
summary: 'CLAUDE.md - CODITECT Compliance & Regulatory Framework CRITICAL SAFETY DIRECTIVE
  NEVER USE , , , OR ANY DELETE COMMAND WITHOUT EXPLICIT USER PERMISSION. This directive
  applies to: - This repository and ALL subfolders - ANY folder containing a or...'
moe_confidence: 0.950
moe_classified: 2025-12-31
---
# CLAUDE.md - CODITECT Compliance & Regulatory Framework

## CRITICAL SAFETY DIRECTIVE

> **NEVER USE `rm`, `rm -f`, `rm -rf`, OR ANY DELETE COMMAND WITHOUT EXPLICIT USER PERMISSION.**
>
> This directive applies to:
> - This repository and ALL subfolders
> - ANY folder containing a `.coditect` or `.claude` reference
> - ALL compliance documentation, evidence, and artifacts
> - ALL projects using CODITECT framework without exception
>
> **Before deleting ANY file or folder:**
> 1. ASK the user first
> 2. WAIT for explicit confirmation
> 3. NEVER assume files are safe to delete
> 4. NEVER perform "cleanup" deletions autonomously
>
> **Compliance evidence and documentation are CRITICAL - deletion may cause audit failures.**
>
> **Violation of this directive is unacceptable.**

---

## 🔗 Symlink Architecture

**From rollout-master:** `.claude` → `.coditect` → `submodules/core/coditect-core`

**This submodule:** Inherits CODITECT framework components via symlink chain. See `.coditect/` for available agents, commands, and skills.

---

## Project Overview

**CODITECT-COMPLIANCE v1.0** - Enterprise-grade compliance and regulatory oversight management system with autonomous AI agent orchestration for continuous compliance monitoring and evidence collection.

**Mission:** Rival Vanta/Drata/Hyperproof while providing unique multi-agent compliance automation capabilities for regulated industries.

**Status:** Development (Phase 1 - Foundation Documents) | **Classification:** Enterprise Plugin Module
**Target Verticals:** Healthcare (HIPAA/FDA), Fintech (SOC2/PCI), AI Governance (EU AI Act/ISO 42001)
**Stack:** Control Graph Database, Multi-Agent Orchestration, Evidence Collection Pipeline

### Strategic Differentiators
- Agentic Compliance Automation (AI agents autonomously monitor/remediate)
- Control Graph Architecture (cross-framework mapping)
- Regulated Industry Focus (healthcare AI, fintech)
- Autonomous Evidence Collection (1,200+ automated tests)
- CODITECT Integration (compliant-by-default software delivery)
- Cross-Framework Mapping (30-35+ standards from single control)

---

## Essential Reading

**Start here in order:**

1. **[00-MASTER-ORCHESTRATION-PROMPT.md](00-MASTER-ORCHESTRATION-PROMPT.md)** - Complete build sequence and dependencies
2. **[docs/BUILD-ORCHESTRATION-GUIDE.md](docs/BUILD-ORCHESTRATION-GUIDE.md)** - Implementation phases and compliance coverage
3. **[vanta_compliance_features.md](vanta_compliance_features.md)** - Competitive landscape analysis
4. **[docs/project-management/PROJECT-PLAN.md](docs/project-management/PROJECT-PLAN.md)** - Implementation strategy
5. **[docs/project-management/TASKLIST.md](docs/project-management/TASKLIST.md)** - Current task progress

---

## 📂 Directory Structure

```
coditect-compliance-core/
├── .coditect/                          # CODITECT framework (symlinked)
├── .claude -> .coditect                # Claude Code compatibility
├── docs/
│   ├── project-management/             # PROJECT-PLAN.md + TASKLIST.md
│   └── BUILD-ORCHESTRATION-GUIDE.md    # Build phases and compliance coverage
├── 00-MASTER-ORCHESTRATION-PROMPT.md   # Build sequence orchestration
├── 01-04-*.md                          # Phase 1: Product/Requirements/Design
├── 05-09-ADR-*.md                      # Phase 2: Architecture Decisions
├── 10-20-BUILD-*.md                    # Phase 3: Component Build
├── vanta_compliance_features.md        # Competitive analysis
└── files/                              # Supporting artifacts
```

---

## 🚀 Quick Start Examples

### Execute Build Sequence
```bash
# Phase 1: Foundation Documents (execute prompts 01-04 in sequence)
# Phase 2: Architecture Decisions (execute prompts 05-09 in sequence)
# Phase 3: Component Build (execute prompts 10-20 in sequence)

# Track progress
cat docs/project-management/TASKLIST.md
```

### Use CODITECT Framework
```bash
# Access inherited framework components
cd .coditect

# Available: 81 agents, 105 commands, 58 skills, 109 scripts
# See: .coditect/config/component-counts.json
```

### Compliance Workflow
```bash
# Create compliance artifact: /new-project "SOC2 compliance implementation"
# Evidence collection: Use compliance-agent for automated evidence gathering
# Cross-framework mapping: Use control-graph-service for multi-standard compliance
# Audit preparation: Use evidence-engine for audit-ready documentation
```

---

## Git Workflow

**This is a submodule** - Always commit here FIRST, then update pointer in rollout-master.

```bash
# 1. Work in this submodule
git checkout main
git pull

# 2. Make changes, commit, push IN THIS REPO
git add .
git commit -m "feat(compliance): Add SOC2 control mappings"
git push

# 3. Update pointer in parent rollout-master
cd /Users/halcasteel/PROJECTS/coditect-rollout-master
git add submodules/compliance/coditect-compliance-core
git commit -m "Update compliance core: Add SOC2 control mappings"
git push
```

---

**Last Updated:** December 9, 2025
**Repository:** coditect-compliance-core (submodule of coditect-rollout-master)
**Owner:** AZ1.AI INC
**Lead:** Hal Casteel, Founder/CEO/CTO
**Module Version:** v1.0 (Development Phase)
