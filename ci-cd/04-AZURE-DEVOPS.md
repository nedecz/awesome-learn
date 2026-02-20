# Azure DevOps Pipelines

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Azure DevOps Overview](#azure-devops-overview)
   - [Azure DevOps Services vs Azure DevOps Server](#azure-devops-services-vs-azure-devops-server)
   - [Key Components](#key-components)
3. [Azure Pipelines Fundamentals](#azure-pipelines-fundamentals)
   - [Classic Pipelines vs YAML Pipelines](#classic-pipelines-vs-yaml-pipelines)
   - [Pipeline Execution Flow](#pipeline-execution-flow)
4. [YAML Pipeline Syntax](#yaml-pipeline-syntax)
   - [Pipeline Structure](#pipeline-structure)
   - [Stages](#stages)
   - [Jobs](#jobs)
   - [Steps and Tasks](#steps-and-tasks)
   - [Complete Pipeline Example](#complete-pipeline-example)
5. [Triggers and Schedules](#triggers-and-schedules)
   - [CI Triggers](#ci-triggers)
   - [PR Triggers](#pr-triggers)
   - [Scheduled Triggers](#scheduled-triggers)
   - [Pipeline Triggers](#pipeline-triggers)
   - [Trigger Comparison](#trigger-comparison)
6. [Variables and Variable Groups](#variables-and-variable-groups)
   - [Pipeline Variables](#pipeline-variables)
   - [Variable Groups](#variable-groups)
   - [Key Vault Integration](#key-vault-integration)
   - [Variable Precedence](#variable-precedence)
7. [Service Connections](#service-connections)
   - [Azure Resource Manager](#azure-resource-manager)
   - [Docker Registry](#docker-registry)
   - [Kubernetes](#kubernetes)
   - [GitHub](#github)
   - [Service Connection Comparison](#service-connection-comparison)
8. [Agent Pools](#agent-pools)
   - [Microsoft-Hosted Agents](#microsoft-hosted-agents)
   - [Self-Hosted Agents](#self-hosted-agents)
   - [Agent Capabilities](#agent-capabilities)
   - [Agent Comparison](#agent-comparison)
9. [Templates](#templates)
   - [Stage Templates](#stage-templates)
   - [Job Templates](#job-templates)
   - [Step Templates](#step-templates)
   - [Variable Templates](#variable-templates)
   - [Template Expressions](#template-expressions)
10. [Environments and Approvals](#environments-and-approvals)
    - [Environment Resources](#environment-resources)
    - [Deployment Strategies](#deployment-strategies)
    - [Gates and Checks](#gates-and-checks)
11. [Azure Artifacts](#azure-artifacts)
    - [Feed Types](#feed-types)
    - [NuGet Feeds](#nuget-feeds)
    - [npm Feeds](#npm-feeds)
    - [Maven Feeds](#maven-feeds)
    - [Python Feeds](#python-feeds)
    - [Universal Packages](#universal-packages)
12. [Multi-Stage Pipelines](#multi-stage-pipelines)
    - [Build-Test-Deploy Pattern](#build-test-deploy-pattern)
    - [Stage Dependencies and Conditions](#stage-dependencies-and-conditions)
    - [Parallel Stages](#parallel-stages)
13. [Integration with Azure Services](#integration-with-azure-services)
    - [Azure Kubernetes Service (AKS)](#azure-kubernetes-service-aks)
    - [Azure App Service](#azure-app-service)
    - [Azure Functions](#azure-functions)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Azure DevOps Pipelines** — from the platform's core components and YAML pipeline syntax, through service connections and agent pools, to multi-stage deployment patterns and integration with Azure services.

Azure DevOps is Microsoft's end-to-end DevOps platform that combines source control, CI/CD, project tracking, artifact management, and testing into a single integrated experience. Azure Pipelines, its CI/CD engine, supports any language, platform, and cloud.

```
Azure DevOps Platform
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │  Boards  │ │  Repos   │ │ Pipelines│ │Test Plans│ │ Artifacts│
  │          │ │          │ │          │ │          │ │          │
  │ Work     │ │ Git      │ │ CI/CD    │ │ Manual & │ │ Package  │
  │ tracking │ │ repos    │ │ build &  │ │ automated│ │ feeds    │
  │ & Agile  │ │ & TFVC   │ │ release  │ │ testing  │ │ (NuGet,  │
  │ boards   │ │          │ │          │ │          │ │ npm...)  │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
       │             │             │             │            │
       └─────────────┴─────────────┴─────────────┴────────────┘
                         Unified DevOps Platform
```

### Target Audience

- **Developers** building CI/CD pipelines to automate builds, tests, and deployments
- **DevOps Engineers** designing multi-stage release pipelines with approvals and gates
- **Platform Engineers** managing agent pools, service connections, and organizational templates
- **Architects** evaluating Azure DevOps as a CI/CD platform for enterprise workloads

### Scope

- Azure DevOps Services vs Server and key platform components
- Classic vs YAML pipeline models and when to use each
- YAML pipeline syntax: stages, jobs, steps, tasks, and expressions
- Trigger configuration: CI, PR, scheduled, and pipeline triggers
- Variables, variable groups, and Azure Key Vault integration
- Service connections for Azure, Docker, Kubernetes, and GitHub
- Microsoft-hosted and self-hosted agent pools
- Template reuse patterns: stage, job, step, and variable templates
- Environments, deployment strategies, approvals, and gates
- Azure Artifacts feeds for NuGet, npm, Maven, Python, and Universal Packages
- Multi-stage pipeline patterns and integration with Azure services

---

## Azure DevOps Overview

Azure DevOps provides a set of developer services for teams to plan work, collaborate on code development, and build and deploy applications. It is available as both a cloud service and an on-premises server.

### Azure DevOps Services vs Azure DevOps Server

| Feature | Azure DevOps Services (Cloud) | Azure DevOps Server (On-Premises) |
|---|---|---|
| **Hosting** | Microsoft-managed SaaS | Self-hosted on your infrastructure |
| **Updates** | Continuous delivery of new features | Periodic releases (yearly cadence) |
| **Scalability** | Automatic, managed by Microsoft | Manual capacity planning required |
| **Authentication** | Microsoft Entra ID (Azure AD) | Active Directory or Windows Auth |
| **Data Residency** | Region selection at org creation | Full control over data location |
| **Microsoft-Hosted Agents** | Available (free tier included) | Not available; self-hosted only |
| **Maintenance** | Zero maintenance | Patching, backups, and upgrades |
| **Pricing** | Per-user with free tier (5 users) | License-based per server/CAL |
| **Compliance** | SOC 1/2/3, ISO 27001, FedRAMP | Depends on your infrastructure |
| **Connectivity** | Requires internet | Can operate fully air-gapped |

### Key Components

Azure DevOps is organized into five core services that can be used together or independently:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Azure DevOps Organization                        │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                        Project                                │   │
│  │                                                              │   │
│  │  ┌────────────┐  Azure Boards                                │   │
│  │  │ Work Items │  Agile planning with Kanban boards,          │   │
│  │  │ Sprints    │  backlogs, sprints, and dashboards.          │   │
│  │  │ Queries    │  Supports Scrum, Kanban, and CMMI.           │   │
│  │  └────────────┘                                              │   │
│  │                                                              │   │
│  │  ┌────────────┐  Azure Repos                                 │   │
│  │  │ Git Repos  │  Unlimited private Git repositories with     │   │
│  │  │ Pull Reqs  │  branch policies, code review, and           │   │
│  │  │ Branches   │  pull request workflows.                     │   │
│  │  └────────────┘                                              │   │
│  │                                                              │   │
│  │  ┌────────────┐  Azure Pipelines                             │   │
│  │  │ Build      │  CI/CD engine supporting any language,       │   │
│  │  │ Release    │  platform, and cloud. YAML-first with        │   │
│  │  │ YAML       │  classic UI fallback.                        │   │
│  │  └────────────┘                                              │   │
│  │                                                              │   │
│  │  ┌────────────┐  Azure Test Plans                            │   │
│  │  │ Manual     │  Manual and exploratory testing tools         │   │
│  │  │ Automated  │  with test case management and               │   │
│  │  │ Tracing    │  traceability to requirements.               │   │
│  │  └────────────┘                                              │   │
│  │                                                              │   │
│  │  ┌────────────┐  Azure Artifacts                             │   │
│  │  │ NuGet      │  Universal package management for            │   │
│  │  │ npm/Maven  │  NuGet, npm, Maven, Python, and              │   │
│  │  │ Python     │  Universal Packages.                         │   │
│  │  └────────────┘                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Azure Boards** provides work tracking with rich support for Agile methodologies. Teams use boards, backlogs, and sprints to plan and track work. Work items can be linked directly to commits, pull requests, and builds for full traceability.

**Azure Repos** provides Git repositories (or Team Foundation Version Control) with branch policies, required reviewers, and build validation. Branch policies ensure that code meets quality standards before merging.

**Azure Pipelines** is the CI/CD engine at the core of Azure DevOps. It supports build and release automation for any language and any platform. Pipelines can be defined in YAML (recommended) or through the classic visual editor.

**Azure Test Plans** provides manual and exploratory testing capabilities integrated with the rest of the platform. Test cases link to user stories and bugs, providing traceability from requirements through testing.

**Azure Artifacts** hosts package feeds for NuGet, npm, Maven, Python, and Universal Packages. Feeds support upstream sources, allowing you to cache packages from public registries while hosting your private packages alongside them.

---

## Azure Pipelines Fundamentals

Azure Pipelines automates the build, test, and deployment of code projects. It works with any language, platform, and cloud provider.

### Classic Pipelines vs YAML Pipelines

Azure Pipelines offers two authoring models:

| Feature | Classic (Visual Editor) | YAML Pipelines |
|---|---|---|
| **Definition** | GUI-based in Azure DevOps portal | Code-based in `azure-pipelines.yml` |
| **Version Control** | Stored in Azure DevOps (not in repo) | Stored alongside code in the repo |
| **Review Process** | Changes take effect immediately | Changes go through PR review |
| **Template Reuse** | Task groups (limited) | Full template system (stages, jobs, steps) |
| **Multi-Stage** | Separate build and release definitions | Single unified YAML file |
| **Branching** | Same pipeline definition for all branches | Pipeline definition can vary per branch |
| **Rollback** | Manual reconfiguration | Git revert to previous YAML version |
| **Recommended For** | Legacy pipelines, quick prototyping | All new pipelines (Microsoft-recommended) |

> **Recommendation**: Use YAML pipelines for all new development. They provide version control, code review, branching, and a single source of truth for your pipeline configuration.

### Pipeline Execution Flow

```
Pipeline Execution Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Trigger               Pipeline              Agent Pool
  ┌──────────┐          ┌──────────┐          ┌──────────┐
  │ • Push   │          │ Parse    │          │ Allocate │
  │ • PR     │ ──────▶  │ YAML     │ ──────▶  │ Agent    │
  │ • Schedule│          │ Expand   │          │          │
  │ • Manual │          │ Templates│          │ Download │
  └──────────┘          └──────────┘          │ Sources  │
                                              └────┬─────┘
                                                   │
                              ┌─────────────────────┘
                              ▼
                        ┌──────────┐    ┌──────────┐    ┌──────────┐
                        │ Stage 1  │    │ Stage 2  │    │ Stage 3  │
                        │ Build    │──▶ │ Test     │──▶ │ Deploy   │
                        │          │    │          │    │          │
                        │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │
                        │ │Job 1 │ │    │ │Job 1 │ │    │ │Job 1 │ │
                        │ │Step 1│ │    │ │Step 1│ │    │ │Step 1│ │
                        │ │Step 2│ │    │ │Step 2│ │    │ │Step 2│ │
                        │ └──────┘ │    │ └──────┘ │    │ └──────┘ │
                        └──────────┘    └──────────┘    └──────────┘
```

---

## YAML Pipeline Syntax

YAML pipelines are defined in a file named `azure-pipelines.yml` at the root of your repository. The YAML schema uses a hierarchical structure: **pipeline → stages → jobs → steps**.

### Pipeline Structure

```
YAML Pipeline Hierarchy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Pipeline
  ├── trigger
  ├── pool
  ├── variables
  └── stages
      ├── stage: Build
      │   └── jobs
      │       ├── job: Compile
      │       │   └── steps
      │       │       ├── script: dotnet build
      │       │       └── task: PublishBuildArtifacts
      │       └── job: UnitTest
      │           └── steps
      │               └── script: dotnet test
      └── stage: Deploy
          └── jobs
              └── deployment: Production
                  └── strategy
                      └── runOnce
                          └── deploy
                              └── steps
```

A minimal pipeline requires only a trigger and steps:

```yaml
# Minimal pipeline — single stage, single job
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - script: echo Hello, Azure Pipelines!
    displayName: 'Run a one-line script'
```

### Stages

Stages are the top-level divisions of a pipeline. Each stage can contain one or more jobs and can depend on other stages.

```yaml
stages:
  - stage: Build
    displayName: 'Build Application'
    jobs:
      - job: BuildJob
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: dotnet build --configuration Release
            displayName: 'Build'

  - stage: Test
    displayName: 'Run Tests'
    dependsOn: Build
    jobs:
      - job: TestJob
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: dotnet test --no-build
            displayName: 'Test'

  - stage: Deploy
    displayName: 'Deploy to Production'
    dependsOn: Test
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Production
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo Deploying to production
                  displayName: 'Deploy'
```

### Jobs

Jobs define a series of steps that run sequentially on an agent. Jobs within a stage run in parallel by default unless dependencies are specified.

```yaml
jobs:
  # Regular job
  - job: Build
    displayName: 'Build Application'
    pool:
      vmImage: 'ubuntu-latest'
    timeoutInMinutes: 60
    steps:
      - script: npm install
      - script: npm run build

  # Deployment job (targets an environment)
  - deployment: DeployWeb
    displayName: 'Deploy Web App'
    environment: 'staging'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
            - script: echo Deploying...

  # Container job (runs inside a container)
  - job: TestInContainer
    displayName: 'Run Tests in Container'
    pool:
      vmImage: 'ubuntu-latest'
    container: node:18-alpine
    steps:
      - script: npm test
```

| Job Type | Keyword | Purpose |
|---|---|---|
| **Agent job** | `job` | Runs on a pipeline agent in a pool |
| **Deployment job** | `deployment` | Deploys to an environment with deployment strategies |
| **Container job** | `job` + `container` | Runs steps inside a specified container image |
| **Server job** | `job` + `pool: server` | Runs on the Azure DevOps server (for gates and checks) |

### Steps and Tasks

Steps are the smallest unit of execution. Each step runs in its own process on the agent.

```yaml
steps:
  # Inline script
  - script: |
      echo "Building project..."
      dotnet build --configuration Release
    displayName: 'Build project'
    workingDirectory: $(Build.SourcesDirectory)/src

  # Bash script (Linux/macOS)
  - bash: |
      set -euo pipefail
      npm install
      npm run lint
    displayName: 'Install and lint'

  # PowerShell script (Windows/cross-platform)
  - pwsh: |
      Write-Host "Running on $($PSVersionTable.OS)"
      dotnet test --logger trx
    displayName: 'Run tests with PowerShell'

  # Built-in task reference
  - task: DotNetCoreCLI@2
    displayName: 'Restore NuGet packages'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'

  # Checkout step
  - checkout: self
    fetchDepth: 0
    clean: true

  # Download artifact
  - download: current
    artifact: 'drop'

  # Publish artifact
  - publish: $(Build.ArtifactStagingDirectory)
    artifact: 'drop'
    displayName: 'Publish build artifact'
```

### Complete Pipeline Example

```yaml
# azure-pipelines.yml — .NET application with build, test, and deploy
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    exclude:
      - docs/*
      - '*.md'

pr:
  branches:
    include:
      - main

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildAndTest
        displayName: 'Build and run tests'
        steps:
          - task: UseDotNet@2
            displayName: 'Install .NET SDK'
            inputs:
              packageType: 'sdk'
              version: $(dotnetVersion)

          - task: DotNetCoreCLI@2
            displayName: 'Restore packages'
            inputs:
              command: 'restore'
              projects: '**/*.csproj'

          - task: DotNetCoreCLI@2
            displayName: 'Build solution'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Run unit tests'
            inputs:
              command: 'test'
              projects: '**/*Tests.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'

          - task: PublishCodeCoverageResults@2
            displayName: 'Publish code coverage'
            inputs:
              summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'

          - task: DotNetCoreCLI@2
            displayName: 'Publish application'
            inputs:
              command: 'publish'
              projects: 'src/MyApp/MyApp.csproj'
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'

          - publish: $(Build.ArtifactStagingDirectory)
            artifact: 'drop'
            displayName: 'Upload build artifact'

  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToStaging
        displayName: 'Deploy to Staging'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy to Azure App Service'
                  inputs:
                    azureSubscription: 'my-azure-connection'
                    appName: 'my-app-staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'

  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployToProduction
        displayName: 'Deploy to Production'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy to Azure App Service'
                  inputs:
                    azureSubscription: 'my-azure-connection'
                    appName: 'my-app-production'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
```

---

## Triggers and Schedules

Triggers define when a pipeline runs. Azure Pipelines supports several trigger types that can be combined for flexible automation.

### CI Triggers

CI triggers run the pipeline when changes are pushed to specified branches or paths:

```yaml
# Branch-based trigger with path filters
trigger:
  branches:
    include:
      - main
      - release/*
      - feature/*
    exclude:
      - feature/experimental/*
  paths:
    include:
      - src/*
      - tests/*
    exclude:
      - docs/*
      - '*.md'
  tags:
    include:
      - v*
```

To disable CI triggers entirely:

```yaml
trigger: none
```

### PR Triggers

PR triggers run the pipeline when a pull request targets specified branches:

```yaml
# PR trigger configuration
pr:
  branches:
    include:
      - main
      - release/*
  paths:
    include:
      - src/*
    exclude:
      - docs/*
  drafts: false    # Do not trigger on draft PRs
```

> **Note**: For Azure Repos, PR triggers are configured in the YAML file. For GitHub repos, PR triggers are configured via branch protection rules and the YAML `pr` key.

### Scheduled Triggers

Scheduled triggers run the pipeline on a cron-based schedule:

```yaml
schedules:
  - cron: '0 2 * * 1-5'         # 2:00 AM UTC, Monday through Friday
    displayName: 'Nightly build'
    branches:
      include:
        - main
    always: true                  # Run even if no code changes

  - cron: '0 8 * * 0'           # 8:00 AM UTC, every Sunday
    displayName: 'Weekly full test'
    branches:
      include:
        - main
        - release/*
    always: true
```

The cron syntax follows the standard five-field format:

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6, Sunday = 0)
│ │ │ │ │
│ │ │ │ │
* * * * *
```

### Pipeline Triggers

Pipeline triggers (resource triggers) run a pipeline when another pipeline completes:

```yaml
resources:
  pipelines:
    - pipeline: buildPipeline         # Internal name for reference
      source: 'MyApp-CI'              # Name of the triggering pipeline
      trigger:
        branches:
          include:
            - main
        stages:
          - Build                     # Only trigger after this stage completes

# Access artifacts from the triggering pipeline
steps:
  - download: buildPipeline
    artifact: 'drop'
```

### Trigger Comparison

| Trigger Type | Fires When | Configuration Key | Use Case |
|---|---|---|---|
| **CI Trigger** | Code is pushed to a branch | `trigger` | Build and test on every commit |
| **PR Trigger** | Pull request is created/updated | `pr` | Validate PRs before merge |
| **Scheduled** | Cron schedule matches | `schedules` | Nightly builds, periodic scans |
| **Pipeline** | Another pipeline completes | `resources.pipelines.trigger` | Chain build → deploy pipelines |
| **Manual** | User clicks "Run pipeline" | (always available) | On-demand deployments |

---

## Variables and Variable Groups

Variables allow you to store values that can be reused throughout your pipeline. Azure Pipelines provides several scopes and mechanisms for managing variables.

### Pipeline Variables

Variables can be defined at the pipeline, stage, or job level:

```yaml
# Pipeline-level variables
variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

# Using name/value syntax (allows additional properties)
variables:
  - name: buildConfiguration
    value: 'Release'
  - name: isMainBranch
    value: $[eq(variables['Build.SourceBranch'], 'refs/heads/main')]
    readonly: true

# Stage-level variables
stages:
  - stage: Build
    variables:
      stageVar: 'build-value'
    jobs:
      - job: BuildJob
        variables:
          jobVar: 'job-value'
        steps:
          - script: echo $(buildConfiguration) $(stageVar) $(jobVar)
```

Azure Pipelines provides many **predefined variables**:

| Variable | Description | Example Value |
|---|---|---|
| `Build.SourceBranch` | Full ref of the triggering branch | `refs/heads/main` |
| `Build.SourceBranchName` | Short branch name | `main` |
| `Build.BuildId` | Unique ID of the build run | `12345` |
| `Build.BuildNumber` | Human-readable build number | `20250115.1` |
| `Build.Repository.Name` | Repository name | `my-app` |
| `Build.ArtifactStagingDirectory` | Local path for staging artifacts | `/home/vsts/work/1/a` |
| `System.DefaultWorkingDirectory` | Default working directory | `/home/vsts/work/1/s` |
| `Agent.TempDirectory` | Temp directory on the agent | `/home/vsts/work/_temp` |
| `Pipeline.Workspace` | Workspace directory for the pipeline | `/home/vsts/work/1` |

### Variable Groups

Variable groups store collections of variables that can be shared across multiple pipelines:

```yaml
variables:
  - group: 'my-variable-group'        # Reference a variable group
  - name: localVar                      # Combine with inline variables
    value: 'local-value'
```

Variable groups are created in the Azure DevOps portal under **Pipelines → Library** or via the CLI:

```bash
# Create a variable group with the Azure CLI
az pipelines variable-group create \
  --name "app-settings" \
  --variables \
    API_URL="https://api.example.com" \
    LOG_LEVEL="info" \
  --authorize true

# Add a secret variable
az pipelines variable-group variable create \
  --group-id 1 \
  --name "DB_PASSWORD" \
  --value "s3cret" \
  --secret true
```

### Key Vault Integration

Variable groups can be linked to Azure Key Vault to pull secrets at pipeline runtime:

```yaml
# Reference a variable group linked to Key Vault
variables:
  - group: 'keyvault-secrets'

steps:
  - script: |
      echo "Using secret from Key Vault"
      curl -H "Authorization: Bearer $(api-key)" https://api.example.com
    displayName: 'Use Key Vault secret'
    env:
      API_KEY: $(api-key)    # Map secret to environment variable
```

```
Key Vault Integration Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Variable Group              Azure Key Vault
  ┌──────────────┐            ┌──────────────┐
  │ Linked to    │ ─────────▶ │ Secrets:     │
  │ Key Vault    │  Fetches   │ • api-key    │
  │              │  at runtime│ • db-conn    │
  └──────┬───────┘            │ • cert-pwd   │
         │                    └──────────────┘
         ▼
  Pipeline Variables
  $(api-key), $(db-conn), $(cert-pwd)
```

### Variable Precedence

When the same variable is defined in multiple places, Azure Pipelines follows a precedence order (highest to lowest):

1. **Queue-time variables** — Set when manually triggering a run
2. **Pipeline-level YAML variables** — Defined in the YAML file
3. **Variable groups** — Linked variable groups
4. **Pipeline settings variables** — Set in the pipeline UI
5. **Predefined variables** — System-provided variables

---

## Service Connections

Service connections define authenticated connections from Azure Pipelines to external services. They are managed at the project level under **Project Settings → Service connections**.

### Azure Resource Manager

The most common service connection type, used to deploy to Azure subscriptions:

```yaml
# Using an Azure Resource Manager service connection
steps:
  - task: AzureCLI@2
    displayName: 'Run Azure CLI commands'
    inputs:
      azureSubscription: 'my-azure-subscription'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az group list --output table
        az webapp list --output table

  - task: AzureResourceManagerTemplateDeployment@3
    displayName: 'Deploy ARM template'
    inputs:
      azureResourceManagerConnection: 'my-azure-subscription'
      subscriptionId: '$(subscriptionId)'
      resourceGroupName: 'my-rg'
      location: 'East US'
      templateLocation: 'LinkedArtifact'
      csmFile: 'infra/main.bicep'
```

Azure Resource Manager connections support three authentication methods:

| Method | Description | Best For |
|---|---|---|
| **Workload Identity Federation** | OIDC-based, no secrets to manage | Recommended for all new connections |
| **Service Principal (automatic)** | Azure DevOps creates and manages the SP | Quick setup for team projects |
| **Service Principal (manual)** | You provide existing SP credentials | Enterprise environments with strict policies |

### Docker Registry

Docker Registry service connections authenticate to container registries:

```yaml
steps:
  - task: Docker@2
    displayName: 'Build and push Docker image'
    inputs:
      containerRegistry: 'my-acr-connection'
      repository: 'myapp'
      command: 'buildAndPush'
      Dockerfile: '**/Dockerfile'
      tags: |
        $(Build.BuildId)
        latest
```

### Kubernetes

Kubernetes service connections enable deployment to Kubernetes clusters:

```yaml
steps:
  - task: KubernetesManifest@1
    displayName: 'Deploy to Kubernetes'
    inputs:
      action: 'deploy'
      connectionType: 'kubernetesServiceConnection'
      kubernetesServiceConnection: 'my-aks-connection'
      namespace: 'production'
      manifests: |
        k8s/deployment.yml
        k8s/service.yml
      containers: |
        myregistry.azurecr.io/myapp:$(Build.BuildId)
```

### GitHub

GitHub service connections allow pipelines to access GitHub repositories:

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: myorg/pipeline-templates
      endpoint: 'my-github-connection'
      ref: refs/heads/main
```

### Service Connection Comparison

| Connection Type | Authentication | Use Case |
|---|---|---|
| **Azure Resource Manager** | Workload Identity / Service Principal | Deploy to Azure resources |
| **Docker Registry** | Username/password or service principal | Push/pull container images |
| **Kubernetes** | Kubeconfig, Azure subscription, or service account | Deploy to K8s clusters |
| **GitHub** | GitHub App, OAuth, or PAT | Access GitHub repositories |
| **Generic** | Username/password or token | Connect to any REST API |
| **SSH** | SSH key pair | Deploy via SSH to servers |
| **NuGet/npm/Maven** | API key or token | Publish to package registries |

---

## Agent Pools

Agents are compute resources that run pipeline jobs. They are organized into pools for management and access control.

### Microsoft-Hosted Agents

Microsoft-hosted agents are managed by Azure DevOps. Each job gets a fresh virtual machine that is discarded after the job completes.

```yaml
# Specifying a Microsoft-hosted agent
pool:
  vmImage: 'ubuntu-latest'

# Or with a specific version
pool:
  vmImage: 'ubuntu-22.04'
```

Available Microsoft-hosted agent images:

| Image | Label | OS | Pre-installed Tools |
|---|---|---|---|
| `ubuntu-latest` | `ubuntu-24.04` | Ubuntu 24.04 LTS | Docker, .NET, Node.js, Python, Java, Go |
| `ubuntu-22.04` | `ubuntu-22.04` | Ubuntu 22.04 LTS | Docker, .NET, Node.js, Python, Java, Go |
| `windows-latest` | `windows-2022` | Windows Server 2022 | Visual Studio, .NET, Node.js, Python |
| `macOS-latest` | `macOS-14` | macOS 14 Sonoma | Xcode, .NET, Node.js, Python, CocoaPods |

### Self-Hosted Agents

Self-hosted agents run on infrastructure you manage. They provide persistent caches, custom tools, and network access to internal resources.

```yaml
# Targeting a self-hosted agent pool
pool:
  name: 'My-Self-Hosted-Pool'
  demands:
    - Agent.OS -equals Linux
    - docker

# Or a specific agent
pool:
  name: 'My-Self-Hosted-Pool'
  demands:
    - Agent.Name -equals build-agent-01
```

### Agent Capabilities

Agent capabilities are name-value pairs that describe what an agent can do. **System capabilities** are auto-detected; **user capabilities** are manually configured.

```yaml
# Demanding specific capabilities
pool:
  name: 'Build-Pool'
  demands:
    - npm                        # Agent must have npm installed
    - Agent.OS -equals Linux     # Must be a Linux agent
    - docker                     # Must have Docker
    - Agent.OSArchitecture -equals X64
```

### Agent Comparison

| Feature | Microsoft-Hosted | Self-Hosted |
|---|---|---|
| **Maintenance** | Fully managed by Microsoft | You manage updates and patching |
| **Clean Environment** | Fresh VM per job (no state) | Persistent (caches survive across jobs) |
| **Build Speed** | Cold start every job | Warm caches improve build times |
| **Cost** | Free tier (1800 min/month), then per-minute | Free unlimited minutes; you pay for infra |
| **Customization** | Use pre-installed tools or install in pipeline | Full control over installed software |
| **Network Access** | Public internet only | Can access internal networks and resources |
| **Parallel Jobs** | 1 free (up to 10 paid) | 1 free (unlimited with self-hosted) |
| **Scale** | Auto-scales to demand | Manual or VMSS-based auto-scaling |

---

## Templates

Templates enable reuse of pipeline logic across multiple pipelines. They support parameters for customization and are the primary mechanism for enforcing organizational standards.

```
Template Types
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌────────────────┐   Reuse entire deployment stages
  │ Stage Template │   across pipelines (build, test,
  └────────────────┘   deploy patterns)

  ┌────────────────┐   Reuse job definitions (build
  │ Job Template   │   jobs, test jobs, security scan
  └────────────────┘   jobs)

  ┌────────────────┐   Reuse step sequences (install
  │ Step Template  │   tools, run tests, publish
  └────────────────┘   results)

  ┌────────────────┐   Reuse variable definitions
  │ Variable       │   (environment configs, shared
  │ Template       │   settings)
  └────────────────┘
```

### Stage Templates

Stage templates encapsulate entire stages for reuse:

```yaml
# templates/deploy-stage.yml
parameters:
  - name: environment
    type: string
  - name: azureSubscription
    type: string
  - name: appName
    type: string

stages:
  - stage: Deploy_${{ parameters.environment }}
    displayName: 'Deploy to ${{ parameters.environment }}'
    jobs:
      - deployment: Deploy
        environment: ${{ parameters.environment }}
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: ${{ parameters.azureSubscription }}
                    appName: ${{ parameters.appName }}
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
```

```yaml
# azure-pipelines.yml — consuming the stage template
trigger:
  - main

stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: dotnet publish -o $(Build.ArtifactStagingDirectory)
          - publish: $(Build.ArtifactStagingDirectory)
            artifact: 'drop'

  - template: templates/deploy-stage.yml
    parameters:
      environment: 'staging'
      azureSubscription: 'my-azure-connection'
      appName: 'my-app-staging'

  - template: templates/deploy-stage.yml
    parameters:
      environment: 'production'
      azureSubscription: 'my-azure-connection'
      appName: 'my-app-production'
```

### Job Templates

Job templates encapsulate job definitions:

```yaml
# templates/build-job.yml
parameters:
  - name: buildConfiguration
    type: string
    default: 'Release'
  - name: projects
    type: string
    default: '**/*.csproj'

jobs:
  - job: Build
    displayName: 'Build (${{ parameters.buildConfiguration }})'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - task: DotNetCoreCLI@2
        displayName: 'Restore'
        inputs:
          command: 'restore'
          projects: ${{ parameters.projects }}

      - task: DotNetCoreCLI@2
        displayName: 'Build'
        inputs:
          command: 'build'
          projects: ${{ parameters.projects }}
          arguments: '--configuration ${{ parameters.buildConfiguration }} --no-restore'
```

```yaml
# azure-pipelines.yml — consuming the job template
stages:
  - stage: Build
    jobs:
      - template: templates/build-job.yml
        parameters:
          buildConfiguration: 'Release'
          projects: 'src/**/*.csproj'
```

### Step Templates

Step templates are the most granular level of reuse:

```yaml
# templates/steps/install-dotnet.yml
parameters:
  - name: version
    type: string
    default: '8.0.x'

steps:
  - task: UseDotNet@2
    displayName: 'Install .NET SDK ${{ parameters.version }}'
    inputs:
      packageType: 'sdk'
      version: ${{ parameters.version }}

  - script: dotnet --info
    displayName: 'Display .NET info'
```

```yaml
# azure-pipelines.yml — consuming the step template
steps:
  - template: templates/steps/install-dotnet.yml
    parameters:
      version: '8.0.x'

  - script: dotnet build
    displayName: 'Build application'
```

### Variable Templates

Variable templates centralize variable definitions:

```yaml
# templates/variables/production.yml
variables:
  environment: 'production'
  azureSubscription: 'prod-azure-connection'
  appServicePlan: 'P1v3'
  minInstances: 2
  maxInstances: 10
```

```yaml
# azure-pipelines.yml — consuming the variable template
variables:
  - template: templates/variables/production.yml
  - name: buildConfiguration
    value: 'Release'

steps:
  - script: echo "Deploying to $(environment) with $(appServicePlan)"
```

### Template Expressions

Templates support compile-time expressions using `${{ }}` syntax:

```yaml
# templates/conditional-steps.yml
parameters:
  - name: runSecurityScan
    type: boolean
    default: true
  - name: environments
    type: object
    default:
      - name: staging
        vmImage: 'ubuntu-latest'
      - name: production
        vmImage: 'ubuntu-latest'

stages:
  - ${{ each env in parameters.environments }}:
    - stage: Deploy_${{ env.name }}
      displayName: 'Deploy to ${{ env.name }}'
      jobs:
        - deployment: Deploy
          environment: ${{ env.name }}
          pool:
            vmImage: ${{ env.vmImage }}
          strategy:
            runOnce:
              deploy:
                steps:
                  - script: echo "Deploying to ${{ env.name }}"

                  - ${{ if eq(parameters.runSecurityScan, true) }}:
                    - script: echo "Running security scan"
                      displayName: 'Security Scan'
```

---

## Environments and Approvals

Environments represent deployment targets (such as staging or production) and provide traceability, approval gates, and deployment history.

### Environment Resources

Environments can target specific resource types:

```yaml
# Deployment to a Kubernetes environment resource
jobs:
  - deployment: DeployToK8s
    environment:
      name: 'production'
      resourceType: Kubernetes
      resourceName: 'my-aks-namespace'
    strategy:
      runOnce:
        deploy:
          steps:
            - task: KubernetesManifest@1
              inputs:
                action: 'deploy'
                manifests: 'k8s/*.yml'
```

| Resource Type | Description |
|---|---|
| **Kubernetes** | Namespace in a Kubernetes cluster |
| **Virtual Machine** | VM registered as a deployment target |
| **Generic** | Default; no specific resource mapping |

### Deployment Strategies

Deployment jobs support several built-in strategies:

```yaml
# RunOnce strategy (default)
strategy:
  runOnce:
    preDeploy:
      steps:
        - script: echo "Pre-deployment validation"
    deploy:
      steps:
        - script: echo "Deploying application"
    routeTraffic:
      steps:
        - script: echo "Routing traffic"
    postRouteTraffic:
      steps:
        - script: echo "Smoke tests after traffic routing"
    on:
      failure:
        steps:
          - script: echo "Deployment failed — rolling back"
      success:
        steps:
          - script: echo "Deployment succeeded"
```

```yaml
# Rolling strategy
strategy:
  rolling:
    maxParallel: 2            # Deploy to 2 targets at a time
    preDeploy:
      steps:
        - script: echo "Preparing deployment"
    deploy:
      steps:
        - script: echo "Deploying to $(Environment.ResourceName)"
    on:
      failure:
        steps:
          - script: echo "Rolling back on failure"
```

```yaml
# Canary strategy
strategy:
  canary:
    increments: [10, 20]     # Route 10%, then 20% of traffic
    preDeploy:
      steps:
        - script: echo "Canary pre-deployment"
    deploy:
      steps:
        - script: echo "Deploying canary ($(Strategy.CycleName))"
    routeTraffic:
      steps:
        - script: echo "Routing $(Strategy.Increment)% traffic to canary"
    postRouteTraffic:
      steps:
        - script: echo "Validating canary health"
    on:
      failure:
        steps:
          - script: echo "Canary failed — routing all traffic to stable"
      success:
        steps:
          - script: echo "Promoting canary to stable"
```

| Strategy | Description | Best For |
|---|---|---|
| **runOnce** | Deploy to all targets at once | Simple deployments, single-target environments |
| **rolling** | Deploy in batches with `maxParallel` | VM-based deployments, gradual rollout |
| **canary** | Route incremental traffic to new version | Risk-sensitive production deployments |

### Gates and Checks

Environment checks add approval and validation gates before deployment:

```
Environment Checks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Pipeline Run
       │
       ▼
  ┌──────────────────┐
  │ Environment:     │
  │ production       │
  │                  │
  │ Checks:          │
  │ ✅ Approval      │ ── Manual approval from designated reviewers
  │ ✅ Branch Control│ ── Only allow deployments from main branch
  │ ✅ Business Hours│ ── Only deploy Mon-Fri 9am-5pm
  │ ✅ Azure Monitor │ ── Check no active alerts
  │ ✅ REST API      │ ── Call external validation endpoint
  │ ✅ Template      │ ── Require extends from approved template
  └──────────────────┘
       │
       ▼ (all checks pass)
  ┌──────────────────┐
  │ Deployment Job   │
  │ Runs             │
  └──────────────────┘
```

Available check types:

| Check Type | Description |
|---|---|
| **Approvals** | One or more users must approve before deployment proceeds |
| **Branch Control** | Restrict which branches can deploy to the environment |
| **Business Hours** | Only allow deployments during specified time windows |
| **Azure Monitor Alerts** | Gate on the absence of active Azure Monitor alerts |
| **REST API (Invoke)** | Call an external API and check the response |
| **Azure Function** | Invoke an Azure Function for custom validation logic |
| **Required Template** | Enforce that the pipeline extends from an approved template |
| **Exclusive Lock** | Only one deployment to the environment at a time |

---

## Azure Artifacts

Azure Artifacts provides integrated package management for your development workflow. It supports multiple package types and can serve as both a private registry and a caching proxy for public packages.

### Feed Types

| Feed Scope | Visibility | Use Case |
|---|---|---|
| **Project-scoped** | Visible within a single project | Team-specific packages |
| **Organization-scoped** | Visible across all projects in the org | Shared packages across teams |
| **Public feeds** | Visible to everyone (for public projects) | Open-source packages |

### NuGet Feeds

```yaml
# Restore and push NuGet packages
steps:
  - task: NuGetAuthenticate@1
    displayName: 'Authenticate to NuGet feed'

  - task: DotNetCoreCLI@2
    displayName: 'Restore from private feed'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'
      feedsToUse: 'select'
      vstsFeed: 'my-project/my-nuget-feed'

  - task: DotNetCoreCLI@2
    displayName: 'Pack NuGet package'
    inputs:
      command: 'pack'
      packagesToPack: 'src/MyLibrary/*.csproj'
      versioningScheme: 'byBuildNumber'

  - task: NuGetCommand@2
    displayName: 'Push to feed'
    inputs:
      command: 'push'
      publishVstsFeed: 'my-project/my-nuget-feed'
      packagesToPush: '$(Build.ArtifactStagingDirectory)/**/*.nupkg'
```

### npm Feeds

```yaml
# Publish npm packages to Azure Artifacts
steps:
  - task: npmAuthenticate@0
    displayName: 'Authenticate to npm feed'
    inputs:
      workingFile: '.npmrc'

  - script: |
      npm ci
      npm run build
      npm publish
    displayName: 'Build and publish npm package'
```

The `.npmrc` file points to the Azure Artifacts feed:

```ini
registry=https://pkgs.dev.azure.com/myorg/myproject/_packaging/my-npm-feed/npm/registry/
always-auth=true
```

### Maven Feeds

```yaml
# Publish Maven packages
steps:
  - task: MavenAuthenticate@0
    displayName: 'Authenticate to Maven feed'
    inputs:
      artifactsFeeds: 'my-maven-feed'

  - task: Maven@4
    displayName: 'Build and deploy Maven package'
    inputs:
      mavenPomFile: 'pom.xml'
      goals: 'deploy'
      publishJUnitResults: true
      testResultsFiles: '**/surefire-reports/TEST-*.xml'
```

### Python Feeds

```yaml
# Publish Python packages
steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.12'

  - script: |
      pip install build twine
      python -m build
    displayName: 'Build Python package'

  - task: TwineAuthenticate@1
    displayName: 'Authenticate to Python feed'
    inputs:
      artifactFeed: 'my-project/my-python-feed'

  - script: |
      twine upload -r my-python-feed --config-file $(PYPIRC_PATH) dist/*
    displayName: 'Publish to feed'
```

### Universal Packages

Universal Packages are a generic package type for any file or folder:

```yaml
# Publish and download Universal Packages
steps:
  - task: UniversalPackages@0
    displayName: 'Publish Universal Package'
    inputs:
      command: 'publish'
      publishDirectory: '$(Build.ArtifactStagingDirectory)/output'
      feedsToUsePublish: 'internal'
      vstsFeedPublish: 'my-project/my-universal-feed'
      vstsFeedPackagePublish: 'my-tool'
      versionOption: 'patch'

  - task: UniversalPackages@0
    displayName: 'Download Universal Package'
    inputs:
      command: 'download'
      downloadDirectory: '$(System.DefaultWorkingDirectory)/tools'
      feedsToUse: 'internal'
      vstsFeed: 'my-project/my-universal-feed'
      vstsFeedPackage: 'my-tool'
      vstsPackageVersion: '*'
```

---

## Multi-Stage Pipelines

Multi-stage pipelines combine build, test, and deployment into a single YAML definition, providing a complete view of the delivery pipeline.

### Build-Test-Deploy Pattern

```yaml
# Multi-stage pipeline with parallel test jobs
trigger:
  - main

variables:
  buildConfiguration: 'Release'

stages:
  # ── Stage 1: Build ──────────────────────────────
  - stage: Build
    displayName: 'Build'
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'publish'
              projects: 'src/MyApp/*.csproj'
              arguments: '-c $(buildConfiguration) -o $(Build.ArtifactStagingDirectory)'
          - publish: $(Build.ArtifactStagingDirectory)
            artifact: 'app'

  # ── Stage 2: Test (parallel jobs) ───────────────
  - stage: Test
    displayName: 'Test'
    dependsOn: Build
    jobs:
      - job: UnitTests
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'test'
              projects: 'tests/Unit/**/*.csproj'

      - job: IntegrationTests
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'test'
              projects: 'tests/Integration/**/*.csproj'

      - job: SecurityScan
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: echo "Running security scan..."
            displayName: 'SAST Scan'

  # ── Stage 3: Deploy Staging ─────────────────────
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Test
    jobs:
      - deployment: Staging
        environment: 'staging'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: 'app'
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'azure-connection'
                    appName: 'myapp-staging'
                    package: '$(Pipeline.Workspace)/app/**/*.zip'

  # ── Stage 4: Deploy Production ──────────────────
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Production
        environment: 'production'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: 'app'
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'azure-connection'
                    appName: 'myapp-production'
                    package: '$(Pipeline.Workspace)/app/**/*.zip'
```

### Stage Dependencies and Conditions

Stages can express complex dependency graphs:

```yaml
stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - script: echo Build

  - stage: TestA
    dependsOn: Build
    jobs:
      - job: Test
        steps:
          - script: echo TestA

  - stage: TestB
    dependsOn: Build
    jobs:
      - job: Test
        steps:
          - script: echo TestB

  # Fan-in: Deploy depends on both TestA and TestB
  - stage: Deploy
    dependsOn:
      - TestA
      - TestB
    condition: |
      and(
        succeeded('TestA'),
        succeeded('TestB')
      )
    jobs:
      - job: Deploy
        steps:
          - script: echo Deploy
```

```
Stage Dependency Graph
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

          ┌───────┐
          │ Build │
          └───┬───┘
         ╱         ╲
    ┌───────┐   ┌───────┐
    │ TestA │   │ TestB │     ← Fan-out (parallel)
    └───┬───┘   └───┬───┘
         ╲         ╱
          ┌───────┐
          │Deploy │               ← Fan-in (both must pass)
          └───────┘
```

### Parallel Stages

Independent stages run in parallel automatically when they share the same (or no) dependency:

```yaml
stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - script: echo Build

  # These three stages run in parallel after Build
  - stage: LintCheck
    dependsOn: Build
    jobs:
      - job: Lint
        steps:
          - script: echo Lint

  - stage: SecurityScan
    dependsOn: Build
    jobs:
      - job: Scan
        steps:
          - script: echo Security Scan

  - stage: UnitTests
    dependsOn: Build
    jobs:
      - job: Test
        steps:
          - script: echo Unit Tests
```

---

## Integration with Azure Services

Azure Pipelines provides first-class integration with Azure services through dedicated tasks and service connections.

### Azure Kubernetes Service (AKS)

```yaml
# Deploy to AKS with manifest files
stages:
  - stage: DeployToAKS
    jobs:
      - deployment: DeployApp
        environment:
          name: 'aks-production'
          resourceType: Kubernetes
          resourceName: 'my-namespace'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  displayName: 'Create image pull secret'
                  inputs:
                    action: 'createSecret'
                    connectionType: 'azureResourceManager'
                    azureSubscriptionConnection: 'azure-connection'
                    azureResourceGroup: 'my-aks-rg'
                    kubernetesCluster: 'my-aks-cluster'
                    secretType: 'dockerRegistry'
                    dockerRegistryEndpoint: 'my-acr-connection'
                    secretName: 'acr-secret'

                - task: KubernetesManifest@1
                  displayName: 'Deploy to AKS'
                  inputs:
                    action: 'deploy'
                    connectionType: 'azureResourceManager'
                    azureSubscriptionConnection: 'azure-connection'
                    azureResourceGroup: 'my-aks-rg'
                    kubernetesCluster: 'my-aks-cluster'
                    namespace: 'my-namespace'
                    manifests: |
                      k8s/deployment.yml
                      k8s/service.yml
                    containers: |
                      myregistry.azurecr.io/myapp:$(Build.BuildId)
                    imagePullSecrets: |
                      acr-secret
```

### Azure App Service

```yaml
# Deploy to Azure App Service with slot swap
stages:
  - stage: Deploy
    jobs:
      - deployment: DeployAppService
        environment: 'production'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                # Deploy to staging slot first
                - task: AzureWebApp@1
                  displayName: 'Deploy to staging slot'
                  inputs:
                    azureSubscription: 'azure-connection'
                    appType: 'webAppLinux'
                    appName: 'my-web-app'
                    deployToSlotOrASE: true
                    resourceGroupName: 'my-rg'
                    slotName: 'staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'

                # Swap staging slot to production
                - task: AzureAppServiceManage@0
                  displayName: 'Swap staging to production'
                  inputs:
                    azureSubscription: 'azure-connection'
                    action: 'Swap Slots'
                    webAppName: 'my-web-app'
                    resourceGroupName: 'my-rg'
                    sourceSlot: 'staging'
                    targetSlot: 'production'
```

### Azure Functions

```yaml
# Build and deploy Azure Functions
stages:
  - stage: Build
    jobs:
      - job: BuildFunctions
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '8.0.x'

          - task: DotNetCoreCLI@2
            displayName: 'Build Functions'
            inputs:
              command: 'publish'
              projects: 'src/MyFunctions/*.csproj'
              arguments: '-c Release -o $(Build.ArtifactStagingDirectory)'

          - publish: $(Build.ArtifactStagingDirectory)
            artifact: 'functions'

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: DeployFunctions
        environment: 'production'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureFunctionApp@2
                  displayName: 'Deploy Azure Functions'
                  inputs:
                    azureSubscription: 'azure-connection'
                    appType: 'functionAppLinux'
                    appName: 'my-function-app'
                    package: '$(Pipeline.Workspace)/functions/**/*.zip'
                    runtimeStack: 'DOTNET-ISOLATED|8.0'
```

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Overview | Core CI/CD concepts, pipeline anatomy, and tool landscape |
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Build automation, testing, artifact management, and quality gates |
| [02-CONTINUOUS-DELIVERY.md](02-CONTINUOUS-DELIVERY.md) | Continuous Delivery | Deployment pipelines, release strategies, and environment management |
| [03-GITHUB-ACTIONS.md](03-GITHUB-ACTIONS.md) | GitHub Actions | Workflow syntax, runners, secrets, and reusable workflows |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Azure DevOps Pipelines documentation |
