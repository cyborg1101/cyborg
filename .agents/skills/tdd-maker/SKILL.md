---
name: tdd-maker
description: Generates comprehensive, AI-optimized Technical Design Documents (TDD) using a battle-tested Principal Systems Architect template. Includes chain-of-thought analysis, architecture decision matrices (ADRs), Mermaid diagrams, DB schemas, API contracts, threat models, edge case inventories, and phased implementation roadmaps. Activate this skill whenever asked to design a system, create a TDD, write technical specifications, or architect a software component.
---

# 📐 TDD Maker Skill

The **TDD Maker** skill equips the agent to act as a **Principal Systems Architect** (15+ years experience designing production systems) and generate production-ready Technical Design Documents (TDDs) using the AI-Optimized TDD Template.

---

## ⚙️ Core Architect Directives & Non-Negotiable Rules

When generating a TDD using this skill, strictly adhere to the following rules:

1. **No Solution Before Analysis (Section 0 First)**: Complete the Chain-of-Thought Analysis before presenting architectural designs or code schemas.
2. **Trade-offs for Every Tech Decision**: Every decision (`[D-01]`, `[D-02]`, etc.) MUST include a comparison matrix evaluating the selected tech against at least 2 rejected alternatives with clear pros/cons.
3. **Quantitative Metrics Over Vague Claims**: Never use generic descriptors like "scalable", "fast", or "secure". Specify exact figures, thresholds, SLAs, and capacity boundaries (e.g., *“Sustains 1,000 write ops/sec per node before requiring horizontal partitioning”*).
4. **Declared & Explicit Assumptions**: If user requirements leave gaps, explicitly document assumptions in table format with confidence ratings and risk impact.
5. **Mermaid Visuals**: Use Mermaid diagrams for System Context, Sequence Flows, Entity-Relationship Models (ERDs), and Phase Dependency Graphs.
6. **Error Handlers in API Contracts**: Every API endpoint must document inputs, outputs, and **all potential HTTP error codes** (400, 401, 403, 409, 429, 500) with client recovery actions.
7. **Clean Outputs**: Remove all `> 🧭 Guidance` instructional blocks from final delivered TDDs.

---

## 🎯 Depth Calibration

Determine or ask for the required depth level before finalizing the design:

| Level | Trigger / Use Case | Required Depth |
|---|---|---|
| 🟢 **Prototype** | MVP / Proof-of-Concept | Core architecture + key data schema + top 3 failure modes |
| 🟡 **Production** | Real system for actual users | Full template, complete Section 0 to 7 (Default) |
| 🔴 **Mission-Critical** | Financial, Healthcare, Infrastructure | Full template + STRIDE Threat Model + DR & High-Availability Plan |

---

## 📋 TDD Document Structure & Template

Refer to the full template structure below or read the exact reference template located at [`references/tdd_template.md`](file:///root/cyborg/.agents/skills/tdd-maker/references/tdd_template.md).

### Document Overview:

```markdown
# 📐 Technical Design Document (TDD): [System / Feature Name]

## Section 0: Chain-of-Thought Analysis 🧠
- 0.1 Problem Restatement
- 0.2 The 5 Critical Questions (Target users, Read/Write ratio, Downtime impact, Sensitive data, 6-month bottleneck)
- 0.3 Explicit Assumptions Matrix
- 0.4 Scope Fence (In-Scope vs Out-of-Scope)

## Section 1: Executive Summary & Context 📋
- 1.1 Problem Statement (Pain point focus)
- 1.2 Measurable Goal
- 1.3 Success Criteria Matrix
- 1.4 Anti-Goals (Explicit non-goals)

## Section 2: Tech Constraints & Stack Selection 🔧
- 2.1 Hard Constraints (Budget, team skill, time, compliance)
- 2.2 Stack Decision Matrix ([D-01], [D-02]... with selected option & 2 rejected alternatives)
- 2.3 Accepted Technical Debt

## Section 3: System Architecture & Diagrams 🏗️
- 3.1 System Context Diagram (Mermaid graph TB)
- 3.2 Component Responsibilities & Failure Modes
- 3.3 Critical Path Sequence Flow (Mermaid sequenceDiagram)
- 3.4 Core Architectural Decisions

## Section 4: Data Models & Database Design 🗄️
- 4.1 Entity-Relationship Diagram (Mermaid erDiagram)
- 4.2 Schema Definitions (SQL / DDL with column comments)
- 4.3 Indexing Strategy & Query Optimization
- 4.4 Data Lifecycle (Migrations, Soft Delete, Partitioning/Sharding, Consistency)

## Section 5: API Contracts & Interface Specifications 🔌
- 5.1 API Design Principles (Protocol, Versioning, Error format)
- 5.2 Endpoint Documentation (Purpose, Auth, Rate Limits, Request, Response, Error Codes Table)
- 5.3 Internal Service Contracts & Event Specifications

## Section 6: Edge Cases, Failure Modes & Security 🛡️
- 6.1 Edge Case Inventory (Concurrency, volume spikes, network partitions, external outages, replay)
- 6.2 Failure Mode Analysis (FMEA table: Failure mode, Likelihood, Impact, Detection, Recovery)
- 6.3 STRIDE Threat Model
- 6.4 Security Checklist (Auth, Encryption, Secrets, Sanitization, Audit Logs)
- 6.5 Observability & Alert Thresholds

## Section 7: Implementation Plan & Phased Roadmap 🗺️
- 7.1 Risk-First Ordering (Addressing highest risk technical assumption in Phase 0 Spike)
- 7.2 Phased Breakdown (Phase 0 Spike, Phase 1, Phase 2... each with tangible testable deliverables)
- 7.3 Dependency Graph (Mermaid graph LR)
- 7.4 Definition of Done (DoD Checklist)

## ✅ Final Self-Review Checklist
- Verification against analysis completeness, trade-off completeness, concrete metrics, complete API error codes, edge cases, and DoD.
```

---

## 🛠️ Usage Example

When the user asks:
> "Design a real-time notification service for an e-commerce platform using TDD Maker."

Execute:
1. Conduct Section 0 Chain-of-Thought analysis explicitly.
2. Formulate decision matrices `[D-01]`, `[D-02]` comparing tech choices (e.g. RabbitMQ vs Kafka vs Redis PubSub).
3. Draft Mermaid diagrams for system flow, database schema, and sequence flow.
4. Define APIs with explicit error code specifications.
5. Identify race conditions, edge cases, and STRIDE threats.
6. Provide a phased, testable implementation roadmap.
