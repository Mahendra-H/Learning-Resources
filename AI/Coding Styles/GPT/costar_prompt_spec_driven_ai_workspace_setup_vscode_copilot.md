# COSTAR Prompt — Enterprise Spec-Driven AI-Assisted Development Workspace Setup

## Objective

This prompt is designed to be pasted directly into GitHub Copilot Chat inside VS Code.

It will instruct Copilot (Claude/Codex models) to:
- Analyze the current project workspace.
- Detect the technology stack.
- Create a complete spec-driven development structure.
- Create AI context engineering artifacts.
- Create reusable prompt templates.
- Create architecture and governance standards.
- Create implementation workflows.
- Create review workflows.
- Create onboarding and usage documentation.
- Optimize AI context persistence.
- Reduce repetitive prompting.
- Improve token efficiency.
- Improve implementation quality.
- Establish architecture-first engineering workflow.

This setup is specifically designed for:
- VS Code
- GitHub Copilot
- Claude models via Copilot
- Codex models via Copilot
- Enterprise-restricted environments
- No external AI tools
- No standalone LLMs
- Security-constrained organizations
- Python projects
- Node.js projects
- TypeScript projects
- API-heavy backend systems
- IAM/security-oriented enterprise systems

---

# MASTER COSTAR PROMPT

```text
# CONTEXT

You are acting as a Senior Principal Software Architect, Enterprise AI Workflow Engineer, Security Reviewer, and Spec-Driven Development Expert.

I am using:
- VS Code
- GitHub Copilot Chat
- Claude models through GitHub Copilot
- Codex models through GitHub Copilot

Constraints:
- No external AI tools allowed.
- No standalone models allowed.
- No Cursor IDE.
- No Windsurf.
- No local LLM.
- No MCP servers.
- No external orchestration tools.
- Only GitHub Copilot inside VS Code is allowed.

Project characteristics:
- Some projects use Python.
- Some projects use Node.js.
- Some projects use TypeScript.
- Enterprise environment.
- Security-sensitive environment.
- Architecture governance required.
- Long-term maintainability required.
- AI-assisted development optimization required.

Primary problem:
I repeatedly lose tokens and productivity because I have to explain the project context, architecture, standards, requirements, and implementation expectations again and again.

Primary goal:
Set up this repository/workspace using modern enterprise-grade Spec-Driven Development (SDD) and AI Context Engineering best practices so that:
- project context becomes persistent,
- implementation becomes structured,
- AI prompting becomes smaller and more focused,
- architecture quality improves,
- review quality improves,
- token consumption reduces,
- implementation quality increases,
- AI-generated output becomes more deterministic,
- future onboarding becomes easier,
- maintenance becomes easier,
- planning becomes the primary engineering activity.

The workspace must support:
- Architecture-first development
- Contract-first development
- Test-first workflows
- Incremental AI-assisted implementation
- Security-first engineering
- Human-governed AI development
- Review-first workflows
- Reusable AI prompts
- Context persistence
- Documentation-driven engineering


# OBJECTIVE

Analyze the current repository and automatically set up a complete enterprise-grade AI-assisted spec-driven development framework.

The final result must include:
- AI context folders
- Spec templates
- Architecture templates
- ADR templates
- Prompt templates
- Workflow templates
- Governance rules
- Review checklists
- Security standards
- Coding standards
- Documentation standards
- AI operating procedures
- Development lifecycle guidance
- Project onboarding documentation
- Workspace usage documentation
- State management guidance
- Context management guidance

The repository should become fully optimized for long-term AI-assisted development using GitHub Copilot.


# STYLE

Operate with:
- Enterprise-grade engineering discipline
- Architecture-first mindset
- Security-first mindset
- Maintainability-first mindset
- AI-context engineering mindset
- Token optimization mindset
- Spec-driven mindset
- Incremental implementation mindset
- Strong documentation standards
- Clean folder organization
- Clear separation of concerns
- Deterministic engineering workflows

All generated artifacts must:
- Be production-grade
- Be reusable
- Be scalable
- Be maintainable
- Be easy for AI models to understand
- Be easy for humans to maintain
- Reduce future prompting needs
- Improve Copilot output quality


# TONE

Use:
- Clear enterprise engineering language
- Direct instructions
- Structured formatting
- Concise but complete explanations
- Architecture terminology
- Security terminology where applicable
- AI workflow terminology where applicable

Avoid:
- Generic filler text
- Vague recommendations
- Overly simplified explanations
- Toy examples


# AUDIENCE

Primary audience:
- Staff/Principal engineers
- Architects
- Backend developers
- IAM engineers
- Enterprise developers
- AI-assisted developers

Secondary audience:
- Future onboarding developers
- Reviewers
- Security reviewers
- Operations engineers

Assume the repository may eventually be maintained by multiple teams.


# RESPONSE REQUIREMENTS

Perform ALL the following tasks.

DO NOT ask for confirmation.
DO NOT ask unnecessary questions.
Use intelligent assumptions wherever possible.

If the tech stack is detected:
- tailor standards accordingly.

If multiple tech stacks exist:
- create stack-specific standards.


# TASK 1 — ANALYZE CURRENT REPOSITORY

Analyze:
- folder structure
- source code organization
- frameworks
- runtime
- language
- APIs
- tests
- configuration files
- deployment files
- CI/CD files
- Docker files
- infra files
- architecture patterns
- naming conventions
- logging patterns
- security patterns
- dependency management

Then:
- infer best engineering structure.


# TASK 2 — CREATE ENTERPRISE AI WORKSPACE STRUCTURE

Create the following structure if not already present:

project-root/
│
├── .ai/
│   ├── project-context.md
│   ├── architecture.md
│   ├── coding-standards.md
│   ├── security-rules.md
│   ├── api-guidelines.md
│   ├── testing-standards.md
│   ├── logging-standards.md
│   ├── deployment-standards.md
│   ├── glossary.md
│   ├── tech-stack.md
│   ├── review-checklist.md
│   ├── ai-operating-guidelines.md
│   ├── workflow-guide.md
│   ├── prompt-engineering-guide.md
│   ├── context-management.md
│   ├── task-breakdown-strategy.md
│   ├── implementation-lifecycle.md
│   ├── state-management-guide.md
│   ├── prompts/
│   ├── skills/
│   ├── templates/
│   └── examples/
│
├── specs/
│   ├── feature-template/
│   ├── bugfix-template/
│   ├── enhancement-template/
│   ├── maintenance-template/
│   └── shared/
│
├── docs/
│   ├── architecture/
│   ├── diagrams/
│   ├── runbooks/
│   ├── onboarding/
│   ├── troubleshooting/
│   ├── reviews/
│   └── adr/
│
├── tests/
│
├── src/
│
└── README-AI-WORKFLOW.md


# TASK 3 — CREATE AI CONTEXT FILES

Populate all .ai/*.md files with meaningful enterprise-grade content.

The content must:
- reduce repetitive prompting,
- preserve project memory,
- guide future AI conversations,
- improve implementation consistency.

Examples:

.ai/project-context.md:
- business purpose
- architecture overview
- important modules
- authentication model
- deployment model
- integration systems
- security expectations
- coding philosophy
- testing expectations
- operational expectations
- review expectations

.ai/coding-standards.md:
- naming conventions
- file structure standards
- function standards
- API standards
- dependency injection rules
- exception handling rules
- logging rules
- async programming rules
- TypeScript rules
- Python typing rules
- Node.js module standards

.ai/security-rules.md:
- no secrets in logs
- fail closed
- zero trust
- audit logging mandatory
- input validation mandatory
- OWASP API Top 10 rules
- secure token handling
- JWT validation expectations
- encryption expectations
- secrets handling

.ai/testing-standards.md:
- test pyramid
- unit test expectations
- integration test expectations
- contract test expectations
- acceptance testing
- mocking guidance
- coverage expectations
- naming conventions


# TASK 4 — CREATE REUSABLE AI PROMPT LIBRARY

Create reusable prompts inside:

.ai/prompts/

Create prompts for:
- architecture-review.md
- security-review.md
- performance-review.md
- code-review.md
- implementation-plan.md
- feature-development.md
- defect-fix.md
- root-cause-analysis.md
- api-development.md
- unit-test-generation.md
- refactoring.md
- documentation-generation.md
- threat-modeling.md
- acceptance-test-generation.md
- migration-planning.md
- dependency-review.md
- PR-review.md

Each prompt must:
- follow structured prompting,
- be optimized for GitHub Copilot,
- minimize token waste,
- maximize deterministic output,
- encourage planning before coding.


# TASK 5 — CREATE AI SKILL FILES

Create:

.ai/skills/

Create specialized skill files such as:
- architect.skill.md
- backend-engineer.skill.md
- python-engineer.skill.md
- nodejs-engineer.skill.md
- typescript-engineer.skill.md
- security-reviewer.skill.md
- performance-reviewer.skill.md
- api-designer.skill.md
- test-engineer.skill.md
- documentation-engineer.skill.md
- iam-engineer.skill.md
- reviewer.skill.md

Each skill file must define:
- role responsibilities
- operating principles
- constraints
- review expectations
- quality standards
- anti-patterns
- preferred workflows
- expected output structure

These skill files will later be referenced during prompting.


# TASK 6 — CREATE SPEC-DRIVEN DEVELOPMENT TEMPLATES

Inside:

/specs/

Create templates for:
- new feature
- enhancement
- maintenance
- defect fix
- migration
- refactor
- API implementation

Each template must include:
- business context
- requirements
- non-functional requirements
- architecture impact
- dependencies
- assumptions
- constraints
- acceptance criteria
- edge cases
- security considerations
- observability requirements
- rollback considerations
- deployment impact
- test strategy
- implementation phases
- review checklist
- completion checklist


# TASK 7 — CREATE CONTRACT-FIRST TEMPLATES

Create templates for:
- OpenAPI contracts
- API versioning
- request/response standards
- error handling standards
- pagination standards
- correlation IDs
- audit logging
- structured logging

If REST APIs exist:
- create OpenAPI starter templates.


# TASK 8 — CREATE ARCHITECTURE DECISION RECORDS (ADR)

Create:

docs/adr/

Include:
- ADR template
- example ADRs
- naming standards
- lifecycle guidance

Example ADRs:
- architecture-pattern-selection
- logging-strategy
- authentication-strategy
- testing-strategy
- deployment-strategy
- API-versioning-strategy


# TASK 9 — CREATE GOLDEN EXAMPLES

Create:

.ai/examples/

Include ideal examples for:
- controller
- service
- repository
- middleware
- validation
- structured logging
- exception handling
- unit testing
- integration testing
- configuration management
- API response model

Ensure examples match detected tech stack.

These examples should become the implementation reference standard.


# TASK 10 — CREATE AI WORKFLOW DOCUMENTATION

Create:

README-AI-WORKFLOW.md

This file is CRITICAL.

It must explain:

1. What Spec-Driven Development is
2. Why the repository is structured this way
3. How to use the .ai folder
4. How to use prompts
5. How to use skill files
6. How to create specs
7. How to maintain context
8. How to reduce token consumption
9. How to plan before implementation
10. How to run architecture-first workflows
11. How to use review workflows
12. How to run test-first workflows
13. How to use incremental implementation
14. How to use feature templates
15. How to use bugfix templates
16. How to review AI-generated code
17. How to avoid AI anti-patterns
18. How to maintain project memory
19. How to onboard new developers
20. How to evolve the AI context over time

Also include:

- example workflows,
- example prompts,
- example feature lifecycle,
- example bugfix lifecycle,
- example architecture-review lifecycle,
- recommended daily workflow.


# TASK 11 — CREATE DEVELOPMENT LIFECYCLE GUIDE

Create lifecycle guidance for:
- feature development
- defect fixing
- enhancement work
- maintenance work
- migration work
- refactoring work

Preferred lifecycle:

Requirements
→ Specs
→ Architecture review
→ API contracts
→ Acceptance criteria
→ Tests
→ Implementation plan
→ Incremental implementation
→ Validation
→ Review
→ Merge


# TASK 12 — CREATE AI GOVERNANCE RULES

Create mandatory governance rules such as:
- never generate giant uncontrolled refactors
- never bypass security reviews
- always explain plan before implementation
- always review generated code
- always validate contracts
- always write tests
- always maintain specs
- always maintain ADRs
- never hardcode secrets
- never log sensitive data
- fail closed
- maintain traceability
- preserve backward compatibility unless explicitly approved


# TASK 13 — CREATE STACK-SPECIFIC GUIDANCE

If Python detected:
- create Python best practices
- typing guidance
- FastAPI/Flask/Django guidance if relevant
- virtual environment guidance
- linting guidance
- pytest guidance

If Node.js detected:
- async handling standards
- package management standards
- modularization standards
- logging standards
- dependency governance

If TypeScript detected:
- strict typing standards
- interface-first design
- DTO standards
- API contract typing
- ESLint/Prettier guidance


# TASK 14 — CREATE REVIEW CHECKLISTS

Create review checklists for:
- architecture review
- security review
- code review
- API review
- performance review
- dependency review
- observability review
- operational readiness review


# TASK 15 — CREATE CONTEXT MANAGEMENT STRATEGY

Create guidance for:
- how to maintain AI context
- how to update project-context.md
- how to avoid token waste
- how to keep prompts small
- how to reference specs instead of rewriting context
- how to use skill files effectively
- how to chain prompts effectively
- how to keep AI conversations deterministic


# TASK 16 — CREATE COPILOT OPERATING MODEL

Create documentation specifically for GitHub Copilot usage in VS Code.

Include:
- best prompting strategies
- Claude usage patterns
- Codex usage patterns
- when to use chat vs inline generation
- how to scope prompts
- how to reference files
- how to avoid context overflow
- how to keep implementation incremental
- how to reduce hallucinations
- how to review generated code efficiently


# TASK 17 — CREATE TASK EXECUTION STRATEGY

Create guidance for:

Small task
→ validate
→ review
→ merge
→ next task

Avoid:
- giant implementations
- giant prompts
- uncontrolled edits
- repository-wide rewrites


# TASK 18 — CREATE IMPLEMENTATION PHASE STRATEGY

Define standard implementation phases:

Phase 1:
- Requirements clarification
- Spec creation

Phase 2:
- Architecture review
- Contract definition

Phase 3:
- Test planning
- Acceptance criteria

Phase 4:
- Incremental implementation

Phase 5:
- Validation
- Review
- Documentation

Phase 6:
- Merge
- ADR updates
- Context updates


# TASK 19 — CREATE TOKEN OPTIMIZATION STRATEGY

Create best practices to:
- minimize repetitive prompting
- reduce unnecessary context repetition
- reuse persistent project context
- use file references instead of long prompts
- use structured prompts
- use reusable templates
- avoid giant conversations
- break work into deterministic tasks


# TASK 20 — FINAL VALIDATION

After generating all artifacts:

1. Validate folder structure.
2. Validate markdown quality.
3. Validate consistency.
4. Validate naming conventions.
5. Validate readability.
6. Validate enterprise standards.
7. Validate AI usability.
8. Validate maintainability.
9. Validate token optimization.
10. Ensure all files are interconnected logically.

Finally:
- summarize generated structure,
- summarize intended workflows,
- summarize how developers should use the repository.

IMPORTANT:
The generated setup must feel like a mature enterprise-grade AI-native engineering workspace optimized specifically for GitHub Copilot inside VS Code.
```

---

# Recommended Daily Workflow

## Recommended AI-Assisted Engineering Flow

```text
1. Create spec first
   ↓
2. Create acceptance criteria
   ↓
3. Ask Copilot for implementation plan only
   ↓
4. Review plan
   ↓
5. Generate tests first
   ↓
6. Implement incrementally
   ↓
7. Run validations
   ↓
8. Perform AI review
   ↓
9. Perform human review
   ↓
10. Update ADR/spec/docs
```

---

# Recommended Prompting Strategy

## Good Prompt Pattern

```text
Use:
- .ai/project-context.md
- .ai/coding-standards.md
- specs/authentication/requirements.md
- specs/authentication/acceptance-tests.md

Task:
Implement JWT validation middleware.

Requirements:
- Validate PingFederate JWKS
- Support key rotation
- Fail closed
- Add structured logging
- Add unit tests only

Do not modify unrelated files.
Explain implementation plan first.
```

---

# Recommended Spec Lifecycle

## Feature Lifecycle

```text
Requirement
→ Spec
→ Architecture Review
→ API Contract
→ Acceptance Criteria
→ Test Generation
→ Incremental Implementation
→ Validation
→ Security Review
→ Merge
→ ADR Update
→ Context Update
```

---

# Recommended Repository Discipline

## Keep AI Effective

Always:
- keep files small,
- maintain naming consistency,
- maintain documentation,
- maintain specs,
- maintain ADRs,
- remove dead code,
- maintain architecture boundaries,
- use typed contracts,
- keep modules isolated.

AI quality degrades significantly in:
- messy repositories,
- unclear architecture,
- giant files,
- inconsistent naming,
- missing docs,
- missing contracts.

---

# Recommended Usage of Skill Files

## Example

```text
Use:
- .ai/skills/security-reviewer.skill.md
- .ai/security-rules.md

Review authentication middleware implementation for:
- OWASP API Top 10
- JWT vulnerabilities
- logging leaks
- fail-open risks
- improper exception handling
```

---

# Recommended Context Management Strategy

## DO

- Reference files instead of rewriting context.
- Keep reusable prompts.
- Keep architecture docs updated.
- Keep ADRs updated.
- Use feature specs.
- Keep prompts task-scoped.
- Use incremental development.

## DO NOT

- Paste huge repository context repeatedly.
- Use giant prompts.
- Ask AI to rewrite entire systems.
- Accept code without review.
- Skip specs.
- Skip tests.
- Skip validation.

---

# Expected Outcome

After using this setup:

You should have:
- persistent AI project memory,
- reduced repetitive prompting,
- lower token waste,
- better architecture discipline,
- better implementation quality,
- deterministic AI workflows,
- scalable documentation,
- reusable prompts,
- reusable specs,
- reusable engineering patterns,
- improved onboarding,
- enterprise-grade AI-assisted engineering workflow.

The repository should operate like a modern AI-native engineering workspace optimized specifically for GitHub Copilot in VS Code.

