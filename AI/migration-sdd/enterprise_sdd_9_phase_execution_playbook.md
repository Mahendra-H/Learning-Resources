# Enterprise SDD Execution Playbook
## PingFederate AWS Migration — AI Assisted Spec-Driven Development Framework

---

# Overview

This document defines a complete multi-phase execution strategy for establishing an enterprise-grade Spec-Driven Development (SDD) workspace for a PingFederate AWS migration project.

The goal of this framework is to:

- Establish an architecture-first engineering workflow
- Maximize productivity using AI-assisted development
- Efficiently utilize Claude and Codex models through GitHub Copilot
- Reduce token wastage and repetitive prompting
- Improve planning, review, and implementation quality
- Create a scalable long-term engineering operating model
- Enable faster onboarding into an unfamiliar enterprise IAM ecosystem
- Build reusable AI engineering workflows and context management strategies

This document is intentionally designed as a practical operational playbook rather than theoretical guidance.

---

# Project Context

## Existing Environment

The organization currently uses:

- PingFederate for SSO and federation
- PingDirectory for identity-related services
- AWS for cloud infrastructure migration
- GitHub Actions for CI/CD
- Terraform for Infrastructure as Code
- Ansible for PingFederate automation and configuration

The current Ping infrastructure is hosted on-premises.

The migration initiative involves:

- Migrating PingFederate IdP infrastructure to AWS
- Decommissioning the on-prem environment after migration
- Supporting enterprise-grade scaling, resiliency, and high availability
- Maintaining production-grade CI/CD and automation workflows

## Current State

Already implemented:

- Stage environment
- Production environment
- Production-ready CI/CD
- Production-ready Terraform infrastructure
- Production-ready Ansible automation

Not implemented:

- Development environment

The Dev environment is being intentionally introduced as:

- A learning platform
- A sandbox for experimentation
- A feature development environment
- A platform for understanding architecture deeply
- A safe environment for operational learning

---

# Why Multi-Phase SDD Instead of One Giant Prompt

Attempting to generate an entire enterprise SDD ecosystem in one massive AI prompt usually causes:

- Context drift
- Shallow artifacts
- Generic documentation
- Inconsistent workflows
- Weak architecture decisions
- Poor maintainability
- Repetitive outputs
- Token waste
- Reduced quality over long responses

Instead, this framework separates responsibilities into focused execution phases.

Each phase:

- Has a clear purpose
- Uses the most suitable AI model
- Produces reusable artifacts
- Preserves context incrementally
- Feeds the next phase
- Reduces hallucinations and drift

---

# Recommended AI Model Strategy

| Phase | Primary Goal | Recommended Model |
|---|---|---|
| Phase 0 | Project understanding + knowledge capture | Claude Opus |
| Phase 1 | SDD architecture and workflow design | Claude Opus |
| Phase 2 | Workspace structure and blueprint | Claude Sonnet |
| Phase 3 | Template and artifact generation | Claude Sonnet |
| Phase 4 | AI context engineering and skill system | Claude Opus |
| Phase 5 | Terraform/Ansible/GitHub scaffolding | Codex |
| Phase 6 | README and onboarding guides | Claude Sonnet |
| Phase 7 | Enterprise review and optimization | Claude Opus |
| Phase 8 | Daily execution workflows | Claude Sonnet + Codex |
| Phase 9 | Long-term governance and sustainability | Claude Opus |
| Small repetitive tasks | Summaries and lightweight updates | Claude Haiku |

---

# Why Each Model Is Used

## Claude Opus

Best suited for:

- Systems thinking
- Architecture design
- SDD workflow orchestration
- Enterprise reasoning
- IAM operational thinking
- Context engineering
- Governance design
- Long-form reasoning
- Review frameworks
- Multi-dimensional planning

Use Opus whenever:

- The problem is ambiguous
- Architecture decisions matter
- Long-term maintainability matters
- Workflow design matters
- AI orchestration matters
- Context preservation matters

---

## Claude Sonnet

Best suited for:

- Structured markdown generation
- Playbooks
- Templates
- Operational guides
- Documentation systems
- Reusable engineering artifacts
- Onboarding guides

Use Sonnet when:

- Generating large document sets
- Creating reusable templates
- Producing structured documentation
- Writing onboarding material
- Producing implementation guides

---

## Claude Haiku

Best suited for:

- Lightweight summaries
- Small rewrites
- Fast reviews
- Minor updates
- Quick comparisons
- Token-efficient refinements

Use Haiku for:

- Daily quick operations
- Small context refreshes
- Fast transformations
- Reformatting content

---

## Codex

Best suited for:

- Terraform implementation
- GitHub Actions implementation
- Ansible scaffolding
- Script generation
- Boilerplate generation
- Refactoring
- CI/CD implementation
- Infrastructure automation

Use Codex when:

- The work becomes implementation-heavy
- Code generation is primary
- Infrastructure scaffolding is required
- Repetitive execution is needed

---

# Critical Context Management Principle

Do NOT repeatedly paste the entire project context into every prompt.

Instead create reusable foundational context documents.

Recommended foundational files:

- PROJECT_CONTEXT.md
- CURRENT_STATE.md
- GOALS.md
- ARCHITECTURE_BASELINE.md
- KNOWN_RISKS.md
- FEATURE_CONTEXT.md

This significantly improves:

- Token efficiency
- AI consistency
- Long-term maintainability
- Workflow quality
- Architectural continuity

---

# Master Execution Flow

```text
Phase 0 → Understand Project
Phase 1 → Design SDD System
Phase 2 → Design Workspace Structure
Phase 3 → Generate Templates
Phase 4 → Build AI Context System
Phase 5 → Generate Infra/Automation Scaffolding
Phase 6 → Generate README + Usage Guides
Phase 7 → Review and Refine Entire System
Phase 8 → Establish Daily Engineering Workflow
Phase 9 → Long-Term Governance Model
```

---

# PHASE 0 — PROJECT KNOWLEDGE FOUNDATION

## Recommended Model
Claude Opus

## Goal

Create foundational project understanding artifacts before generating any implementation structure.

This phase establishes:

- Project understanding
- Enterprise constraints
- Technical baseline
- Domain understanding
- IAM architecture awareness
- Operational awareness
- Learning roadmap

Without this phase:

- Context quality degrades
- AI workflows become inconsistent
- Architecture decisions become shallow
- Documentation becomes fragmented

---

## Expected Outputs

Generate:

1. PROJECT_CONTEXT.md
2. CURRENT_SYSTEM_STATE.md
3. TECHNOLOGY_STACK_OVERVIEW.md
4. ENTERPRISE_CONSTRAINTS.md
5. KNOWN_RISKS.md
6. LEARNING_ROADMAP.md
7. DOMAIN_GLOSSARY.md
8. IAM_ARCHITECTURE_BASELINE.md

---

## Prompt

```text
You are an enterprise IAM architect and AI-assisted engineering consultant.

I am taking ownership of a PingFederate AWS migration project in a large enterprise environment.

Context:
- PingFederate + PingDirectory used for SSO/MFA
- Existing infrastructure hosted on-prem
- PingFederate IdP being migrated to AWS
- Stage and Prod already implemented
- Dev environment not yet created
- CI/CD already production-ready
- Terraform used for IaC
- Ansible used for PingFed automation
- GitHub Actions used for CI/CD
- Infrastructure already supports enterprise-scale resiliency/load/scaling
- On-prem infrastructure will eventually be decommissioned

I am new to this ecosystem and want to establish a long-term Spec-Driven Development workflow optimized for AI-assisted engineering using Claude and Codex.

Task:
Generate foundational project understanding documents:

1. PROJECT_CONTEXT.md
2. CURRENT_SYSTEM_STATE.md
3. TECHNOLOGY_STACK_OVERVIEW.md
4. ENTERPRISE_CONSTRAINTS.md
5. KNOWN_RISKS.md
6. LEARNING_ROADMAP.md
7. DOMAIN_GLOSSARY.md
8. IAM_ARCHITECTURE_BASELINE.md

Focus on:
- preserving long-term project understanding
- minimizing future AI context repetition
- helping a new owner deeply understand the platform
- building reusable AI context foundations

Use enterprise-grade structure and practical explanations.
```

---

# PHASE 1 — DESIGN THE SDD SYSTEM

## Recommended Model
Claude Opus

## Goal

Design the overall Spec-Driven Development operating model.

This phase defines:

- Engineering philosophy
- AI workflow integration
- Context lifecycle
- Planning standards
- Review lifecycle
- Governance structure

This becomes the brain of the entire ecosystem.

---

## Prompt

```text
Using the previously generated project context documents, design a complete enterprise-grade Spec-Driven Development operating model for this PingFederate AWS migration ecosystem.

The SDD system must support:
- Terraform workflows
- Ansible workflows
- GitHub Actions workflows
- AWS infrastructure management
- PingFederate operational management
- Dev environment experimentation
- Architecture-first engineering
- AI-assisted development
- Long-term maintainability

Design:
1. SDD philosophy
2. Engineering lifecycle
3. Requirement lifecycle
4. Design review lifecycle
5. AI-assisted workflow lifecycle
6. Context management lifecycle
7. Review and validation lifecycle
8. Documentation lifecycle
9. Operational handoff lifecycle
10. Governance lifecycle

Additionally define:
- token optimization strategy
- context loading strategy
- feature decomposition strategy
- architecture review standards
- implementation planning standards
- AI interaction standards

Output detailed markdown documents and workflow diagrams in text form.
```

---

# PHASE 2 — WORKSPACE STRUCTURE DESIGN

## Recommended Model
Claude Sonnet

## Goal

Generate the enterprise-grade workspace and repository structure.

This phase creates:

- Folder hierarchy
- Naming conventions
- Artifact organization
- Context organization
- Documentation organization
- Feature organization

---

## Prompt

```text
Using the previously defined SDD operating model, generate a complete enterprise-grade repository and workspace structure for this PingFederate AWS migration project.

The structure must support:
- Spec-Driven Development
- AI-assisted engineering
- Terraform
- Ansible
- GitHub Actions
- Architecture reviews
- Context preservation
- Operational runbooks
- Feature lifecycle tracking
- Decision records
- AI skill files
- Token-efficient workflows

Generate:
1. Full folder hierarchy
2. Naming conventions
3. File organization strategy
4. Context organization strategy
5. Feature organization strategy
6. Documentation organization strategy
7. AI artifact organization strategy
8. State/session tracking strategy

Also explain WHY each folder exists and how it should be used.
```

---

# PHASE 3 — TEMPLATE GENERATION

## Recommended Model
Claude Sonnet

## Goal

Generate reusable enterprise templates.

---

## Prompt

```text
Generate enterprise-grade reusable templates for this Spec-Driven Development workspace.

Generate templates for:
- feature requirements
- architecture design
- Terraform planning
- Ansible planning
- GitHub Actions workflow planning
- implementation plans
- review documents
- ADRs
- runbooks
- RCA documents
- validation checklists
- deployment plans
- operational support
- troubleshooting workflows

Each template must include:
- purpose
- required inputs
- review criteria
- architecture considerations
- security considerations
- rollback planning
- operational considerations
- AI prompting hints
- token optimization hints

Use realistic enterprise IAM examples where appropriate.
```

---

# PHASE 4 — AI CONTEXT + SKILL SYSTEM

## Recommended Model
Claude Opus

## Goal

Create the AI engineering operating system.

This phase defines:

- AI memory strategy
- Context preservation
- Reusable AI skills
- Token optimization strategy
- Prompt engineering patterns
- Session continuation strategy

This phase has one of the highest long-term impacts.

---

## Prompt

```text
Design a complete AI context engineering and reusable skill system for this enterprise PingFederate AWS migration project.

The system must support:
- Claude
- Codex
- GitHub Copilot workflows
- long-running engineering tasks
- token-efficient prompting
- architecture-first engineering
- Spec-Driven Development

Generate:
1. AI context loading strategy
2. Session management strategy
3. Project state preservation strategy
4. Feature context templates
5. AI conversation continuation templates
6. Token optimization workflows
7. Context summarization workflows
8. Reusable engineering skill files

Generate skill files for:
- requirement analysis
- architecture reviews
- Terraform reviews
- Ansible reviews
- GitHub Actions reviews
- implementation planning
- troubleshooting
- AWS reviews
- PingFederate reviews
- operational readiness
- production readiness
- security reviews

For every skill file include:
- when to use
- inputs required
- expected outputs
- prompting examples
- token optimization guidance
- common mistakes
```

---

# PHASE 5 — IMPLEMENTATION SCAFFOLDING

## Recommended Model
Codex

## Goal

Generate execution-oriented implementation scaffolding.

This is where:

- Terraform structure
- GitHub Actions
- Ansible roles
- Bootstrap scripts
- Validation scripts

should be generated.

---

## Prompt

```text
Using the previously defined enterprise SDD architecture and repository structure:

Generate implementation scaffolding for:
- GitHub Actions
- Terraform
- Ansible
- validation scripts
- bootstrap scripts
- local development helpers
- repository initialization

Requirements:
- enterprise-grade structure
- modular Terraform design
- reusable GitHub Actions
- reusable Ansible roles
- environment separation for Dev/Stage/Prod
- Dev environment enablement
- maintainability
- scalability
- validation hooks
- linting
- documentation comments

Do NOT generate shallow examples.

Generate production-oriented scaffolding with explanatory comments and best-practice structure.
```

---

# PHASE 6 — README + ONBOARDING

## Recommended Model
Claude Sonnet

## Goal

Create beginner-friendly but enterprise-grade onboarding documentation.

This phase becomes your long-term operational handbook.

---

## Prompt

```text
Generate a comprehensive beginner-friendly but enterprise-grade onboarding and operational guide for this Spec-Driven Development workspace.

Generate:
1. README.md
2. QUICK_START.md
3. DAILY_WORKFLOW.md
4. AI_USAGE_GUIDE.md
5. TOKEN_OPTIMIZATION_GUIDE.md
6. CONTEXT_MANAGEMENT_GUIDE.md

Explain:
- how to use every folder
- how to use every template
- how to use AI skill files
- how to preserve context
- how to interact with Claude and Codex
- how to review AI-generated code
- how to avoid blind AI dependency
- how to think architecturally
- how to onboard new engineers

Assume the user is new to Spec-Driven Development.
Use practical examples and workflows.
```

---

# PHASE 7 — SYSTEM REVIEW

## Recommended Model
Claude Opus

## Goal

Perform a brutal enterprise-grade review of the entire SDD ecosystem.

This phase validates:

- consistency
- maintainability
- scalability
- governance
- operational practicality
- token optimization
- AI workflow quality

---

## Prompt

```text
Perform a brutal enterprise-grade review of the entire generated Spec-Driven Development ecosystem.

Review:
- architecture consistency
- workflow consistency
- context management quality
- token optimization quality
- Terraform structure
- GitHub Actions structure
- Ansible structure
- maintainability
- scalability
- onboarding quality
- operational practicality
- IAM operational alignment
- AWS operational alignment

Identify:
- weaknesses
- missing areas
- unnecessary complexity
- overengineering
- underengineering
- workflow bottlenecks
- context risks
- governance gaps

Provide:
- prioritized improvements
- architectural refinements
- simplification opportunities
- long-term scaling recommendations
```

---

# PHASE 8 — DAILY EXECUTION WORKFLOWS

## Recommended Model
Claude Sonnet + Codex

## Goal

Turn the SDD framework into practical daily engineering workflows.

This phase operationalizes the system.

---

## Prompt

```text
Using the generated SDD ecosystem, create practical daily engineering workflows for:

- adding new features
- modifying Terraform
- updating Ansible
- reviewing GitHub Actions
- troubleshooting production issues
- creating Dev environment changes
- performing architecture reviews
- using AI efficiently
- preserving context
- reducing token usage
- reviewing AI-generated code
- operational support

Create:
- step-by-step workflows
- reusable command patterns
- reusable prompting patterns
- session management examples
- context loading examples
- review checklists
- validation checklists
```

---

# PHASE 9 — LONG-TERM GOVERNANCE

## Recommended Model
Claude Opus

## Goal

Prevent long-term degradation of the AI-assisted engineering ecosystem.

Most AI-generated systems fail because governance is missing.

This phase creates sustainability.

---

## Prompt

```text
Design a long-term governance and maintenance model for this AI-assisted Spec-Driven Development ecosystem.

Design:
- documentation maintenance strategy
- template evolution strategy
- AI skill maintenance strategy
- context archive strategy
- session archival strategy
- knowledge preservation strategy
- onboarding governance
- operational governance
- review governance
- architecture governance

Also define:
- anti-patterns
- warning signs
- context bloat prevention
- token waste prevention
- AI dependency risks
- periodic review strategy

Provide practical operational recommendations for sustaining this system long-term in a large enterprise IAM environment.
```

---

# Recommended Real-World Execution Order

## Week 1

Focus:
- Phase 0
- Phase 1
- Understanding architecture deeply

Do not rush implementation.

---

## Week 2

Focus:
- Phase 2
- Phase 3
- Workspace structure
- Templates
- Documentation standards

---

## Week 3

Focus:
- Phase 4
- Context system
- AI workflows
- Token optimization

This phase provides massive long-term value.

---

## Week 4

Focus:
- Phase 5
- Dev environment scaffolding
- Terraform understanding
- GitHub Actions understanding
- Ansible role understanding

---

## Week 5+

Focus:
- Phase 6 onward
- Refinement
- Governance
- Operationalization
- Scaling the workflow

---

# Most Important Principles

## 1. Planning Before Coding

Your instinct is correct.

For IAM and infrastructure systems:

- Good planning reduces operational risk
- Good architecture reduces future complexity
- Good documentation reduces onboarding pain
- Good review processes reduce production issues

---

## 2. Never Blindly Trust AI

Always:

- review outputs
- validate Terraform
- validate workflows
- review security implications
- validate rollback strategy
- review IAM impacts carefully

---

## 3. Preserve Context Aggressively

Context loss creates:

- repeated explanations
- wasted tokens
- inconsistent architecture
- duplicated workflows
- hallucinations

Invest heavily in reusable context files.

---

## 4. Separate Planning From Implementation

Do not mix:

- architecture thinking
- implementation generation
- review workflows
- operational decisions

Keep them modular.

---

## 5. Optimize for Long-Term Maintainability

This system should still make sense:

- after 6 months
- after ownership transitions
- after infrastructure changes
- after onboarding new engineers

Build for longevity.

---

# Final Recommendation

For your specific enterprise IAM migration scenario:

- Claude should be your architecture and reasoning engine
- Codex should be your implementation and scaffolding engine
- SDD should become your operational discipline
- Context management should become your biggest investment area

The biggest productivity gains will not come from generating more code.

They will come from:

- better planning
- better architectural thinking
- reusable workflows
- context preservation
- disciplined reviews
- operational clarity

That is what will allow you to successfully take ownership of this enterprise PingFederate AWS ecosystem long-term.

