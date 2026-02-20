# Jenkins

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Jenkins Overview](#jenkins-overview)
   - [History and Evolution](#history-and-evolution)
   - [Architecture](#architecture)
   - [Master/Agent Model](#masteragent-model)
3. [Pipeline as Code](#pipeline-as-code)
   - [Why Pipeline as Code](#why-pipeline-as-code)
   - [Declarative vs Scripted Pipelines](#declarative-vs-scripted-pipelines)
   - [Jenkinsfile](#jenkinsfile)
4. [Declarative Pipeline Syntax](#declarative-pipeline-syntax)
   - [pipeline](#pipeline)
   - [agent](#agent)
   - [stages and stage](#stages-and-stage)
   - [steps](#steps)
   - [post](#post)
   - [environment](#environment)
   - [options](#options)
   - [parameters](#parameters)
   - [triggers](#triggers)
   - [when](#when)
   - [tools](#tools)
   - [input](#input)
5. [Scripted Pipeline](#scripted-pipeline)
   - [Groovy DSL Fundamentals](#groovy-dsl-fundamentals)
   - [node and stage Blocks](#node-and-stage-blocks)
   - [Flow Control in Scripted Pipelines](#flow-control-in-scripted-pipelines)
6. [Jenkinsfile Best Practices](#jenkinsfile-best-practices)
   - [Keep Pipelines Readable](#keep-pipelines-readable)
   - [Fail Fast](#fail-fast)
   - [Idempotent Builds](#idempotent-builds)
   - [Avoid Hardcoded Values](#avoid-hardcoded-values)
7. [Shared Libraries](#shared-libraries)
   - [Library Structure](#library-structure)
   - [Global vs Folder-Level Libraries](#global-vs-folder-level-libraries)
   - [Versioning Shared Libraries](#versioning-shared-libraries)
   - [@Library Annotation](#library-annotation)
   - [Writing Custom Steps](#writing-custom-steps)
   - [Shared Library Example](#shared-library-example)
8. [Plugins Ecosystem](#plugins-ecosystem)
   - [Key Plugins](#key-plugins)
   - [Plugin Management Best Practices](#plugin-management-best-practices)
9. [Agents and Executors](#agents-and-executors)
   - [Labels and Node Selection](#labels-and-node-selection)
   - [Docker Agents](#docker-agents)
   - [Kubernetes Agents](#kubernetes-agents)
   - [Ephemeral Agents](#ephemeral-agents)
   - [Agent Comparison](#agent-comparison)
10. [Credentials Management](#credentials-management)
    - [Credential Types](#credential-types)
    - [Credential Scopes](#credential-scopes)
    - [withCredentials](#withcredentials)
    - [Credentials Best Practices](#credentials-best-practices)
11. [Multibranch Pipelines](#multibranch-pipelines)
    - [Branch Discovery](#branch-discovery)
    - [Pull Request Detection](#pull-request-detection)
    - [Organization Folders](#organization-folders)
12. [Parallel Execution and Matrix Builds](#parallel-execution-and-matrix-builds)
    - [Parallel Stages](#parallel-stages)
    - [Matrix Directive](#matrix-directive)
    - [failFast and Parallel Control](#failfast-and-parallel-control)
13. [Jenkins Configuration as Code (JCasC)](#jenkins-configuration-as-code-jcasc)
    - [JCasC Overview](#jcasc-overview)
    - [Configuration File Structure](#configuration-file-structure)
    - [Managing JCasC in Version Control](#managing-jcasc-in-version-control)
14. [Jenkins in Docker and Kubernetes](#jenkins-in-docker-and-kubernetes)
    - [Running Jenkins in Docker](#running-jenkins-in-docker)
    - [Jenkins on Kubernetes](#jenkins-on-kubernetes)
    - [Helm Chart Deployment](#helm-chart-deployment)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Jenkins** — the most widely adopted open-source automation server for continuous integration and continuous delivery (CI/CD). It covers Jenkins architecture, Pipeline as Code with Jenkinsfiles, shared libraries for reusable pipeline logic, the plugin ecosystem, agent management, and modern deployment patterns using Docker and Kubernetes.

Jenkins has been a cornerstone of CI/CD since its inception as Hudson in 2004. While newer platforms have emerged, Jenkins remains dominant in enterprises due to its extensibility, massive plugin ecosystem, and ability to orchestrate virtually any automation workflow.

```
Jenkins Pipeline Lifecycle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Jenkinsfile              Jenkins Controller          Agents
  ┌──────────────┐         ┌──────────────────┐       ┌──────────┐
  │ pipeline {   │         │  Parses          │       │ Execute  │
  │   agent any  │ ──────▶ │  Schedules       │ ────▶ │ Build    │
  │   stages {   │         │  Orchestrates    │       │ Test     │
  │     ...      │         │  Reports         │       │ Deploy   │
  │   }          │         └──────────────────┘       └──────────┘
  └──────────────┘
```

### Target Audience

- **Developers** writing Jenkinsfiles to automate build, test, and deployment workflows
- **DevOps Engineers** designing shared libraries and managing Jenkins infrastructure
- **Platform Engineers** operating Jenkins at scale with Docker/Kubernetes agents and JCasC
- **SREs** maintaining Jenkins availability, performance, and security in production

### Scope

- Jenkins architecture: controller, agents, executors, and the master/agent model
- Pipeline as Code: Declarative and Scripted pipeline syntax with practical examples
- Shared libraries: structure, versioning, and reusable pipeline steps
- Plugin ecosystem: essential plugins and management strategies
- Agent configuration: static agents, Docker agents, and Kubernetes pods
- Credentials management, multibranch pipelines, and parallel execution
- Jenkins Configuration as Code (JCasC) for reproducible environments
- Running Jenkins in Docker and Kubernetes

---

## Jenkins Overview

### History and Evolution

Jenkins began life as **Hudson**, created by Kohsuke Kawaguchi at Sun Microsystems in 2004. After a governance dispute following Oracle's acquisition of Sun, the community forked Hudson and renamed it **Jenkins** in 2011. Since then, Jenkins has become the most widely used automation server in the world.

| Year | Milestone |
|---|---|
| 2004 | Hudson created at Sun Microsystems |
| 2005 | Hudson 1.0 released as open source |
| 2010 | Oracle acquires Sun Microsystems |
| 2011 | Community forks Hudson → Jenkins |
| 2014 | Jenkins Workflow plugin (predecessor to Pipeline) |
| 2016 | Jenkins 2.0 — Pipeline as first-class citizen, improved UX |
| 2017 | Blue Ocean UI, Declarative Pipeline syntax |
| 2019 | Jenkins Configuration as Code (JCasC) 1.0 |
| 2022 | Jenkins requires Java 11+, Spring Security update |
| 2024 | Jenkins requires Java 17+, continued modernization |

### Architecture

Jenkins uses a **controller-agent architecture** where the controller (historically called "master") manages configuration, scheduling, and the web UI, while agents execute the actual build workloads.

```
                         Jenkins Architecture
┌────────────────────────────────────────────────────────────────┐
│                     Jenkins Controller                         │
│                                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Web UI   │  │ REST API │  │ Scheduler│  │  Plugin Mgr  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Job/Pipeline Definitions                 │  │
│  │        Build History  │  Credential Store  │  Logs       │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────┬──────────────────┬──────────────────┬───────────────┘
           │ JNLP / SSH       │ JNLP / SSH       │ JNLP / SSH
           ▼                  ▼                  ▼
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │   Agent 1    │   │   Agent 2    │   │   Agent 3    │
  │              │   │              │   │              │
  │ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │
  │ │Executor 1│ │   │ │Executor 1│ │   │ │Executor 1│ │
  │ │Executor 2│ │   │ │Executor 2│ │   │ │Executor 2│ │
  │ └──────────┘ │   │ └──────────┘ │   │ └──────────┘ │
  │  Linux/amd64 │   │  Windows     │   │  Linux/arm64 │
  └──────────────┘   └──────────────┘   └──────────────┘
```

**Key components:**

| Component | Responsibility |
|---|---|
| **Controller** | Hosts the web UI, REST API, job definitions, build history, and plugin management. Schedules builds and dispatches them to agents. |
| **Agent** | A machine (physical, virtual, or container) that connects to the controller and executes build steps. Each agent can run one or more executors. |
| **Executor** | A single thread on an agent that runs a build. An agent with 2 executors can run 2 concurrent builds. |
| **Node** | A generic term for any machine in the Jenkins environment (controller or agent). |
| **Workspace** | A directory on the agent's filesystem where Jenkins checks out source code and runs build steps. |

### Master/Agent Model

The controller (master) delegates build execution to agents via one of several connection protocols:

| Protocol | Direction | Use Case |
|---|---|---|
| **SSH** | Controller → Agent | Linux/macOS agents with SSH access |
| **JNLP (Inbound)** | Agent → Controller | Agents behind firewalls, Kubernetes pods |
| **JNLP (WebSocket)** | Agent → Controller | Agents behind proxies, newer Jenkins versions |
| **Built-in node** | Local | Small setups only — **not recommended for production** |

> **Best Practice:** Never run builds on the controller node in production. Always delegate to dedicated agents for security and resource isolation.

---

## Pipeline as Code

### Why Pipeline as Code

Before Jenkins 2.0, jobs were configured through the web UI using "Freestyle" projects. This approach had significant limitations:

| Aspect | Freestyle (UI-Configured) | Pipeline as Code |
|---|---|---|
| **Version control** | Configuration stored in Jenkins XML | Jenkinsfile lives in the repository alongside code |
| **Code review** | No review process for pipeline changes | Pull request workflow applies to pipeline changes |
| **Reproducibility** | Difficult to recreate if Jenkins fails | Pipeline definition is in Git — fully recoverable |
| **Branching** | One job per branch, manually created | Multibranch pipeline auto-discovers branches |
| **Complexity** | Limited to linear build steps | Full programming constructs: loops, conditionals, functions |
| **Reusability** | Copy-paste between jobs | Shared libraries and functions |

### Declarative vs Scripted Pipelines

Jenkins offers two pipeline syntaxes:

```
Pipeline Syntax Comparison
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Declarative                         Scripted
  ┌─────────────────────┐            ┌─────────────────────┐
  │ pipeline {          │            │ node {              │
  │   agent any         │            │   stage('Build') {  │
  │   stages {          │            │     sh 'make'       │
  │     stage('Build') {│            │   }                 │
  │       steps {       │            │   stage('Test') {   │
  │         sh 'make'   │            │     sh 'make test'  │
  │       }             │            │   }                 │
  │     }               │            │ }                   │
  │   }                 │            │                     │
  │ }                   │            │                     │
  └─────────────────────┘            └─────────────────────┘
  ● Structured, opinionated          ● Full Groovy power
  ● Easier to learn                  ● Maximum flexibility
  ● Built-in validation              ● No structural guard rails
  ● Blue Ocean compatible            ● Harder to maintain
```

| Feature | Declarative | Scripted |
|---|---|---|
| **Syntax wrapper** | `pipeline { }` | `node { }` |
| **Learning curve** | Lower — structured, guided | Higher — requires Groovy knowledge |
| **Flexibility** | Constrained to defined directives | Unrestricted Groovy programming |
| **Validation** | Syntax checked before execution | Errors found at runtime |
| **Blue Ocean support** | Full editor and visualization | Visualization only |
| **Error handling** | `post { }` block with conditions | `try/catch/finally` blocks |
| **Recommended for** | Most pipelines | Complex orchestration, dynamic logic |

> **Recommendation:** Start with Declarative pipelines. Use Scripted pipelines only when Declarative syntax cannot express your requirements. You can embed Scripted code inside a Declarative pipeline using the `script { }` step.

### Jenkinsfile

A **Jenkinsfile** is a text file containing the pipeline definition. It is committed to the root of the source code repository and treated like any other source file — versioned, reviewed, and audited.

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
        stage('Test') {
            steps {
                sh './gradlew test'
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {
        always {
            junit '**/build/test-results/**/*.xml'
        }
        failure {
            mail to: 'team@example.com',
                 subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Check: ${env.BUILD_URL}"
        }
    }
}
```

---

## Declarative Pipeline Syntax

Declarative pipelines follow a strict, pre-defined structure. Every Declarative pipeline must be enclosed in a `pipeline { }` block.

```
Declarative Pipeline Structure
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

pipeline {
  ├── agent           (where to run)
  ├── environment     (env variables)
  ├── options         (pipeline options)
  ├── parameters      (user inputs)
  ├── triggers        (automatic triggers)
  ├── tools           (auto-install tools)
  ├── stages {
  │     ├── stage('...') {
  │     │     ├── agent      (override)
  │     │     ├── when       (conditional)
  │     │     ├── environment
  │     │     ├── steps { }
  │     │     └── post { }
  │     │   }
  │     └── stage('...') { ... }
  │   }
  └── post            (always/success/failure/...)
}
```

### pipeline

The top-level block that wraps the entire Declarative pipeline.

```groovy
pipeline {
    // All directives and stages go here
}
```

### agent

Specifies where the pipeline or a stage runs. The `agent` directive is **required** at the top level.

```groovy
// Run on any available agent
pipeline {
    agent any
    stages { /* ... */ }
}

// Run on an agent with a specific label
pipeline {
    agent { label 'linux && docker' }
    stages { /* ... */ }
}

// Run inside a Docker container
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /tmp:/tmp'
        }
    }
    stages { /* ... */ }
}

// Run inside a Kubernetes pod
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.9-eclipse-temurin-21
                    command: ['sleep']
                    args: ['infinity']
            '''
            defaultContainer 'maven'
        }
    }
    stages { /* ... */ }
}

// No global agent — each stage defines its own
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux' }
            steps { sh 'make' }
        }
        stage('Test on Windows') {
            agent { label 'windows' }
            steps { bat 'run-tests.bat' }
        }
    }
}
```

### stages and stage

`stages` contains one or more `stage` blocks. Each `stage` represents a logical phase of the pipeline (Build, Test, Deploy).

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh './gradlew assemble'
            }
        }
        stage('Unit Tests') {
            steps {
                sh './gradlew test'
            }
        }
        stage('Integration Tests') {
            steps {
                sh './gradlew integrationTest'
            }
        }
        stage('Deploy to Staging') {
            steps {
                sh './deploy.sh staging'
            }
        }
    }
}
```

### steps

The `steps` block contains the actual work — shell commands, plugin steps, and script blocks.

```groovy
steps {
    // Shell commands (Linux/macOS)
    sh 'echo "Hello from the build"'
    sh '''
        npm ci
        npm run build
        npm test
    '''

    // Windows batch commands
    bat 'echo Hello from Windows'

    // PowerShell
    powershell 'Write-Output "Hello from PowerShell"'

    // Built-in steps
    echo 'This is a Jenkins echo step'
    retry(3) {
        sh './flaky-script.sh'
    }
    timeout(time: 10, unit: 'MINUTES') {
        sh './long-running-test.sh'
    }
    sleep(time: 30, unit: 'SECONDS')

    // Embed Scripted Pipeline logic
    script {
        def version = sh(script: 'cat VERSION', returnStdout: true).trim()
        echo "Building version: ${version}"
    }
}
```

### post

The `post` block defines actions that run after all stages complete. Conditions determine which actions fire.

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'make' }
        }
    }
    post {
        always {
            // Runs regardless of outcome
            archiveArtifacts artifacts: 'build/**/*.jar', fingerprint: true
            junit 'build/reports/**/*.xml'
        }
        success {
            echo 'Build succeeded!'
            slackSend channel: '#builds', color: 'good',
                      message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo 'Build failed!'
            mail to: 'team@example.com',
                 subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Check console: ${env.BUILD_URL}"
        }
        unstable {
            echo 'Build is unstable (test failures).'
        }
        changed {
            echo 'Build status changed from previous run.'
        }
        cleanup {
            // Runs after all other post conditions — use for workspace cleanup
            cleanWs()
        }
    }
}
```

| Post Condition | Triggers When |
|---|---|
| `always` | Every build, regardless of status |
| `success` | Build completes successfully |
| `failure` | Build fails |
| `unstable` | Build is marked unstable (e.g., test failures) |
| `changed` | Build status differs from the previous build |
| `fixed` | Previous build was unsuccessful, current build succeeds |
| `regression` | Previous build was successful, current build fails |
| `aborted` | Build was manually aborted |
| `cleanup` | Runs after all other `post` conditions — guaranteed final step |

### environment

Defines environment variables for the entire pipeline or a single stage.

```groovy
pipeline {
    agent any

    environment {
        APP_NAME    = 'my-service'
        DEPLOY_ENV  = 'staging'
        // Retrieve credentials as environment variables
        DB_CREDS    = credentials('database-credentials')
        // DB_CREDS_USR and DB_CREDS_PSW are automatically created
    }

    stages {
        stage('Build') {
            environment {
                // Stage-level variables override pipeline-level
                BUILD_VERSION = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
            }
            steps {
                sh 'echo "Building ${APP_NAME} version ${BUILD_VERSION}"'
            }
        }
        stage('Deploy') {
            environment {
                DEPLOY_ENV = 'production'
            }
            steps {
                sh 'echo "Deploying to ${DEPLOY_ENV}"'
            }
        }
    }
}
```

### options

Pipeline-level and stage-level options control build behavior.

```groovy
pipeline {
    agent any
    options {
        timeout(time: 1, unit: 'HOURS')            // Abort if pipeline exceeds 1 hour
        retry(2)                                     // Retry entire pipeline up to 2 times
        timestamps()                                 // Prepend timestamps to console output
        buildDiscarder(logRotator(numToKeepStr: '10')) // Keep only last 10 builds
        disableConcurrentBuilds()                    // No parallel runs of same pipeline
        skipDefaultCheckout()                        // Do not auto-checkout SCM
        ansiColor('xterm')                           // Enable ANSI color in console
    }
    stages {
        stage('Build') {
            options {
                timeout(time: 15, unit: 'MINUTES')  // Stage-level timeout
            }
            steps { sh 'make' }
        }
    }
}
```

### parameters

Define user inputs that can be supplied when triggering a build manually.

```groovy
pipeline {
    agent any
    parameters {
        string(name: 'DEPLOY_TARGET',
               defaultValue: 'staging',
               description: 'Target environment')
        booleanParam(name: 'RUN_INTEGRATION_TESTS',
                     defaultValue: true,
                     description: 'Run integration test suite')
        choice(name: 'JAVA_VERSION',
               choices: ['21', '17', '11'],
               description: 'Java version for the build')
        text(name: 'RELEASE_NOTES',
             defaultValue: '',
             description: 'Release notes for this build')
        password(name: 'API_KEY',
                 description: 'API key for deployment')
    }
    stages {
        stage('Build') {
            steps {
                sh "echo Building with Java ${params.JAVA_VERSION}"
            }
        }
        stage('Integration Tests') {
            when {
                expression { params.RUN_INTEGRATION_TESTS }
            }
            steps {
                sh './gradlew integrationTest'
            }
        }
        stage('Deploy') {
            steps {
                sh "./deploy.sh ${params.DEPLOY_TARGET}"
            }
        }
    }
}
```

### triggers

Automate pipeline execution on a schedule or in response to external events.

```groovy
pipeline {
    agent any
    triggers {
        // Poll SCM every 5 minutes
        pollSCM('H/5 * * * *')

        // Run on a schedule (nightly at 2 AM)
        cron('H 2 * * *')

        // Trigger when upstream job completes
        upstream(upstreamProjects: 'build-library', threshold: hudson.model.Result.SUCCESS)
    }
    stages {
        stage('Build') {
            steps { sh 'make' }
        }
    }
}
```

### when

Conditional stage execution based on branch, environment, or custom expressions.

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps { sh './deploy.sh staging' }
        }
        stage('Deploy to Production') {
            when {
                allOf {
                    branch 'main'
                    environment name: 'DEPLOY_APPROVED', value: 'true'
                }
            }
            steps { sh './deploy.sh production' }
        }
        stage('PR Checks') {
            when {
                changeRequest()
            }
            steps { sh './run-pr-checks.sh' }
        }
        stage('Release') {
            when {
                tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
            }
            steps { sh './release.sh' }
        }
        stage('Custom Condition') {
            when {
                expression {
                    return env.GIT_BRANCH == 'main' && currentBuild.number > 10
                }
            }
            steps { sh 'echo "Custom condition met"' }
        }
    }
}
```

| When Condition | Description |
|---|---|
| `branch 'main'` | Matches branch name (supports wildcards: `'release/*'`) |
| `tag 'v*'` | Matches tag names |
| `environment name: 'X', value: 'Y'` | Matches environment variable |
| `expression { ... }` | Evaluates a Groovy expression |
| `changeRequest()` | Runs only for pull/merge requests |
| `changeset '**/*.java'` | Runs if matching files changed |
| `allOf { ... }` | All nested conditions must be true |
| `anyOf { ... }` | At least one nested condition must be true |
| `not { ... }` | Negates the nested condition |
| `triggeredBy 'TimerTrigger'` | Matches the trigger source |

### tools

Automatically installs and configures tools from Jenkins Global Tool Configuration.

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk-21'
        maven 'maven-3.9'
        nodejs 'node-20'
    }
    stages {
        stage('Build') {
            steps {
                sh 'java -version'
                sh 'mvn --version'
                sh 'node --version'
            }
        }
    }
}
```

### input

Pauses the pipeline and waits for human approval before proceeding.

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy to Production') {
            input {
                message 'Deploy to production?'
                ok 'Yes, deploy!'
                submitter 'admin,release-team'
                parameters {
                    string(name: 'RELEASE_VERSION', defaultValue: '1.0.0',
                           description: 'Version to deploy')
                }
            }
            steps {
                sh "echo Deploying version ${RELEASE_VERSION}"
            }
        }
    }
}
```

---

## Scripted Pipeline

Scripted pipelines use full **Groovy** programming language constructs, giving maximum flexibility at the cost of readability and validation.

### Groovy DSL Fundamentals

```groovy
// Variables and string interpolation
def appName = 'my-service'
def version = '1.2.3'
echo "Building ${appName} version ${version}"

// Lists and iteration
def environments = ['dev', 'staging', 'prod']
for (env in environments) {
    echo "Deploying to ${env}"
}

// Maps
def config = [
    appName: 'my-service',
    port   : 8080,
    debug  : false
]
echo "App: ${config.appName}, Port: ${config.port}"

// Closures
def runTests = { String suite ->
    sh "./gradlew test --tests ${suite}"
}
runTests('unit')
runTests('integration')
```

### node and stage Blocks

The `node` block allocates an agent and workspace. The `stage` block defines a logical pipeline phase.

```groovy
node('linux') {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        sh './gradlew clean assemble'
    }

    stage('Test') {
        try {
            sh './gradlew test'
        } finally {
            junit '**/build/test-results/**/*.xml'
        }
    }

    stage('Archive') {
        archiveArtifacts artifacts: 'build/libs/**/*.jar', fingerprint: true
    }
}
```

### Flow Control in Scripted Pipelines

```groovy
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        sh 'make'
    }

    // Conditional logic
    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            // Production deployment with approval
            input message: 'Deploy to production?', ok: 'Deploy'
            sh './deploy.sh production'
        } else if (env.BRANCH_NAME == 'develop') {
            sh './deploy.sh staging'
        } else {
            echo 'Skipping deploy for feature branch'
        }
    }

    // Error handling
    stage('Integration Tests') {
        try {
            sh './run-integration-tests.sh'
        } catch (Exception e) {
            currentBuild.result = 'UNSTABLE'
            echo "Integration tests failed: ${e.message}"
        } finally {
            junit 'test-results/**/*.xml'
        }
    }

    // Retry with timeout
    stage('Deploy with Retry') {
        retry(3) {
            timeout(time: 5, unit: 'MINUTES') {
                sh './deploy.sh'
            }
        }
    }
}
```

---

## Jenkinsfile Best Practices

### Keep Pipelines Readable

```groovy
// GOOD — clear stage names, minimal nesting
pipeline {
    agent any
    stages {
        stage('Build')  { steps { sh './gradlew build' } }
        stage('Test')   { steps { sh './gradlew test' } }
        stage('Deploy') { steps { sh './deploy.sh' } }
    }
}

// AVOID — deeply nested script blocks, unclear structure
pipeline {
    agent any
    stages {
        stage('Everything') {
            steps {
                script {
                    // Dozens of lines of Groovy logic...
                }
            }
        }
    }
}
```

### Fail Fast

Place quick validation stages first so builds fail early and provide fast feedback.

```groovy
pipeline {
    agent any
    stages {
        stage('Lint')           { steps { sh './gradlew checkstyleMain' } }
        stage('Compile')        { steps { sh './gradlew compileJava' } }
        stage('Unit Tests')     { steps { sh './gradlew test' } }
        stage('Integration')    { steps { sh './gradlew integrationTest' } }
        stage('Deploy')         { steps { sh './deploy.sh' } }
    }
}
```

### Idempotent Builds

Ensure that running the same pipeline multiple times produces the same result. Avoid side effects that accumulate.

```groovy
stage('Deploy') {
    steps {
        // GOOD — idempotent: creates or updates
        sh 'kubectl apply -f k8s/deployment.yaml'

        // AVOID — not idempotent: fails if resource already exists
        // sh 'kubectl create -f k8s/deployment.yaml'
    }
}
```

### Avoid Hardcoded Values

```groovy
// GOOD — use parameters and environment variables
pipeline {
    agent any
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE    = "${REGISTRY}/my-app:${env.BUILD_NUMBER}"
    }
    stages {
        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE} ."
            }
        }
    }
}
```

---

## Shared Libraries

Shared libraries allow you to extract common pipeline logic into a reusable Git repository, keeping Jenkinsfiles concise and consistent across projects.

### Library Structure

```
(root)
├── vars/                          # Global variables / custom steps
│   ├── buildApp.groovy            # Callable as buildApp() in pipelines
│   ├── buildApp.txt               # Help text for buildApp (optional)
│   ├── deployService.groovy
│   └── notifySlack.groovy
├── src/                           # Groovy source (class-based code)
│   └── com/
│       └── example/
│           └── pipeline/
│               ├── Docker.groovy
│               └── Kubernetes.groovy
├── resources/                     # Non-Groovy files (configs, templates)
│   └── com/
│       └── example/
│           └── pipeline/
│               └── deployment.yaml
└── README.md
```

| Directory | Purpose | Access Pattern |
|---|---|---|
| `vars/` | Global variables exposed as pipeline steps | Called directly: `buildApp()` |
| `src/` | Groovy classes with full OOP support | Imported: `import com.example.pipeline.Docker` |
| `resources/` | Static files loaded at runtime | `libraryResource('com/example/pipeline/deployment.yaml')` |

### Global vs Folder-Level Libraries

| Scope | Configuration Location | Available To |
|---|---|---|
| **Global** | Manage Jenkins → System → Global Pipeline Libraries | All pipelines on the instance |
| **Folder-level** | Folder → Configure → Pipeline Libraries | Pipelines within that folder only |
| **Pipeline-level** | `@Library` annotation in Jenkinsfile | The specific pipeline only |

### Versioning Shared Libraries

Libraries are loaded from a Git repository. You specify a version (branch, tag, or commit SHA):

```groovy
// Use a specific branch
@Library('my-shared-lib@main') _

// Use a semantic version tag
@Library('my-shared-lib@v2.1.0') _

// Use a specific commit
@Library('my-shared-lib@a1b2c3d') _

// Load multiple libraries
@Library(['my-shared-lib@v2.0', 'other-lib@main']) _
```

> **Best Practice:** Pin shared libraries to tags or specific commits for production pipelines. Use branch references only in development.

### @Library Annotation

The `@Library` annotation loads a shared library into the pipeline. The trailing `_` is required when no import follows.

```groovy
// Load and use global variables from vars/
@Library('my-shared-lib@v2.0') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                buildApp(language: 'java', jdkVersion: '21')
            }
        }
        stage('Deploy') {
            steps {
                deployService(environment: 'staging', region: 'us-east-1')
            }
        }
        stage('Notify') {
            steps {
                notifySlack(channel: '#deployments', status: 'SUCCESS')
            }
        }
    }
}
```

### Writing Custom Steps

Custom steps are defined in `vars/` as Groovy scripts. Each file defines a `call()` method that becomes the step invocation.

```groovy
// vars/buildApp.groovy
def call(Map config = [:]) {
    def language   = config.get('language', 'java')
    def jdkVersion = config.get('jdkVersion', '17')

    pipeline {
        agent {
            docker {
                image "eclipse-temurin:${jdkVersion}"
            }
        }
        stages {
            stage('Compile') {
                steps {
                    sh './gradlew compileJava'
                }
            }
            stage('Test') {
                steps {
                    sh './gradlew test'
                }
            }
            stage('Package') {
                steps {
                    sh './gradlew jar'
                    archiveArtifacts artifacts: 'build/libs/**/*.jar'
                }
            }
        }
    }
}
```

```groovy
// vars/notifySlack.groovy
def call(Map config = [:]) {
    def channel = config.get('channel', '#builds')
    def status  = config.get('status', currentBuild.currentResult)
    def color   = status == 'SUCCESS' ? 'good' : 'danger'
    def message = "${env.JOB_NAME} #${env.BUILD_NUMBER} — ${status}\n${env.BUILD_URL}"

    slackSend(channel: channel, color: color, message: message)
}
```

### Shared Library Example

A complete example showing class-based code in `src/`:

```groovy
// src/com/example/pipeline/Docker.groovy
package com.example.pipeline

class Docker implements Serializable {
    def steps

    Docker(steps) {
        this.steps = steps
    }

    def buildImage(String imageName, String tag = 'latest') {
        steps.sh "docker build -t ${imageName}:${tag} ."
    }

    def pushImage(String imageName, String tag = 'latest', String registry) {
        steps.sh "docker push ${registry}/${imageName}:${tag}"
    }

    def scanImage(String imageName, String tag = 'latest') {
        steps.sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${imageName}:${tag}"
    }
}
```

Using the class in a pipeline:

```groovy
@Library('my-shared-lib@v2.0') _
import com.example.pipeline.Docker

pipeline {
    agent any
    stages {
        stage('Build Image') {
            steps {
                script {
                    def docker = new Docker(this)
                    docker.buildImage('my-app', env.BUILD_NUMBER)
                    docker.scanImage('my-app', env.BUILD_NUMBER)
                    docker.pushImage('my-app', env.BUILD_NUMBER, 'registry.example.com')
                }
            }
        }
    }
}
```

---

## Plugins Ecosystem

Jenkins' extensibility is driven by its plugin architecture. With over 1,800 plugins available, virtually any tool or service can be integrated.

### Key Plugins

| Plugin | Purpose | Use Case |
|---|---|---|
| **Pipeline** | Enables Pipeline as Code | Core plugin for Jenkinsfile support |
| **Blue Ocean** | Modern UI for pipelines | Visual pipeline editor and run visualization |
| **Docker Pipeline** | Docker integration in pipelines | `agent { docker { ... } }` syntax |
| **Kubernetes** | Dynamic pod-based agents | Ephemeral build agents in Kubernetes clusters |
| **Credentials** | Secure secret storage | Manage passwords, tokens, SSH keys, certificates |
| **Credentials Binding** | Expose credentials in builds | `withCredentials([...])` step |
| **Git** | Git SCM integration | Checkout, polling, branch discovery |
| **GitHub Branch Source** | GitHub integration | Multibranch pipelines, PR detection, webhooks |
| **JUnit** | Test result reporting | `junit` step for test result visualization |
| **Slack Notification** | Slack messaging | `slackSend` step for build notifications |
| **Configuration as Code** | Jenkins config in YAML | JCasC for reproducible Jenkins setups |
| **Warnings Next Generation** | Static analysis reporting | Aggregate reports from linters and analyzers |
| **Pipeline Utility Steps** | File operations in pipelines | `readJSON`, `readYaml`, `writeFile`, `findFiles` |
| **Timestamper** | Timestamp console output | `timestamps()` option in pipelines |
| **Workspace Cleanup** | Clean workspaces | `cleanWs()` step in post blocks |

### Plugin Management Best Practices

```
Plugin Management Strategy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌───────────────────────────────────────────┐
  │         plugins.txt / plugin-catalog      │
  │                                           │
  │  pipeline:latest                          │
  │  docker-workflow:latest                   │
  │  kubernetes:latest                        │
  │  credentials-binding:latest               │
  │  configuration-as-code:latest             │
  └─────────────────┬─────────────────────────┘
                    │
                    ▼
  ┌───────────────────────────────────────────┐
  │     jenkins-plugin-cli (install tool)     │
  │     or Docker image with pre-installed    │
  │     plugins                               │
  └───────────────────────────────────────────┘
```

- **Pin plugin versions** in production to avoid unexpected changes
- **Test plugin upgrades** in a staging Jenkins instance before rolling to production
- **Minimize plugin count** — each plugin adds attack surface and upgrade complexity
- **Use `plugins.txt`** for Docker-based Jenkins installations to declare plugins declaratively

```text
# plugins.txt — used with jenkins-plugin-cli
pipeline-model-definition:2.2198.v1a_5df1a_027a_f
docker-workflow:572.v950f58993843
kubernetes:4246.v5a_12b_1fe120e
credentials-binding:677.vdc9c38cb_254d
configuration-as-code:1810.v9b_c30a_249a_4c
git:5.2.2
junit:1265.v65b_14fa_f12f0
slack:684.v833089650554
```

---

## Agents and Executors

### Labels and Node Selection

Labels are tags assigned to agents that allow pipelines to target specific machines or capabilities.

```groovy
// Run on any agent with both labels
agent { label 'linux && java21' }

// Run on agents matching either label
agent { label 'docker || podman' }

// Run on a specific named agent
agent { label 'build-server-01' }
```

### Docker Agents

The Docker Pipeline plugin allows running each stage inside a container, providing clean, reproducible build environments.

```groovy
pipeline {
    agent none
    stages {
        stage('Build Frontend') {
            agent {
                docker {
                    image 'node:20-alpine'
                    args '-v $HOME/.npm:/root/.npm'  // Cache npm packages
                }
            }
            steps {
                sh 'npm ci && npm run build'
            }
        }
        stage('Build Backend') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                    args '-v $HOME/.m2:/root/.m2'  // Cache Maven dependencies
                }
            }
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Build Docker Image') {
            agent { label 'docker' }
            steps {
                sh 'docker build -t my-app:${BUILD_NUMBER} .'
            }
        }
    }
}
```

### Kubernetes Agents

The Kubernetes plugin dynamically provisions pods as Jenkins agents. Each build runs in an ephemeral pod that is destroyed after completion.

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                metadata:
                  labels:
                    app: jenkins-agent
                spec:
                  containers:
                  - name: maven
                    image: maven:3.9-eclipse-temurin-21
                    command: ['sleep']
                    args: ['infinity']
                    resources:
                      requests:
                        cpu: "500m"
                        memory: "512Mi"
                      limits:
                        cpu: "2"
                        memory: "2Gi"
                    volumeMounts:
                    - name: maven-cache
                      mountPath: /root/.m2
                  - name: docker
                    image: docker:27-dind
                    securityContext:
                      privileged: true
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                  volumes:
                  - name: maven-cache
                    persistentVolumeClaim:
                      claimName: maven-cache-pvc
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            '''
            defaultContainer 'maven'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t my-app:${BUILD_NUMBER} .'
                }
            }
        }
    }
}
```

### Ephemeral Agents

Ephemeral agents are created on demand and destroyed after the build completes. This model offers strong isolation and consistent environments.

```
Ephemeral Agent Lifecycle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Build triggered
       │
       ▼
  ┌──────────────┐
  │ Provision    │──── Spin up Docker container or K8s pod
  │ Agent        │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Execute      │──── Run build steps in clean environment
  │ Build        │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Destroy      │──── Remove container/pod, reclaim resources
  │ Agent        │
  └──────────────┘
```

### Agent Comparison

| Agent Type | Provisioning | Isolation | Startup Time | Cost Model |
|---|---|---|---|---|
| **Static (SSH/JNLP)** | Pre-provisioned VMs | Shared filesystem | Instant | Always running |
| **Docker** | On-demand containers | Container-level | Seconds | Per-build |
| **Kubernetes** | On-demand pods | Pod-level | 10–30 seconds | Per-build |
| **Cloud (EC2/GCE)** | On-demand VMs | Full VM | 1–5 minutes | Per-build |

---

## Credentials Management

### Credential Types

| Type | Description | Example Use |
|---|---|---|
| **Username with Password** | Username and password pair | Git, Docker registry, database |
| **Secret text** | Single secret string | API tokens, webhook secrets |
| **Secret file** | A file uploaded as a secret | Kubeconfig, service account keys |
| **SSH Username with Private Key** | SSH key pair | Git over SSH, server access |
| **Certificate** | PKCS#12 certificate | TLS client authentication |
| **AWS Credentials** | Access Key ID + Secret Key | AWS API calls |

### Credential Scopes

| Scope | Visibility |
|---|---|
| **Global** | Available to all jobs and pipelines on the Jenkins instance |
| **System** | Available only to Jenkins system operations (e.g., agent connections) — not exposed to builds |
| **Folder** | Available only to jobs within a specific folder |

### withCredentials

The `withCredentials` step binds credentials to environment variables within a block.

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                // Username/password credential
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push my-app:latest
                    '''
                }

                // Secret text
                withCredentials([string(
                    credentialsId: 'slack-webhook',
                    variable: 'WEBHOOK_URL'
                )]) {
                    sh 'curl -X POST -d "Build done" ${WEBHOOK_URL}'
                }

                // SSH key
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'deploy-ssh-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        ssh -i "${SSH_KEY}" ${SSH_USER}@server.example.com \
                            "cd /opt/app && ./deploy.sh"
                    '''
                }

                // Secret file
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh 'kubectl --kubeconfig=${KUBECONFIG} apply -f k8s/'
                }
            }
        }
    }
}
```

### Credentials Best Practices

- **Never print credentials** in console output — Jenkins masks them, but avoid `echo $SECRET` patterns
- **Use the narrowest scope** — folder-level credentials are preferred over global
- **Rotate credentials regularly** and update them in Jenkins Credential Store
- **Audit credential usage** via the Credentials plugin's usage tracking
- **Use short-lived tokens** (e.g., OIDC) instead of long-lived static credentials when possible

---

## Multibranch Pipelines

Multibranch pipelines automatically discover branches (and pull requests) in a repository and create a pipeline job for each branch that contains a `Jenkinsfile`.

### Branch Discovery

```
Multibranch Pipeline Discovery
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Git Repository                    Jenkins Multibranch
  ┌─────────────────┐              ┌─────────────────────┐
  │ main            │──────────────│ main (pipeline)     │
  │ develop         │──────────────│ develop (pipeline)  │
  │ feature/login   │──────────────│ feature/login       │
  │ release/2.0     │──────────────│ release/2.0         │
  │ PR #42          │──────────────│ PR-42 (pipeline)    │
  └─────────────────┘              └─────────────────────┘

  Scan triggers:
  • Periodic scan (e.g., every 5 minutes)
  • Webhook from GitHub/GitLab/Bitbucket
  • Manual "Scan Now" button
```

**Configuration options:**

- **Branch sources** — GitHub, Bitbucket, GitLab, or generic Git
- **Discover branches** — all branches, only branches with PRs, or only branches without PRs
- **Filter by name** — include/exclude branches by regex pattern
- **Jenkinsfile path** — defaults to `Jenkinsfile` in the repository root, but can be customized
- **Build strategies** — build all branches, only specific branches, or only branches with changes

### Pull Request Detection

Multibranch pipelines can detect pull requests and run validation builds:

```groovy
// Jenkinsfile with PR-specific behavior
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
        stage('PR Checks') {
            when { changeRequest() }
            steps {
                sh './gradlew checkstyleMain spotbugsMain'
                sh './gradlew jacocoTestReport'
                publishChecks name: 'Code Quality', summary: 'All checks passed'
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
                not { changeRequest() }
            }
            steps {
                sh './deploy.sh production'
            }
        }
    }
}
```

### Organization Folders

Organization Folders automatically scan an entire GitHub or Bitbucket organization and create multibranch pipeline jobs for every repository that contains a `Jenkinsfile`.

```
Organization Folder Structure
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GitHub Organization: my-org
  ├── repo-a (has Jenkinsfile) ──▶ Multibranch Pipeline: repo-a
  │   ├── main
  │   ├── develop
  │   └── PR-15
  ├── repo-b (has Jenkinsfile) ──▶ Multibranch Pipeline: repo-b
  │   ├── main
  │   └── feature/api
  ├── repo-c (no Jenkinsfile)  ──▶ (skipped)
  └── repo-d (has Jenkinsfile) ──▶ Multibranch Pipeline: repo-d
      └── main
```

---

## Parallel Execution and Matrix Builds

### Parallel Stages

Run independent stages concurrently to reduce total pipeline time.

```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux' }
            steps { sh './gradlew build' }
        }
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    agent { label 'linux' }
                    steps { sh './gradlew test' }
                }
                stage('Integration Tests') {
                    agent { label 'linux' }
                    steps { sh './gradlew integrationTest' }
                }
                stage('Security Scan') {
                    agent { label 'linux' }
                    steps { sh './gradlew dependencyCheckAnalyze' }
                }
                stage('Lint') {
                    agent { label 'linux' }
                    steps { sh './gradlew checkstyleMain' }
                }
            }
        }
        stage('Deploy') {
            agent { label 'linux' }
            steps { sh './deploy.sh' }
        }
    }
}
```

```
Pipeline Execution Timeline
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Sequential             With Parallel
  ─────────              ──────────────
  Build   ████           Build   ████
  Unit    ██████          Test    ┌─ Unit    ██████
  Integ   ████████               ├─ Integ   ████████
  Scan    ██████                  ├─ Scan    ██████
  Lint    ███                     └─ Lint    ███
  Deploy  ████           Deploy  ████

  Total: ~31 min         Total: ~16 min
```

### Matrix Directive

The `matrix` directive creates a cartesian product of axes and runs a stage for each combination.

```groovy
pipeline {
    agent none
    stages {
        stage('Test Matrix') {
            matrix {
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'windows', 'macos'
                    }
                    axis {
                        name 'JAVA_VERSION'
                        values '17', '21'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'PLATFORM'
                            values 'macos'
                        }
                        axis {
                            name 'JAVA_VERSION'
                            values '17'
                        }
                    }
                }
                agent { label "${PLATFORM}" }
                stages {
                    stage('Build and Test') {
                        tools {
                            jdk "jdk-${JAVA_VERSION}"
                        }
                        steps {
                            sh 'java -version'
                            sh './gradlew clean test'
                        }
                    }
                }
            }
        }
    }
}
```

The above matrix produces these combinations (excluding macOS + Java 17):

| Combination | Platform | Java Version |
|---|---|---|
| 1 | linux | 17 |
| 2 | linux | 21 |
| 3 | windows | 17 |
| 4 | windows | 21 |
| 5 | macos | 21 |

### failFast and Parallel Control

```groovy
pipeline {
    agent any
    stages {
        stage('Parallel Tests') {
            failFast true  // Abort all parallel branches if any one fails
            parallel {
                stage('Test Suite A') {
                    steps { sh './test-a.sh' }
                }
                stage('Test Suite B') {
                    steps { sh './test-b.sh' }
                }
            }
        }
    }
}
```

---

## Jenkins Configuration as Code (JCasC)

### JCasC Overview

The **Configuration as Code** plugin allows you to define the entire Jenkins configuration in YAML files, eliminating manual UI configuration and enabling reproducible Jenkins environments.

```
Traditional Setup                 JCasC Setup
━━━━━━━━━━━━━━━━━                ━━━━━━━━━━━━━━━
Manual UI clicks                  jenkins.yaml in Git
Point-and-click config            Declarative YAML
Configuration drift               Version-controlled
Hard to reproduce                 Reproducible
Undocumented changes              Auditable changes
```

### Configuration File Structure

```yaml
# jenkins.yaml — JCasC configuration
jenkins:
  systemMessage: "Jenkins configured via JCasC"
  numExecutors: 0  # No builds on controller
  mode: EXCLUSIVE

  securityRealm:
    ldap:
      configurations:
        - server: "ldap.example.com"
          rootDN: "dc=example,dc=com"
          userSearch: "uid={0}"

  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            entries:
              - group: "jenkins-admins"
          - name: "developer"
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Read"
              - "Job/Workspace"
            entries:
              - group: "developers"

  nodes:
    - permanent:
        name: "build-agent-01"
        remoteFS: "/var/jenkins"
        launcher:
          ssh:
            host: "agent01.example.com"
            credentialsId: "agent-ssh-key"
            sshHostKeyVerificationStrategy:
              manuallyProvidedKeyVerificationStrategy:
                key: "ssh-ed25519 AAAA..."
        labelString: "linux docker java"
        numExecutors: 4

  globalNodeProperties:
    - envVars:
        env:
          - key: "JAVA_HOME"
            value: "/usr/lib/jvm/java-21"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "docker-hub"
              username: "${DOCKER_HUB_USER}"
              password: "${DOCKER_HUB_PASS}"
          - string:
              scope: GLOBAL
              id: "slack-token"
              secret: "${SLACK_TOKEN}"

unclassified:
  location:
    url: "https://jenkins.example.com/"
    adminAddress: "admin@example.com"

  globalLibraries:
    libraries:
      - name: "my-shared-lib"
        defaultVersion: "main"
        retriever:
          modernSCM:
            scm:
              git:
                remote: "https://github.com/my-org/jenkins-shared-lib.git"
                credentialsId: "github-token"

  slackNotifier:
    teamDomain: "my-team"
    tokenCredentialId: "slack-token"

tool:
  jdk:
    installations:
      - name: "jdk-21"
        properties:
          - installSource:
              installers:
                - adoptOpenJdkInstaller:
                    id: "jdk-21.0.2+13"
  maven:
    installations:
      - name: "maven-3.9"
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.9.6"
```

### Managing JCasC in Version Control

```
Repository Structure for Jenkins Infrastructure
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

jenkins-config/
├── jenkins.yaml              # Main JCasC config
├── credentials.yaml          # Credential definitions (secrets via env vars)
├── plugins.txt               # Plugin list with pinned versions
├── Dockerfile                # Custom Jenkins image
├── docker-compose.yaml       # Local Jenkins for testing
├── helm/
│   └── values.yaml           # Kubernetes Helm values
└── tests/
    └── test-config.sh        # Validate configuration
```

---

## Jenkins in Docker and Kubernetes

### Running Jenkins in Docker

```dockerfile
# Dockerfile for a custom Jenkins controller
FROM jenkins/jenkins:lts-jdk21

# Skip the setup wizard
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

# Install plugins
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

# Apply JCasC configuration
ENV CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs
COPY jenkins.yaml /var/jenkins_home/casc_configs/jenkins.yaml
```

```yaml
# docker-compose.yaml — Jenkins with Docker-in-Docker
services:
  jenkins:
    build: .
    ports:
      - "8080:8080"
      - "50000:50000"   # Agent JNLP port
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      DOCKER_HUB_USER: "${DOCKER_HUB_USER}"
      DOCKER_HUB_PASS: "${DOCKER_HUB_PASS}"
      SLACK_TOKEN: "${SLACK_TOKEN}"

volumes:
  jenkins_home:
```

### Jenkins on Kubernetes

```yaml
# kubernetes/jenkins-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: ci
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk21
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 50000
              name: jnlp
          env:
            - name: JAVA_OPTS
              value: "-Djenkins.install.runSetupWizard=false"
            - name: CASC_JENKINS_CONFIG
              value: "/var/jenkins_home/casc_configs"
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
            - name: casc-config
              mountPath: /var/jenkins_home/casc_configs
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 120
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-home-pvc
        - name: casc-config
          configMap:
            name: jenkins-casc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: ci
spec:
  type: ClusterIP
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: 8080
    - name: jnlp
      port: 50000
      targetPort: 50000
```

### Helm Chart Deployment

The official Jenkins Helm chart provides a production-ready deployment:

```bash
# Add the Jenkins Helm repository
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Install Jenkins with custom values
helm install jenkins jenkins/jenkins \
    --namespace ci \
    --create-namespace \
    --values values.yaml
```

```yaml
# values.yaml — Helm chart values for Jenkins on Kubernetes
controller:
  image:
    tag: "lts-jdk21"
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  installPlugins:
    - kubernetes:latest
    - pipeline-model-definition:latest
    - docker-workflow:latest
    - configuration-as-code:latest
    - credentials-binding:latest
    - git:latest
    - github-branch-source:latest
    - junit:latest
    - slack:latest
  JCasC:
    configScripts:
      main: |
        jenkins:
          systemMessage: "Jenkins on Kubernetes via Helm"
          numExecutors: 0
  serviceType: ClusterIP
  ingress:
    enabled: true
    hostName: jenkins.example.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
      - secretName: jenkins-tls
        hosts:
          - jenkins.example.com

agent:
  enabled: true
  podTemplates:
    java: |
      - name: java
        label: java
        containers:
          - name: maven
            image: maven:3.9-eclipse-temurin-21
            command: sleep
            args: infinity
            resourceRequestCpu: "500m"
            resourceRequestMemory: "512Mi"
    node: |
      - name: node
        label: node
        containers:
          - name: node
            image: node:20-alpine
            command: sleep
            args: infinity
            resourceRequestCpu: "500m"
            resourceRequestMemory: "512Mi"

persistence:
  enabled: true
  size: 50Gi
  storageClass: gp3
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
| [04-AZURE-DEVOPS.md](04-AZURE-DEVOPS.md) | Azure DevOps Pipelines | YAML pipelines, service connections, templates, and Azure integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Jenkins documentation — Jenkinsfile, shared libraries, pipeline as code |
