# CI/CD Anti-Patterns

A catalogue of the most common CI/CD pipeline mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Snowflake Pipelines](#1-snowflake-pipelines)
- [2. Long-Running Branches](#2-long-running-branches)
- [3. Manual Deployments](#3-manual-deployments)
- [4. No Rollback Strategy](#4-no-rollback-strategy)
- [5. Secrets in Pipeline Code](#5-secrets-in-pipeline-code)
- [6. Ignoring Flaky Tests](#6-ignoring-flaky-tests)
- [7. Monolithic Pipelines](#7-monolithic-pipelines)
- [8. No Environment Parity](#8-no-environment-parity)
- [9. Building Different Artifacts Per Environment](#9-building-different-artifacts-per-environment)
- [10. Insufficient Pipeline Observability](#10-insufficient-pipeline-observability)
- [11. Over-Reliance on Manual Approval Gates](#11-over-reliance-on-manual-approval-gates)
- [12. Not Versioning Pipeline Configuration](#12-not-versioning-pipeline-configuration)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Anti-patterns are recurring practices that seem reasonable at first glance but create significant problems over time. In CI/CD pipelines, the consequences of anti-patterns are typically felt during production deployments, incident response, or scaling efforts — the exact moments when your delivery infrastructure needs to work flawlessly.

The patterns documented here represent real failures observed across production CI/CD environments. Each one is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates deployment failures, security vulnerabilities, slow feedback loops, or operational overhead
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Pre-production review:** Use the [Quick Reference Checklist](#quick-reference-checklist) before deploying a new pipeline to production
2. **Code review:** Reference specific sections when reviewing pipeline configuration PRs
3. **Incident retros:** After a deployment incident, check which anti-patterns contributed to the failure
4. **Team onboarding:** Assign this document to new engineers before they write production pipelines
5. **Periodic audit:** Run through the checklist quarterly for existing pipelines and deployment workflows

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|--------------|----------|----------|
| 1 | Snowflake Pipelines | Maintainability | 🔴 Critical |
| 2 | Long-Running Branches | Integration | 🔴 Critical |
| 3 | Manual Deployments | Reliability | 🔴 Critical |
| 4 | No Rollback Strategy | Reliability | 🔴 Critical |
| 5 | Secrets in Pipeline Code | Security | 🔴 Critical |
| 6 | Ignoring Flaky Tests | Quality | 🟠 High |
| 7 | Monolithic Pipelines | Performance | 🟠 High |
| 8 | No Environment Parity | Reliability | 🟠 High |
| 9 | Building Different Artifacts Per Environment | Reliability | 🟠 High |
| 10 | Insufficient Pipeline Observability | Operations | 🟡 Medium |
| 11 | Over-Reliance on Manual Approval Gates | Performance | 🟡 Medium |
| 12 | Not Versioning Pipeline Configuration | Maintainability | 🟡 Medium |

---

## 1. Snowflake Pipelines

### Problem

Every team (or worse, every service) has its own unique, manually configured pipeline that evolved organically over months. No two pipelines work the same way — different CI tools, different folder structures, different deployment scripts, different notification channels. When something breaks, only the original author knows how to fix it. When that author leaves, the pipeline becomes an untouchable artifact that nobody dares to modify.

### Example of the Bad Pattern

```yaml
# Team A: .github/workflows/deploy.yml — hand-crafted artisanal pipeline
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install
      - run: npm run build
      - run: |
          ssh deploy@prod-server-1 "cd /var/www/app && git pull && npm install && pm2 restart all"
```

```yaml
# Team B: Jenkinsfile — completely different approach
pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }
    stage('Deploy') {
      steps {
        sh './scripts/deploy-legacy.sh'   // 800-line bash script nobody understands
      }
    }
  }
}
```

```yaml
# Team C: .gitlab-ci.yml — yet another approach
deploy:
  script:
    - docker build -t myapp .
    - docker push myapp
    - kubectl apply -f k8s/
  only:
    - master   # different default branch name
```

**What happens:**

```
Team A: Deploys via SSH + git pull on the server (no image, no rollback)
Team B: Deploys via an 800-line bash script that calls rsync
Team C: Deploys via kubectl apply with no health checks

Incident occurs — nobody can help debug another team's pipeline.
New engineer onboards — must learn 3 completely different deployment systems.
Platform team wants to add security scanning — must modify 47 unique pipelines.
```

### Why It's Harmful

- Knowledge is siloed — only the author can debug or modify the pipeline
- Security policies and compliance checks must be added to every pipeline individually
- No consistency in deployment behaviour — same org, different reliability guarantees
- Onboarding takes weeks because every team has a bespoke setup
- Platform improvements (caching, scanning, notifications) require N separate implementations
- Incident response is slow because responders must learn each pipeline from scratch

### Recommended Fix

```yaml
# Shared reusable workflow: .github/workflows/deploy-template.yml
# All teams consume this — one place to update, audit, and secure
name: Reusable Deploy Pipeline
on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      dockerfile-path:
        required: false
        type: string
        default: "./Dockerfile"
      deploy-environment:
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: |
          docker build -f ${{ inputs.dockerfile-path }} \
            -t registry.mycompany.com/${{ inputs.service-name }}:${{ github.sha }} .
      - name: Push image
        run: docker push registry.mycompany.com/${{ inputs.service-name }}:${{ github.sha }}

  security-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: registry.mycompany.com/${{ inputs.service-name }}:${{ github.sha }}
          exit-code: 1
          severity: CRITICAL,HIGH

  deploy:
    needs: security-scan
    runs-on: ubuntu-latest
    environment: ${{ inputs.deploy-environment }}
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/${{ inputs.service-name }} \
            app=registry.mycompany.com/${{ inputs.service-name }}:${{ github.sha }}
          kubectl rollout status deployment/${{ inputs.service-name }} --timeout=300s
```

```yaml
# Team A's pipeline — three lines to consume the shared template
name: Deploy My Service
on:
  push:
    branches: [main]
jobs:
  deploy:
    uses: mycompany/platform/.github/workflows/deploy-template.yml@v2
    with:
      service-name: team-a-api
      deploy-environment: production
```

### Quick Check

- [ ] All pipelines use shared templates or reusable workflows
- [ ] Pipeline changes go through code review (not manual UI edits)
- [ ] A platform team owns and maintains the shared pipeline templates
- [ ] New services can onboard to CI/CD in under 30 minutes using existing templates
- [ ] Security scanning and compliance checks are built into the shared template

---

## 2. Long-Running Branches

### Problem

Feature branches that live for days, weeks, or months diverge significantly from the main branch. The longer a branch lives, the larger the eventual merge becomes, the higher the probability of merge conflicts, and the greater the risk that the integrated code introduces regressions. Teams that rely on long-running branches delay integration pain rather than eliminating it.

### Example of the Bad Pattern

```
main:        A ─── B ─── C ─── D ─── E ─── F ─── G ─── H ─── I
                    \                                           
feature/big:         B1 ── B2 ── B3 ── B4 ── B5 ── B6 ── B7 ── B8 ── B9
                     (3 weeks, 47 files changed, 2,800 lines added)

$ git merge feature/big-redesign
Auto-merging src/api/router.js
CONFLICT (content): Merge conflict in src/api/router.js
CONFLICT (content): Merge conflict in src/services/auth.js
CONFLICT (content): Merge conflict in src/models/user.js
CONFLICT (content): Merge conflict in src/utils/validator.js
CONFLICT (content): Merge conflict in tests/integration/api.test.js
... 12 more conflicts ...
Automatic merge failed; fix conflicts and then commit the result.
```

**What happens:**

```
Day 1:    Branch created — "I'll merge it in a few days"
Day 5:    Main branch has moved on — conflicts starting to appear
Day 10:   Developer rebases — spends 2 hours resolving conflicts
Day 15:   Feature is "almost done" — 40+ files changed
Day 20:   Merge attempted — 17 conflicts, 3 broken tests
Day 22:   Merge completed — 2 new bugs discovered in production
Day 25:   Hotfix branch created to fix the bugs from the merge
```

### Why It's Harmful

- Merge conflicts grow exponentially with branch lifetime and team size
- Integration bugs are discovered late — after weeks of divergent development
- Code reviews become impossibly large — reviewers rubber-stamp 2,000-line diffs
- CI pipeline runs take longer due to massive changesets
- Rollbacks are difficult because the merge commit touches dozens of files
- Developers avoid pulling from main, creating a vicious cycle of further divergence

### Recommended Fix

```bash
# Good: trunk-based development with short-lived branches
# Branch lives for hours, not weeks

# 1. Create a short-lived branch
$ git checkout -b feature/add-email-validation
# ... make small, focused changes (< 200 lines) ...

# 2. Push and create PR within 1-2 days
$ git push origin feature/add-email-validation
$ gh pr create --title "Add email validation to signup form"

# 3. Use feature flags for incomplete features
```

```python
# Use feature flags to merge incomplete work safely
# The code is in main but disabled in production
def process_signup(email: str) -> dict:
    if feature_flags.is_enabled("new_email_validation"):
        return validate_email_v2(email)    # new code — off in prod
    return validate_email_v1(email)         # existing code — still active
```

```yaml
# CI pipeline enforces short branch lifetimes
name: Branch Hygiene
on:
  schedule:
    - cron: "0 9 * * 1"   # Every Monday at 9 AM

jobs:
  stale-branches:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Find stale branches
        run: |
          echo "## Branches older than 7 days:" >> $GITHUB_STEP_SUMMARY
          git for-each-ref --sort=-committerdate refs/remotes/origin \
            --format='%(refname:short) %(committerdate:relative)' \
            | grep -E '(weeks|months) ago' \
            | head -20 >> $GITHUB_STEP_SUMMARY
```

**Trunk-based development flow:**

```
main:  A ── B ── C ── D ── E ── F ── G ── H ── I ── J ── K
            \   /    \  /        \  /    \  /
feat1:       B1─     D1─        G1─     I1─
             (4h)    (6h)       (3h)    (2h)

Each branch: < 200 lines, < 2 days, easy to review, easy to roll back
```

### Quick Check

- [ ] Feature branches live for less than 2 days on average
- [ ] Pull requests change fewer than 400 lines of code
- [ ] Developers integrate with main at least once per day
- [ ] Feature flags are used for incomplete features instead of long branches
- [ ] Stale branches older than 7 days trigger automated notifications

---

## 3. Manual Deployments

### Problem

Bypassing the CI/CD pipeline to deploy "quick fixes" directly to production — via SSH, manual kubectl commands, or drag-and-drop uploads — is one of the most dangerous anti-patterns in software delivery. It undermines every safeguard the pipeline provides: automated tests, security scans, audit logs, and reproducible deployments.

### Example of the Bad Pattern

```bash
# "Quick fix" at 11 PM on a Friday
$ ssh deploy@production-server
deploy@prod:~$ cd /var/www/api
deploy@prod:/var/www/api$ git pull origin main
deploy@prod:/var/www/api$ npm install
deploy@prod:/var/www/api$ pm2 restart all
# "Fixed! Going to bed."

# Monday morning: "Why is production running a different version than what CI built?"
```

```bash
# "Just this once" kubectl edit in production
$ kubectl edit deployment api-server -n production
# ... manually change the image tag ...
# ... no record of who changed what or why ...
```

```bash
# Manual S3 upload for a frontend "hotfix"
$ aws s3 sync ./dist s3://production-website --delete
# Skipped: tests, linting, security scan, cache invalidation, version tagging
```

**What happens:**

```
11:00 PM — Developer SSHes into production and applies a "one-line fix"
11:05 PM — Fix works, developer goes to bed
 9:00 AM — CI pipeline runs and deploys the latest main branch
 9:01 AM — The manual fix is overwritten by the automated deployment
 9:15 AM — The original bug is back — but now nobody knows what the fix was
 9:30 AM — Developer is asleep, the rest of the team has no context
```

### Why It's Harmful

- No audit trail — impossible to determine who deployed what and when
- Automated tests, linting, and security scans are completely bypassed
- Manual steps are error-prone — typos, missed steps, wrong environment
- Deployments are not reproducible — "it worked when I ran it from my laptop"
- CI/CD pipeline and production state diverge, causing cascading failures
- Compliance violations — most standards require auditable deployment processes

### Recommended Fix

```yaml
# Good: all deployments go through the pipeline — no exceptions
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      - run: npm run lint

  security-scan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        run: npm audit --audit-level=high

  deploy:
    needs: security-scan
    runs-on: ubuntu-latest
    environment: production    # requires approval for production
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: |
          echo "Deploying commit ${{ github.sha }}"
          # All deployments are automated, auditable, and reproducible
```

**Enforce pipeline-only deployments:**

```yaml
# Kubernetes RBAC — developers cannot modify production directly
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-production
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]    # read-only — no create, update, delete
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]    # cannot edit deployments
```

```bash
# For genuine emergencies: a documented, auditable break-glass process
# 1. Create an incident ticket
# 2. Get approval from on-call lead
# 3. Use the emergency pipeline (still automated, still auditable)
# 4. Post-incident review within 48 hours

# Emergency pipeline — same safeguards, faster approval
gh workflow run emergency-deploy.yml \
  --field commit_sha=abc123 \
  --field incident_ticket=INC-4521 \
  --field approver=oncall-lead
```

### Quick Check

- [ ] All production deployments go through the CI/CD pipeline — no SSH, no manual kubectl
- [ ] Production environments have RBAC that prevents direct modification by developers
- [ ] Emergency deployments use a documented break-glass process with audit logging
- [ ] The CI/CD service account is the only identity with deploy permissions
- [ ] Post-incident reviews check whether manual deployments contributed to the incident

---

## 4. No Rollback Strategy

### Problem

Deploying to production without a tested, automated rollback mechanism means that when a bad deployment occurs — and it will — the only options are "fix forward under pressure" or "manually undo changes while the site is down." Without a rollback strategy, a simple bug fix can become a multi-hour outage.

### Example of the Bad Pattern

```yaml
# Bad: deploy with no rollback mechanism
deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to production
      run: |
        kubectl set image deployment/api app=mycompany/api:${{ github.sha }}
        # No health check wait
        # No rollback on failure
        # No previous version recorded
        # If this breaks, good luck
```

```bash
# The incident timeline without rollback
09:00 — Deploy v2.3.1 to production
09:01 — 500 errors spike to 40% of requests
09:05 — Alert fires — on-call engineer paged
09:15 — Engineer identifies the bad deployment
09:20 — "What was the previous version?" — nobody knows
09:25 — Engineer tries to rebuild v2.3.0 from Git tag
09:40 — Build completes — push to registry
09:50 — Redeploy v2.3.0 — but database migrations ran on v2.3.1
09:55 — v2.3.0 crashes because the database schema has changed
10:30 — Manual database rollback attempted
11:00 — Service partially restored after 2-hour outage
```

### Why It's Harmful

- Mean Time to Recovery (MTTR) increases from minutes to hours
- Engineers must debug and fix forward under pressure during an outage
- Database migrations may not be reversible, compounding the problem
- Previous artifact versions are often not retained or easily accessible
- "Fix forward" during an incident introduces additional risk of further breakage
- Customer trust erodes with every prolonged outage

### Recommended Fix

```yaml
# Good: deploy with automated rollback on failure
deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Record current version
      id: current
      run: |
        CURRENT=$(kubectl get deployment/api -o jsonpath='{.spec.template.spec.containers[0].image}')
        echo "image=$CURRENT" >> "$GITHUB_OUTPUT"
        echo "Rolling back to $CURRENT if deployment fails"

    - name: Deploy new version
      run: |
        kubectl set image deployment/api app=mycompany/api:${{ github.sha }}
        kubectl rollout status deployment/api --timeout=300s

    - name: Rollback on failure
      if: failure()
      run: |
        echo "Deployment failed — rolling back to ${{ steps.current.outputs.image }}"
        kubectl rollout undo deployment/api
        kubectl rollout status deployment/api --timeout=300s
```

**Kubernetes native rollback with revision history:**

```bash
# Kubernetes keeps a revision history — rollback is instant
$ kubectl rollout history deployment/api
REVISION  CHANGE-CAUSE
1         Initial deployment
2         Deploy v2.3.0 (commit abc1234)
3         Deploy v2.3.1 (commit def5678)   ← bad deployment

$ kubectl rollout undo deployment/api --to-revision=2
deployment.apps/api rolled back

$ kubectl rollout status deployment/api
deployment "api" successfully rolled out   # < 30 seconds
```

**Database migrations must be backward-compatible:**

```sql
-- Good: additive migration — safe to roll back the application
-- v2.3.1 adds a new column with a default value
ALTER TABLE users ADD COLUMN display_name VARCHAR(255) DEFAULT '';

-- The old code (v2.3.0) simply ignores the new column
-- No breaking change — rollback is safe

-- Bad: destructive migration — rollback breaks the old code
-- ALTER TABLE users DROP COLUMN legacy_name;   ← v2.3.0 still reads this column!
```

### Quick Check

- [ ] Every deployment records the previous version for instant rollback
- [ ] Pipeline automatically rolls back on failed health checks
- [ ] `kubectl rollout undo` or equivalent is tested and documented
- [ ] Database migrations are backward-compatible (additive only, no destructive changes)
- [ ] Rollback procedure is tested in staging at least monthly

---

## 5. Secrets in Pipeline Code

### Problem

Hardcoding credentials, API keys, tokens, and passwords directly in pipeline YAML files, shell scripts, or environment variables committed to the repository means that anyone with repository access — including every past contributor, every CI log viewer, and every fork — can read your production secrets. Pipeline logs may also print secret values in plain text.

### Example of the Bad Pattern

```yaml
# Bad: secrets hardcoded in pipeline YAML
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE
      AWS_SECRET_ACCESS_KEY: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
      DATABASE_URL: postgres://admin:s3cretP@ss@prod-db.rds.amazonaws.com:5432/myapp
      SLACK_WEBHOOK: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
    steps:
      - uses: actions/checkout@v4
      - run: |
          echo "Deploying with DB: $DATABASE_URL"    # ← printed in CI logs!
          npm run deploy
```

```bash
# Bad: secrets in a deploy script committed to the repo
#!/bin/bash
# scripts/deploy.sh
export DOCKER_REGISTRY_PASSWORD="MyS3cur3P@ssw0rd!"
export API_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

docker login -u deploy -p "$DOCKER_REGISTRY_PASSWORD" registry.mycompany.com
docker push registry.mycompany.com/api:latest
```

**What happens:**

```
1. Developer commits secrets to the repo — "I'll move them to env vars later"
2. Secret appears in Git history forever (even after deletion)
3. Another developer forks the repo — secrets are in the fork
4. CI logs print the DATABASE_URL with credentials
5. An attacker finds the leaked AWS keys in a public fork
6. Attacker spins up crypto miners on your AWS account — $47,000 bill
```

### Why It's Harmful

- Secrets in Git history persist forever — `git revert` does not remove them from history
- CI logs are often accessible to all team members and sometimes to external contributors
- Forked repositories carry all secrets from the original repository
- A single leaked AWS key can result in thousands of dollars of unauthorized usage
- Compliance frameworks (SOC 2, PCI-DSS, HIPAA) treat hardcoded secrets as critical findings
- Automated secret scanners (GitHub, GitLab, trufflehog) flag these as high-severity alerts

### Recommended Fix

```yaml
# Good: secrets managed through the CI platform's secret store
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/deploy-role
          aws-region: us-east-1
          # No static keys — uses OIDC federation

      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          # Secrets are masked in logs automatically
          npm run deploy
```

**Use OIDC federation instead of static credentials:**

```yaml
# GitHub Actions OIDC — no long-lived credentials to leak
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS via OIDC
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
      aws-region: us-east-1
      # Short-lived token — expires in 1 hour
      # No secret stored in GitHub — role trust policy validates the repo
```

**Prevent secret leakage in logs:**

```yaml
# Good: mask values and avoid printing secrets
steps:
  - name: Deploy
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    run: |
      # Never echo secrets
      # GitHub Actions automatically masks ${{ secrets.* }} in logs
      # For extra safety, register additional masks:
      echo "::add-mask::$DATABASE_URL"
      npm run deploy
```

### Quick Check

- [ ] No secrets, tokens, or passwords are committed to the repository
- [ ] All secrets are stored in the CI platform's encrypted secret store
- [ ] OIDC federation is used instead of static cloud credentials where possible
- [ ] CI logs do not contain any secret values — verified by manual inspection
- [ ] A secret scanning tool (e.g., `trufflehog`, `gitleaks`, GitHub secret scanning) runs in CI

---

## 6. Ignoring Flaky Tests

### Problem

Flaky tests — tests that pass and fail intermittently without code changes — erode confidence in the entire CI/CD pipeline. When teams respond by disabling, skipping, or re-running failed tests until they pass, the test suite becomes meaningless. Developers learn to ignore red builds, and real bugs slip through because "it's probably just a flaky test."

### Example of the Bad Pattern

```yaml
# Bad: auto-retry failing tests until they pass
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Run tests (retry up to 5 times)
      run: |
        for i in 1 2 3 4 5; do
          npm test && break
          echo "Attempt $i failed — retrying..."
          sleep 10
        done
        # If it passes once out of 5 tries, we call it "green"
```

```javascript
// Bad: test skipped because it's "flaky"
describe("Payment processing", () => {
  it.skip("should charge the customer correctly", () => {
    // TODO: Fix this flaky test — sometimes fails on CI
    // Skipped since March 2024
    // Nobody remembers why it fails
  });

  it.skip("should handle refunds", () => {
    // Also flaky — disabled "temporarily"
  });
});
```

**What happens:**

```
Month 1:   1 flaky test — developer adds .skip() — "I'll fix it later"
Month 3:   7 tests skipped — "the test suite is unreliable anyway"
Month 6:   23 tests skipped — payment processing has zero test coverage
Month 7:   Refund bug ships to production — the test that would have caught it was skipped
Month 7+:  $180,000 in incorrect refunds processed before the bug is discovered
```

### Why It's Harmful

- Developers lose trust in the test suite — red builds are ignored
- Real failures are masked by the assumption that "it's just a flaky test"
- Skipped tests create blind spots in code coverage — critical paths go untested
- Retry loops hide genuine regressions by brute-forcing a passing result
- CI pipeline time increases significantly with retry delays
- The cost of fixing flaky tests grows over time as root causes compound

### Recommended Fix

```yaml
# Good: quarantine flaky tests — track them separately without blocking the pipeline
name: Tests
on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - name: Run stable tests
        run: npm test -- --exclude-tag=flaky

  flaky-tests:
    runs-on: ubuntu-latest
    continue-on-error: true    # does not block the pipeline
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - name: Run quarantined tests
        run: npm test -- --tag=flaky
      - name: Report flaky test results
        if: failure()
        run: |
          echo "::warning::Flaky tests failed — see quarantine dashboard"
          # Post results to a tracking system for prioritised fixing
```

```javascript
// Good: tag flaky tests for quarantine — do not skip them
describe("Payment processing", () => {
  // Quarantined: tracked in JIRA-1234, assigned to @payments-team
  // Root cause: race condition in database seeding
  it("should charge the customer correctly", { tags: ["flaky"] }, () => {
    // Test still runs — just in a separate non-blocking job
    // Fix deadline: 2025-02-15
  });
});
```

**Establish a flaky test policy:**

```markdown
## Flaky Test Policy

1. A test that fails without code changes is immediately tagged as `flaky`
2. A JIRA ticket is created and assigned to the owning team
3. Flaky tests are quarantined — they run separately and do not block deployments
4. Flaky tests must be fixed within 14 days or deleted with a justification
5. A dashboard tracks flaky test count over time — target: zero
```

### Quick Check

- [ ] No tests are skipped with `.skip()`, `@Ignore`, or `xit()` without a tracking ticket
- [ ] Flaky tests are quarantined in a separate CI job, not disabled
- [ ] A dashboard tracks the number of flaky tests over time
- [ ] Flaky tests have an assigned owner and a fix deadline (< 14 days)
- [ ] The team reviews flaky test metrics weekly

---

## 7. Monolithic Pipelines

### Problem

A single, massive pipeline that builds, tests, scans, deploys, and validates everything in one sequential chain — often taking 45 minutes to an hour — destroys developer productivity. Every commit triggers the entire pipeline, even if only a README was changed. Developers wait for long feedback loops, stack up commits, and eventually start merging without waiting for green builds.

### Example of the Bad Pattern

```yaml
# Bad: one giant sequential pipeline for everything
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:

jobs:
  everything:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: npm ci                                  # 3 min
      - name: Lint
        run: npm run lint                            # 2 min
      - name: Unit tests
        run: npm test                                # 5 min
      - name: Build
        run: npm run build                           # 4 min
      - name: Integration tests
        run: npm run test:integration                # 12 min
      - name: E2E tests
        run: npm run test:e2e                        # 15 min
      - name: Security scan
        run: npm audit                               # 2 min
      - name: Build Docker image
        run: docker build -t myapp .                 # 5 min
      - name: Push Docker image
        run: docker push myapp                       # 3 min
      - name: Deploy to staging
        run: kubectl apply -f k8s/staging/           # 2 min
      - name: Smoke tests on staging
        run: npm run test:smoke                      # 5 min
      - name: Deploy to production
        run: kubectl apply -f k8s/production/        # 2 min
      # Total: ~60 minutes, fully sequential, no parallelism
```

**What happens:**

```
09:00 — Developer pushes a typo fix to README.md
09:00 — Full pipeline starts: install, lint, test, build, scan, deploy...
10:00 — Pipeline finishes — 60 minutes to deploy a README change
09:15 — Another developer pushes a real code change — queued behind the README build
10:05 — Second pipeline starts — developer has been waiting 50 minutes
11:05 — Second pipeline finishes — the fix took 2 hours from commit to deploy
```

### Why It's Harmful

- 60-minute feedback loops kill developer productivity and flow state
- Sequential steps cannot exploit parallelism — unit tests wait for lint to finish
- A failure in E2E tests (step 6 of 12) wastes 40 minutes of prior work
- Unrelated changes (docs, config) trigger the full pipeline unnecessarily
- Developers batch commits to avoid waiting, creating larger and riskier changesets
- Pipeline queuing creates bottlenecks during peak development hours

### Recommended Fix

```yaml
# Good: parallel jobs with path-based triggers
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm audit --audit-level=high

  build-and-push:
    needs: [lint, unit-tests, security-scan]   # parallel deps resolve first
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp:${{ github.sha }} .
      - run: docker push myapp:${{ github.sha }}

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - run: kubectl set image deployment/api app=myapp:${{ github.sha }}

  e2e-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:e2e -- --base-url=https://staging.mycompany.com

  deploy-production:
    needs: e2e-tests
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: kubectl set image deployment/api app=myapp:${{ github.sha }}
```

**Use path filters to skip unnecessary work:**

```yaml
# Only run tests when source code changes — not for docs or config
on:
  push:
    paths:
      - "src/**"
      - "tests/**"
      - "package.json"
      - "package-lock.json"
    paths-ignore:
      - "docs/**"
      - "*.md"
      - ".github/ISSUE_TEMPLATE/**"
```

**Pipeline time comparison:**

```
Sequential (bad):
  lint → unit → integration → e2e → scan → build → push → deploy
  [======================================================] 60 min

Parallel (good):
  lint ────────┐
  unit ────────┤
  integration ─┤→ build → push → deploy-staging → e2e → deploy-prod
  scan ────────┘
  [=========================] 28 min (53% faster)
```

### Quick Check

- [ ] Pipeline jobs run in parallel where there are no dependencies between them
- [ ] Path filters skip the pipeline for documentation-only or config-only changes
- [ ] Lint, unit tests, and security scans run concurrently
- [ ] Total pipeline time for a typical code change is under 15 minutes
- [ ] Pipeline stages have clear dependency graphs — no unnecessary sequencing

---

## 8. No Environment Parity

### Problem

When development, staging, and production environments differ in infrastructure, configuration, operating system, runtime versions, or backing services, bugs that appear in production cannot be reproduced in lower environments. "Works on my machine" becomes "works in staging, breaks in production," and the team spends hours debugging environment-specific issues instead of application logic.

### Example of the Bad Pattern

```
Development:
  - macOS with Node.js 18 installed via Homebrew
  - SQLite for the database
  - Files served from localhost:3000
  - No SSL, no load balancer, no CDN

Staging:
  - Ubuntu 22.04 with Node.js 20 (different major version)
  - MySQL 8.0 (different database engine entirely)
  - Single server behind nginx
  - Self-signed SSL certificate

Production:
  - Amazon Linux 2023 with Node.js 20.11.1 (specific patch version)
  - Aurora PostgreSQL 15.4 (yet another database engine)
  - Auto-scaling group behind ALB with WAF
  - ACM-managed TLS certificate, CloudFront CDN
```

**What happens:**

```
Developer:  "All tests pass on my machine"          (macOS + SQLite)
Staging:    "All tests pass in staging"              (Ubuntu + MySQL)
Production: "TypeError: column 'created_at' does     (Amazon Linux + PostgreSQL)
             not exist"

Root cause: SQLite auto-creates columns on INSERT.
            MySQL silently truncates long strings.
            PostgreSQL enforces strict schema validation.

Three different database engines = three different behaviours.
The bug only manifests in the production database.
```

### Why It's Harmful

- Bugs are discovered in production instead of during development or staging testing
- "Works in staging" provides false confidence — the environments are fundamentally different
- Debugging production-only issues requires production access, increasing security risk
- Infrastructure differences mask application bugs and vice versa
- Cost of fixing bugs increases 10–100× when discovered in production
- Team velocity drops as engineers spend time on environment-specific debugging

### Recommended Fix

```yaml
# Good: identical infrastructure defined as code for all environments
# docker-compose.yml — same services, same versions, every environment
services:
  api:
    image: mycompany/api:${IMAGE_TAG}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}

  postgres:
    image: postgres:16.2-alpine    # same version everywhere
    environment:
      - POSTGRES_DB=myapp

  redis:
    image: redis:7.2.4-alpine      # same version everywhere
```

```hcl
# Terraform modules — same infrastructure, different sizes
# modules/api-service/main.tf
module "api" {
  source = "../modules/api-service"

  environment    = var.environment          # dev, staging, production
  instance_type  = var.instance_type        # t3.small, t3.medium, t3.large
  min_replicas   = var.min_replicas         # 1, 2, 4
  max_replicas   = var.max_replicas         # 2, 4, 20

  # These are IDENTICAL across all environments:
  node_version   = "20.11.1"
  postgres_version = "16.2"
  redis_version  = "7.2.4"
  os_image       = "ubuntu-22.04"
}
```

**Enforce parity in CI:**

```yaml
# CI pipeline uses the same Docker image that runs in production
test:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:16.2-alpine    # same as production
      env:
        POSTGRES_DB: myapp_test
        POSTGRES_PASSWORD: test
    redis:
      image: redis:7.2.4-alpine      # same as production

  container:
    image: node:20.11.1-slim         # same Node.js version as production

  steps:
    - uses: actions/checkout@v4
    - run: npm ci
    - run: npm test
```

### Quick Check

- [ ] All environments use the same OS, runtime version, and backing service versions
- [ ] Infrastructure is defined as code (Terraform, Pulumi, CloudFormation) with shared modules
- [ ] CI tests run against the same database engine and version as production
- [ ] Environment differences are limited to scale (instance size, replica count) and secrets
- [ ] A parity audit is performed quarterly — any drift is treated as a bug

---

## 9. Building Different Artifacts Per Environment

### Problem

Building a separate artifact (binary, Docker image, package) for each environment instead of following "build once, deploy many" means the artifact tested in staging is not the same artifact deployed to production. A different build may include different dependencies, different compiler optimisations, or different bundled assets — introducing subtle, impossible-to-reproduce bugs.

### Example of the Bad Pattern

```yaml
# Bad: separate build per environment
jobs:
  build-staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          npm ci
          NODE_ENV=staging npm run build    # different build configuration
          docker build --build-arg ENV=staging -t myapp:staging .
          docker push myapp:staging

  build-production:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          npm ci
          NODE_ENV=production npm run build  # different build configuration
          docker build --build-arg ENV=production -t myapp:production .
          docker push myapp:production
```

```dockerfile
# Bad: different build output per environment
FROM node:20 AS builder
ARG ENV=production
WORKDIR /app
COPY . .
RUN NODE_ENV=${ENV} npm run build
# staging build includes source maps and debug logging
# production build has different minification and tree-shaking
# They are NOT the same artifact
```

**What happens:**

```
Staging build:   NODE_ENV=staging  → includes source maps, debug endpoints, verbose logging
Production build: NODE_ENV=production → minified, no source maps, no debug endpoints

Tests pass in staging against the staging build.
Production build has a different code path due to tree-shaking.
A function used in production was shaken out of the bundle.

Result: "TypeError: processPayment is not a function" — only in production.
The staging artifact never had this bug because it was a different build.
```

### Why It's Harmful

- The artifact tested in staging is literally a different binary than what runs in production
- Build tools may produce different output depending on environment variables
- Tree-shaking, minification, and dead code elimination behave differently per build
- Debugging production issues is impossible if the staging artifact differs
- Build reproducibility is lost — two builds from the same commit produce different results
- Compliance audits require proof that the tested artifact is the deployed artifact

### Recommended Fix

```yaml
# Good: build once, deploy to all environments
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - run: |
          docker build -t mycompany/api:${{ github.sha }} .
          docker push mycompany/api:${{ github.sha }}
          # One image — used everywhere

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: |
          kubectl set image deployment/api \
            app=mycompany/api:${{ github.sha }}    # same image as production

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: |
          kubectl set image deployment/api \
            app=mycompany/api:${{ github.sha }}    # exact same image
```

**Inject environment-specific configuration at runtime, not build time:**

```yaml
# Kubernetes ConfigMap — environment config separate from the artifact
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  LOG_LEVEL: "warn"
  FEATURE_NEW_CHECKOUT: "true"
  API_BASE_URL: "https://api.mycompany.com"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
        - name: api
          image: mycompany/api:abc123    # same image in every environment
          envFrom:
            - configMapRef:
                name: api-config         # environment config injected at runtime
```

**The build-once-deploy-many principle:**

```
Build:    mycompany/api:abc123   ← built once from commit abc123

Deploy:
  Dev:        mycompany/api:abc123 + dev config
  Staging:    mycompany/api:abc123 + staging config
  Production: mycompany/api:abc123 + production config

Same bytes. Same binary. Same image digest. Different config.
```

### Quick Check

- [ ] The CI pipeline builds exactly one artifact per commit
- [ ] The same image tag or digest is deployed to staging and production
- [ ] No `--build-arg ENV=` or `NODE_ENV=` at build time that changes the output
- [ ] Environment-specific configuration is injected at runtime via env vars or config maps
- [ ] Image digests are compared between staging and production to verify they match

---

## 10. Insufficient Pipeline Observability

### Problem

Running CI/CD pipelines without metrics, dashboards, or alerting means that failures, slowdowns, and reliability degradation go unnoticed until they cause a visible problem. Teams have no idea how long their pipelines take, what the failure rate is, or which step is the bottleneck — they only notice when a build has been red for three days or a deploy has been stuck for hours.

### Example of the Bad Pattern

```yaml
# Bad: pipeline runs, results go into a black hole
name: CI
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      - run: npm run build
      # No timing metrics
      # No failure notifications
      # No success rate tracking
      # No bottleneck identification
```

**What happens:**

```
Week 1:   Pipeline takes 8 minutes — nobody tracks this
Week 4:   Pipeline takes 12 minutes — nobody notices
Week 8:   Pipeline takes 22 minutes — a developer complains in Slack
Week 10:  Pipeline takes 35 minutes — "it's always been slow"
Week 12:  Pipeline failure rate is 40% — "tests are just flaky"

Nobody can answer:
  - What is our average build time?
  - What is our deployment frequency?
  - What percentage of builds fail?
  - Which step is the slowest?
  - How long does it take to recover from a failed deployment?
```

### Why It's Harmful

- Pipeline degradation is invisible until it becomes a crisis
- Cannot identify bottlenecks without timing data for each step
- Failure trends are missed — a gradually increasing failure rate goes unnoticed
- DORA metrics (deployment frequency, lead time, MTTR, change failure rate) cannot be measured
- No alerting means failed deployments can go unnoticed for hours
- Teams cannot demonstrate CI/CD improvement over time without historical data

### Recommended Fix

```yaml
# Good: pipeline with metrics, notifications, and observability
name: CI/CD
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Record pipeline start
        run: echo "PIPELINE_START=$(date +%s)" >> "$GITHUB_ENV"

      - name: Install dependencies
        run: |
          START=$(date +%s)
          npm ci
          DURATION=$(($(date +%s) - START))
          echo "::notice::npm ci completed in ${DURATION}s"

      - name: Run tests
        run: |
          START=$(date +%s)
          npm test
          DURATION=$(($(date +%s) - START))
          echo "::notice::Tests completed in ${DURATION}s"

      - name: Report metrics
        if: always()
        run: |
          TOTAL=$(($(date +%s) - PIPELINE_START))
          echo "Total pipeline duration: ${TOTAL}s"
          # Send to your metrics platform (Datadog, Prometheus, etc.)

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: "deploys"
          slack-message: |
            ❌ Pipeline failed for `${{ github.ref_name }}`
            Commit: ${{ github.sha }}
            Author: ${{ github.actor }}
            Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

**Track DORA metrics:**

```
┌─────────────────────────────────────────────────────────────┐
│ DORA Metrics Dashboard                                       │
├─────────────────────────────┬───────────┬───────────────────┤
│ Metric                      │ Current   │ Target            │
├─────────────────────────────┼───────────┼───────────────────┤
│ Deployment Frequency         │ 3/week    │ Daily             │
│ Lead Time for Changes        │ 4 days    │ < 1 day           │
│ Mean Time to Recovery (MTTR) │ 2 hours   │ < 1 hour          │
│ Change Failure Rate          │ 22%       │ < 15%             │
├─────────────────────────────┼───────────┼───────────────────┤
│ Pipeline Success Rate        │ 78%       │ > 95%             │
│ Average Build Time           │ 18 min    │ < 10 min          │
│ Flaky Test Count             │ 12        │ 0                 │
└─────────────────────────────┴───────────┴───────────────────┘
```

### Quick Check

- [ ] Pipeline failures send notifications to the team (Slack, email, PagerDuty)
- [ ] Build duration, success rate, and failure rate are tracked over time
- [ ] DORA metrics are measured and reviewed monthly
- [ ] A dashboard visualises pipeline health, bottlenecks, and trends
- [ ] Alerts fire when pipeline duration exceeds a defined threshold

---

## 11. Over-Reliance on Manual Approval Gates

### Problem

Requiring manual human approval for every deployment — even routine changes to non-critical services — creates a bottleneck that slows delivery to a crawl. When a single manager or change advisory board (CAB) must approve every release, deployments queue up, developers context-switch while waiting, and the organisation optimises for fewer, larger (and riskier) releases instead of frequent, small, safe ones.

### Example of the Bad Pattern

```yaml
# Bad: every deployment requires manual approval, even for non-critical services
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      # url: https://staging.mycompany.com
    steps:
      - run: kubectl apply -f k8s/staging/

  # Manual gate — someone must click "Approve" in the GitHub UI
  approve-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production-approval    # requires manual approval

  deploy-production:
    needs: approve-production
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - run: kubectl apply -f k8s/production/
```

**What happens:**

```
Monday 09:00 — Developer A pushes a bug fix. Pipeline reaches approval gate.
Monday 09:15 — Developer A requests approval in Slack.
Monday 11:00 — Approver is in meetings. No response.
Monday 14:00 — Developer B pushes a feature. Queued behind Developer A.
Monday 16:30 — Approver returns. Approves Developer A's change.
Monday 16:35 — Developer A's fix deploys — 7.5 hours after commit.
Monday 16:40 — Developer B's change reaches approval gate.
Tuesday 09:00 — Approver sees the request. Approves.
Tuesday 09:05 — Developer B's feature deploys — 19 hours after commit.

Both changes were low-risk, fully tested, and passed all automated checks.
The only bottleneck was a human clicking "Approve."
```

### Why It's Harmful

- Deployments queue behind approval gates, increasing lead time from hours to days
- Approvers become bottlenecks — their calendar controls the entire organisation's deployment cadence
- Developers batch changes to reduce approval requests, creating larger and riskier releases
- Context switching — developers must wait, move to another task, then return when approved
- The approval itself adds no safety if the approver rubber-stamps without reviewing
- Compliance requirements are better met by automated evidence (tests, scans) than by a human checkbox

### Recommended Fix

```yaml
# Good: risk-based deployment strategy
# Low risk (internal tools, docs, non-critical services) → auto-deploy
# Medium risk (backend APIs with automated tests) → progressive rollout
# High risk (payment systems, auth, data migrations) → manual approval

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      # Manual approval only for high-risk services
    steps:
      - name: Progressive rollout (no manual gate needed)
        run: |
          # Deploy to 5% of traffic first
          kubectl set image deployment/api app=mycompany/api:${{ github.sha }}
          kubectl rollout status deployment/api --timeout=120s

          # Monitor error rate for 5 minutes
          sleep 300
          ERROR_RATE=$(curl -s "https://metrics.mycompany.com/api/error-rate?last=5m")
          if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
            echo "Error rate too high ($ERROR_RATE%) — rolling back"
            kubectl rollout undo deployment/api
            exit 1
          fi

          # Promote to 100% of traffic
          echo "Error rate nominal ($ERROR_RATE%) — promoting to full rollout"
```

**Use automated quality gates instead of manual approval:**

```yaml
# Automated gates that replace manual approval
quality-gate:
  runs-on: ubuntu-latest
  steps:
    - name: Verify test coverage
      run: |
        COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
        if (( $(echo "$COVERAGE < 80" | bc -l) )); then
          echo "Coverage $COVERAGE% is below 80% threshold"
          exit 1
        fi

    - name: Verify no critical vulnerabilities
      run: npm audit --audit-level=critical

    - name: Verify performance regression
      run: |
        # Compare against baseline — automated, no human needed
        npm run test:performance -- --threshold=p95:500ms
```

**When manual approval is appropriate:**

```
Auto-deploy (no gate):     Documentation, internal tools, config changes
Progressive rollout:       Standard backend and frontend services
Manual approval required:  Payment processing, authentication, database migrations,
                           changes affecting >10,000 users, infrastructure changes
```

### Quick Check

- [ ] Manual approval gates are reserved for high-risk changes only
- [ ] Low-risk and medium-risk deployments use automated quality gates
- [ ] Progressive rollout (canary, blue-green) replaces manual approval for most services
- [ ] Approval requests include automated evidence (test results, scan reports, coverage)
- [ ] Time-to-approve is tracked — alert if approvals consistently take more than 1 hour

---

## 12. Not Versioning Pipeline Configuration

### Problem

Maintaining pipeline configuration through a CI/CD tool's web UI — clicking through forms, editing inline scripts, and configuring triggers in a dashboard — means the pipeline has no version history, no code review, no audit trail, and no way to reproduce or roll back changes. When a UI edit breaks the pipeline, there is no diff to inspect and no previous version to restore.

### Example of the Bad Pattern

```
Jenkins UI:
  Dashboard → My Project → Configure
  → Build Triggers: [✓] Poll SCM: H/5 * * * *
  → Build Steps:
      Execute Shell:
        #!/bin/bash
        npm install
        npm test
        npm run build
        scp -r dist/ deploy@prod:/var/www/app/
  → Post-build Actions:
      E-mail Notification: ops@mycompany.com

Changes made by: someone (no audit log)
Last modified: unknown
Previous version: gone forever
```

**What happens:**

```
Tuesday 14:00 — Engineer edits the Jenkins job to add a new build step via the UI
Tuesday 14:05 — Typo in the shell script breaks the pipeline
Tuesday 14:10 — All builds fail — team cannot deploy
Tuesday 14:20 — "What changed?" — no diff, no history, no audit log
Tuesday 14:30 — Engineer tries to remember the old configuration
Tuesday 15:00 — Pipeline restored from memory — but is it exactly what it was before?
Tuesday 15:30 — Subtle difference causes a staging deploy to skip tests for 2 weeks
```

### Why It's Harmful

- No version history — cannot see what changed, when, or by whom
- No code review — pipeline changes bypass the team's review process
- No rollback — if a change breaks the pipeline, the previous version is gone
- Configuration drift — the "source of truth" lives in a UI that cannot be diffed
- Cannot reproduce the pipeline on a new CI server without manual reconfiguration
- Disaster recovery is impossible — if the CI server dies, all pipeline configuration is lost
- Compliance audits require auditable change history for deployment processes

### Recommended Fix

```yaml
# Good: pipeline configuration lives in the repository alongside the code
# .github/workflows/ci.yml — version-controlled, reviewed, auditable
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: |
          docker build -t myapp:${{ github.sha }} .
          docker push myapp:${{ github.sha }}
          kubectl set image deployment/api app=myapp:${{ github.sha }}
```

**Protect pipeline configuration with the same rigor as application code:**

```yaml
# CODEOWNERS — require review from the platform team for pipeline changes
# .github/CODEOWNERS
.github/workflows/    @mycompany/platform-team
Jenkinsfile           @mycompany/platform-team
.gitlab-ci.yml        @mycompany/platform-team
```

```yaml
# Branch protection — prevent unreviewed changes to pipeline config
# Repository Settings → Branches → Branch protection rules → main
# [✓] Require a pull request before merging
# [✓] Require approvals: 1
# [✓] Require review from Code Owners
# [✓] Include administrators
```

**Migrate from UI-based to code-based pipelines:**

```bash
# Step 1: Export current Jenkins config
$ java -jar jenkins-cli.jar -s http://jenkins:8080 get-job my-project > Jenkinsfile.xml

# Step 2: Convert to a Jenkinsfile (pipeline-as-code)
# Step 3: Commit the Jenkinsfile to the repository
# Step 4: Configure Jenkins to use "Pipeline from SCM"
# Step 5: Remove the old UI-configured job
```

### Quick Check

- [ ] All pipeline configuration is stored in version-controlled files (not UI-only)
- [ ] Pipeline changes go through the same code review process as application code
- [ ] CODEOWNERS file requires platform team approval for pipeline file changes
- [ ] Pipeline configuration can be fully reproduced from the repository alone
- [ ] CI server can be rebuilt from scratch without manual configuration

---

## Quick Reference Checklist

Use this checklist before deploying any new pipeline or service to production.

### Pipeline Design

- [ ] All pipelines use shared templates or reusable workflows — no snowflakes
- [ ] Pipeline jobs run in parallel where there are no dependencies
- [ ] Path filters skip unnecessary work for documentation and config changes
- [ ] Total pipeline time for a typical code change is under 15 minutes
- [ ] Pipeline configuration is version-controlled and code-reviewed

### Integration and Branching

- [ ] Feature branches live for less than 2 days on average
- [ ] Pull requests change fewer than 400 lines of code
- [ ] Developers integrate with main at least once per day
- [ ] Feature flags are used for incomplete features — not long-lived branches
- [ ] Stale branches older than 7 days trigger automated notifications

### Deployment Safety

- [ ] All production deployments go through the CI/CD pipeline — no manual deploys
- [ ] Every deployment records the previous version for instant rollback
- [ ] Rollback is automated and tested regularly
- [ ] Database migrations are backward-compatible (additive only)
- [ ] Progressive rollout (canary, blue-green) is used for critical services

### Security

- [ ] No secrets, tokens, or passwords are committed to the repository
- [ ] All secrets are stored in the CI platform's encrypted secret store
- [ ] OIDC federation is used instead of static cloud credentials where possible
- [ ] A secret scanning tool runs in CI on every push
- [ ] CI logs do not contain any secret values

### Testing and Quality

- [ ] Flaky tests are quarantined — not skipped or disabled
- [ ] Flaky tests have an assigned owner and a fix deadline
- [ ] Test coverage is tracked and enforced by automated quality gates
- [ ] Automated quality gates replace manual approval for low-risk changes
- [ ] Manual approval is reserved for high-risk changes only

### Artifacts and Environments

- [ ] One artifact is built per commit — deployed to all environments
- [ ] The same image tag or digest is used in staging and production
- [ ] Environment-specific configuration is injected at runtime, not build time
- [ ] All environments use the same OS, runtime, and backing service versions
- [ ] Environment parity is audited quarterly — drift is treated as a bug

### Observability and Operations

- [ ] Pipeline failures send notifications to the team
- [ ] Build duration, success rate, and failure rate are tracked over time
- [ ] DORA metrics are measured and reviewed monthly
- [ ] Alerts fire when pipeline duration exceeds a defined threshold
- [ ] Pipeline configuration changes are auditable and code-reviewed

---

## Next Steps

1. **Score yourself** — Use the [Quick Reference Checklist](#quick-reference-checklist) and score your current pipelines. Any unchecked item is a potential anti-pattern.
2. **Fix critical severity first** — Address 🔴 Critical anti-patterns (#1–#5) before high and medium ones.
3. **Read the companion guide** — [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) describes the correct patterns to replace these anti-patterns.
4. **Secure your pipelines** — [08-SECURITY-IN-PIPELINES.md](08-SECURITY-IN-PIPELINES.md) covers pipeline security in depth, including secret management and supply chain hardening.
5. **Optimise pipeline testing** — [07-TESTING-IN-PIPELINES.md](07-TESTING-IN-PIPELINES.md) covers test strategies, parallelism, and quality gates.
6. **Follow the learning path** — [LEARNING-PATH.md](LEARNING-PATH.md) provides a structured curriculum through all CI/CD topics.

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-01-01 | Initial document — 12 anti-patterns with examples and fixes | Platform Team |
