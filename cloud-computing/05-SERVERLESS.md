# Serverless Computing

## Table of Contents

- [Overview](#overview)
- [What is Serverless](#what-is-serverless)
- [Functions as a Service (FaaS)](#functions-as-a-service-faas)
- [Event Sources and Triggers](#event-sources-and-triggers)
- [Cold Starts](#cold-starts)
- [Serverless Patterns](#serverless-patterns)
- [Serverless Databases and Storage](#serverless-databases-and-storage)
- [Orchestration](#orchestration)
- [Testing and Debugging](#testing-and-debugging)
- [Serverless Security](#serverless-security)
- [Limitations and Trade-offs](#limitations-and-trade-offs)
- [Next Steps](#next-steps)
- [Version History](#version-history)

## Overview

### Target Audience

This guide is intended for developers, architects, and cloud engineers who want to understand serverless computing concepts, evaluate when serverless is appropriate, and learn best practices for building serverless applications on major cloud platforms.

### Scope

This document covers the fundamentals of serverless computing including Functions as a Service (FaaS), event-driven architectures, cold start optimization, common patterns, security considerations, and the trade-offs involved in adopting a serverless approach. Examples reference AWS, Azure, and Google Cloud Platform.

---

## What is Serverless

Serverless computing is a cloud execution model where the cloud provider dynamically manages the allocation and provisioning of servers. Despite the name, servers still exist — but the developer no longer needs to think about them. The provider handles all infrastructure concerns including scaling, patching, and availability.

### Key Characteristics

- **No server management** — No provisioning, maintaining, or administering servers. The cloud provider handles the underlying infrastructure entirely.
- **Automatic scaling** — Functions scale horizontally and automatically in response to demand, from zero to thousands of concurrent executions.
- **Pay-per-use pricing** — You are billed only for the compute time consumed. When your code is not running, you are not charged.
- **Event-driven execution** — Functions are triggered by events such as HTTP requests, database changes, file uploads, or scheduled timers.
- **Stateless by design** — Each function invocation is independent. State must be managed externally through databases, caches, or storage services.

### Serverless vs Containers vs Virtual Machines

| Aspect | Serverless | Containers | Virtual Machines |
|---|---|---|---|
| Provisioning | Fully managed | User manages orchestration | User manages OS and runtime |
| Scaling | Automatic, per-request | Auto-scaling with orchestrator | Manual or auto-scaling groups |
| Startup time | Milliseconds to seconds | Seconds | Minutes |
| Billing | Per invocation + duration | Per running container | Per running instance |
| Max execution time | Minutes (varies by provider) | Unlimited | Unlimited |
| State | Stateless | Stateful or stateless | Stateful |
| Customization | Limited to runtime options | Full container control | Full OS control |
| Operational overhead | Minimal | Moderate | High |

---

## Functions as a Service (FaaS)

Functions as a Service is the core compute model in serverless. Each function is a small, single-purpose piece of code that runs in response to an event. The three major cloud providers each offer their own FaaS platform.

### AWS Lambda

AWS Lambda was the first major FaaS offering, launched in 2014. It supports a wide range of runtimes and integrates deeply with the AWS ecosystem.

```python
# AWS Lambda handler example (Python)
import json

def handler(event, context):
    name = event.get("queryStringParameters", {}).get("name", "World")
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"message": f"Hello, {name}!"})
    }
```

### Azure Functions

Azure Functions integrates with the broader Azure ecosystem and supports multiple hosting plans including a consumption (serverless) plan.

```csharp
// Azure Functions example (C#)
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;

public static class HelloFunction
{
    [FunctionName("Hello")]
    public static IActionResult Run(
        [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req)
    {
        string name = req.Query["name"];
        return new OkObjectResult($"Hello, {name ?? "World"}!");
    }
}
```

### Google Cloud Functions

Google Cloud Functions offers a lightweight FaaS experience tightly integrated with Google Cloud services and Firebase.

```javascript
// Google Cloud Functions example (Node.js)
const functions = require('@google-cloud/functions-framework');

functions.http('hello', (req, res) => {
  const name = req.query.name || 'World';
  res.json({ message: `Hello, ${name}!` });
});
```

### FaaS Provider Comparison

| Feature | AWS Lambda | Azure Functions | Google Cloud Functions |
|---|---|---|---|
| Supported runtimes | Python, Node.js, Java, Go, .NET, Ruby, custom | C#, JavaScript, Python, Java, PowerShell, TypeScript | Node.js, Python, Go, Java, .NET, Ruby, PHP |
| Max execution time | 15 minutes | 10 min (Consumption), 60 min (Premium) | 9 minutes (1st gen), 60 min (2nd gen) |
| Max memory | 10,240 MB | 1,536 MB (Consumption), 14 GB (Premium) | 32 GB (2nd gen) |
| Triggers | 200+ AWS service integrations | HTTP, Timer, Queue, Blob, Cosmos DB, Event Hub | HTTP, Pub/Sub, Cloud Storage, Firestore |
| Deployment package size | 250 MB (unzipped) | No hard limit (Consumption: ~1 GB) | 100 MB (compressed), 500 MB (uncompressed) |
| Concurrency model | Per-invocation (up to 1000 default) | Per-instance (multiple concurrent executions) | Per-invocation (default) or per-instance |

---

## Event Sources and Triggers

Serverless functions are activated by events. Understanding the available event sources is essential for designing event-driven architectures.

### Common Event Types

- **HTTP requests** — The most common trigger. Functions act as API endpoints behind an API gateway or HTTP load balancer.
- **Message queues** — Functions process messages from queues like SQS, Azure Service Bus, or Pub/Sub for asynchronous workloads.
- **Storage events** — File uploads, modifications, or deletions in object storage trigger functions for processing pipelines.
- **Scheduled events** — Cron-like schedules trigger functions at regular intervals for batch jobs and maintenance tasks.
- **Database streams** — Changes in databases like DynamoDB Streams or Cosmos DB Change Feed trigger real-time processing.
- **IoT events** — Messages from IoT devices trigger functions for data ingestion and processing.

### Event Sources by Provider

| Event Source | AWS | Azure | GCP |
|---|---|---|---|
| HTTP | API Gateway, Function URL | HTTP Trigger, API Management | HTTP Trigger, API Gateway |
| Queue | SQS, SNS | Service Bus, Storage Queue | Pub/Sub |
| Storage | S3 events | Blob Storage trigger | Cloud Storage trigger |
| Database | DynamoDB Streams | Cosmos DB Change Feed | Firestore triggers |
| Schedule | EventBridge Scheduler | Timer trigger | Cloud Scheduler |
| Stream | Kinesis | Event Hubs | Dataflow |
| Authentication | Cognito triggers | N/A (via Event Grid) | Firebase Authentication |
| IoT | IoT Core rules | IoT Hub | IoT Core |

```yaml
# AWS SAM template — S3 trigger example
Resources:
  ImageProcessor:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      Runtime: python3.12
      Events:
        S3Upload:
          Type: S3
          Properties:
            Bucket: !Ref ImageBucket
            Events: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: suffix
                    Value: .jpg
```

---

## Cold Starts

### What Are Cold Starts?

A cold start occurs when a serverless function is invoked after a period of inactivity or when new instances are needed to handle increased load. During a cold start, the provider must allocate a container, download the function code, initialize the runtime, and execute any initialization logic before handling the request.

### Why Cold Starts Happen

1. **No active instances** — The function has not been invoked recently and all previous instances have been recycled.
2. **Scale-up events** — Current instances are fully utilized and new instances must be created to handle additional requests.
3. **Code deployment** — After deploying new code, existing warm instances are replaced with new ones.

### Impact on Latency

Cold starts can add anywhere from 100 milliseconds to several seconds of latency depending on the runtime, package size, and initialization complexity. This can be significant for user-facing APIs where consistent response times are expected.

### Cold Start Duration by Language

| Language | Typical Cold Start | Notes |
|---|---|---|
| Python | 200–500 ms | Fast startup, lightweight runtime |
| Node.js | 200–500 ms | Fast startup, common for API workloads |
| Go | 100–300 ms | Compiled binary, very fast cold starts |
| Rust | 100–300 ms | Compiled binary, similar to Go |
| Java | 1–5 seconds | JVM startup is heavy, GraalVM can reduce this |
| .NET | 500 ms–2 seconds | Improves significantly with AOT compilation |
| Ruby | 300–700 ms | Moderate cold start times |

### Mitigation Strategies

- **Provisioned concurrency** (AWS) / **Pre-warmed instances** (Azure Premium) — Keep a set number of instances initialized and ready to handle requests, eliminating cold starts for those instances.
- **Keep-warm scheduling** — Use a scheduled ping every few minutes to keep at least one instance warm. This is a cost-effective workaround but not a guarantee under load.
- **Smaller deployment packages** — Reduce the size of your deployment artifact. Fewer dependencies means faster initialization.
- **Lazy initialization** — Defer expensive setup operations (database connections, SDK clients) until they are actually needed, or initialize them outside the handler to reuse across invocations.
- **Language choice** — Choose compiled languages like Go or Rust for latency-sensitive workloads. Avoid Java unless using GraalVM native images or SnapStart.

```python
# Initialize outside the handler to reuse across warm invocations
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def handler(event, context):
    # This reuses the existing connection on warm invocations
    response = table.get_item(Key={'userId': event['userId']})
    return response.get('Item')
```

---

## Serverless Patterns

### API Backend

The most common serverless pattern. Functions serve as the backend for REST or GraphQL APIs, sitting behind an API gateway that handles routing, authentication, and rate limiting.

```
Client → API Gateway → Lambda Function → DynamoDB
```

### Event Processing

Functions consume events from queues or streams and process them asynchronously. This pattern decouples producers from consumers and naturally handles backpressure through queue depth.

```
Producer → SQS Queue → Lambda → Database
                     ↘ Dead Letter Queue (on failure)
```

### Scheduled Tasks

Functions triggered on a schedule replace traditional cron jobs. Useful for report generation, data cleanup, health checks, and periodic synchronization tasks.

```
CloudWatch Schedule (rate: 1 hour) → Lambda → Generate Report → S3
```

### Data Pipeline

Functions process data as it flows through a pipeline. Each stage transforms or enriches the data before passing it downstream. Object storage events often initiate the pipeline.

```
S3 Upload → Lambda (Validate) → Lambda (Transform) → Lambda (Load) → Database
```

### Fan-Out / Fan-In

A single event triggers multiple parallel function executions. Each function processes a subset of the work independently, and results are aggregated afterward. This pattern is effective for parallel processing of large datasets.

```
SNS Topic → Lambda Instance 1 ↘
          → Lambda Instance 2 → Aggregation → Result
          → Lambda Instance 3 ↗
```

---

## Serverless Databases and Storage

Serverless databases complement serverless compute by eliminating the need to provision or manage database capacity.

### Amazon DynamoDB

A fully managed NoSQL database that scales automatically. It offers single-digit millisecond performance and integrates natively with Lambda through DynamoDB Streams for change-driven processing.

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

def handler(event, context):
    table.put_item(Item={
        'orderId': event['orderId'],
        'customer': event['customer'],
        'total': event['total'],
        'status': 'pending'
    })
    return {'statusCode': 201, 'body': 'Order created'}
```

### Azure Cosmos DB (Serverless Mode)

Cosmos DB offers a serverless capacity mode where you pay only for the request units consumed. It provides multi-model support (document, graph, key-value) and global distribution with the Change Feed for event-driven processing.

### Google Firestore

Firestore is a serverless document database that integrates with Cloud Functions. It offers real-time synchronization, offline support for mobile clients, and automatic scaling.

### Storage Triggers

Object storage services (S3, Azure Blob Storage, Cloud Storage) can trigger functions on file events:

| Storage Service | Trigger Events | Common Use Cases |
|---|---|---|
| Amazon S3 | ObjectCreated, ObjectRemoved | Image processing, ETL, backups |
| Azure Blob Storage | BlobCreated, BlobDeleted | Document processing, archiving |
| Google Cloud Storage | Finalize, Delete, MetadataUpdate | Media transcoding, analysis |

---

## Orchestration

Individual functions handle single tasks well, but complex workflows require coordination between multiple functions. Orchestration services manage the sequencing, error handling, and state of multi-step workflows.

### AWS Step Functions

Step Functions uses a JSON-based Amazon States Language to define state machines. Each state can invoke a Lambda function, wait for input, branch on conditions, or run parallel tasks.

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:ValidateOrder",
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:ProcessPayment",
      "Next": "ShipOrder",
      "Catch": [{
        "ErrorEquals": ["PaymentFailed"],
        "Next": "NotifyCustomer"
      }]
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:ShipOrder",
      "End": true
    },
    "NotifyCustomer": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:NotifyCustomer",
      "End": true
    }
  }
}
```

### Azure Durable Functions

Durable Functions extends Azure Functions with stateful orchestrations using code rather than JSON configuration. Orchestrations are written in the same language as the functions they coordinate.

```csharp
[FunctionName("OrderOrchestrator")]
public static async Task RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<Order>();

    await context.CallActivityAsync("ValidateOrder", order);
    await context.CallActivityAsync("ProcessPayment", order);
    await context.CallActivityAsync("ShipOrder", order);
}
```

### Google Cloud Workflows

Workflows is a serverless orchestration service that connects Google Cloud and HTTP-based API services into automated workflows defined in YAML or JSON.

### When to Use Orchestration

- Workflows with multiple sequential or parallel steps that depend on each other
- Long-running processes that exceed single function timeout limits
- Workflows requiring built-in retry logic and error handling across steps
- Processes that need human approval steps or external wait conditions
- Scenarios where you need visibility into workflow execution state

---

## Testing and Debugging

Testing serverless applications requires a combination of local emulation, unit testing, and cloud-based observability tools.

### Local Testing Tools

- **AWS SAM CLI** — Emulates Lambda locally using Docker containers, supporting local API Gateway and event simulation.
- **Azure Functions Core Tools** — Runs Azure Functions locally with full trigger support including HTTP, Timer, and Queue triggers.
- **Google Functions Framework** — Provides a local development server for testing Cloud Functions with HTTP and event triggers.

```bash
# Local testing with AWS SAM CLI
sam local invoke MyFunction --event events/test-event.json

# Start a local API for testing HTTP-triggered functions
sam local start-api --port 3000

# Local testing with Azure Functions Core Tools
func start --port 7071

# Local testing with Google Functions Framework
npx @google-cloud/functions-framework --target=hello --port 8080
```

### Unit Testing

Write unit tests for your function handlers by treating them as regular functions. Mock external service calls to isolate the function logic.

```python
# test_handler.py
from unittest.mock import patch, MagicMock
from handler import handler

def test_handler_returns_user():
    mock_table = MagicMock()
    mock_table.get_item.return_value = {
        'Item': {'userId': '123', 'name': 'Alice'}
    }

    with patch('handler.table', mock_table):
        result = handler({'userId': '123'}, None)
        assert result['name'] == 'Alice'
        mock_table.get_item.assert_called_once_with(Key={'userId': '123'})
```

### Logging and Distributed Tracing

- **Structured logging** — Use JSON-formatted log output so logs can be queried and filtered efficiently in CloudWatch, Application Insights, or Cloud Logging.
- **Distributed tracing** — Use AWS X-Ray, Azure Application Insights, or Google Cloud Trace to visualize request flows across multiple functions and services.
- **Correlation IDs** — Pass a unique identifier through all function invocations in a workflow to correlate logs across services.

---

## Serverless Security

### Least Privilege Permissions

Every function should have an IAM role or managed identity with only the permissions it needs. Avoid shared roles across functions.

```yaml
# AWS SAM — scoped IAM policy for a function
Resources:
  ReadUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      Runtime: python3.12
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref UsersTable
```

### Input Validation

All function inputs must be validated. Functions are often exposed to the internet via API gateways, making them targets for injection attacks, oversized payloads, and malformed data.

```python
import json

def handler(event, context):
    body = json.loads(event.get('body', '{}'))

    email = body.get('email', '')
    if not email or '@' not in email or len(email) > 254:
        return {'statusCode': 400, 'body': 'Invalid email address'}

    name = body.get('name', '')
    if not name or len(name) > 100:
        return {'statusCode': 400, 'body': 'Invalid name'}

    # Proceed with validated input
    return {'statusCode': 200, 'body': 'OK'}
```

### Secrets Management

Never hardcode secrets in function code or environment variables. Use dedicated secrets services:

- **AWS Secrets Manager** or **SSM Parameter Store**
- **Azure Key Vault**
- **Google Secret Manager**

Cache secrets in memory during the function's warm lifetime to reduce API calls and latency.

### VPC Integration

When functions need to access private resources (databases, internal APIs), they can be deployed inside a VPC. Be aware that VPC-attached functions may experience additional cold start latency due to network interface provisioning, though this has been significantly reduced by providers in recent years.

---

## Limitations and Trade-offs

### Vendor Lock-in

Serverless functions rely heavily on provider-specific services and APIs. Migrating from one provider to another requires significant effort. Mitigation strategies include using abstraction layers, the Serverless Framework, or keeping business logic separate from provider-specific integration code.

### Cold Starts

As discussed in the [Cold Starts](#cold-starts) section, the initialization latency for new function instances can be problematic for latency-sensitive applications. This is an inherent characteristic of the serverless model.

### Execution Limits

| Limit | AWS Lambda | Azure Functions (Consumption) | Google Cloud Functions |
|---|---|---|---|
| Max execution time | 15 minutes | 10 minutes | 9 minutes (1st gen) |
| Max memory | 10 GB | 1.5 GB | 32 GB (2nd gen) |
| Max payload size | 6 MB (sync), 256 KB (async) | 100 MB | 10 MB (HTTP) |
| Temp disk storage | 10 GB (/tmp) | 500 MB | In-memory only |
| Concurrent executions | 1,000 (default, increasable) | 200 per instance | 1,000 (default) |

### Debugging Complexity

Debugging distributed serverless applications is harder than debugging monolithic applications. Requests flow through multiple functions and services, making it difficult to reproduce issues locally. Invest in structured logging, distributed tracing, and comprehensive monitoring from the start.

### Statelessness

Functions are stateless between invocations. Any data that needs to persist must be stored externally. This adds latency and complexity compared to in-memory state in long-running processes but improves scalability and fault tolerance.

### Cost at Scale

While serverless is cost-effective for sporadic and unpredictable workloads, sustained high-throughput workloads may be cheaper on reserved containers or virtual machines. Evaluate your workload profile before committing fully to serverless.

---

## Next Steps

| Topic | Resource | Description |
|---|---|---|
| AWS Lambda | [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/) | Official AWS Lambda developer guide |
| Azure Functions | [Azure Functions Documentation](https://learn.microsoft.com/azure/azure-functions/) | Official Azure Functions documentation |
| Cloud Functions | [Google Cloud Functions Docs](https://cloud.google.com/functions/docs) | Official Google Cloud Functions guide |
| Serverless Framework | [serverless.com](https://www.serverless.com/) | Multi-provider serverless deployment framework |
| AWS SAM | [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/) | Infrastructure as code for serverless on AWS |
| Step Functions | [AWS Step Functions Docs](https://docs.aws.amazon.com/step-functions/) | Serverless orchestration on AWS |
| Serverless Patterns | [serverlessland.com](https://serverlessland.com/) | Community patterns and best practices |
| Cold Start Analysis | [Mikhail Shilkov's Research](https://mikhail.io/serverless/coldstarts/aws/) | Detailed cold start benchmarks across providers |

---

## Version History

| Version | Date | Description |
|---|---|---|
| 1.0 | 2025 | Initial serverless computing documentation |
