# README — How to Use Your Spec-Driven Development Workspace
> A practical guide for someone new to SDD, AI context engineering, and structured AI-assisted development.

---

## What This Document Covers

After running the COSTAR setup prompt, your workspace will contain a set of folders, Markdown files, prompt templates, and skill files. This README explains exactly what each one is for, when to use it, and how to use it — with step-by-step instructions for every common scenario.

---

## Part 1 — Understanding the Folder Structure

```
your-project/
│
├── .ai/                          ← AI brain of your project
│   ├── project-context.md        ← Master context file (read by Copilot every session)
│   ├── architecture.md           ← Architecture map and boundaries
│   ├── coding-standards.md       ← Language-specific standards
│   ├── api-guidelines.md         ← REST/API design rules
│   ├── security-rules.md         ← Non-negotiable security rules
│   ├── glossary.md               ← Domain and technical terms
│   ├── prompts/                  ← Reusable prompt templates
│   │   ├── 01-architect-plan.md
│   │   ├── 02-spec-writer.md
│   │   ├── 03-test-writer.md
│   │   ├── 04-implementer.md
│   │   ├── 05-security-reviewer.md
│   │   ├── 06-code-reviewer.md
│   │   ├── 07-adr-writer.md
│   │   └── 08-doc-generator.md
│   └── skills/
│       ├── feature-development-skill.md
│       ├── bug-fix-skill.md
│       ├── api-development-skill.md
│       ├── refactor-skill.md
│       └── test-writing-skill.md
│
├── .github/
│   └── copilot-instructions.md   ← Auto-loaded by Copilot on every session
│
├── specs/                        ← One folder per feature/task
│   ├── _template/                ← Copy this to start every new spec
│   └── README.md
│
├── docs/
│   ├── adr/                      ← Architecture Decision Records
│   └── README.md
│
├── src/
│   └── examples/                 ← Golden reference implementations
│
├── SPEC-WORKFLOW.md              ← Visual workflow reference card
└── README-SDD.md                 ← This file
```

---

## Part 2 — The Core Principle (Read This First)

The entire setup is built around one idea:

> **Never explain your project to Copilot again. The `.ai/` folder does it for you.**

Before this setup, every Copilot session started cold — you had to re-explain context, tech stack, standards, and constraints. Now:

1. `.github/copilot-instructions.md` is automatically loaded by Copilot in VS Code.
2. Every prompt template references `.ai/project-context.md` and `.ai/architecture.md`.
3. Copilot reads these files and arrives at your session pre-loaded with full project knowledge.

**Token savings**: You stop repeating the same context every session. You only describe the specific task.

---

## Part 3 — Starting a New Feature (Step-by-Step)

### Step 1 — Write the Spec First (Never Skip This)

1. Copy `specs/_template/` to a new folder: `specs/your-feature-name/`
2. Open Copilot Chat in VS Code (`Ctrl+Shift+I`)
3. Open `.ai/prompts/02-spec-writer.md`
4. Copy the prompt, replace `{{FEATURE_OR_TASK_NAME}}` with your feature name
5. Paste into Copilot Chat and run it
6. Copilot will generate all spec files inside `specs/your-feature-name/`
7. **Review every spec file yourself** — you own the requirements, not the AI

### Step 2 — Get the Architecture Plan

1. Open `.ai/prompts/01-architect-plan.md`
2. Copy the prompt, replace `{{TASK_DESCRIPTION}}` with a brief task description
3. Reference your spec: "Spec is at specs/your-feature-name/"
4. Run in Copilot Chat
5. Copilot will produce a plan — **review it critically before proceeding**
6. If the plan looks wrong, correct it here — it's much cheaper to fix a plan than to fix code

### Step 3 — Generate Tests Before Code

1. Open `.ai/prompts/03-test-writer.md`
2. Fill in `{{COMPONENT_OR_FUNCTION_NAME}}` and spec location
3. Run in Copilot Chat
4. Copilot writes tests based on the spec's acceptance criteria
5. Review the tests — they represent your acceptance contract

### Step 4 — Implement

1. Open `.ai/prompts/04-implementer.md`
2. Fill in `{{TASK_DESCRIPTION}}` with the specific, scoped task (e.g., "Implement only the token validation function")
3. **Be specific and narrow** — do NOT say "build the whole feature at once"
4. Run in Copilot Chat
5. Copilot implements following your spec and golden examples

### Step 5 — Review

1. Open `.ai/prompts/06-code-reviewer.md`, paste the generated diff/file, run it
2. Open `.ai/prompts/05-security-reviewer.md`, paste the same diff, run it
3. Fix all Critical and High findings before merging
4. Run your tests to confirm they pass

---

## Part 4 — Fixing a Bug (Step-by-Step)

1. **Describe the bug clearly first** — use `.ai/skills/bug-fix-skill.md` as your guide
2. Open Copilot Chat, type: `Read .ai/project-context.md. I have a bug: [describe symptom, not cause]`
3. Ask Copilot to **diagnose first, do NOT fix yet**: "Identify likely root cause and affected files. Do not write any code yet."
4. Review the diagnosis — correct it if wrong
5. If the fix is small: proceed with `.ai/prompts/04-implementer.md`
6. If the fix touches architecture: write a spec in `specs/bugfix-your-issue/` first
7. Always run security review if the bug was in authentication, input handling, or data access

---

## Part 5 — How to Use Each Prompt Template

| Prompt File | When to Use | What You Provide |
|---|---|---|
| `01-architect-plan.md` | Before any implementation — always | Task description |
| `02-spec-writer.md` | To generate a full spec folder | Feature/task name |
| `03-test-writer.md` | After spec, before implementation | Component name + spec path |
| `04-implementer.md` | After tests exist | Specific scoped task + spec path |
| `05-security-reviewer.md` | After every implementation | Code file or diff |
| `06-code-reviewer.md` | After every implementation | Code file or diff |
| `07-adr-writer.md` | When making an architecture decision | Decision title |
| `08-doc-generator.md` | After completing a feature | Module or feature name |

**How to run a prompt:**
1. Open the prompt file in VS Code
2. Copy its full content
3. Replace all `{{PLACEHOLDER}}` values with your specifics
4. Open Copilot Chat (`Ctrl+Shift+I`)
5. Paste and send
6. Review Copilot's output before accepting anything

---

## Part 6 — How to Use Each Skill File

Skill files are **workflow checklists** — they tell you the sequence of steps, which prompts to use, and what conditions must be true before you start and after you finish.

| Skill File | Use It When |
|---|---|
| `feature-development-skill.md` | Building any new functionality |
| `bug-fix-skill.md` | Diagnosing and fixing a defect |
| `api-development-skill.md` | Adding a new API endpoint |
| `refactor-skill.md` | Restructuring or cleaning up existing code |
| `test-writing-skill.md` | Writing tests for existing or new code |

**How to use a skill file:**
1. Open the relevant skill file before starting any task
2. Read the pre-conditions — make sure they are all met
3. Follow the steps in order — do not skip steps
4. Check off the quality checklist at the end before considering the task done

---

## Part 7 — How to Create and Use a Spec

A spec is a folder inside `specs/` that defines everything about a feature before any code is written. Think of it as the contract between you and the AI.

### Creating a Spec Manually

1. Copy `specs/_template/` to `specs/your-feature-name/`
2. Fill in `requirements.md` — write the what, not the how
3. Fill in `acceptance-tests.md` — write the exact conditions that must be true when the feature is done
4. Fill in `edge-cases.md` — list every failure mode and boundary condition you can think of
5. Fill in `api-contract.yaml` if the feature includes API changes
6. Fill in `sequence-diagram.md` if the feature has complex multi-step flows

### Creating a Spec Using Copilot

Use `02-spec-writer.md` prompt and let Copilot generate the initial draft, then review and edit every file yourself.

### Using a Spec During Implementation

When running any implementation prompt, always include the spec path:
```
Spec is at: specs/your-feature-name/
Read requirements.md, edge-cases.md, and acceptance-tests.md before generating any code.
```

This is the single most effective way to reduce hallucinations and off-target implementations.

---

## Part 8 — Architecture Decision Records (ADRs)

An ADR records *why* an important decision was made — so that you (or Copilot) never questions or accidentally reverses a deliberate architectural choice.

### When to Write an ADR

Write an ADR whenever you make a decision that:
- Affects more than one module
- Chooses between two or more viable approaches
- Introduces a new library, pattern, or integration
- Changes how authentication, data storage, or APIs work
- Would be confusing to someone reading the code months later

### How to Write an ADR

1. Open `.ai/prompts/07-adr-writer.md`
2. Replace `{{DECISION_TITLE}}` with a concise title (e.g., "Use JWT for API authentication")
3. Run in Copilot Chat
4. Edit the generated draft to ensure the Context and Consequences sections are accurate
5. Save to `docs/adr/ADR-XXX-your-title.md`
6. Update the index in `docs/adr/ADR-000-index.md`

### Referencing ADRs in Prompts

When working in an area covered by an ADR, reference it:
```
This work is governed by ADR-002. Read docs/adr/ADR-002-jwt-authentication.md before proceeding.
```

---

## Part 9 — Maintaining `.ai/project-context.md` Over Time

This file is the most important file in the entire setup. It degrades in value if not maintained. Update it when:

| Change | What to Update |
|---|---|
| New library or framework added | Tech stack section |
| Naming convention changed | Naming conventions section |
| New module or service added | Module structure map |
| New external integration | External dependencies section |
| Security rule added | Security rules section |
| Architecture pattern changed | Link to new ADR + update architecture section |

**Tip**: After every sprint or significant change, spend 5 minutes reviewing `project-context.md` for accuracy. An outdated context file causes Copilot to generate inconsistent code.

---

## Part 10 — How `.github/copilot-instructions.md` Works

GitHub Copilot in VS Code reads `.github/copilot-instructions.md` automatically at the start of every chat session. It acts as a persistent system prompt.

**What this means for you:**
- You no longer need to paste context at the start of every session
- Copilot will follow your workflow rules and coding standards automatically
- If you add a new rule (e.g., "always use Zod for validation"), add it here and it takes effect immediately

**Updating the file:**
1. Open `.github/copilot-instructions.md`
2. Add your rule under the appropriate section
3. Save — it takes effect on your next Copilot Chat session

---

## Part 11 — Token Optimisation Tips

The entire SDD setup is designed to reduce wasted token usage. Here is how:

| Old Behaviour | New Behaviour | Token Saving |
|---|---|---|
| Re-explain project context every session | Context auto-loaded from `.ai/` files | High |
| Giant vague prompts ("build the whole feature") | Small scoped prompts per step | High |
| No spec → lots of back-and-forth clarification | Spec provided upfront → fewer corrections | High |
| No examples → inconsistent style → rewrites | Golden examples → correct style first time | Medium |
| No security review → bugs found late → costly fixes | Security review prompt run immediately | Medium |

**Golden rule**: Invest tokens in the planning phase (spec + architect plan). This always pays off by reducing implementation retries.

---

## Part 12 — Common Mistakes to Avoid

| Mistake | Why It Hurts | What to Do Instead |
|---|---|---|
| Skipping the spec | Copilot has no contract, generates off-target code | Always create the spec folder first |
| Giant prompts ("build everything") | Unpredictable output, hard to review | One scoped task at a time |
| Accepting code without review | AI output is untrusted by default | Always run code reviewer + security reviewer prompts |
| Letting `project-context.md` get stale | Copilot follows outdated conventions | Update it after every significant change |
| Not writing ADRs for decisions | Future-you (or Copilot) will undo deliberate decisions | Write ADR for every significant decision |
| Using `print()` / `console.log()` in production | Violates logging standard | Always use structured logger |
| Hardcoding config values | Security and maintenance risk | Always use environment variables / config files |

---

## Part 13 — Quick Reference Card

```
NEW FEATURE:
  1. specs/_template/ → specs/feature-name/
  2. Prompt: 02-spec-writer.md
  3. Prompt: 01-architect-plan.md  ← Plan, don't code yet
  4. Prompt: 03-test-writer.md     ← Tests before code
  5. Prompt: 04-implementer.md     ← Scoped implementation
  6. Prompt: 06-code-reviewer.md
  7. Prompt: 05-security-reviewer.md
  8. Prompt: 08-doc-generator.md

BUG FIX:
  1. Skill: bug-fix-skill.md
  2. Copilot: diagnose first, no code yet
  3. Prompt: 04-implementer.md (narrow scope)
  4. Prompt: 05-security-reviewer.md

ARCHITECTURE DECISION:
  1. Prompt: 07-adr-writer.md
  2. Save to docs/adr/ADR-XXX.md
  3. Update ADR-000-index.md

CONTEXT SESSION STARTUP:
  .github/copilot-instructions.md loads automatically.
  You only need to provide:
  - Task type (Feature / Bug / Enhancement)
  - Spec path
  - Scope (which files/modules)

KEY FILES:
  .ai/project-context.md     ← Master context
  .ai/architecture.md        ← Architecture map
  .github/copilot-instructions.md ← Auto-loaded by Copilot
  specs/_template/           ← Starting point for every spec
  src/examples/              ← Golden style reference
```

---

*Maintained as a living document. Update this README whenever the workspace structure changes.*
