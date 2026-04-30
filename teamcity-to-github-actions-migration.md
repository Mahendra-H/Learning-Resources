# TeamCity → GitHub Actions Migration Guide
### TypeScript / Node.js APIs on AWS | 15 Services | Dev + Prod

---

## Part 1 — Copilot Prompting Strategy (Claude Models)

Use these prompts in GitHub Copilot Chat with the Claude model selected.

---

### Prompt 1 — Understand an Existing TeamCity Pipeline

Use this when you open a TeamCity build config and need to understand what it actually does.

```
You are a senior DevOps engineer. I am a developer with no deep DevOps knowledge.

I have this TeamCity build configuration. Explain it to me in plain English:
- What does each build step do?
- What triggers the pipeline?
- What environment variables or parameters are used?
- What is the deployment target and mechanism?
- Are there any implicit dependencies I should know about?

TeamCity config:
[PASTE YOUR TEAMCITY CONFIG XML OR BUILD STEPS HERE]
```

---

### Prompt 2 — Convert a TeamCity Pipeline to GitHub Actions

Use this after you understand the TC pipeline.

```
You are a senior DevOps engineer. Convert the following TeamCity build configuration
into a GitHub Actions workflow for a TypeScript Node.js API deployed to AWS.

Requirements:
- Use GitHub Actions best practices (reusable workflows where applicable)
- The pipeline must support two environments: dev and prod
- Use GitHub Environments for secrets separation (dev secrets vs prod secrets)
- Use OIDC-based AWS authentication (no long-lived AWS keys)
- The Node.js version should be parameterized
- Include caching for node_modules
- Add a manual approval gate before prod deployment

TeamCity config:
[PASTE CONFIG]

AWS deployment mechanism currently used:
[e.g., CDK deploy / ECS / Lambda / Elastic Beanstalk — fill this in]
```

---

### Prompt 3 — Audit Generated Workflow for Security and Best Practices

Run this on any workflow YAML Copilot generates before you use it.

```
Audit this GitHub Actions workflow for:
1. Security issues (exposed secrets, overly broad IAM permissions, missing OIDC constraints)
2. Missing best practices (pinned action versions, timeout settings, concurrency controls)
3. Anything that would cause silent failures in a prod deployment
4. Cost or performance improvements

Flag each issue with: [CRITICAL] [WARNING] [SUGGESTION]

Workflow:
[PASTE YAML]
```

---

### Prompt 4 — Generate Reusable Workflow for 15 APIs

Use this to avoid copy-pasting 15 near-identical YAML files.

```
I have 15 TypeScript Node.js APIs deployed to AWS. Each has nearly the same
CI/CD steps. Design a GitHub Actions reusable workflow strategy where:

- A caller workflow in each repo passes service-specific parameters
- The reusable workflow handles: install, lint, test, build, deploy
- Secrets are injected via GitHub Environments (dev / prod)
- The reusable workflow lives in a shared repo called [YOUR-ORG/shared-workflows]

Show me:
1. The reusable workflow file (.github/workflows/node-aws-deploy.yml)
2. An example caller workflow for one of my APIs
3. What inputs and secrets the reusable workflow should accept
```

---

### Prompt 5 — Validate a Workflow Without Running It

```
Without running this GitHub Actions workflow, validate it for:
- YAML syntax errors
- Incorrect GitHub Actions syntax (wrong keys, missing required fields)
- Logic errors (steps that will never run, wrong condition expressions)
- Missing permissions blocks

Workflow:
[PASTE YAML]
```

---

## Part 2 — Understanding TeamCity vs GitHub Actions

### Core Concept Mapping

| TeamCity Concept | GitHub Actions Equivalent | Notes |
|---|---|---|
| Build Configuration | Workflow file (`.github/workflows/*.yml`) | One YAML file per workflow |
| Build Step | `step` in a job | Sequential by default |
| Build Agent | `runs-on` runner | Use `ubuntu-latest` for most cases |
| VCS Root | `on: push / pull_request` trigger | Configured in the workflow |
| Build Trigger | `on:` block | Push, PR, schedule, manual (`workflow_dispatch`) |
| Build Parameters | `inputs` (manual) or `env` (static) | |
| Agent Requirements | `runs-on` labels or self-hosted runner tags | |
| Build Chain (dependencies) | `needs:` between jobs | Explicit DAG |
| Artifact Publishing | `actions/upload-artifact` | |
| Artifact Downloading | `actions/download-artifact` | |
| Shared Build Template | Reusable Workflow (`.github/workflows/`) | Called with `uses:` |
| Environment Variables | `env:` at workflow / job / step level | |
| Build Feature: SSH Agent | `webfactory/ssh-agent` action | |
| Deployment environments | GitHub Environments (with protection rules) | Replaces TC deployment configs |
| Approval gates | GitHub Environment required reviewers | |
| TeamCity secrets | GitHub Secrets (repo or environment level) | |

---

### How to Read a TeamCity Build Config (Quick Guide)

When you open a TeamCity project, look for these things in order:

**1. VCS Settings** — tells you which branch triggers the build and what the checkout rules are.

**2. Build Steps** — the ordered list of commands. Each step has a Runner Type:
- `Command Line` → raw shell commands
- `Gradle`, `Maven`, `Node.js` → language-specific runners
- `AWS CodeDeploy`, `S3 Upload` → deployment steps

**3. Parameters** — `%parameter.name%` syntax. These are either:
- System properties (`system.*`)
- Environment variables (`env.*`)
- Config parameters (used in step definitions)

**4. Build Features** — things like SSH Agent, automatic merge, or Docker support running alongside the build.

**5. Dependencies** — if a build waits for another build to finish first (Snapshot or Artifact dependency).

**6. Triggers** — VCS trigger (on code push), schedule trigger, or dependency trigger.

**7. Failure Conditions** — what counts as a failed build (non-zero exit code, test failure count, etc.).

---

### GitHub Actions Workflow Anatomy

```yaml
name: Deploy API                        # Workflow name

on:                                     # TRIGGERS (replaces TC VCS Trigger)
  push:
    branches: [main]
  workflow_dispatch:                    # Manual trigger

env:                                    # GLOBAL ENV VARS
  NODE_VERSION: '20'

jobs:
  build:                                # JOB 1
    runs-on: ubuntu-latest              # AGENT (replaces TC Agent Requirement)
    steps:
      - uses: actions/checkout@v4       # STEP: checkout code

      - uses: actions/setup-node@v4     # STEP: set up runtime
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci                     # STEP: shell command
      - run: npm test

  deploy-dev:
    needs: build                        # DEPENDENCY (replaces TC Snapshot Dependency)
    runs-on: ubuntu-latest
    environment: dev                    # GITHUB ENVIRONMENT (secrets + approval rules)
    steps:
      - run: npm run deploy

  deploy-prod:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: prod                   # Prod requires manual approval
    steps:
      - run: npm run deploy
```

---

### AWS Authentication — Do It Right (OIDC, No Static Keys)

**Never store `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` as GitHub secrets.**

Use OIDC instead — GitHub proves its identity to AWS and gets a short-lived token.

**Step 1 — One-time AWS setup (do this once per account)**

```bash
# Create the OIDC identity provider in AWS IAM
# Provider URL: https://token.actions.githubusercontent.com
# Audience: sts.amazonaws.com
```

**Step 2 — Create an IAM Role with this trust policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:YOUR-ORG/*:environment:prod"
      }
    }
  }]
}
```

**Step 3 — Use in workflow**

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      aws-region: us-east-1
```

Store `AWS_ROLE_ARN` in your GitHub Environment (dev / prod separately).

---

## Part 3 — Migration Plan: 15 APIs

### Phase 0 — Preparation (Before Touching Any Pipeline)

**Checklist:**

- [ ] Export all 15 TeamCity build configs (XML export from TC admin)
- [ ] For each API, document: trigger branch, build steps, deploy target, env vars used
- [ ] Identify which APIs share identical (or near-identical) pipelines → candidates for reusable workflow
- [ ] Set up GitHub Environments in each repo: `dev` and `prod`
- [ ] Set up OIDC in AWS (one-time)
- [ ] Create dev IAM role and prod IAM role with least-privilege policies
- [ ] Create a shared workflow repo: `your-org/shared-workflows` (if applicable)
- [ ] Agree on branch strategy: is `main` → prod and `develop` → dev? Confirm this.

---

### Phase 1 — Build the Reusable Workflow Template (Week 1)

Do this once. All 15 APIs will use it.

**File:** `your-org/shared-workflows/.github/workflows/node-aws-deploy.yml`

```yaml
name: Node.js AWS Deploy (Reusable)

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      node-version:
        required: false
        type: string
        default: '20'
      working-directory:
        required: false
        type: string
        default: '.'
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'

      - run: npm ci

      - run: npm run lint

      - run: npm test

      - run: npm run build

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - run: npm run deploy  # Replace with your actual deploy command (CDK, SAM, etc.)
```

---

### Phase 2 — Pilot Migration: 1 Low-Risk API (Week 1-2)

Pick one API that is: low-traffic, has good test coverage, not business-critical.

Run the TC pipeline and new GA workflow **in parallel** for 2–3 deploys. Compare:
- Build times
- Deployment artifacts
- Any env var mismatches

Only retire TC for this API after parallel validation passes.

---

### Phase 3 — Migrate Remaining 14 APIs (Week 2-4)

**Batch by similarity.** Group APIs that share the same deploy mechanism (e.g., all Lambda, or all ECS). Migrate one group at a time.

For each API, the caller workflow is minimal:

```yaml
# .github/workflows/deploy.yml (in each API repo)
name: Deploy

on:
  push:
    branches: [main, develop]

jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    uses: your-org/shared-workflows/.github/workflows/node-aws-deploy.yml@main
    with:
      environment: dev
      node-version: '20'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    uses: your-org/shared-workflows/.github/workflows/node-aws-deploy.yml@main
    with:
      environment: prod
      node-version: '20'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

---

### Phase 4 — Decommission TeamCity

Only after all 15 APIs have had at least 2 successful prod deployments via GitHub Actions.

- [ ] Disable TC build triggers (don't delete yet)
- [ ] Monitor for 1 sprint
- [ ] Delete TC configs and retire the TC server

---

### Risk Register

| Risk | Mitigation |
|---|---|
| Hidden env vars in TC not documented | Run every TC build with verbose logging before migration; compare env dumps |
| Different Node.js version between TC agent and GA runner | Pin `node-version` explicitly in both |
| Deploy command differences (TC plugin vs CLI) | Document exact deploy commands per API before starting |
| Prod approval gate not configured → accidental deploy | Set up GitHub Environment protection rules before any prod workflow exists |
| OIDC misconfiguration → deploy fails silently | Test OIDC auth on a non-destructive AWS CLI call first (`aws sts get-caller-identity`) |

---

### Key GitHub Actions Concepts to Know

**Contexts** — variables provided by GitHub at runtime:
- `github.ref` — branch or tag that triggered the workflow
- `github.sha` — commit SHA
- `github.actor` — who triggered it
- `secrets.*` — encrypted secrets
- `env.*` — environment variables

**Expressions** — used in `if:` conditions:
```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

**Concurrency** — prevent two deploys running simultaneously:
```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false  # false for prod; true acceptable for dev
```

**Job outputs** — pass data between jobs:
```yaml
jobs:
  build:
    outputs:
      image-tag: ${{ steps.tag.outputs.value }}
```

---

*Document version: 1.0 | Use as living document — update as each API migrates.*
