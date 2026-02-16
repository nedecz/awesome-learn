# Cloud-Native Payment Systems with AWS

## Overview

This document covers cloud-native architectures for payment systems using AWS services. We'll explore serverless, microservices, and event-driven patterns with AWS-managed services.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [AWS Services for Payments](#aws-services-for-payments)
3. [DynamoDB vs RDS](#dynamodb-vs-rds)
4. [SQS Message Patterns](#sqs-message-patterns)
5. [Lambda Functions](#lambda-functions)
6. [EventBridge Architecture](#eventbridge-architecture)
7. [Step Functions Workflows](#step-functions-workflows)
8. [API Gateway](#api-gateway)
9. [Security & Compliance](#security--compliance)
10. [Cost Optimization](#cost-optimization)

---

## Architecture Overview

### Cloud-Native Payment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Applications                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  API Gateway (REST/HTTP API)                                 │
│  - Rate Limiting                                              │
│  - Authentication (Cognito/Lambda Authorizer)                │
│  - Request Validation                                         │
└────────────────────┬────────────────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  Lambda Function │    │  Lambda Function │
│  (Payments API)  │    │  (Webhooks)      │
└────────┬─────────┘    └────────┬─────────┘
         │                       │
         ▼                       ▼
┌──────────────────────────────────────────┐
│           DynamoDB Tables                 │
│  - Payments                               │
│  - PaymentEvents                          │
│  - IdempotencyKeys                        │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  SQS Queues                               │
│  - PaymentProcessingQueue                 │
│  - DeadLetterQueue                        │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  EventBridge                              │
│  - PaymentCompleted                       │
│  - PaymentFailed                          │
│  - RefundProcessed                        │
└────────┬─────────────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌─────────┐ ┌─────────┐
│  SNS    │ │  SES    │
│(Alerts) │ │(Email)  │
└─────────┘ └─────────┘
```

---

## AWS Services for Payments

### Service Selection Matrix

| Requirement | AWS Service | Alternative | When to Use |
|-------------|-------------|-------------|-------------|
| **API Layer** | API Gateway | ALB + Lambda | REST/WebSocket APIs |
| **Compute** | Lambda | ECS/Fargate | Event-driven, short tasks |
| **Primary DB** | RDS (PostgreSQL) | DynamoDB | ACID transactions critical |
| **NoSQL DB** | DynamoDB | DocumentDB | High-scale, key-value access |
| **Queues** | SQS | Kinesis | Async processing, decoupling |
| **Events** | EventBridge | SNS | Event routing, scheduling |
| **Workflows** | Step Functions | Custom orchestration | Multi-step processes |
| **Secrets** | Secrets Manager | Parameter Store | API keys, credentials |
| **Cache** | ElastiCache | DAX | Session data, hot data |
| **Storage** | S3 | EFS | Receipts, exports |
| **Monitoring** | CloudWatch | X-Ray | Logs, metrics, traces |

---

## DynamoDB vs RDS

### When to Use Each

**Use RDS (PostgreSQL/MySQL) for:**
- ✅ ACID transactions are critical
- ✅ Complex queries and joins
- ✅ Payment processing (primary database)
- ✅ Strong consistency required
- ✅ Relational data models

**Use DynamoDB for:**
- ✅ High-throughput operations (10K+ TPS)
- ✅ Simple key-value access patterns
- ✅ Event sourcing / audit logs
- ✅ Idempotency tracking
- ✅ Session data
- ✅ Global distribution required

### Hybrid Approach (Recommended)

```
RDS (Primary)          DynamoDB (Supporting)
├─ Payments            ├─ PaymentEvents (audit log)
├─ Refunds             ├─ IdempotencyKeys
├─ Customers           ├─ WebhookDeliveries
└─ PaymentMethods      └─ SessionData
```

### DynamoDB Table Design

**Payments Table:**

```csharp
public class PaymentDynamoDbTable
{
    // Single-table design
    public string PK { get; set; }  // PAYMENT#{PaymentId}
    public string SK { get; set; }  // METADATA or EVENT#{Timestamp}
    
    // GSI for queries
    public string GSI1PK { get; set; }  // ORDER#{OrderId}
    public string GSI1SK { get; set; }  // PAYMENT#{CreatedAt}
    
    // Attributes
    public string PaymentId { get; set; }
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string Status { get; set; }
    public long CreatedAt { get; set; }  // Unix timestamp
    public long TTL { get; set; }  // For automatic cleanup
}

// Table definition (CloudFormation/CDK)
{
    "TableName": "Payments",
    "KeySchema": [
        { "AttributeName": "PK", "KeyType": "HASH" },
        { "AttributeName": "SK", "KeyType": "RANGE" }
    ],
    "GlobalSecondaryIndexes": [
        {
            "IndexName": "GSI1",
            "KeySchema": [
                { "AttributeName": "GSI1PK", "KeyType": "HASH" },
                { "AttributeName": "GSI1SK", "KeyType": "RANGE" }
            ]
        }
    ],
    "BillingMode": "PAY_PER_REQUEST",
    "StreamSpecification": {
        "StreamEnabled": true,
        "StreamViewType": "NEW_AND_OLD_IMAGES"
    },
    "TimeToLiveSpecification": {
        "Enabled": true,
        "AttributeName": "TTL"
    }
}
```

**Access Patterns:**

```csharp
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.Model;

public class DynamoDbPaymentRepository
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private const string TableName = "Payments";
    
    // Pattern 1: Get payment by ID
    public async Task<Payment> GetByIdAsync(string paymentId)
    {
        var response = await _dynamoDb.GetItemAsync(new GetItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = $"PAYMENT#{paymentId}" },
                ["SK"] = new AttributeValue { S = "METADATA" }
            }
        });
        
        return MapToPayment(response.Item);
    }
    
    // Pattern 2: Get payments by order ID
    public async Task<List<Payment>> GetByOrderIdAsync(string orderId)
    {
        var response = await _dynamoDb.QueryAsync(new QueryRequest
        {
            TableName = TableName,
            IndexName = "GSI1",
            KeyConditionExpression = "GSI1PK = :orderId",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":orderId"] = new AttributeValue { S = $"ORDER#{orderId}" }
            }
        });
        
        return response.Items.Select(MapToPayment).ToList();
    }
    
    // Pattern 3: Create payment with event
    public async Task CreatePaymentAsync(Payment payment)
    {
        var timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        
        await _dynamoDb.TransactWriteItemsAsync(new TransactWriteItemsRequest
        {
            TransactItems = new List<TransactWriteItem>
            {
                // Write payment metadata
                new TransactWriteItem
                {
                    Put = new Put
                    {
                        TableName = TableName,
                        Item = new Dictionary<string, AttributeValue>
                        {
                            ["PK"] = new AttributeValue { S = $"PAYMENT#{payment.Id}" },
                            ["SK"] = new AttributeValue { S = "METADATA" },
                            ["GSI1PK"] = new AttributeValue { S = $"ORDER#{payment.OrderId}" },
                            ["GSI1SK"] = new AttributeValue { S = $"PAYMENT#{timestamp}" },
                            ["PaymentId"] = new AttributeValue { S = payment.Id },
                            ["OrderId"] = new AttributeValue { S = payment.OrderId },
                            ["Amount"] = new AttributeValue { N = payment.Amount.ToString() },
                            ["Currency"] = new AttributeValue { S = payment.Currency },
                            ["Status"] = new AttributeValue { S = payment.Status },
                            ["CreatedAt"] = new AttributeValue { N = timestamp.ToString() }
                        }
                    }
                },
                // Write creation event
                new TransactWriteItem
                {
                    Put = new Put
                    {
                        TableName = TableName,
                        Item = new Dictionary<string, AttributeValue>
                        {
                            ["PK"] = new AttributeValue { S = $"PAYMENT#{payment.Id}" },
                            ["SK"] = new AttributeValue { S = $"EVENT#{timestamp}" },
                            ["EventType"] = new AttributeValue { S = "PaymentCreated" },
                            ["EventData"] = new AttributeValue { S = JsonSerializer.Serialize(payment) }
                        }
                    }
                }
            }
        });
    }
    
    // Pattern 4: Query payment events (audit trail)
    public async Task<List<PaymentEvent>> GetEventsAsync(string paymentId)
    {
        var response = await _dynamoDb.QueryAsync(new QueryRequest
        {
            TableName = TableName,
            KeyConditionExpression = "PK = :paymentId AND begins_with(SK, :eventPrefix)",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":paymentId"] = new AttributeValue { S = $"PAYMENT#{paymentId}" },
                [":eventPrefix"] = new AttributeValue { S = "EVENT#" }
            }
        });
        
        return response.Items.Select(MapToPaymentEvent).ToList();
    }
}
```

### DynamoDB Streams for Event Sourcing

```csharp
// Lambda function triggered by DynamoDB Stream
public class DynamoDbStreamHandler
{
    public async Task HandleStreamAsync(DynamoDBEvent dynamoEvent)
    {
        foreach (var record in dynamoEvent.Records)
        {
            if (record.EventName == "INSERT" || record.EventName == "MODIFY")
            {
                var newImage = record.Dynamodb.NewImage;
                
                // Check if this is a payment status change
                if (newImage.ContainsKey("Status"))
                {
                    var paymentId = newImage["PaymentId"].S;
                    var newStatus = newImage["Status"].S;
                    var oldStatus = record.Dynamodb.OldImage?["Status"]?.S;
                    
                    if (oldStatus != newStatus)
                    {
                        // Publish event to EventBridge
                        await PublishStatusChangeEventAsync(paymentId, oldStatus, newStatus);
                    }
                }
            }
        }
    }
}
```

---

## SQS Message Patterns

### Standard Queue for Payment Processing

```csharp
using Amazon.SQS;
using Amazon.SQS.Model;

public class SqsPaymentQueue
{
    private readonly IAmazonSQS _sqs;
    private const string QueueUrl = "https://sqs.us-east-1.amazonaws.com/123456789/payment-processing";
    
    // Producer: Send payment to queue
    public async Task EnqueuePaymentAsync(PaymentRequest payment)
    {
        var messageBody = JsonSerializer.Serialize(payment);
        
        await _sqs.SendMessageAsync(new SendMessageRequest
        {
            QueueUrl = QueueUrl,
            MessageBody = messageBody,
            MessageAttributes = new Dictionary<string, MessageAttributeValue>
            {
                ["PaymentId"] = new MessageAttributeValue 
                { 
                    DataType = "String", 
                    StringValue = payment.PaymentId 
                },
                ["Priority"] = new MessageAttributeValue 
                { 
                    DataType = "Number", 
                    StringValue = payment.Amount > 1000 ? "1" : "2" 
                }
            },
            // Deduplication for FIFO queues
            MessageDeduplicationId = payment.IdempotencyKey,
            MessageGroupId = payment.OrderId // FIFO only
        });
    }
    
    // Consumer: Process payments from queue
    public async Task ProcessPaymentsAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            var response = await _sqs.ReceiveMessageAsync(new ReceiveMessageRequest
            {
                QueueUrl = QueueUrl,
                MaxNumberOfMessages = 10,
                WaitTimeSeconds = 20, // Long polling
                MessageAttributeNames = new List<string> { "All" },
                VisibilityTimeout = 300 // 5 minutes to process
            });
            
            var tasks = response.Messages.Select(async message =>
            {
                try
                {
                    var payment = JsonSerializer.Deserialize<PaymentRequest>(message.Body);
                    
                    // Process payment
                    await ProcessPaymentAsync(payment);
                    
                    // Delete from queue on success
                    await _sqs.DeleteMessageAsync(new DeleteMessageRequest
                    {
                        QueueUrl = QueueUrl,
                        ReceiptHandle = message.ReceiptHandle
                    });
                }
                catch (Exception ex)
                {
                    // Message will become visible again after VisibilityTimeout
                    // Or go to DLQ after max retries
                    Console.WriteLine($"Failed to process message: {ex.Message}");
                }
            });
            
            await Task.WhenAll(tasks);
        }
    }
}
```

### FIFO Queue for Order Preservation

```csharp
public class FifoPaymentQueue
{
    private readonly IAmazonSQS _sqs;
    private const string QueueUrl = "https://sqs.us-east-1.amazonaws.com/123456789/payment-processing.fifo";
    
    public async Task EnqueuePaymentAsync(PaymentRequest payment)
    {
        await _sqs.SendMessageAsync(new SendMessageRequest
        {
            QueueUrl = QueueUrl,
            MessageBody = JsonSerializer.Serialize(payment),
            
            // Required for FIFO
            MessageGroupId = payment.CustomerId, // Process same customer sequentially
            MessageDeduplicationId = payment.IdempotencyKey, // Prevent duplicates
            
            // Optional: Content-based deduplication
            // AWS will generate hash of message body
        });
    }
}
```

### Dead Letter Queue Setup

```json
{
  "RedrivePolicy": {
    "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789:payment-processing-dlq",
    "maxReceiveCount": 3
  }
}
```

**Process DLQ Messages:**

```csharp
public class DeadLetterQueueHandler
{
    private readonly IAmazonSQS _sqs;
    private readonly INotificationService _notifications;
    
    public async Task MonitorDlqAsync()
    {
        var response = await _sqs.ReceiveMessageAsync(new ReceiveMessageRequest
        {
            QueueUrl = DlqUrl,
            MaxNumberOfMessages = 10
        });
        
        if (response.Messages.Any())
        {
            // Alert on DLQ messages
            await _notifications.SendAlertAsync(
                $"Payment processing DLQ has {response.Messages.Count} messages");
            
            // Log for manual investigation
            foreach (var message in response.Messages)
            {
                await LogFailedPaymentAsync(message);
            }
        }
    }
}
```

### Batch Processing

```csharp
public async Task EnqueueBatchAsync(List<PaymentRequest> payments)
{
    // SQS supports up to 10 messages per batch
    var batches = payments.Chunk(10);
    
    foreach (var batch in batches)
    {
        var entries = batch.Select((p, index) => new SendMessageBatchRequestEntry
        {
            Id = index.ToString(),
            MessageBody = JsonSerializer.Serialize(p),
            MessageDeduplicationId = p.IdempotencyKey,
            MessageGroupId = p.OrderId
        }).ToList();
        
        await _sqs.SendMessageBatchAsync(new SendMessageBatchRequest
        {
            QueueUrl = QueueUrl,
            Entries = entries
        });
    }
}
```

---

## Lambda Functions

### Payment Processing Lambda

```csharp
using Amazon.Lambda.Core;
using Amazon.Lambda.APIGatewayEvents;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

public class PaymentLambdaHandler
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly IAmazonSQS _sqs;
    private readonly IPaymentGateway _gateway;
    
    public PaymentLambdaHandler()
    {
        _dynamoDb = new AmazonDynamoDBClient();
        _sqs = new AmazonSQSClient();
        _gateway = new StripeGateway();
    }
    
    // API Gateway handler
    public async Task<APIGatewayProxyResponse> HandleApiRequestAsync(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        context.Logger.LogLine($"Processing payment request: {request.RequestContext.RequestId}");
        
        try
        {
            var paymentRequest = JsonSerializer.Deserialize<PaymentRequest>(request.Body);
            
            // Validate request
            if (!ValidateRequest(paymentRequest, out var errors))
            {
                return new APIGatewayProxyResponse
                {
                    StatusCode = 400,
                    Body = JsonSerializer.Serialize(new { errors })
                };
            }
            
            // Create payment record
            var payment = new Payment
            {
                Id = Guid.NewGuid().ToString(),
                OrderId = paymentRequest.OrderId,
                Amount = paymentRequest.Amount,
                Currency = paymentRequest.Currency,
                Status = "Pending"
            };
            
            // Store in DynamoDB
            var repository = new DynamoDbPaymentRepository(_dynamoDb);
            await repository.CreatePaymentAsync(payment);
            
            // Queue for async processing
            await _sqs.SendMessageAsync(new SendMessageRequest
            {
                QueueUrl = Environment.GetEnvironmentVariable("PAYMENT_QUEUE_URL"),
                MessageBody = JsonSerializer.Serialize(payment)
            });
            
            return new APIGatewayProxyResponse
            {
                StatusCode = 202, // Accepted
                Body = JsonSerializer.Serialize(new 
                { 
                    paymentId = payment.Id,
                    status = payment.Status
                }),
                Headers = new Dictionary<string, string>
                {
                    ["Content-Type"] = "application/json"
                }
            };
        }
        catch (Exception ex)
        {
            context.Logger.LogLine($"Error: {ex.Message}");
            
            return new APIGatewayProxyResponse
            {
                StatusCode = 500,
                Body = JsonSerializer.Serialize(new { error = "Internal server error" })
            };
        }
    }
    
    // SQS handler
    public async Task HandleSqsMessageAsync(
        SQSEvent sqsEvent,
        ILambdaContext context)
    {
        foreach (var record in sqsEvent.Records)
        {
            try
            {
                var payment = JsonSerializer.Deserialize<Payment>(record.Body);
                
                context.Logger.LogLine($"Processing payment: {payment.Id}");
                
                // Process with payment gateway
                var result = await _gateway.ProcessPaymentAsync(new PaymentRequest
                {
                    Amount = payment.Amount,
                    Currency = payment.Currency,
                    PaymentMethodId = payment.PaymentMethodId
                });
                
                // Update status
                payment.Status = result.IsSuccess ? "Completed" : "Failed";
                payment.ProviderTransactionId = result.TransactionId;
                
                var repository = new DynamoDbPaymentRepository(_dynamoDb);
                await repository.UpdatePaymentAsync(payment);
                
                // Publish event
                if (result.IsSuccess)
                {
                    await PublishEventAsync("PaymentCompleted", payment);
                }
            }
            catch (Exception ex)
            {
                context.Logger.LogLine($"Error processing payment: {ex.Message}");
                throw; // Will retry or send to DLQ
            }
        }
    }
}
```

### Lambda Best Practices

**1. Cold Start Optimization:**

```csharp
public class OptimizedLambdaHandler
{
    // Initialize outside handler (reused across invocations)
    private static readonly IAmazonDynamoDB _dynamoDb = new AmazonDynamoDBClient();
    private static readonly HttpClient _httpClient = new HttpClient();
    
    static OptimizedLambdaHandler()
    {
        // Warm up connections
        _ = _dynamoDb.ListTablesAsync();
    }
    
    public async Task<APIGatewayProxyResponse> Handler(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        // Handler code uses static clients
    }
}
```

**2. Environment Variables:**

```csharp
private static readonly string TableName = Environment.GetEnvironmentVariable("TABLE_NAME");
private static readonly string QueueUrl = Environment.GetEnvironmentVariable("QUEUE_URL");
private static readonly string StripeKey = GetSecretAsync("stripe-api-key").Result;

private static async Task<string> GetSecretAsync(string secretName)
{
    using var client = new AmazonSecretsManagerClient();
    var response = await client.GetSecretValueAsync(new GetSecretValueRequest
    {
        SecretId = secretName
    });
    return response.SecretString;
}
```

**3. Proper Error Handling:**

```csharp
public async Task Handler(SQSEvent sqsEvent, ILambdaContext context)
{
    var failures = new List<SQSBatchItemFailure>();
    
    foreach (var record in sqsEvent.Records)
    {
        try
        {
            await ProcessRecordAsync(record);
        }
        catch (Exception ex)
        {
            context.Logger.LogLine($"Failed to process {record.MessageId}: {ex.Message}");
            
            // Report partial batch failure
            failures.Add(new SQSBatchItemFailure
            {
                ItemIdentifier = record.MessageId
            });
        }
    }
    
    // Return failed messages for retry
    return new SQSBatchResponse
    {
        BatchItemFailures = failures
    };
}
```

---

## EventBridge Architecture

### Event-Driven Payment System

```csharp
using Amazon.EventBridge;
using Amazon.EventBridge.Model;

public class EventBridgePublisher
{
    private readonly IAmazonEventBridge _eventBridge;
    private const string EventBusName = "payment-events";
    
    public async Task PublishPaymentEventAsync(string eventType, Payment payment)
    {
        var @event = new PutEventsRequestEntry
        {
            EventBusName = EventBusName,
            Source = "payment.service",
            DetailType = eventType,
            Detail = JsonSerializer.Serialize(new
            {
                paymentId = payment.Id,
                orderId = payment.OrderId,
                amount = payment.Amount,
                currency = payment.Currency,
                status = payment.Status,
                timestamp = DateTime.UtcNow
            })
        };
        
        await _eventBridge.PutEventsAsync(new PutEventsRequest
        {
            Entries = new List<PutEventsRequestEntry> { @event }
        });
    }
}
```

### EventBridge Rules (Infrastructure as Code)

```yaml
# CloudFormation / SAM Template
PaymentCompletedRule:
  Type: AWS::Events::Rule
  Properties:
    EventBusName: payment-events
    EventPattern:
      source:
        - payment.service
      detail-type:
        - PaymentCompleted
    Targets:
      - Arn: !GetAtt EmailNotificationLambda.Arn
        Id: EmailTarget
      - Arn: !GetAtt InventoryUpdateLambda.Arn
        Id: InventoryTarget
      - Arn: !Ref PaymentCompletedTopic
        Id: SnsTarget

PaymentFailedRule:
  Type: AWS::Events::Rule
  Properties:
    EventBusName: payment-events
    EventPattern:
      source:
        - payment.service
      detail-type:
        - PaymentFailed
    Targets:
      - Arn: !GetAtt AlertingLambda.Arn
        Id: AlertTarget
```

### Event Handlers

```csharp
// Lambda triggered by EventBridge
public class PaymentEventHandler
{
    public async Task HandlePaymentCompletedAsync(
        CloudWatchEvent<PaymentEvent> eventBridgeEvent,
        ILambdaContext context)
    {
        var payment = eventBridgeEvent.Detail;
        
        context.Logger.LogLine($"Payment completed: {payment.PaymentId}");
        
        // Multiple actions can happen in parallel
        await Task.WhenAll(
            SendConfirmationEmailAsync(payment),
            UpdateInventoryAsync(payment),
            CreateShipmentAsync(payment),
            UpdateAnalyticsAsync(payment)
        );
    }
    
    public async Task HandlePaymentFailedAsync(
        CloudWatchEvent<PaymentEvent> eventBridgeEvent,
        ILambdaContext context)
    {
        var payment = eventBridgeEvent.Detail;
        
        context.Logger.LogLine($"Payment failed: {payment.PaymentId}");
        
        await Task.WhenAll(
            SendFailureNotificationAsync(payment),
            LogFailureAsync(payment),
            TriggerRetryIfApplicableAsync(payment)
        );
    }
}

public class PaymentEvent
{
    public string PaymentId { get; set; }
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string Status { get; set; }
    public DateTime Timestamp { get; set; }
}
```

---

## Step Functions Workflows

### Payment Processing Workflow

```json
{
  "Comment": "Payment Processing Workflow",
  "StartAt": "ValidatePayment",
  "States": {
    "ValidatePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:ValidatePayment",
      "Next": "CheckInventory",
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "Next": "PaymentFailed"
        }
      ]
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:CheckInventory",
      "Next": "ProcessPayment",
      "Catch": [
        {
          "ErrorEquals": ["InsufficientStock"],
          "Next": "PaymentFailed"
        }
      ]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:ProcessPayment",
      "Retry": [
        {
          "ErrorEquals": ["NetworkError", "TimeoutError"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Next": "WaitForConfirmation"
    },
    "WaitForConfirmation": {
      "Type": "Wait",
      "Seconds": 30,
      "Next": "CheckPaymentStatus"
    },
    "CheckPaymentStatus": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:CheckPaymentStatus",
      "Next": "IsPaymentComplete"
    },
    "IsPaymentComplete": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.status",
          "StringEquals": "completed",
          "Next": "PaymentSuccess"
        },
        {
          "Variable": "$.status",
          "StringEquals": "failed",
          "Next": "PaymentFailed"
        }
      ],
      "Default": "WaitForConfirmation"
    },
    "PaymentSuccess": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "CreateShipment",
          "States": {
            "CreateShipment": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789:function:CreateShipment",
              "End": true
            }
          }
        },
        {
          "StartAt": "SendConfirmation",
          "States": {
            "SendConfirmation": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789:function:SendConfirmation",
              "End": true
            }
          }
        },
        {
          "StartAt": "UpdateAnalytics",
          "States": {
            "UpdateAnalytics": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789:function:UpdateAnalytics",
              "End": true
            }
          }
        }
      ],
      "End": true
    },
    "PaymentFailed": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:HandlePaymentFailure",
      "End": true
    }
  }
}
```

### C# Step Functions Integration

```csharp
using Amazon.StepFunctions;
using Amazon.StepFunctions.Model;

public class PaymentWorkflowService
{
    private readonly IAmazonStepFunctions _stepFunctions;
    private const string StateMachineArn = "arn:aws:states:us-east-1:123456789:stateMachine:PaymentProcessing";
    
    public async Task<string> StartPaymentWorkflowAsync(PaymentRequest payment)
    {
        var input = JsonSerializer.Serialize(payment);
        
        var response = await _stepFunctions.StartExecutionAsync(new StartExecutionRequest
        {
            StateMachineArn = StateMachineArn,
            Input = input,
            Name = $"payment-{payment.PaymentId}-{DateTime.UtcNow.Ticks}"
        });
        
        return response.ExecutionArn;
    }
    
    public async Task<ExecutionStatus> GetWorkflowStatusAsync(string executionArn)
    {
        var response = await _stepFunctions.DescribeExecutionAsync(new DescribeExecutionRequest
        {
            ExecutionArn = executionArn
        });
        
        return new ExecutionStatus
        {
            Status = response.Status,
            StartDate = response.StartDate,
            StopDate = response.StopDate,
            Output = response.Output
        };
    }
}
```

---

## API Gateway

### REST API Configuration

```yaml
# SAM Template
PaymentApi:
  Type: AWS::Serverless::Api
  Properties:
    StageName: prod
    Auth:
      DefaultAuthorizer: CognitoAuthorizer
      Authorizers:
        CognitoAuthorizer:
          UserPoolArn: !GetAtt UserPool.Arn
    MethodSettings:
      - ResourcePath: "/*"
        HttpMethod: "*"
        ThrottlingBurstLimit: 1000
        ThrottlingRateLimit: 500
    Cors:
      AllowOrigin: "'*'"
      AllowHeaders: "'Content-Type,Authorization'"
      AllowMethods: "'GET,POST,PUT,DELETE'"
```

### Lambda Authorizer

```csharp
public class CustomAuthorizerHandler
{
    public async Task<APIGatewayCustomAuthorizerResponse> HandleAsync(
        APIGatewayCustomAuthorizerRequest request,
        ILambdaContext context)
    {
        try
        {
            var token = request.AuthorizationToken.Replace("Bearer ", "");
            
            // Validate token (JWT, API key, etc.)
            var isValid = await ValidateTokenAsync(token);
            
            if (!isValid)
            {
                throw new UnauthorizedException("Invalid token");
            }
            
            // Extract user info
            var userId = GetUserIdFromToken(token);
            
            return new APIGatewayCustomAuthorizerResponse
            {
                PrincipalID = userId,
                PolicyDocument = new APIGatewayCustomAuthorizerPolicy
                {
                    Version = "2012-10-17",
                    Statement = new List<APIGatewayCustomAuthorizerPolicy.IAMPolicyStatement>
                    {
                        new APIGatewayCustomAuthorizerPolicy.IAMPolicyStatement
                        {
                            Action = new HashSet<string> { "execute-api:Invoke" },
                            Effect = "Allow",
                            Resource = new HashSet<string> { request.MethodArn }
                        }
                    }
                },
                Context = new APIGatewayCustomAuthorizerContextOutput
                {
                    ["userId"] = userId
                }
            };
        }
        catch
        {
            throw new UnauthorizedException("Unauthorized");
        }
    }
}
```

### Request Validation

```yaml
PaymentRequestModel:
  Type: AWS::ApiGateway::Model
  Properties:
    RestApiId: !Ref PaymentApi
    ContentType: application/json
    Schema:
      type: object
      required:
        - orderId
        - amount
        - currency
      properties:
        orderId:
          type: string
          minLength: 1
        amount:
          type: number
          minimum: 0.01
        currency:
          type: string
          pattern: "^[A-Z]{3}$"
        paymentMethodId:
          type: string
```

---

## Security & Compliance

### Secrets Management

```csharp
using Amazon.SecretsManager;
using Amazon.SecretsManager.Model;

public class SecretsService
{
    private readonly IAmazonSecretsManager _secretsManager;
    private readonly IMemoryCache _cache;
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        // Check cache first
        if (_cache.TryGetValue(secretName, out string cachedValue))
        {
            return cachedValue;
        }
        
        var response = await _secretsManager.GetSecretValueAsync(new GetSecretValueRequest
        {
            SecretId = secretName
        });
        
        // Cache for 5 minutes
        _cache.Set(secretName, response.SecretString, TimeSpan.FromMinutes(5));
        
        return response.SecretString;
    }
}

// Usage
var stripeKey = await _secretsService.GetSecretAsync("stripe-api-key");
```

### KMS Encryption

```csharp
using Amazon.KeyManagementService;

public class EncryptionService
{
    private readonly IAmazonKeyManagementService _kms;
    private const string KeyId = "arn:aws:kms:us-east-1:123456789:key/...";
    
    public async Task<string> EncryptAsync(string plaintext)
    {
        var response = await _kms.EncryptAsync(new EncryptRequest
        {
            KeyId = KeyId,
            Plaintext = new MemoryStream(Encoding.UTF8.GetBytes(plaintext))
        });
        
        return Convert.ToBase64String(response.CiphertextBlob.ToArray());
    }
    
    public async Task<string> DecryptAsync(string ciphertext)
    {
        var response = await _kms.DecryptAsync(new DecryptRequest
        {
            CiphertextBlob = new MemoryStream(Convert.FromBase64String(ciphertext))
        });
        
        return Encoding.UTF8.GetString(response.Plaintext.ToArray());
    }
}
```

### VPC Configuration

```yaml
PaymentLambda:
  Type: AWS::Serverless::Function
  Properties:
    VpcConfig:
      SecurityGroupIds:
        - !Ref LambdaSecurityGroup
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
    Environment:
      Variables:
        RDS_ENDPOINT: !GetAtt PaymentDatabase.Endpoint.Address
```

---

## Cost Optimization

### Strategies

**1. Right-Size Lambda Functions:**
```yaml
# Start small, monitor, adjust
PaymentLambda:
  MemorySize: 512  # MB (CPU scales with memory)
  Timeout: 30      # seconds
```

**2. Use Provisioned Concurrency Sparingly:**
```yaml
# Only for critical, latency-sensitive functions
PaymentLambda:
  ProvisionedConcurrencyConfig:
    ProvisionedConcurrentExecutions: 5  # Warmed instances
```

**3. DynamoDB On-Demand vs Provisioned:**
```
On-Demand: Unpredictable traffic, dev/test
Provisioned: Predictable traffic, can save 60%+ with reserved capacity
```

**4. S3 Intelligent-Tiering:**
```yaml
ReceiptsBucket:
  LifecycleConfiguration:
    Rules:
      - Id: ArchiveOldReceipts
        Status: Enabled
        Transitions:
          - Days: 90
            StorageClass: INTELLIGENT_TIERING
```

**5. CloudWatch Logs Retention:**
```yaml
PaymentLambdaLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    RetentionInDays: 7  # Don't keep logs forever
```

### Cost Monitoring

```csharp
// Tag all resources for cost allocation
public class ResourceTagger
{
    public Dictionary<string, string> GetPaymentSystemTags()
    {
        return new Dictionary<string, string>
        {
            ["Application"] = "PaymentSystem",
            ["Environment"] = "Production",
            ["CostCenter"] = "Engineering",
            ["Owner"] = "payments-team@company.com"
        };
    }
}
```

---

## Complete Architecture Example (CDK)

```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;
using Amazon.CDK.AWS.DynamoDB;
using Amazon.CDK.AWS.SQS;
using Amazon.CDK.AWS.APIGateway;

public class PaymentSystemStack : Stack
{
    public PaymentSystemStack(Construct scope, string id) : base(scope, id)
    {
        // DynamoDB Table
        var paymentsTable = new Table(this, "PaymentsTable", new TableProps
        {
            PartitionKey = new Attribute { Name = "PK", Type = AttributeType.STRING },
            SortKey = new Attribute { Name = "SK", Type = AttributeType.STRING },
            BillingMode = BillingMode.PAY_PER_REQUEST,
            Stream = StreamViewType.NEW_AND_OLD_IMAGES,
            TimeToLiveAttribute = "TTL"
        });
        
        // Add GSI
        paymentsTable.AddGlobalSecondaryIndex(new GlobalSecondaryIndexProps
        {
            IndexName = "GSI1",
            PartitionKey = new Attribute { Name = "GSI1PK", Type = AttributeType.STRING },
            SortKey = new Attribute { Name = "GSI1SK", Type = AttributeType.STRING }
        });
        
        // SQS Queue
        var dlq = new Queue(this, "PaymentDLQ");
        var paymentQueue = new Queue(this, "PaymentQueue", new QueueProps
        {
            VisibilityTimeout = Duration.Seconds(300),
            DeadLetterQueue = new DeadLetterQueue
            {
                Queue = dlq,
                MaxReceiveCount = 3
            }
        });
        
        // Lambda Functions
        var paymentApiLambda = new Function(this, "PaymentApiLambda", new FunctionProps
        {
            Runtime = Runtime.DOTNET_10,
            Handler = "PaymentApi::PaymentApi.Handler::HandleAsync",
            Code = Code.FromAsset("./src/PaymentApi/bin/Release/net10.0/publish"),
            Environment = new Dictionary<string, string>
            {
                ["TABLE_NAME"] = paymentsTable.TableName,
                ["QUEUE_URL"] = paymentQueue.QueueUrl
            },
            Timeout = Duration.Seconds(30),
            MemorySize = 512
        });
        
        // Permissions
        paymentsTable.GrantReadWriteData(paymentApiLambda);
        paymentQueue.GrantSendMessages(paymentApiLambda);
        
        // API Gateway
        var api = new RestApi(this, "PaymentApi", new RestApiProps
        {
            RestApiName = "Payment API",
            Description = "Payment processing API"
        });
        
        var payments = api.Root.AddResource("payments");
        payments.AddMethod("POST", new LambdaIntegration(paymentApiLambda));
        
        // Outputs
        new CfnOutput(this, "ApiUrl", new CfnOutputProps
        {
            Value = api.Url
        });
    }
}
```

---

## Summary

### AWS Service Selection

| Use Case | Primary Service | Alternative |
|----------|----------------|-------------|
| **Payment Records** | RDS PostgreSQL | Aurora Serverless |
| **Event Log** | DynamoDB | OpenSearch |
| **Job Queue** | SQS | EventBridge |
| **Workflows** | Step Functions | Lambda + SQS |
| **API** | API Gateway | ALB |
| **Compute** | Lambda | Fargate |
| **Cache** | ElastiCache Redis | DynamoDB DAX |
| **Secrets** | Secrets Manager | Parameter Store |

### Cost Estimate (100K payments/month)

```
Lambda (1M invocations):        $0.20
DynamoDB (on-demand):           $25-50
SQS (1M messages):              $0.40
API Gateway (1M requests):      $3.50
CloudWatch Logs:                $5
Total:                          ~$35-60/month
```

## Next Steps

- [Database Design](06-DATABASE-DESIGN.md)
- [Performance Optimization](08-PERFORMANCE-OPTIMIZATION.md)
- [Security Patterns & Practices](15-SECURITY-PATTERNS-PRACTICES.md)
