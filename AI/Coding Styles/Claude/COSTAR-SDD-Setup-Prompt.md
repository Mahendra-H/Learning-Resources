# COSTAR Prompt — Spec-Driven Development Workspace Setup
> Copy everything inside the prompt block below and paste it into GitHub Copilot Chat (VS Code) to run the full workspace setup.

---

```
## CONTEXT

I am a solo developer managing multiple projects — some built with Python, others with Node.js and TypeScript — all inside VS Code using GitHub Copilot (Claude and Codex models). No external AI tools, standalone models, or third-party platforms are permitted due to company security constraints.

My core problem: Every time I work on a new feature, bug fix, or enhancement, I waste significant time and tokens re-explaining the project context, architecture, and conventions to Copilot from scratch. This hurts my productivity and inflates token usage.

I follow — or want to follow — these engineering principles:
- Spec-Driven Development (SDD)
- Contract-First Engineering
- Architecture-aware AI generation
- Test-first workflows
- Incremental AI orchestration
- Human-governed AI development
- Multi-role AI usage (Architect → Reviewer → Implementer)

My workflow must follow this sequence:
Requirements → Structured Specs → Architecture Plan → Task Decomposition → Acceptance Tests → AI-Assisted Implementation → Automated Validation → Human Review

---

## OBJECTIVE

Scan the current project workspace, detect the tech stack (Python or Node.js/TypeScript), and then scaffold a complete Spec-Driven Development (SDD) workspace inside this project. The setup must include:

1. A `.ai/` folder containing persistent AI context files that eliminate the need to re-explain project context on every Copilot session.
2. A `specs/` folder structure for structured feature/task specifications.
3. A `docs/` folder for Architecture Decision Records (ADRs) and project documentation.
4. A `.github/copilot-instructions.md` file that auto-loads context into every Copilot Chat session.
5. Reusable prompt templates stored in `.ai/prompts/` covering all common engineering workflows.
6. A `src/examples/` folder with golden reference implementations matching the detected tech stack.
7. Skill files in `.ai/skills/` for common reusable task patterns.
8. A `SPEC-WORKFLOW.md` guide at the project root documenting the full workflow.
9. A `README-SDD.md` at the project root explaining how to use every artifact created.

All files must be production-grade, filled with meaningful, project-specific content — NOT generic placeholders. Adapt every file to the detected tech stack.

---

## STYLE

- All Markdown files: clean, structured, with clear H1/H2/H3 hierarchy, tables where useful, and code blocks where relevant.
- All example code: idiomatic, production-quality, fully commented, following best practices for the detected language (PEP8 + type hints for Python; ESLint/Prettier-compatible TypeScript with strict types for Node.js/TS).
- Folder structure: flat and navigable — avoid deep nesting beyond 3 levels.
- Naming: kebab-case for all files and folders.
- Prompts: reusable, parameterised where possible, using `{{PLACEHOLDER}}` syntax for variable parts.

---

## TONE

Precise, professional, and pragmatic. Write as a Senior Staff Engineer setting up standards for a production codebase. No fluff, no over-explanation. Every file must earn its place.

---

## AUDIENCE

A solo developer who is new to Spec-Driven Development. All guidance must be self-contained and actionable. Assume the developer knows their tech stack well but is unfamiliar with AI context engineering, ADR patterns, and structured spec workflows.

---

## RESULT — EXACT DELIVERABLES

Perform the following steps IN ORDER. After each major step, confirm what was created before proceeding.

---

### STEP 1 — Detect Tech Stack

Scan the project root for:
- `package.json` + `.ts` files → Node.js / TypeScript project
- `requirements.txt` OR `pyproject.toml` OR `setup.py` OR `*.py` files → Python project
- Both present → Mixed project (scaffold for both)

Print detected stack before proceeding.

---

### STEP 2 — Create `.ai/` Folder (Persistent AI Context Layer)

Create the following files inside `.ai/`:

#### `.ai/project-context.md`
Fill with:
- Project name (infer from package.json "name" or folder name)
- Detected tech stack + key framework versions
- High-level purpose of this project (leave a `{{PURPOSE}}` placeholder if not inferable)
- Architecture style (infer from folder structure: MVC, layered, hexagonal, etc.)
- Module/package structure map
- Coding conventions (language-specific best practices)
- Security rules: no secrets in logs, no hardcoded credentials, fail-closed design, zero-trust where applicable
- Testing conventions: unit test file naming, test runner, coverage target
- Logging standards: structured logging only, no PII in logs
- Naming conventions: files, classes, functions, environment variables
- Git branch naming convention: `feature/`, `fix/`, `chore/`, `ai-spike/`
- AI usage rules for this project: always spec-first, always test-first, never modify unrelated files

#### `.ai/architecture.md`
Fill with:
- Current folder/module structure with purpose of each directory
- Data flow diagram (ASCII or Mermaid)
- Key design patterns in use
- Boundaries and constraints (what must NOT be changed without ADR)
- External dependencies and integrations

#### `.ai/coding-standards.md`
Fill with tech-stack-specific standards:
- **Python**: PEP8, type hints mandatory, docstrings (Google style), no bare `except`, prefer `pathlib` over `os.path`, use `dataclasses` or `pydantic` for models
- **Node.js/TS**: strict TypeScript config, ESLint + Prettier, async/await only (no raw callbacks), Zod or class-validator for runtime validation, barrel exports pattern, no `any` types
- Error handling patterns
- Logging patterns
- Testing patterns

#### `.ai/api-guidelines.md`
Fill with:
- REST API design rules (resource naming, HTTP verbs, status codes)
- OpenAPI/Swagger documentation requirement
- Request/response envelope standards
- Versioning strategy
- Pagination standard
- Error response schema
- Correlation ID / trace ID requirement
- Rate limiting considerations

#### `.ai/security-rules.md`
Fill with:
- Non-negotiable rules (secrets never in logs, credentials never hardcoded, fail-closed on auth errors)
- Input validation requirements
- Authentication / authorisation patterns
- Sensitive data handling
- OWASP API Top 10 checklist reference
- Audit logging requirements

#### `.ai/glossary.md`
Create a starter glossary with:
- Common domain terms (with `{{ADD YOUR DOMAIN TERMS HERE}}` placeholder)
- Technical acronyms used in the project
- AI-specific terms used in this workflow (SDD, ADR, contract-first, etc.)

---

### STEP 3 — Create `.ai/prompts/` Folder (Reusable Prompt Templates)

Create the following prompt template files:

#### `.ai/prompts/01-architect-plan.md`
```
# Architect Plan Prompt

Role: Senior Software Architect

Task: Before writing any code, produce a detailed implementation plan for the following task:

{{TASK_DESCRIPTION}}

Output must include:
1. Summary of what needs to be built
2. Files to be created or modified (list each with reason)
3. Data models or interfaces required
4. API contracts (if applicable)
5. Sequence of implementation steps
6. Edge cases and error scenarios to handle
7. Security considerations
8. Test scenarios (unit + integration)
9. Open questions or assumptions

Do NOT generate any code yet. Plan only.
Reference: .ai/project-context.md, .ai/architecture.md
```

#### `.ai/prompts/02-spec-writer.md`
```
# Spec Writer Prompt

Role: Technical Spec Writer

Task: Write a structured spec document for the following feature or task:

{{FEATURE_OR_TASK_NAME}}

Context: Read .ai/project-context.md before writing.

Spec must follow this structure:
- /specs/{{feature-name}}/requirements.md
- /specs/{{feature-name}}/api-contract.yaml (if API work involved)
- /specs/{{feature-name}}/sequence-diagram.md
- /specs/{{feature-name}}/data-model.md (if data changes involved)
- /specs/{{feature-name}}/edge-cases.md
- /specs/{{feature-name}}/acceptance-tests.md

Generate all applicable files.
```

#### `.ai/prompts/03-test-writer.md`
```
# Test Writer Prompt

Role: QA Engineer / Test Designer

Task: Generate tests BEFORE implementation for:

{{COMPONENT_OR_FUNCTION_NAME}}

Spec location: /specs/{{feature-name}}/

Rules:
- Generate unit tests first
- Cover all acceptance criteria from acceptance-tests.md
- Cover all edge cases from edge-cases.md
- Follow testing conventions in .ai/coding-standards.md
- Do NOT generate implementation code yet

Output: Test file(s) only.
```

#### `.ai/prompts/04-implementer.md`
```
# Implementer Prompt

Role: Senior Developer

Task: Implement the following — and ONLY the following:

{{TASK_DESCRIPTION}}

Rules:
- Read .ai/project-context.md and .ai/coding-standards.md first
- Follow spec at /specs/{{feature-name}}/
- Make tests pass from /tests/{{feature-name}}/
- Do NOT modify files outside the scope of this task
- Follow the golden example pattern at /src/examples/
- Add structured logging
- Add input validation
- Handle all error cases from edge-cases.md
- Do NOT hardcode any values — use config/env
```

#### `.ai/prompts/05-security-reviewer.md`
```
# Security Review Prompt

Role: Application Security Reviewer

Task: Review the following code changes for security issues:

{{FILE_OR_DIFF}}

Checklist:
- OWASP API Top 10 violations
- Hardcoded secrets or credentials
- Missing input validation
- Improper error handling (stack traces exposed)
- Missing authentication or authorisation checks
- SQL/NoSQL injection risks
- Logging of sensitive data (PII, tokens, passwords)
- Insecure dependencies
- Fail-open vs fail-closed behaviour

Reference: .ai/security-rules.md

Output: Numbered list of findings with severity (Critical / High / Medium / Low) and fix recommendation.
```

#### `.ai/prompts/06-code-reviewer.md`
```
# Code Review Prompt

Role: Staff Engineer / Code Reviewer

Task: Review the following code for quality, correctness, and standards compliance:

{{FILE_OR_DIFF}}

Review for:
1. Standards compliance (ref: .ai/coding-standards.md)
2. Architecture alignment (ref: .ai/architecture.md)
3. Missing tests or poor test coverage
4. Performance concerns
5. Error handling gaps
6. Documentation gaps
7. Naming and readability
8. Dead code or over-engineering

Output: Structured review with Must Fix / Should Fix / Nice to Have categories.
```

#### `.ai/prompts/07-adr-writer.md`
```
# ADR Writer Prompt

Role: Senior Architect

Task: Write an Architecture Decision Record (ADR) for the following decision:

{{DECISION_TITLE}}

ADR format:
# ADR-{{NUMBER}}: {{TITLE}}

**Date**: {{DATE}}
**Status**: Proposed | Accepted | Deprecated | Superseded

## Context
{{What problem or situation prompted this decision?}}

## Decision
{{What was decided?}}

## Consequences
**Positive**:
**Negative**:
**Risks**:

## Alternatives Considered
{{What else was evaluated?}}

Save to: /docs/adr/ADR-{{NUMBER}}-{{kebab-title}}.md
```

#### `.ai/prompts/08-doc-generator.md`
```
# Documentation Generator Prompt

Role: Technical Writer

Task: Generate documentation for:

{{MODULE_OR_FEATURE_NAME}}

Generate:
1. Module-level README (purpose, usage, configuration)
2. API documentation (if applicable)
3. Sequence diagram (Mermaid format)
4. Runbook (how to operate, monitor, troubleshoot)
5. Onboarding section (how a new developer would understand this module)

Reference: /specs/{{feature-name}}/ and existing source code.
```

---

### STEP 4 — Create `specs/` Folder Structure

Create:
```
specs/
├── _template/
│   ├── requirements.md
│   ├── api-contract.yaml
│   ├── sequence-diagram.md
│   ├── data-model.md
│   ├── edge-cases.md
│   └── acceptance-tests.md
└── README.md
```

Fill `specs/_template/requirements.md` with a fully populated example template showing all required sections:
- Feature Name
- Business Requirement
- Functional Requirements (numbered)
- Non-Functional Requirements
- Out of Scope
- Dependencies
- Acceptance Criteria

Fill `specs/_template/acceptance-tests.md` with a filled example showing test scenario table format.

Fill `specs/README.md` explaining how to create a new spec folder for every feature/task.

---

### STEP 5 — Create `docs/` Folder

Create:
```
docs/
├── adr/
│   ├── README.md
│   └── ADR-000-index.md
└── README.md
```

Fill `docs/adr/ADR-000-index.md` as an ADR index table tracking all future ADRs.
Fill `docs/adr/README.md` explaining ADR format and when to create one.

---

### STEP 6 — Create `.github/copilot-instructions.md`

This file is automatically read by GitHub Copilot on every session. Fill it as follows:

```markdown
# GitHub Copilot Instructions — {{PROJECT_NAME}}

## Read Before Every Session
Always read the following files before generating any code or plan:
- `.ai/project-context.md` — full project context, conventions, rules
- `.ai/architecture.md` — architecture boundaries and patterns
- `.ai/coding-standards.md` — language-specific coding rules
- `.ai/security-rules.md` — non-negotiable security constraints

## Workflow Rules
1. NEVER generate code without a spec. Ask for spec location first.
2. ALWAYS generate tests before implementation.
3. NEVER modify files outside the declared scope of the current task.
4. ALWAYS follow coding standards in `.ai/coding-standards.md`.
5. ALWAYS follow security rules in `.ai/security-rules.md`.
6. ALWAYS produce structured logging — never `print()` or `console.log()` in production code.
7. ALWAYS reference golden examples in `/src/examples/` for implementation style.
8. When in doubt about architecture decisions, ask the developer before proceeding.

## Session Startup Prompt
At the start of any session, the developer will provide:
- Task type: Feature | Bug Fix | Enhancement | Maintenance
- Spec location: `/specs/{{feature-name}}/`
- Scope: exact files or modules in scope

Do not proceed without this information.
```

---

### STEP 7 — Create `src/examples/` Folder (Golden Reference)

Detect tech stack and generate appropriate golden examples:

**If Python:**
```
src/examples/
├── README.md
├── example_service.py         # ideal service class with DI, logging, error handling
├── example_repository.py      # ideal repository/data access pattern
├── example_controller.py      # ideal API route handler (FastAPI or Flask)
├── example_models.py          # ideal Pydantic model definitions
└── example_test.py            # ideal unit test with fixtures and mocking
```

**If Node.js/TypeScript:**
```
src/examples/
├── README.md
├── example.service.ts         # ideal service class with DI, logging, error handling
├── example.repository.ts      # ideal repository/data access pattern
├── example.controller.ts      # ideal REST controller
├── example.dto.ts             # ideal DTO with validation decorators
├── example.interface.ts       # ideal TypeScript interface definitions
└── example.spec.ts            # ideal unit test with Jest
```

Fill every example file with fully implemented, heavily commented production-quality code. These are the canonical style references Copilot must follow.

---

### STEP 8 — Create `.ai/skills/` Folder (Skill Files)

Create the following skill files:

#### `.ai/skills/feature-development-skill.md`
Full workflow for implementing a new feature from scratch.

#### `.ai/skills/bug-fix-skill.md`
Full workflow for diagnosing and fixing a defect.

#### `.ai/skills/api-development-skill.md`
Full workflow for designing and implementing a new API endpoint contract-first.

#### `.ai/skills/refactor-skill.md`
Full workflow for refactoring existing code safely.

#### `.ai/skills/test-writing-skill.md`
Full workflow for writing unit, integration, and acceptance tests.

Each skill file must include:
- When to use this skill
- Pre-conditions (what must exist before starting)
- Step-by-step workflow with exact Copilot prompt references
- Post-conditions (what must be true when done)
- Quality checklist

---

### STEP 9 — Create Root-Level Workflow and README Files

#### `SPEC-WORKFLOW.md` (project root)
Visual end-to-end SDD workflow guide with:
- Full pipeline: Requirements → Spec → Architecture Plan → Tests → Implementation → Review
- Which prompt to use at each stage
- Which skill file to reference
- Decision tree: "Is this a new feature? Go here. Is this a bug fix? Go here."
- Token optimisation tips

#### `README-SDD.md` (project root)
Complete beginner's guide covering:
1. What is Spec-Driven Development and why it matters
2. Folder structure map with purpose of every directory and file
3. How to start a new feature (step-by-step with exact Copilot commands)
4. How to start a bug fix (step-by-step)
5. How to use each prompt template file
6. How to use each skill file
7. How to create and use a spec
8. How to write an ADR and when
9. How to maintain and update `.ai/project-context.md` over time
10. How to use `.github/copilot-instructions.md` effectively
11. Token optimisation tips — how this setup reduces wasted tokens
12. Common mistakes to avoid
13. Quick reference card (one-page cheat sheet at the end)

---

### STEP 10 — Final Confirmation

After all files are created, output:
1. Complete folder tree of everything created
2. Summary of what each top-level folder/file does
3. Recommended first action: "Open README-SDD.md to begin"

---

## CONSTRAINTS

- Do NOT install any packages or run any shell commands.
- Do NOT modify existing source code files.
- Do NOT create files outside the project root.
- All files must be filled with real, meaningful content — no `TODO` placeholders except where explicitly marked with `{{PLACEHOLDER}}` syntax.
- All code examples must be syntactically correct and follow the detected tech stack conventions.
- If the tech stack cannot be determined, ask before proceeding.
- This is a VS Code + GitHub Copilot only environment. No external tools.
```

---

*End of COSTAR Prompt. Copy the block above into GitHub Copilot Chat and run it in your project workspace.*
