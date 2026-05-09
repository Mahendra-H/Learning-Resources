# TeamCity → GitHub Actions Migration Guide
### TypeScript / Node.js APIs on AWS | 15 Services | Dev + Prod

---

## Part 1 — Copilot Prompting Strategy (Claude Models)

Paste these directly into GitHub Copilot Chat with a Claude model selected.

---

### Prompt 1 — Decode a TeamCity Build Config

```
You are a senior DevOps engineer. I am a developer with no DevOps background.

Analyse this TeamCity build configuration for a TypeScript Node.js API on AWS.
Tell me in plain English:
- What each build step does and why it exists
- What triggers this pipeline and on which branches
- Every environment variable / parameter used — what it is and where it comes from
- The exact deployment mechanism and target (Lambda / ECS / EB / S3 etc.)
- Any hidden dependencies, shared templates, or implicit behaviour I must know before migrating

TeamCity config:
[PASTE XML OR BUILD STEPS]
```

---

### Prompt 2 — Convert TC Pipeline to GitHub Actions

Run after Prompt 1 so you understand what you're converting.

```
You are a senior DevOps engineer. Convert the TeamCity config below into a
GitHub Actions workflow for a TypeScript Node.js API deployed to AWS.

Hard requirements:
- Two environments: dev (auto-deploy on push) and prod (manual approval gate)
- GitHub Environments for secret isolation — no repo-level secrets for deploy creds
- OIDC-based AWS auth — no static AWS keys anywhere
- Node.js version as a workflow input, not hardcoded
- npm ci with node_modules cache
- Pinned action versions (@v4 minimum, never @latest)
- timeout-minutes on every job
- concurrency block to prevent parallel deploys to the same environment

TeamCity config: [PASTE]
AWS deploy mechanism: [CDK / SAM / Lambda CLI / ECS / EB — fill in]
```

---

### Prompt 3 — Security & Quality Audit

Run on every generated workflow before committing.

```
Audit this GitHub Actions workflow. Flag every issue as [CRITICAL] [WARNING] [SUGGESTION].

Check for:
- Secrets exposed via echo, logs, or env vars printed in steps
- Actions pinned to @latest or unpinned (supply chain risk)
- Overly broad IAM permissions or missing permissions: block
- Missing timeout-minutes (runaway job risk)
- Missing concurrency control (parallel deploy risk)
- Conditions that will never evaluate true
- Steps that will silently succeed even when they fail
- Any prod deploy reachable without a manual approval gate

Workflow: [PASTE YAML]
```

---

### Prompt 4 — Pre-Flight Validation Without Running

```
Validate this GitHub Actions workflow without executing it.

Check for:
- YAML syntax errors
- Invalid GitHub Actions keys or missing required fields
- if: expressions that are syntactically wrong or will never be true
- Jobs referencing secrets or inputs that are never declared
- Missing permissions blocks required for OIDC or artifact access

Workflow: [PASTE YAML]
```

---

## Part 2 — TeamCity vs GitHub Actions: What You Need to Know

### Concept Mapping

| TeamCity | GitHub Actions | Notes |
|---|---|---|
| Build Configuration | `.github/workflows/*.yml` | One YAML file = one workflow |
| Build Step | `step` inside a job | Sequential within a job |
| Build Agent | `runs-on: ubuntu-latest` | GitHub-hosted; no infrastructure to manage |
| VCS Trigger | `on: push / pull_request` | Defined inside the workflow |
| Build Parameters (`%name%`) | `inputs:` or `env:` | `inputs` for manual triggers; `env` for static values |
| Build Chain | `needs:` between jobs | Explicit dependency graph |
| Shared Build Template | Reusable Workflow (`workflow_call`) | Called with `uses:` from any repo |
| Deployment Environment | GitHub Environment | Holds secrets + approval rules per environment |
| Approval Gate | Environment → Required Reviewers | Blocks job until a human approves in GitHub UI |
| TC Secrets | GitHub Secrets (Environment-scoped) | Never use repo-level secrets for deploy credentials |
| Artifact publish/download | `upload-artifact` / `download-artifact` | Passes files between jobs in the same run |

---

### How to Read a TeamCity Build Config

Open TC admin and inspect these **in this order** before you migrate anything:

**1. VCS Settings** — source repo, branch filter, checkout rules. Tells you *what code* and *when*.

**2. Build Steps** — ordered list of commands. Each has a Runner Type:
- `Command Line` → raw shell
- `Node.js` → npm/yarn steps
- `AWS CodeDeploy / S3 Upload` → deployment steps

**3. Parameters** — `%parameter.name%` syntax. Three types:
- `env.*` → become environment variables at runtime
- `system.*` → passed as system properties to the build tool
- Config params → used inside step definitions only, not at runtime

**4. Build Features** — things running *alongside* the build (SSH agent, Docker support, auto-merge). Easy to miss — check this tab explicitly.

**5. Dependencies** — does this build wait for another build's artifact or completion? If yes, replicate that ordering with `needs:` in GA.

**6. Triggers** — VCS (code push), schedule, or dependency trigger. Map each to the `on:` block.

**7. Failure Conditions** — GA fails on non-zero exit by default. Custom thresholds (test count, metric gates) need scripting in GA.

---

### GitHub Actions Workflow Anatomy

```yaml
name: Deploy API

on:                                       # TRIGGERS — replaces TC VCS Trigger
  push:
    branches: [main, develop]
  workflow_dispatch:                      # Manual trigger from GitHub UI

env:
  NODE_VERSION: '20'                      # Global constant — change once, applies everywhere

jobs:

  build:                                  # JOB 1: CI gate — must pass before any deploy runs
    runs-on: ubuntu-latest
    timeout-minutes: 15                   # Always set — prevents runaway billing
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'                    # Caches based on package-lock.json hash
      - run: npm ci                       # Clean install — never mutates the lockfile
      - run: npm run lint
      - run: npm test

  deploy-dev:
    needs: build                          # DEPENDENCY — replaces TC Snapshot Dependency
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment: dev                      # Pulls dev secrets; no approval needed
    concurrency:
      group: deploy-dev
      cancel-in-progress: true           # Cancel stale runs on fast pushes
    permissions:
      id-token: write                     # Required for OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - run: npm ci
      - run: npm run deploy

  deploy-prod:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment: prod                     # Blocks here until a required reviewer approves
    concurrency:
      group: deploy-prod
      cancel-in-progress: false          # Never cancel an in-flight prod deploy
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - run: npm ci
      - run: npm run deploy
```

---

### AWS Authentication via OIDC (One-Time Setup)

**Never store `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` as secrets.**
OIDC gives GitHub a short-lived token from AWS per run — no rotating keys, no leak risk.

**Step 1 — AWS Console → IAM → Identity Providers → Add Provider**
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

**Step 2 — Create one IAM Role per environment with this trust policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:YOUR-ORG/*:environment:prod"
      }
    }
  }]
}
```

Change `environment:prod` to `environment:dev` for the dev role. Store each ARN in its GitHub Environment as `AWS_ROLE_ARN`.

**Step 3 — Verify OIDC before writing any deploy logic**

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-east-1
- run: aws sts get-caller-identity    # If this prints your role ARN, OIDC is working
```

---

### Key Runtime Concepts

**Contexts** — variables GitHub injects per run:

| Context | Value |
|---|---|
| `github.ref` | `refs/heads/main` |
| `github.sha` | Full commit SHA |
| `github.actor` | Who triggered the run |
| `secrets.NAME` | Encrypted secret value |
| `inputs.NAME` | Value passed to reusable or manual workflow |

**Expressions** — `if:` conditions:
```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

**Job outputs** — pass values between jobs (e.g., image tag from build to deploy):
```yaml
jobs:
  build:
    outputs:
      image-tag: ${{ steps.tag.outputs.value }}
  deploy:
    needs: build
    env:
      IMAGE: ${{ needs.build.outputs.image-tag }}
```

---

## Part 3 — Migration Plan: 15 APIs

### Phase 0 — Audit Before You Touch Anything

- [ ] Export all 15 TC build configs (TC Admin → Project → Export as XML)
- [ ] For each API record: trigger branch, exact build steps, deploy target, all env vars
- [ ] Group APIs by deploy pattern → each group calls one reusable workflow
- [ ] Create GitHub Environments (`dev`, `prod`) in every repo before writing any workflow
- [ ] Add required reviewers to every `prod` environment immediately
- [ ] Create dev IAM role and prod IAM role in AWS with least-privilege policies
- [ ] Create `your-org/shared-workflows` repo for the reusable template
- [ ] Confirm branch strategy with team: `develop` → dev, `main` → prod

---

### Phase 1 — Build the Reusable Workflow (Week 1)

Write this once. All 15 APIs call it. Changes to the pipeline logic happen here, not in 15 repos.

**`your-org/shared-workflows/.github/workflows/node-aws-deploy.yml`**

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
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment: ${{ inputs.environment }}
    concurrency:
      group: deploy-${{ inputs.environment }}
      cancel-in-progress: ${{ inputs.environment != 'prod' }}
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

      - run: npm run deploy    # Replace: cdk deploy / sam deploy / aws lambda update-function-code
```

**Caller workflow in each API repo (`.github/workflows/deploy.yml`):**

```yaml
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

### GitHub Environments — How to Set Them Up (Do This Before Phase 2)

This is referenced throughout the document. Here is exactly how to create them.

**For every API repo, repeat these steps:**

**Create the `dev` environment:**
1. GitHub repo → Settings → Environments → New environment → name it `dev`
2. No approval required — deploys automatically on push to `develop`
3. Add secret: `AWS_ROLE_ARN` → paste the dev IAM role ARN
4. Add any other API-specific env secrets here (DB connection strings, API keys etc.)

**Create the `prod` environment:**
1. GitHub repo → Settings → Environments → New environment → name it `prod`
2. Check **Required reviewers** → add yourself and at least one other person
3. Set **Wait timer** to 0 (no delay, just approval)
4. Add secret: `AWS_ROLE_ARN` → paste the prod IAM role ARN (different from dev)
5. Add prod-specific secrets here

**Result:** When a workflow job targets `environment: prod`, GitHub pauses it and sends a Slack/email notification to the required reviewers. Nobody can skip this — it is enforced by GitHub, not by convention.

---

### AWS Deploy Commands — Replace the Placeholder

The reusable workflow has `npm run deploy` as a placeholder. Your `package.json` deploy script must call one of these. Use whichever matches your setup:

**AWS CDK**
```json
// package.json
"scripts": {
  "deploy": "cdk deploy --require-approval never"
}
```

**AWS SAM**
```json
"scripts": {
  "deploy": "sam deploy --no-confirm-changeset --no-fail-on-empty-changeset"
}
```

**Lambda — Direct code update (no infra tool)**
```json
"scripts": {
  "deploy": "aws lambda update-function-code --function-name MY-FUNCTION-NAME --zip-file fileb://dist/function.zip"
}
```

**ECS — Force new deployment after pushing image to ECR**
```json
"scripts": {
  "deploy": "aws ecs update-service --cluster MY-CLUSTER --service MY-SERVICE --force-new-deployment"
}
```

The environment (dev vs prod) is controlled by the `AWS_ROLE_ARN` OIDC role, not by a flag in the deploy command. The same script runs in both environments — it deploys to whichever AWS account the role belongs to.

---

### Phase 2 — Pilot: 1 Low-Risk API (Week 1–2)

Pick one API: low-traffic, has tests, not customer-facing.

Run **TC and GA in parallel** for 3 consecutive deploys. Validate:
- Same code lands in the same AWS target
- All env vars present and correct in GA
- No timing or ordering regressions

Retire TC for this API only after parallel validation passes. This becomes the reference pattern for all 14 remaining APIs.

---

### Phase 3 — Migrate Remaining 14 APIs (Week 2–4)

**Batch by deploy mechanism** — all Lambda together, all ECS together, all CDK together. Each batch shares the same deploy command in the reusable workflow, so you configure once and apply across the batch.

**For each API in a batch, follow this checklist in order:**

- [ ] Run Prompt 1 (Copilot) on its TC config — document what it actually does
- [ ] Confirm the deploy command matches the batch's mechanism (see AWS Deploy Commands below)
- [ ] Create `.github/workflows/deploy.yml` using the caller workflow template from Phase 1
- [ ] Add `AWS_ROLE_ARN` to the repo's `dev` and `prod` GitHub Environments
- [ ] Push to `develop` → watch the GA run → compare deployment output against the TC run side-by-side
- [ ] If GA deploy matches, push to `main` → approve the prod gate → confirm prod deploy lands correctly
- [ ] Disable TC trigger for this API only after both dev and prod validate
- [ ] Move to the next API in the batch

**Batch sizing:** 2–3 APIs per day is safe. Don't migrate an entire batch in one day — if something is misconfigured, you want to catch it on API 1, not after all 5 are broken.

**Tracking sheet** — maintain a simple table while migrating (add to this doc or a separate file):

| API | TC Config Documented | GA Workflow Created | Dev Validated | Prod Validated | TC Disabled |
|---|---|---|---|---|---|
| api-name-1 | ✅ | ✅ | ✅ | ✅ | ✅ |
| api-name-2 | ✅ | ✅ | ⬜ | ⬜ | ⬜ |

---

### Phase 4 — Decommission TeamCity

Only after all 15 APIs have ≥2 successful prod deployments via GitHub Actions.

- [ ] Disable TC build triggers (do not delete yet)
- [ ] Run one full sprint with TC triggers off
- [ ] Confirm no rollbacks were caused by GA issues
- [ ] Delete TC configs → retire TC server

---

### Risk Register

| Risk | Mitigation |
|---|---|
| Hidden env vars in TC not visible in UI | Run TC build with verbose logging; capture env dump before migration |
| Node.js version mismatch (TC agent vs GA runner) | Check `node --version` on TC agent; pin exact version in GA |
| Deploy command differs (TC plugin vs CLI flag) | Document exact CLI command per API in Phase 0; test in dev first |
| Accidental prod deploy before approval gate is configured | Create GitHub Environments with required reviewers *before* the first workflow commit |
| OIDC fails silently on first run | Verify with `aws sts get-caller-identity` before any deploy step |
| Parallel deploys collide during migration window | Keep TC trigger and GA trigger on different branches during overlap |

---

## Part 4 — GitHub Actions Best Practices (Simple & Doable)

Every item here applies directly to your setup and takes minutes to implement.

---

### 1. Pin Action Versions — Never Use `@latest`

```yaml
# Risk: behaviour changes without warning, supply chain attacks possible
- uses: actions/checkout@latest        # Bad

# Safe: locked to major version
- uses: actions/checkout@v4            # Good

# Safest: locked to exact commit SHA (use for security-critical steps)
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # Best
```

---

### 2. Set `timeout-minutes` on Every Job

Without it, a hung deploy runs for up to 6 hours, burns billable minutes, and locks the environment.

```yaml
jobs:
  build:
    timeout-minutes: 15
  deploy:
    timeout-minutes: 20
```

---

### 3. Use `concurrency` to Prevent Parallel Deploys

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true    # true for dev (cancel stale), false for prod (never interrupt)
```

---

### 4. Minimum `permissions` on Every Job

GA workflows run with broad token permissions by default. Lock them down explicitly:

```yaml
permissions:
  contents: read      # Always needed for checkout
  id-token: write     # Only if using OIDC — remove otherwise
```

---

### 5. `npm ci` — Not `npm install`

```yaml
- run: npm ci   # Installs exactly what's in package-lock.json. Never mutates it.
                # npm install can silently upgrade packages — never use in CI.
```

---

### 6. Cache `node_modules` — One Line, Free Speed

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'    # Caches based on package-lock.json hash. Saves ~40–60s per run.
```

---

### 7. Secrets at Environment Level, Not Repo Level

**Repo-level secrets** → available to every workflow, every branch, every PR.
**Environment-level secrets** → only available when a job explicitly targets that environment.

Store `AWS_ROLE_ARN` and any deploy credentials inside the `dev` / `prod` GitHub Environments. A workflow triggered from a feature branch cannot access prod credentials — even if someone writes a workflow that tries.

---

### 8. Never Echo Secrets

```yaml
# This leaks the value into the workflow log
- run: echo "Token is ${{ secrets.MY_SECRET }}"   # Never do this

# GA masks known secrets automatically, but masking is not foolproof.
# If a secret appears in a log at all, rotate it.
```

---

### 9. Protect the `main` Branch

GitHub → Settings → Branches → Add rule for `main`:
- Require a pull request before merging
- Require the build status check to pass before merge
- Block direct pushes

This means nothing reaches prod without a reviewed PR and a green build — the simplest prod safety net you can add.

---

### 10. Add a Status Badge to Every API's README

```markdown
![Deploy](https://github.com/YOUR-ORG/YOUR-REPO/actions/workflows/deploy.yml/badge.svg)
```

Instant visibility into pipeline health. When the badge goes red, the team knows before anyone checks the logs.

---

*Document version: 2.0 | Living document — tick off checklist items as each API migrates.*
