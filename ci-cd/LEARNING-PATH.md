# CI/CD Learning Path

A structured, self-paced training guide to mastering Continuous Integration, Continuous Delivery, and Continuous Deployment — from pipeline fundamentals and build automation to platform deep dives, GitOps, security scanning, and production-grade deployment strategies. Each phase builds on the previous one, progressing from core concepts to advanced production patterns.

> **Time Estimate:** 8–10 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior build automation or scripting experience may complete Phase 1 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how pipeline concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all six phases and produces a portfolio artifact you can reference in engineering conversations

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand the core concepts of Continuous Integration, Continuous Delivery, and Continuous Deployment
- Learn pipeline anatomy: stages, jobs, steps, triggers, and artifacts
- Grasp the build-test-deploy lifecycle and how feedback loops drive quality
- Understand branching strategies and their impact on CI/CD pipeline design
- Master the CI feedback loop: commit, build, test, report

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | CI/CD fundamentals, pipeline anatomy, execution models, environments, branching strategies, artifact management |
| 2 | [01-CONTINUOUS-INTEGRATION](01-CONTINUOUS-INTEGRATION.md) | CI feedback loop, build automation, source code integration, automated testing, code quality gates |

### Exercises

**1. CI/CD Concept Mapping:**

Choose a project you are familiar with — a personal project, a work service, or an open-source repository — and map out how code changes currently flow from commit to production:

- Draw a diagram showing every step between a developer committing code and the change reaching production
- For each step, identify whether it is manual or automated
- Identify the feedback loops: how long does it take a developer to know their change broke something?
- List the environments the code passes through (development, staging, production)
- Estimate the total lead time from commit to production deployment
- Identify three bottlenecks in the current process that CI/CD could eliminate

Document your findings. You will revisit this analysis after Phase 6 and compare it with your capstone pipeline.

**2. Pipeline Anatomy Exploration:**

Examine the following conceptual pipeline definition and identify every component:

```yaml
# Conceptual CI/CD pipeline — identify each component
trigger:
  branches:
    - main
    - release/*

stages:
  - stage: build
    jobs:
      - job: compile
        steps:
          - checkout: self
          - run: npm install
          - run: npm run build
        artifacts:
          - path: dist/

  - stage: test
    depends_on: build
    jobs:
      - job: unit-tests
        steps:
          - run: npm run test:unit -- --coverage
      - job: integration-tests
        steps:
          - run: npm run test:integration

  - stage: deploy-staging
    depends_on: test
    environment: staging
    jobs:
      - job: deploy
        steps:
          - run: ./scripts/deploy.sh staging

  - stage: deploy-production
    depends_on: deploy-staging
    environment: production
    approval: manual
    jobs:
      - job: deploy
        steps:
          - run: ./scripts/deploy.sh production
```

After reviewing, answer:
- What triggers this pipeline, and what branches are included?
- How many stages does this pipeline have, and what is the dependency order?
- Which jobs can run in parallel within the test stage?
- What is the difference between the staging and production deploy stages?
- What is the purpose of the `approval: manual` gate?
- What are artifacts, and why does the build stage produce them?

**3. CI Feedback Loop Timing Exercise:**

Set up a basic CI pipeline for a simple application using any platform you have access to (GitHub Actions, Azure DevOps, Jenkins, or even a local shell script). The pipeline must:

- Trigger on every push to the `main` branch
- Check out the source code
- Install dependencies
- Run a linter
- Run unit tests
- Report pass/fail status

Use a minimal project structure:

```bash
# Create a sample project
mkdir ci-demo && cd ci-demo
git init

# Create a simple Node.js project (or use your preferred language)
cat > package.json << 'EOF'
{
  "name": "ci-demo",
  "version": "1.0.0",
  "scripts": {
    "lint": "echo 'Linting passed'",
    "test": "echo 'All tests passed'"
  }
}
EOF

cat > index.js << 'EOF'
function add(a, b) {
  return a + b;
}
module.exports = { add };
EOF
```

After the pipeline runs successfully:
- Measure the total pipeline execution time
- Identify which step takes the longest
- Consider how you would reduce total pipeline time as the project grows

### Knowledge Check

- [ ] What is the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment?
- [ ] What are the four stages of the CI feedback loop (commit, build, test, report)?
- [ ] What is a pipeline artifact, and why is it important to pass artifacts between stages?
- [ ] How do branching strategies (trunk-based, GitFlow, GitHub Flow) affect pipeline design?

---

## Phase 2: Delivery & Deployment (Week 3–4)

### Learning Objectives

- Understand the difference between Continuous Delivery and Continuous Deployment
- Learn deployment pipeline design: multi-stage pipelines, promotion between environments, pipeline as code
- Master release strategies: blue-green, canary, rolling updates, and feature flags
- Implement rollback strategies and understand when each applies
- Configure environment management: standard tiers, environment parity, and ephemeral environments

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-CONTINUOUS-DELIVERY](02-CONTINUOUS-DELIVERY.md) | Delivery vs deployment, multi-stage pipelines, release strategies, rollback, environment management, feature flags |

### Exercises

**1. Release Strategy Comparison:**

For each of the following release strategies, create a one-page summary documenting how it works, when to use it, and the risks involved:

| Strategy | Description |
|----------|-------------|
| Blue-Green | Two identical environments; switch traffic from blue to green after validation |
| Canary | Route a small percentage of traffic to the new version; gradually increase |
| Rolling Update | Replace instances one at a time; no separate environment needed |
| Feature Flags | Deploy code to production but hide features behind runtime toggles |

For each strategy, answer:
- What infrastructure is required?
- How fast is the rollback?
- What monitoring is needed to detect problems?
- What is the blast radius if the deployment fails?
- In what scenario would you choose this strategy over the others?

**2. Multi-Stage Pipeline Design:**

Design a deployment pipeline on paper (or in YAML pseudocode) for a web application with the following requirements:

```
Requirements:
- Source: monorepo with frontend (React) and backend (Python API)
- Environments: dev, staging, production
- Dev deploys automatically on every merge to main
- Staging deploys automatically after dev succeeds and all integration tests pass
- Production requires manual approval after staging has been stable for 1 hour
- Rollback must be possible within 5 minutes
- Frontend and backend can be deployed independently
```

Your design should include:
- A diagram showing the pipeline stages and their dependencies
- The trigger conditions for each stage
- Where artifacts are built and how they are promoted between environments
- The approval gate configuration for production
- The rollback mechanism (redeploy previous version? revert git commit? blue-green switch?)

**3. Environment Parity Audit:**

Examine a real or hypothetical application and document the differences between environments:

```
Environment Parity Checklist:
  [ ] Same operating system and version across all environments
  [ ] Same runtime version (Node.js, Python, JDK) across all environments
  [ ] Same database engine and version (not SQLite in dev, PostgreSQL in prod)
  [ ] Same container images promoted between environments (build once, deploy everywhere)
  [ ] Same configuration mechanism (environment variables, not hardcoded values)
  [ ] Same network topology (or as close as possible)
  [ ] Same secret management approach (not plaintext in dev, vault in prod)
  [ ] Same monitoring and logging setup (or as close as feasible)
```

For each item that is not in parity, write a remediation plan: what to change, estimated effort, and priority.

**4. Rollback Simulation:**

Simulate a failed deployment and practice rollback:

```bash
# Scenario: you deployed v2.0.0 to production, but error rates spiked

# Option A: Revert to previous version
# If using container images:
#   Re-tag the last known good image and redeploy
#   docker tag myapp:1.9.0 myapp:rollback
#   deploy myapp:rollback to production

# Option B: Git revert
#   git revert <bad-commit-sha>
#   Push the revert — pipeline automatically deploys the fix

# Option C: Blue-green switch
#   Switch the load balancer back to the blue environment
#   curl -X POST https://lb.example.com/switch?target=blue
```

Document the time it takes for each rollback method and identify which method your team should standardise on.

### Knowledge Check

- [ ] What is the key difference between Continuous Delivery and Continuous Deployment (manual gate vs automatic)?
- [ ] What is a blue-green deployment, and how does it enable instant rollback?
- [ ] What is environment parity, and why does it matter for deployment confidence?
- [ ] What is the "build once, deploy everywhere" principle, and how does it relate to artifact promotion?

---

## Phase 3: Platform Deep Dives (Week 5–6)

### Learning Objectives

- Understand GitHub Actions: workflows, events, jobs, steps, runners, and the Actions marketplace
- Learn Azure DevOps Pipelines: YAML syntax, stages, variables, service connections, and template reuse
- Understand Jenkins: Declarative and Scripted pipelines, Jenkinsfile, agents, shared libraries, and plugin ecosystem
- Compare platform capabilities and identify when to use each
- Build working pipelines in at least one platform

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [03-GITHUB-ACTIONS](03-GITHUB-ACTIONS.md) | Workflows, events, jobs, steps, actions, runners, secrets, matrix builds, reusable workflows |
| 2 | [04-AZURE-DEVOPS](04-AZURE-DEVOPS.md) | YAML pipelines, stages, jobs, triggers, variables, service connections, templates, environments |
| 3 | [05-JENKINS](05-JENKINS.md) | Declarative and Scripted pipelines, Jenkinsfile, agents, shared libraries, plugins |

### Exercises

**1. GitHub Actions — Build and Test Workflow:**

Create a GitHub Actions workflow that builds and tests a project:

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run test -- --coverage
      - uses: actions/upload-artifact@v4
        if: matrix.node-version == 20
        with:
          name: coverage-report
          path: coverage/

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
```

After creating the workflow:
- Push it to a repository and observe the Actions tab
- Verify that the matrix strategy runs tests across three Node.js versions
- Confirm that the build job waits for all test matrix jobs to complete
- Download the coverage report artifact and review it

Answer:
- What does `needs: lint` do, and how does it affect the job execution order?
- What is a matrix strategy, and when would you use one?
- What is the difference between `actions/checkout@v4` and checking out code manually?
- Why should you use `npm ci` instead of `npm install` in CI pipelines?

**2. Azure DevOps — Multi-Stage Pipeline:**

Write an Azure DevOps YAML pipeline with build, test, and deploy stages:

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: my-app-variables
  - name: buildConfiguration
    value: 'Release'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildJob
        displayName: 'Build Application'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '8.x'
          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: 'restore'
          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              arguments: '--configuration $(buildConfiguration) --no-restore'
          - task: DotNetCoreCLI@2
            displayName: 'Test'
            inputs:
              command: 'test'
              arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'
          - task: PublishBuildArtifacts@1
            displayName: 'Publish Artifacts'
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'drop'

  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployStaging
        displayName: 'Deploy to Staging'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo 'Deploying to staging...'

  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployProduction
        displayName: 'Deploy to Production'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo 'Deploying to production...'
```

After reviewing the pipeline:
- Identify how stages depend on each other using `dependsOn`
- Explain the difference between `jobs` and `deployment` jobs
- Describe how `environment` enables approval gates in Azure DevOps
- Compare variable groups with inline variables

**3. Jenkins — Declarative Pipeline:**

Write a Jenkinsfile that mirrors the GitHub Actions workflow from Exercise 1:

```groovy
// Jenkinsfile
pipeline {
    agent any

    tools {
        nodejs 'NodeJS-20'
    }

    environment {
        CI = 'true'
        HOME = '.'
    }

    stages {
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Test') {
            steps {
                sh 'npm run test -- --coverage'
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                    publishHTML(target: [
                        reportDir: 'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/**', fingerprint: true
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                sh './scripts/deploy.sh staging'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message 'Deploy to production?'
                ok 'Deploy'
            }
            steps {
                sh './scripts/deploy.sh production'
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed — notify the team'
        }
        cleanup {
            cleanWs()
        }
    }
}
```

After reviewing the Jenkinsfile:
- Identify the equivalent of GitHub Actions `needs` (sequential stages in Declarative)
- Compare `input` in Jenkins with manual approval in Azure DevOps
- Explain the `post` block and when `always`, `success`, and `failure` run
- Describe the difference between `agent any` and `agent { docker { image '...' } }`

**4. Platform Comparison Table:**

Complete the following comparison table for all three platforms:

```
| Feature                | GitHub Actions     | Azure DevOps       | Jenkins            |
|------------------------|--------------------|--------------------|---------------------|
| Configuration format   |                    |                    |                     |
| Runner/agent model     |                    |                    |                     |
| Secret management      |                    |                    |                     |
| Matrix builds          |                    |                    |                     |
| Manual approvals       |                    |                    |                     |
| Reusable pipelines     |                    |                    |                     |
| Marketplace/plugins    |                    |                    |                     |
| Self-hosted option     |                    |                    |                     |
| RBAC granularity       |                    |                    |                     |
| Cost model             |                    |                    |                     |
```

### Knowledge Check

- [ ] What is the difference between a GitHub Actions workflow, job, and step?
- [ ] How does Azure DevOps `dependsOn` differ from GitHub Actions `needs`?
- [ ] What is a Jenkins shared library, and how does it enable pipeline reuse?
- [ ] When would you choose self-hosted runners/agents over cloud-hosted ones?

---

## Phase 4: GitOps & Testing (Week 6–7)

### Learning Objectives

- Understand GitOps principles: declarative configuration, Git as single source of truth, reconciliation loops
- Compare push-based and pull-based deployment models
- Learn ArgoCD and Flux fundamentals for Kubernetes-native GitOps
- Master the testing pyramid in CI/CD: unit, integration, end-to-end, contract, and performance tests
- Implement quality gates, test parallelisation, and flaky test management in pipelines

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-GITOPS](06-GITOPS.md) | GitOps principles, push vs pull models, ArgoCD, Flux, repository strategies, secrets in GitOps |
| 2 | [07-TESTING-IN-PIPELINES](07-TESTING-IN-PIPELINES.md) | Testing pyramid, unit/integration/E2E tests, quality gates, test parallelisation, flaky test management |

### Exercises

**1. GitOps Repository Structure:**

Design a GitOps repository structure for a microservices application with three services deployed to two environments (staging and production):

```
# Option A: Monorepo
gitops-config/
├── base/
│   ├── service-a/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── service-b/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── service-c/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
├── overlays/
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   │       ├── replicas.yaml
│   │       └── resources.yaml
│   └── production/
│       ├── kustomization.yaml
│       └── patches/
│           ├── replicas.yaml
│           └── resources.yaml
└── README.md
```

For your chosen structure, answer:
- How does a new image version get promoted from staging to production?
- How do you handle configuration differences between environments (replica count, resource limits, feature flags)?
- How do you roll back a deployment — git revert, or manual image tag change?
- What branch strategy do you use for the config repository (single branch with directory-per-env, or branch-per-env)?

**2. ArgoCD Application Definition:**

Write an ArgoCD Application manifest that deploys a service from a GitOps repository:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: service-a-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-config.git
    targetRevision: main
    path: overlays/staging/service-a
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

After reviewing:
- Explain what `selfHeal: true` does and why it is important for GitOps
- Describe what `prune: true` does when a resource is removed from Git
- What happens if someone manually edits a Kubernetes resource that ArgoCD manages?
- How would you configure a production application that requires manual sync (no `automated` block)?

**3. Testing Pyramid Implementation:**

Implement a testing strategy for a CI pipeline with proper stage ordering:

```yaml
# Testing pipeline — stages ordered by the testing pyramid
stages:
  # Layer 1: Fast feedback (seconds)
  - stage: unit-tests
    jobs:
      - run: npm run test:unit -- --coverage --reporter=junit
      - quality-gate: coverage >= 80%

  # Layer 2: Integration tests (minutes)
  - stage: integration-tests
    depends_on: unit-tests
    services:
      - postgres:16-alpine
      - redis:7.2-alpine
    jobs:
      - run: npm run test:integration

  # Layer 3: Contract tests (minutes)
  - stage: contract-tests
    depends_on: unit-tests
    jobs:
      - run: npm run test:contracts
      - publish: pact-broker

  # Layer 4: End-to-end tests (minutes to hours)
  - stage: e2e-tests
    depends_on: [integration-tests, contract-tests]
    jobs:
      - run: npx playwright test
      - artifacts: playwright-report/
```

Create a plan for each test layer:
- **Unit tests:** What percentage of code coverage is your quality gate? How do you fail the pipeline if coverage drops?
- **Integration tests:** What external services do tests need (databases, queues, caches)? How are they provisioned in the pipeline?
- **Contract tests:** If you have multiple services, how do you verify API compatibility? What tool would you use (Pact, Spectral, openapi-diff)?
- **End-to-end tests:** How do you manage flaky tests? Do you retry failures? How many retries before flagging a test as flaky?

**4. Flaky Test Quarantine Process:**

Design a process for managing flaky tests in your pipeline:

```
Flaky Test Management Process:
1. Detection:
   - Test passes on retry but failed initially → flag as potentially flaky
   - Test fails intermittently across multiple pipeline runs → confirmed flaky
   - Track flaky rate: (flaky failures / total runs) × 100

2. Quarantine:
   - Move flaky tests to a separate test suite
   - Run quarantined tests in a non-blocking pipeline stage
   - Tag quarantined tests: @flaky, @quarantine, or skip marker

3. Remediation:
   - Assign flaky tests to owners on a rotation
   - Set a time limit: fix within 2 weeks or delete the test
   - Common fixes: add waits for async operations, mock external services,
     fix shared state between tests, use unique test data

4. Prevention:
   - Review new E2E tests for timing dependencies before merge
   - Require deterministic test data (no shared mutable state)
   - Use test isolation patterns (unique database schemas, fresh containers)
```

### Knowledge Check

- [ ] What are the four GitOps principles (declarative, versioned, automated, reconciled)?
- [ ] What is the difference between push-based and pull-based deployment models?
- [ ] What is the testing pyramid, and why should the base (unit tests) be larger than the top (E2E)?
- [ ] What is a flaky test, and how does it erode confidence in CI pipelines?

---

## Phase 5: Security & Production Readiness (Week 8–9)

### Learning Objectives

- Understand DevSecOps and shift-left security principles
- Implement SAST, DAST, SCA, and container image scanning in pipelines
- Configure secret detection and prevent secrets from leaking into repositories
- Learn supply chain security: SBOM generation, dependency pinning, and signed artifacts
- Apply CI/CD best practices across pipeline design, build optimisation, and deployment patterns
- Identify and remediate all 12 CI/CD anti-patterns

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-SECURITY-IN-PIPELINES](08-SECURITY-IN-PIPELINES.md) | DevSecOps, SAST, DAST, SCA, container scanning, secret detection, SBOM, supply chain security |
| 2 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Pipeline design principles, trunk-based development, build optimisation, environment management, observability |
| 3 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | All 12 anti-patterns, quick reference checklist |

### Exercises

**1. Security Scanning Pipeline:**

Design a security pipeline that integrates multiple scanning tools:

```yaml
# Security scanning pipeline
stages:
  - stage: secret-detection
    steps:
      # Scan for secrets in source code
      - run: gitleaks detect --source=. --report-format=json --report-path=gitleaks.json
      - quality-gate: zero secrets detected

  - stage: sast
    steps:
      # Static Application Security Testing
      - run: semgrep scan --config=auto --json --output=semgrep.json .
      - quality-gate: no high/critical findings

  - stage: dependency-scan
    steps:
      # Software Composition Analysis
      - run: npm audit --audit-level=high
      - run: snyk test --severity-threshold=high
      - quality-gate: no high/critical vulnerabilities

  - stage: container-scan
    depends_on: build
    steps:
      # Scan the built container image
      - run: trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest
      - run: trivy image --format spdx-json --output sbom.json myapp:latest

  - stage: iac-scan
    steps:
      # Scan Infrastructure as Code templates
      - run: checkov --directory=./infrastructure --output=json
      - quality-gate: no high/critical misconfigurations
```

For each scanning stage:
- Identify the tool used and what category of vulnerability it detects
- Explain the quality gate: what severity threshold should block the pipeline?
- Describe how false positives should be handled (suppression files, inline comments, baseline files)
- Estimate the execution time and whether the stage can run in parallel with other stages

**2. Secret Management Audit:**

Audit a real or sample repository for secret management practices:

```
Secret Management Checklist:
  [ ] No secrets in source code (passwords, API keys, tokens)
  [ ] No secrets in CI/CD pipeline YAML files
  [ ] No secrets in Dockerfiles or container images
  [ ] No secrets in environment variable definitions checked into Git
  [ ] .gitignore includes .env, *.pem, *.key, and other sensitive file patterns
  [ ] Pre-commit hooks prevent secrets from being committed (e.g., gitleaks, detect-secrets)
  [ ] Pipeline secrets are stored in the platform's secret store (GitHub Secrets, Azure Key Vault)
  [ ] Secrets are rotated on a regular schedule
  [ ] Secret access is logged and auditable
  [ ] Secrets are scoped to the minimum necessary (per-environment, per-service)
```

Set up a pre-commit hook for secret detection:

```bash
# Install gitleaks
brew install gitleaks  # macOS
# or download from https://github.com/gitleaks/gitleaks/releases

# Create a pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
gitleaks protect --staged --verbose
EOF
chmod +x .git/hooks/pre-commit

# Test the hook — this should be blocked
echo "AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" > secrets.txt
git add secrets.txt
git commit -m "add secrets"
# Expected: commit blocked by gitleaks
rm secrets.txt
```

**3. Anti-Pattern Identification Exercise:**

Review the following system description and identify at least 6 anti-patterns from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md):

> "Our team has 12 developers working on a monolith with five long-lived feature branches that have been open for 3–8 weeks. Each developer has a slightly different pipeline configuration in their branch. We deploy to production by SSHing into the server and running `git pull && npm install && pm2 restart`. Our staging environment uses SQLite while production uses PostgreSQL. There are no rollback procedures — if a deployment fails, a senior engineer manually debugs it on the production server. We have a 45-minute pipeline, but most of that time is a single large E2E test suite that fails intermittently, so developers re-run pipelines until they pass. Our API keys are stored as plaintext variables in the pipeline YAML file committed to the repository. There are no pipeline run dashboards — when something breaks, we find out from customer complaints. We build separate Docker images for staging and production with different configuration baked in."

For each anti-pattern identified:
- State the anti-pattern name and number from the document
- Quote the specific evidence from the description
- Write the recommended fix in 2–3 sentences

<details>
<summary>Anti-patterns identified (click to reveal suggested answers)</summary>

1. **#2 Long-Running Branches** — "five long-lived feature branches that have been open for 3–8 weeks"
2. **#1 Snowflake Pipelines** — "each developer has a slightly different pipeline configuration in their branch"
3. **#3 Manual Deployments** — "deploy to production by SSHing into the server and running git pull && npm install && pm2 restart"
4. **#8 No Environment Parity** — "staging environment uses SQLite while production uses PostgreSQL"
5. **#4 No Rollback Strategy** — "no rollback procedures — if a deployment fails, a senior engineer manually debugs it on the production server"
6. **#6 Ignoring Flaky Tests** — "E2E test suite that fails intermittently, so developers re-run pipelines until they pass"
7. **#7 Monolithic Pipelines** — "45-minute pipeline" with a single large test suite
8. **#5 Secrets in Pipeline Code** — "API keys are stored as plaintext variables in the pipeline YAML file committed to the repository"
9. **#10 Insufficient Pipeline Observability** — "no pipeline run dashboards — when something breaks, we find out from customer complaints"
10. **#9 Building Different Artifacts Per Environment** — "build separate Docker images for staging and production with different configuration baked in"

</details>

**4. Pipeline Best Practices Audit:**

Evaluate an existing pipeline (your own or the capstone) against the best practices from [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md):

```
Pipeline Best Practices Audit:
  Design:
    [ ] Pipeline runs in under 10 minutes for CI feedback
    [ ] Pipeline fails fast — linting and unit tests run before expensive stages
    [ ] All pipeline configuration is versioned in the repository (pipeline as code)
    [ ] Pipeline is idempotent — running it twice produces the same result

  Build:
    [ ] Dependencies are cached between pipeline runs
    [ ] Build artifacts are produced once and promoted between environments
    [ ] Build matrix is used for cross-platform or cross-version testing
    [ ] Parallel jobs are used where stages are independent

  Testing:
    [ ] Testing follows the pyramid: many unit tests, fewer integration, minimal E2E
    [ ] Code coverage is tracked and enforced as a quality gate
    [ ] Flaky tests are quarantined and tracked
    [ ] Test results are published as pipeline artifacts

  Security:
    [ ] Secrets are stored in the platform's secret manager, not in code
    [ ] Dependencies are scanned for known vulnerabilities
    [ ] Container images are scanned before deployment
    [ ] Pre-commit hooks prevent secrets from being committed

  Deployment:
    [ ] Deployments are automated — no manual SSH or copy commands
    [ ] Rollback is automated and tested
    [ ] Health checks verify deployment success
    [ ] Deployment notifications are sent to the team

  Observability:
    [ ] Pipeline run duration is tracked over time
    [ ] Failure rates are monitored per stage and per branch
    [ ] Deployment frequency and lead time are measured
    [ ] Alerts fire when pipeline failure rate exceeds threshold
```

For each unchecked item, write a one-paragraph remediation plan: what to change, estimated effort, and priority.

### Knowledge Check

- [ ] What is the difference between SAST, DAST, and SCA, and at which pipeline stage does each run?
- [ ] What is shift-left security, and why should security scanning happen in CI rather than after deployment?
- [ ] What are the four key DORA metrics (deployment frequency, lead time, change failure rate, time to recovery)?
- [ ] Name three anti-patterns from the checklist and explain why each is harmful

---

## Phase 6: Capstone Project (Week 10)

### Project Overview

Design and implement a complete CI/CD pipeline for a multi-service application with production-grade practices. Build a task management API with the following components:

### System Architecture

```
                ┌─────────────────────────────────┐
                │         GitHub Repository         │
                │    (source code + pipeline config) │
                └──────────────┬──────────────────┘
                               │
                    push / pull_request
                               │
                ┌──────────────▼──────────────────┐
                │         CI Pipeline              │
                │                                   │
                │  ┌────────┐  ┌────────────────┐  │
                │  │ Lint   │→ │ Unit Tests      │  │
                │  └────────┘  └───────┬────────┘  │
                │                       │           │
                │  ┌────────────────────▼────────┐  │
                │  │ Build & Container Image      │  │
                │  └───────┬──────────┬──────────┘  │
                │          │          │              │
                │  ┌───────▼──────┐ ┌▼───────────┐  │
                │  │ Integration  │ │ Security    │  │
                │  │ Tests        │ │ Scanning    │  │
                │  └───────┬──────┘ └┬───────────┘  │
                │          │         │               │
                │  ┌───────▼─────────▼───────────┐  │
                │  │ Push to Container Registry   │  │
                │  └──────────────┬──────────────┘  │
                └─────────────────┼─────────────────┘
                                  │
                ┌─────────────────▼─────────────────┐
                │         CD Pipeline               │
                │                                    │
                │  ┌──────────────────────────────┐  │
                │  │ Deploy to Staging             │  │
                │  │ (automatic on main branch)    │  │
                │  └──────────────┬───────────────┘  │
                │                 │                   │
                │  ┌──────────────▼───────────────┐  │
                │  │ Smoke Tests on Staging        │  │
                │  └──────────────┬───────────────┘  │
                │                 │                   │
                │  ┌──────────────▼───────────────┐  │
                │  │ Manual Approval Gate          │  │
                │  └──────────────┬───────────────┘  │
                │                 │                   │
                │  ┌──────────────▼───────────────┐  │
                │  │ Deploy to Production          │  │
                │  │ (blue-green or canary)        │  │
                │  └──────────────┬───────────────┘  │
                │                 │                   │
                │  ┌──────────────▼───────────────┐  │
                │  │ Post-Deploy Health Check      │  │
                │  └──────────────────────────────┘  │
                └────────────────────────────────────┘
```

### Requirements

| Requirement | What to Implement |
|-------------|-------------------|
| **Pipeline as Code** | All pipeline configuration lives in the repository; no manual UI configuration |
| **CI Pipeline** | Lint, unit test, build, integration test, security scan — all automated and under 10 minutes |
| **Container Build** | Multi-stage Dockerfile; image under 200 MB; non-root user; pushed to a container registry with SemVer tags |
| **Security Scanning** | Secret detection (gitleaks), dependency scanning (npm audit or snyk), container scanning (trivy), SAST (semgrep or CodeQL) |
| **Testing Strategy** | Unit tests with ≥80% coverage gate, integration tests with service containers, at least one E2E test |
| **CD Pipeline** | Automatic deployment to staging on main branch; manual approval gate for production |
| **Release Strategy** | Implement at least one strategy: blue-green, canary, or rolling update |
| **Rollback** | Automated rollback triggered by health check failure within 5 minutes of deployment |
| **Environment Management** | Same artifact deployed to all environments; configuration via environment variables |
| **Observability** | Pipeline duration tracked; deployment notifications sent (Slack, email, or webhook) |
| **GitOps (bonus)** | Store deployment manifests in a separate config repository; use ArgoCD or Flux for reconciliation |
| **Anti-patterns** | Run the quick reference checklist from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md); zero anti-patterns in final submission |
| **Documentation** | README with architecture diagram, setup instructions, environment variable reference, and runbook |

### Implementation Steps

**Step 1: Set Up the Application**

Create a simple application with the following structure:

```
task-api/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
├── src/
│   ├── app.js
│   ├── routes/
│   │   └── tasks.js
│   └── db.js
├── tests/
│   ├── unit/
│   │   └── tasks.test.js
│   ├── integration/
│   │   └── tasks.integration.test.js
│   └── e2e/
│       └── tasks.e2e.test.js
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .gitignore
├── .gitleaks.toml
├── package.json
└── README.md
```

**Step 2: Write the CI Pipeline**

The CI pipeline should run on every push and pull request:

1. **Lint** — ESLint or equivalent; fail the pipeline on warnings
2. **Unit Tests** — Jest with coverage; fail if coverage drops below 80%
3. **Build** — Build the container image with a unique tag (commit SHA or SemVer)
4. **Integration Tests** — Spin up PostgreSQL as a service container; run integration tests against it
5. **Security Scan** — Run gitleaks, npm audit, trivy, and at least one SAST tool
6. **Push Image** — Push to GitHub Container Registry (ghcr.io) or Docker Hub

**Step 3: Write the CD Pipeline**

The CD pipeline should trigger after CI succeeds on the main branch:

1. **Deploy to Staging** — Automatically deploy the new image to the staging environment
2. **Smoke Tests** — Run a minimal test suite against the staging URL
3. **Manual Approval** — Require a team member to approve the production deployment
4. **Deploy to Production** — Deploy using your chosen release strategy
5. **Health Check** — Verify the `/healthz` endpoint returns 200; trigger rollback if it fails
6. **Notification** — Send a deployment notification with the version, environment, and status

**Step 4: Implement Rollback**

Configure automated rollback:

```yaml
# Post-deployment health check and rollback
- name: Health Check
  run: |
    for i in $(seq 1 30); do
      status=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/healthz)
      if [ "$status" = "200" ]; then
        echo "Health check passed on attempt $i"
        exit 0
      fi
      echo "Health check failed (status: $status), retrying in 10s..."
      sleep 10
    done
    echo "Health check failed after 30 attempts — triggering rollback"
    exit 1

- name: Rollback on Failure
  if: failure()
  run: |
    echo "Rolling back to previous version..."
    # Redeploy the last known good image tag
    kubectl set image deployment/task-api \
      task-api=ghcr.io/myorg/task-api:${{ env.PREVIOUS_VERSION }}
    kubectl rollout status deployment/task-api --timeout=120s
```

**Step 5: Document Everything**

Write a comprehensive README that includes:
- Architecture diagram showing the CI/CD pipeline
- Setup instructions: how to run the application locally, how to run tests, how to trigger a deployment
- Environment variable reference: every configuration value and its purpose
- Runbook: what to do when the pipeline fails at each stage, how to trigger a manual rollback

### Evaluation Criteria

| Area | What to Verify |
|------|----------------|
| **Pipeline as Code** | All configuration is in the repository; `git clone` + `git push` triggers the full pipeline |
| **CI Speed** | Full CI pipeline completes in under 10 minutes |
| **Test Coverage** | Unit test coverage ≥ 80%; integration tests run against real database; at least one E2E test |
| **Security** | At least three security scanning tools integrated; no high/critical findings in final submission |
| **Deployment** | Automatic staging deployment works; manual approval gates production; rollback mechanism tested |
| **Anti-patterns** | All 12 anti-patterns from the checklist evaluated; findings documented |
| **Documentation** | README explains architecture, setup, configuration, and failure runbook |
| **Observability** | Pipeline duration visible; deployment events logged; notifications configured |

---

## Suggested Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Continuous Delivery* | Jez Humble, David Farley | The foundational text on deployment pipelines, release strategies, and automated delivery |
| *Accelerate* | Nicole Forsgren, Jez Humble, Gene Kim | DORA metrics, high-performing teams, and the science of software delivery |
| *The Phoenix Project* | Gene Kim, Kevin Behr, George Spafford | DevOps transformation through narrative — CI/CD culture and flow |
| *The DevOps Handbook* | Gene Kim, Jez Humble, Patrick Debois, John Willis | Practical implementation of DevOps practices including CI/CD |
| *Release It!* | Michael T. Nygard | Stability patterns, deployment strategies, and production readiness |
| *Infrastructure as Code* | Kief Morris | Managing infrastructure through automation — the foundation for CD pipelines |

### Online Resources

- **docs.github.com/en/actions** — Official GitHub Actions documentation, workflow syntax reference
- **learn.microsoft.com/en-us/azure/devops/pipelines** — Azure DevOps Pipelines documentation and tutorials
- **www.jenkins.io/doc/book/pipeline** — Jenkins Pipeline documentation and best practices
- **argo-cd.readthedocs.io** — ArgoCD documentation for GitOps workflows
- **fluxcd.io/docs** — Flux documentation for GitOps on Kubernetes
- **dora.dev** — DORA team research on software delivery performance and metrics
- **owasp.org** — OWASP resources for application security in pipelines

### Tools

| Tool | Purpose | Phase |
|------|---------|-------|
| GitHub Actions | CI/CD platform (GitHub-native) | 3–6, Capstone |
| Azure DevOps Pipelines | CI/CD platform (Microsoft ecosystem) | 3–6, Capstone |
| Jenkins | Self-hosted CI/CD server | 3–6, Capstone |
| ArgoCD | GitOps continuous delivery for Kubernetes | 4, Capstone |
| Flux | GitOps toolkit for Kubernetes | 4, Capstone |
| gitleaks | Secret detection in Git repositories | 5, Capstone |
| trivy | Container image and filesystem vulnerability scanner | 5, Capstone |
| semgrep | Lightweight SAST for multiple languages | 5, Capstone |
| Playwright | End-to-end browser testing framework | 4, Capstone |
| Pact | Consumer-driven contract testing | 4, Capstone |

---

## Quick Reference: Document Map

| # | Document | Phase | Key Topics |
|---|----------|-------|------------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 | CI/CD fundamentals, pipeline anatomy, environments, branching strategies |
| 01 | [CONTINUOUS-INTEGRATION](01-CONTINUOUS-INTEGRATION.md) | 1 | CI feedback loop, build automation, testing, code quality |
| 02 | [CONTINUOUS-DELIVERY](02-CONTINUOUS-DELIVERY.md) | 2 | Delivery vs deployment, release strategies, rollback, environment management |
| 03 | [GITHUB-ACTIONS](03-GITHUB-ACTIONS.md) | 3 | Workflows, events, jobs, runners, matrix, reusable workflows |
| 04 | [AZURE-DEVOPS](04-AZURE-DEVOPS.md) | 3 | YAML pipelines, stages, variables, templates, environments |
| 05 | [JENKINS](05-JENKINS.md) | 3 | Declarative/Scripted pipelines, shared libraries, plugins |
| 06 | [GITOPS](06-GITOPS.md) | 4 | GitOps principles, ArgoCD, Flux, push vs pull models |
| 07 | [TESTING-IN-PIPELINES](07-TESTING-IN-PIPELINES.md) | 4 | Testing pyramid, quality gates, parallelisation, flaky tests |
| 08 | [SECURITY-IN-PIPELINES](08-SECURITY-IN-PIPELINES.md) | 5 | DevSecOps, SAST, DAST, SCA, secret detection, SBOM |
| 09 | [BEST-PRACTICES](09-BEST-PRACTICES.md) | 5 | Pipeline design, build optimisation, deployment patterns, observability |
| 10 | [ANTI-PATTERNS](10-ANTI-PATTERNS.md) | 5 | 12 anti-patterns, quick reference checklist |
| — | [LEARNING-PATH](LEARNING-PATH.md) | All | This document — structured 6-phase curriculum |

---

## Next Steps

1. **Complete the capstone** — The capstone project is the most important part of this learning path; it ties together every phase into a single deliverable.
2. **Audit your existing pipelines** — Apply the [anti-patterns checklist](10-ANTI-PATTERNS.md#quick-reference-checklist) to your production CI/CD pipelines.
3. **Learn Kubernetes** — CI/CD pipelines often deploy to Kubernetes clusters. See the [Kubernetes learning materials](../kubernetes/) in this repository.
4. **Explore observability** — Pipelines and deployed services need monitoring, logging, and tracing. See the [Observability learning materials](../observability/) in this repository.
5. **Contribute** — Found an error, have a better example, or want to add a new exercise? Open a PR following the [CONTRIBUTING.md](../CONTRIBUTING.md) guide.

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-01-01 | Initial document — 6-phase learning path with exercises and capstone | Platform Team |
