# Cloud-Native Payment Systems: Multi-Cloud & Alternatives

## Overview

This document covers cloud-native payment architectures across AWS, Azure, and Google Cloud Platform (GCP), including service equivalents, migration strategies, and multi-cloud patterns.

## Table of Contents

1. [Service Comparison Matrix](#service-comparison-matrix)
2. [Azure Payment Architecture](#azure-payment-architecture)
3. [Google Cloud Payment Architecture](#google-cloud-payment-architecture)
4. [Multi-Cloud Patterns](#multi-cloud-patterns)
5. [Cloud-Agnostic Solutions](#cloud-agnostic-solutions)
6. [Migration Strategies](#migration-strategies)
7. [Kubernetes-Based Approach](#kubernetes-based-approach)
8. [Cost Comparison](#cost-comparison)

---

## Service Comparison Matrix

### Core Services

| Service Category | AWS | Azure | GCP | Cloud-Agnostic |
|------------------|-----|-------|-----|----------------|
| **Compute** | Lambda | Azure Functions | Cloud Functions | Knative, OpenFaaS |
| **Container Orchestration** | EKS | AKS | GKE | Kubernetes |
| **Containers** | ECS/Fargate | Container Instances | Cloud Run | Docker + K8s |
| **API Gateway** | API Gateway | API Management | API Gateway | Kong, Tyk |
| **SQL Database** | RDS | Azure SQL | Cloud SQL | PostgreSQL on K8s |
| **NoSQL Database** | DynamoDB | Cosmos DB | Firestore | MongoDB, Cassandra |
| **Message Queue** | SQS | Service Bus | Pub/Sub | RabbitMQ, Kafka |
| **Event Bus** | EventBridge | Event Grid | Eventarc | Kafka, NATS |
| **Object Storage** | S3 | Blob Storage | Cloud Storage | MinIO |
| **Cache** | ElastiCache | Azure Cache | Memorystore | Redis on K8s |
| **Secrets** | Secrets Manager | Key Vault | Secret Manager | Vault |
| **Monitoring** | CloudWatch | Monitor | Cloud Monitoring | Prometheus + Grafana |
| **Tracing** | X-Ray | Application Insights | Cloud Trace | Jaeger, Zipkin |
| **Service Mesh** | App Mesh | Service Fabric Mesh | Anthos Service Mesh | Istio, Linkerd |

### Detailed Feature Comparison

| Feature | AWS | Azure | GCP |
|---------|-----|-------|-----|
| **Serverless Runtime** | Lambda | Azure Functions | Cloud Functions |
| - Cold Start | ~100-200ms | ~100-300ms | ~100-200ms |
| - Max Execution | 15 min | 10 min (Consumption) | 9 min |
| - Memory Range | 128MB - 10GB | 128MB - 4GB | 128MB - 8GB |
| **Message Queue** | SQS | Service Bus | Pub/Sub |
| - Max Message Size | 256 KB | 1 MB | 10 MB |
| - Ordering | FIFO Queue | Sessions | Ordered delivery |
| - At-least-once | ✅ | ✅ | ✅ |
| **NoSQL Database** | DynamoDB | Cosmos DB | Firestore |
| - Consistency | Eventual/Strong | 5 levels | Strong |
| - Multi-region | ✅ | ✅ | ✅ |
| - Pricing Model | Pay-per-request | RU/s | Document operations |

---

## Azure Payment Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                     Azure Front Door                     │
│              (CDN, WAF, Load Balancing)                  │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Azure API Management                        │
│    - Rate Limiting, Caching, Authentication             │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│ Azure Functions  │    │ Azure Functions  │
│  (Payment API)   │    │  (Webhooks)      │
└────────┬─────────┘    └────────┬─────────┘
         │                       │
         ▼                       ▼
┌──────────────────────────────────────────┐
│     Azure SQL Database (Primary)         │
│     or Azure Cosmos DB                   │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│     Azure Service Bus                    │
│     - Standard Queue                     │
│     - Dead Letter Queue                  │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│     Azure Event Grid                     │
│     - PaymentCompleted Event             │
│     - PaymentFailed Event                │
└────────┬─────────────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌─────────┐ ┌─────────┐
│ Logic   │ │SendGrid │
│ Apps    │ │  (SES)  │
└─────────┘ └─────────┘
```

### Azure Functions Implementation

**Payment API Function:**

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Azure.Messaging.ServiceBus;
using Azure.Data.Tables;

public class PaymentFunction
{
    private readonly ServiceBusClient _serviceBusClient;
    private readonly TableClient _tableClient;
    private readonly ILogger<PaymentFunction> _logger;
    
    public PaymentFunction(
        ServiceBusClient serviceBusClient,
        TableClient tableClient,
        ILogger<PaymentFunction> logger)
    {
        _serviceBusClient = serviceBusClient;
        _tableClient = tableClient;
        _logger = logger;
    }
    
    [Function("ProcessPayment")]
    public async Task<HttpResponseData> ProcessPayment(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "payments")] 
        HttpRequestData req)
    {
        _logger.LogInformation("Processing payment request");
        
        var paymentRequest = await req.ReadFromJsonAsync<PaymentRequest>();
        
        // Create payment entity (Azure Table Storage)
        var payment = new PaymentEntity
        {
            PartitionKey = paymentRequest.CustomerId,
            RowKey = Guid.NewGuid().ToString(),
            OrderId = paymentRequest.OrderId,
            Amount = paymentRequest.Amount,
            Currency = paymentRequest.Currency,
            Status = "Pending",
            Timestamp = DateTimeOffset.UtcNow
        };
        
        await _tableClient.AddEntityAsync(payment);
        
        // Send to Service Bus for async processing
        var sender = _serviceBusClient.CreateSender("payment-processing");
        var message = new ServiceBusMessage(JsonSerializer.Serialize(payment))
        {
            MessageId = payment.RowKey,
            SessionId = paymentRequest.CustomerId // For ordered processing
        };
        
        await sender.SendMessageAsync(message);
        
        var response = req.CreateResponse(HttpStatusCode.Accepted);
        await response.WriteAsJsonAsync(new 
        { 
            paymentId = payment.RowKey,
            status = payment.Status 
        });
        
        return response;
    }
    
    [Function("ProcessPaymentQueue")]
    public async Task ProcessPaymentQueue(
        [ServiceBusTrigger("payment-processing", Connection = "ServiceBusConnection")] 
        ServiceBusReceivedMessage message,
        ServiceBusMessageActions messageActions)
    {
        try
        {
            var payment = JsonSerializer.Deserialize<PaymentEntity>(message.Body.ToString());
            
            _logger.LogInformation("Processing payment: {PaymentId}", payment.RowKey);
            
            // Process with payment gateway
            var result = await ProcessWithGatewayAsync(payment);
            
            // Update status
            payment.Status = result.IsSuccess ? "Completed" : "Failed";
            payment.ProviderTransactionId = result.TransactionId;
            
            await _tableClient.UpdateEntityAsync(payment, payment.ETag);
            
            // Complete message
            await messageActions.CompleteMessageAsync(message);
            
            // Publish event to Event Grid
            await PublishEventAsync("PaymentCompleted", payment);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Payment processing failed");
            
            // Dead-letter the message
            await messageActions.DeadLetterMessageAsync(message);
        }
    }
}
```

### Azure Cosmos DB Implementation

**Using Cosmos DB for Global Distribution:**

```csharp
using Microsoft.Azure.Cosmos;

public class CosmosDbPaymentRepository
{
    private readonly Container _container;
    
    public CosmosDbPaymentRepository(CosmosClient cosmosClient)
    {
        _container = cosmosClient.GetContainer("PaymentDb", "Payments");
    }
    
    // Create payment
    public async Task<Payment> CreatePaymentAsync(Payment payment)
    {
        payment.id = Guid.NewGuid().ToString();
        payment.partitionKey = payment.CustomerId; // Partition by customer
        
        var response = await _container.CreateItemAsync(
            payment,
            new PartitionKey(payment.partitionKey));
        
        return response.Resource;
    }
    
    // Query payments by customer
    public async Task<List<Payment>> GetPaymentsByCustomerAsync(string customerId)
    {
        var query = new QueryDefinition(
            "SELECT * FROM c WHERE c.customerId = @customerId ORDER BY c.createdAt DESC")
            .WithParameter("@customerId", customerId);
        
        var iterator = _container.GetItemQueryIterator<Payment>(
            query,
            requestOptions: new QueryRequestOptions
            {
                PartitionKey = new PartitionKey(customerId),
                MaxItemCount = 100
            });
        
        var results = new List<Payment>();
        while (iterator.HasMoreResults)
        {
            var response = await iterator.ReadNextAsync();
            results.AddRange(response);
        }
        
        return results;
    }
    
    // Update with optimistic concurrency
    public async Task UpdatePaymentAsync(Payment payment)
    {
        await _container.ReplaceItemAsync(
            payment,
            payment.id,
            new PartitionKey(payment.partitionKey),
            new ItemRequestOptions
            {
                IfMatchEtag = payment._etag // Optimistic concurrency
            });
    }
}

public class Payment
{
    public string id { get; set; }
    public string partitionKey { get; set; }
    public string CustomerId { get; set; }
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public string _etag { get; set; } // For optimistic concurrency
    public int ttl { get; set; } = 7776000; // 90 days in seconds
}
```

### Azure Service Bus Patterns

**Session-based Ordering:**

```csharp
using Azure.Messaging.ServiceBus;

public class ServiceBusPaymentQueue
{
    private readonly ServiceBusClient _client;
    private const string QueueName = "payment-processing";
    
    public async Task SendPaymentAsync(PaymentRequest payment)
    {
        var sender = _client.CreateSender(QueueName);
        
        var message = new ServiceBusMessage(JsonSerializer.Serialize(payment))
        {
            MessageId = Guid.NewGuid().ToString(),
            SessionId = payment.CustomerId, // Process per customer in order
            TimeToLive = TimeSpan.FromHours(24),
            ContentType = "application/json"
        };
        
        // Add custom properties
        message.ApplicationProperties.Add("Priority", payment.Amount > 1000 ? "High" : "Normal");
        message.ApplicationProperties.Add("PaymentType", payment.Type);
        
        await sender.SendMessageAsync(message);
    }
    
    // Process with sessions (guaranteed order per customer)
    public async Task ProcessSessionMessagesAsync()
    {
        var processor = _client.CreateSessionProcessor(QueueName, new ServiceBusSessionProcessorOptions
        {
            MaxConcurrentSessions = 10,
            MaxConcurrentCallsPerSession = 1, // Process one at a time per session
            AutoCompleteMessages = false
        });
        
        processor.ProcessMessageAsync += async args =>
        {
            var payment = JsonSerializer.Deserialize<PaymentRequest>(args.Message.Body.ToString());
            
            await ProcessPaymentAsync(payment);
            
            await args.CompleteMessageAsync(args.Message);
        };
        
        processor.ProcessErrorAsync += args =>
        {
            Console.WriteLine($"Error: {args.Exception}");
            return Task.CompletedTask;
        };
        
        await processor.StartProcessingAsync();
    }
}
```

### Azure Durable Functions (Workflow)

**Payment Workflow with Durable Functions:**

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;

public class PaymentOrchestration
{
    [Function("PaymentOrchestrator")]
    public async Task<PaymentResult> RunOrchestrator(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var payment = context.GetInput<PaymentRequest>();
        
        try
        {
            // Step 1: Validate payment
            var isValid = await context.CallActivityAsync<bool>(
                "ValidatePayment", payment);
            
            if (!isValid)
            {
                return PaymentResult.ValidationFailed();
            }
            
            // Step 2: Reserve inventory
            await context.CallActivityAsync("ReserveInventory", payment.OrderId);
            
            // Step 3: Process payment
            var transactionId = await context.CallActivityAsync<string>(
                "ProcessPayment", payment);
            
            // Step 4: Wait for confirmation (with timeout)
            using var cts = new CancellationTokenSource();
            var confirmationTask = context.WaitForExternalEvent<bool>("PaymentConfirmed");
            var timeoutTask = context.CreateTimer(
                context.CurrentUtcDateTime.AddMinutes(5), 
                cts.Token);
            
            var winner = await Task.WhenAny(confirmationTask, timeoutTask);
            
            if (winner == timeoutTask)
            {
                // Timeout - query status
                var status = await context.CallActivityAsync<string>(
                    "CheckPaymentStatus", transactionId);
                
                if (status != "completed")
                {
                    throw new TimeoutException("Payment confirmation timeout");
                }
            }
            
            // Step 5: Complete order (parallel tasks)
            var tasks = new List<Task>
            {
                context.CallActivityAsync("CreateShipment", payment.OrderId),
                context.CallActivityAsync("SendConfirmationEmail", payment),
                context.CallActivityAsync("UpdateAnalytics", payment)
            };
            
            await Task.WhenAll(tasks);
            
            return PaymentResult.Success(transactionId);
        }
        catch (Exception ex)
        {
            // Compensation logic
            await context.CallActivityAsync("RefundPayment", payment);
            await context.CallActivityAsync("ReleaseInventory", payment.OrderId);
            
            return PaymentResult.Failed(ex.Message);
        }
    }
    
    [Function("ValidatePayment")]
    public bool ValidatePayment([ActivityTrigger] PaymentRequest payment)
    {
        return payment.Amount > 0 && !string.IsNullOrEmpty(payment.Currency);
    }
    
    [Function("ProcessPayment")]
    public async Task<string> ProcessPayment([ActivityTrigger] PaymentRequest payment)
    {
        // Call payment gateway
        return await _gateway.ProcessAsync(payment);
    }
}
```

---

## Google Cloud Payment Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│              Cloud Load Balancing                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              API Gateway / Cloud Endpoints              │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│ Cloud Functions  │    │  Cloud Run       │
│  (2nd gen)       │    │  (Containers)    │
└────────┬─────────┘    └────────┬─────────┘
         │                       │
         ▼                       ▼
┌──────────────────────────────────────────┐
│     Cloud SQL (PostgreSQL)               │
│     or Firestore (NoSQL)                 │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│     Cloud Pub/Sub                        │
│     - Topics & Subscriptions             │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│     Eventarc                             │
│     - Event routing                      │
└────────┬─────────────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌─────────┐ ┌─────────┐
│ Cloud   │ │SendGrid │
│Scheduler│ │         │
└─────────┘ └─────────┘
```

### Cloud Functions Implementation

**Payment Function (2nd Gen):**

```csharp
using Google.Cloud.Functions.Framework;
using Google.Cloud.Firestore;
using Google.Cloud.PubSub.V1;
using Microsoft.AspNetCore.Http;

public class PaymentFunction : IHttpFunction
{
    private readonly FirestoreDb _firestore;
    private readonly PublisherClient _publisher;
    
    public PaymentFunction()
    {
        _firestore = FirestoreDb.Create("your-project-id");
        _publisher = PublisherClient.Create();
    }
    
    public async Task HandleAsync(HttpContext context)
    {
        var paymentRequest = await context.Request.ReadFromJsonAsync<PaymentRequest>();
        
        // Create payment document in Firestore
        var paymentsCollection = _firestore.Collection("payments");
        var payment = new Payment
        {
            Id = Guid.NewGuid().ToString(),
            OrderId = paymentRequest.OrderId,
            Amount = paymentRequest.Amount,
            Currency = paymentRequest.Currency,
            Status = "Pending",
            CreatedAt = DateTime.UtcNow
        };
        
        var docRef = await paymentsCollection.AddAsync(payment);
        
        // Publish to Pub/Sub
        var topicName = new TopicName("your-project-id", "payment-processing");
        var message = new PubsubMessage
        {
            Data = ByteString.CopyFromUtf8(JsonSerializer.Serialize(payment)),
            Attributes =
            {
                ["paymentId"] = payment.Id,
                ["priority"] = payment.Amount > 1000 ? "high" : "normal"
            }
        };
        
        await _publisher.PublishAsync(topicName, new[] { message });
        
        context.Response.StatusCode = 202;
        await context.Response.WriteAsJsonAsync(new 
        { 
            paymentId = payment.Id,
            status = payment.Status 
        });
    }
}
```

### Firestore Implementation

**Firestore Data Model:**

```csharp
using Google.Cloud.Firestore;

[FirestoreData]
public class Payment
{
    [FirestoreProperty]
    public string Id { get; set; }
    
    [FirestoreProperty]
    public string CustomerId { get; set; }
    
    [FirestoreProperty]
    public string OrderId { get; set; }
    
    [FirestoreProperty]
    public decimal Amount { get; set; }
    
    [FirestoreProperty]
    public string Currency { get; set; }
    
    [FirestoreProperty]
    public string Status { get; set; }
    
    [FirestoreProperty]
    public Timestamp CreatedAt { get; set; }
}

public class FirestorePaymentRepository
{
    private readonly FirestoreDb _db;
    private const string CollectionName = "payments";
    
    public FirestorePaymentRepository(string projectId)
    {
        _db = FirestoreDb.Create(projectId);
    }
    
    // Create payment
    public async Task<string> CreatePaymentAsync(Payment payment)
    {
        payment.Id = Guid.NewGuid().ToString();
        payment.CreatedAt = Timestamp.FromDateTime(DateTime.UtcNow);
        
        var docRef = _db.Collection(CollectionName).Document(payment.Id);
        await docRef.SetAsync(payment);
        
        return payment.Id;
    }
    
    // Get payment by ID
    public async Task<Payment> GetPaymentAsync(string paymentId)
    {
        var docRef = _db.Collection(CollectionName).Document(paymentId);
        var snapshot = await docRef.GetSnapshotAsync();
        
        return snapshot.Exists ? snapshot.ConvertTo<Payment>() : null;
    }
    
    // Query payments by customer
    public async Task<List<Payment>> GetPaymentsByCustomerAsync(string customerId)
    {
        var query = _db.Collection(CollectionName)
            .WhereEqualTo("CustomerId", customerId)
            .OrderByDescending("CreatedAt")
            .Limit(100);
        
        var snapshot = await query.GetSnapshotAsync();
        
        return snapshot.Documents.Select(d => d.ConvertTo<Payment>()).ToList();
    }
    
    // Update with transaction
    public async Task UpdatePaymentStatusAsync(string paymentId, string newStatus)
    {
        var docRef = _db.Collection(CollectionName).Document(paymentId);
        
        await _db.RunTransactionAsync(async transaction =>
        {
            var snapshot = await transaction.GetSnapshotAsync(docRef);
            
            if (snapshot.Exists)
            {
                transaction.Update(docRef, new Dictionary<string, object>
                {
                    ["Status"] = newStatus,
                    ["UpdatedAt"] = Timestamp.FromDateTime(DateTime.UtcNow)
                });
            }
        });
    }
    
    // Listen to real-time changes
    public void ListenToPaymentChanges(string paymentId, Action<Payment> callback)
    {
        var docRef = _db.Collection(CollectionName).Document(paymentId);
        
        var listener = docRef.Listen(snapshot =>
        {
            if (snapshot.Exists)
            {
                var payment = snapshot.ConvertTo<Payment>();
                callback(payment);
            }
        });
    }
}
```

### Cloud Pub/Sub Patterns

**Publisher & Subscriber:**

```csharp
using Google.Cloud.PubSub.V1;

public class PubSubPaymentQueue
{
    private readonly PublisherClient _publisher;
    private readonly SubscriberClient _subscriber;
    
    // Publish message
    public async Task PublishPaymentAsync(Payment payment)
    {
        var topicName = new TopicName("project-id", "payment-processing");
        
        var message = new PubsubMessage
        {
            Data = ByteString.CopyFromUtf8(JsonSerializer.Serialize(payment)),
            Attributes =
            {
                ["paymentId"] = payment.Id,
                ["customerId"] = payment.CustomerId,
                ["orderingKey"] = payment.CustomerId // For ordered delivery
            },
            OrderingKey = payment.CustomerId // Enable message ordering
        };
        
        var messageId = await _publisher.PublishAsync(topicName, new[] { message });
    }
    
    // Subscribe and process
    public async Task SubscribeToPaymentsAsync(CancellationToken cancellationToken)
    {
        var subscriptionName = new SubscriptionName("project-id", "payment-processing-sub");
        
        await _subscriber.StartAsync(async (message, cancellationToken) =>
        {
            try
            {
                var payment = JsonSerializer.Deserialize<Payment>(
                    message.Data.ToStringUtf8());
                
                await ProcessPaymentAsync(payment);
                
                return SubscriberClient.Reply.Ack; // Acknowledge
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
                return SubscriberClient.Reply.Nack; // Will retry
            }
        });
        
        await Task.Delay(Timeout.Infinite, cancellationToken);
        await _subscriber.StopAsync(CancellationToken.None);
    }
}
```

### Cloud Run (Containerized Services)

**Dockerfile for Payment Service:**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["PaymentService.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "PaymentService.dll"]
```

**Deploy to Cloud Run:**

```bash
# Build and push
gcloud builds submit --tag gcr.io/project-id/payment-service

# Deploy
gcloud run deploy payment-service \
  --image gcr.io/project-id/payment-service \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars "DATABASE_URL=..." \
  --min-instances 1 \
  --max-instances 100 \
  --concurrency 80 \
  --cpu 1 \
  --memory 512Mi
```

---

## Multi-Cloud Patterns

### Pattern 1: Primary Cloud with Failover

```
Primary (AWS)          Failover (Azure)
├─ Active traffic      ├─ Standby
├─ RDS Primary         ├─ RDS Replica (read)
├─ Lambda              ├─ Azure Functions (cold)
└─ SQS                 └─ Service Bus (empty)

DNS/Load Balancer
└─ Route to primary
   └─ Failover to secondary on health check failure
```

**Implementation:**

```csharp
public class MultiCloudPaymentService : IPaymentService
{
    private readonly IPaymentService _primaryService; // AWS
    private readonly IPaymentService _fallbackService; // Azure
    private readonly IHealthCheck _primaryHealthCheck;
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Check primary health
        var isHealthy = await _primaryHealthCheck.CheckHealthAsync();
        
        if (isHealthy)
        {
            try
            {
                return await _primaryService.ProcessPaymentAsync(request);
            }
            catch (Exception ex)
            {
                // Log primary failure
                _logger.LogError(ex, "Primary service failed, failing over");
                
                // Fallback to secondary
                return await _fallbackService.ProcessPaymentAsync(request);
            }
        }
        else
        {
            // Primary unhealthy, use fallback
            return await _fallbackService.ProcessPaymentAsync(request);
        }
    }
}
```

### Pattern 2: Multi-Cloud Active-Active

```
AWS (Region 1)         Azure (Region 2)        GCP (Region 3)
├─ Lambda              ├─ Functions            ├─ Cloud Functions
├─ RDS (Active)        ├─ SQL (Active)         ├─ Cloud SQL (Active)
└─ SQS                 └─ Service Bus          └─ Pub/Sub

Global Load Balancer (Cloudflare, etc.)
└─ Route based on latency/availability
```

**Database Replication:**

```csharp
public class MultiRegionPaymentRepository
{
    private readonly IPaymentRepository _primaryRepo;
    private readonly List<IPaymentRepository> _replicaRepos;
    
    // Write to primary
    public async Task<Payment> CreatePaymentAsync(Payment payment)
    {
        var result = await _primaryRepo.CreatePaymentAsync(payment);
        
        // Async replication to other regions
        _ = Task.Run(async () =>
        {
            foreach (var replica in _replicaRepos)
            {
                try
                {
                    await replica.CreatePaymentAsync(payment);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Replication failed for region");
                }
            }
        });
        
        return result;
    }
    
    // Read from nearest
    public async Task<Payment> GetPaymentAsync(string paymentId, string region)
    {
        var repo = SelectRepositoryByRegion(region);
        return await repo.GetPaymentAsync(paymentId);
    }
}
```

### Pattern 3: Cloud-Agnostic with Abstraction Layer

```csharp
// Abstract interfaces
public interface ICloudStorage
{
    Task<string> UploadAsync(Stream content, string key);
    Task<Stream> DownloadAsync(string key);
}

public interface ICloudQueue
{
    Task SendMessageAsync(string message);
    Task<List<CloudMessage>> ReceiveMessagesAsync(int maxMessages);
}

public interface ICloudDatabase
{
    Task<T> GetAsync<T>(string id);
    Task CreateAsync<T>(T item);
}

// AWS Implementation
public class S3CloudStorage : ICloudStorage
{
    private readonly IAmazonS3 _s3;
    
    public async Task<string> UploadAsync(Stream content, string key)
    {
        await _s3.PutObjectAsync(new PutObjectRequest
        {
            BucketName = _bucketName,
            Key = key,
            InputStream = content
        });
        return key;
    }
}

// Azure Implementation
public class BlobCloudStorage : ICloudStorage
{
    private readonly BlobServiceClient _blobService;
    
    public async Task<string> UploadAsync(Stream content, string key)
    {
        var container = _blobService.GetBlobContainerClient(_containerName);
        var blob = container.GetBlobClient(key);
        await blob.UploadAsync(content);
        return key;
    }
}

// GCP Implementation
public class GcsCloudStorage : ICloudStorage
{
    private readonly StorageClient _storage;
    
    public async Task<string> UploadAsync(Stream content, string key)
    {
        await _storage.UploadObjectAsync(_bucketName, key, null, content);
        return key;
    }
}

// Configuration
services.AddSingleton<ICloudStorage>(sp =>
{
    var cloudProvider = Configuration["CloudProvider"];
    
    return cloudProvider switch
    {
        "AWS" => new S3CloudStorage(/* ... */),
        "Azure" => new BlobCloudStorage(/* ... */),
        "GCP" => new GcsCloudStorage(/* ... */),
        _ => throw new NotSupportedException()
    };
});
```

---

## Cloud-Agnostic Solutions

### Kubernetes-Based Architecture

```yaml
# Payment Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: your-registry/payment-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: payment-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: payment-config
              key: redis-url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Message Queue with RabbitMQ

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

public class RabbitMQPaymentQueue
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    
    public RabbitMQPaymentQueue(string hostname)
    {
        var factory = new ConnectionFactory { HostName = hostname };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
        
        // Declare queue
        _channel.QueueDeclare(
            queue: "payment-processing",
            durable: true,
            exclusive: false,
            autoDelete: false);
    }
    
    // Publish
    public void PublishPayment(Payment payment)
    {
        var body = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(payment));
        
        var properties = _channel.CreateBasicProperties();
        properties.Persistent = true;
        properties.MessageId = payment.Id;
        
        _channel.BasicPublish(
            exchange: "",
            routingKey: "payment-processing",
            basicProperties: properties,
            body: body);
    }
    
    // Consume
    public void StartConsuming(Func<Payment, Task> processFunc)
    {
        var consumer = new EventingBasicConsumer(_channel);
        
        consumer.Received += async (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var message = Encoding.UTF8.GetString(body);
            var payment = JsonSerializer.Deserialize<Payment>(message);
            
            try
            {
                await processFunc(payment);
                _channel.BasicAck(ea.DeliveryTag, false);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
                _channel.BasicNack(ea.DeliveryTag, false, true); // Requeue
            }
        };
        
        _channel.BasicConsume(
            queue: "payment-processing",
            autoAck: false,
            consumer: consumer);
    }
}
```

### PostgreSQL on Kubernetes

```yaml
# StatefulSet for PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: payments
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

---

## Cost Comparison (100K payments/month)

| Service | AWS | Azure | GCP |
|---------|-----|-------|-----|
| **Compute** | Lambda: $0.20 | Functions: $0.15 | Cloud Functions: $0.18 |
| **Database** | RDS t3.micro: $15 | SQL Basic: $5 | Cloud SQL db-f1-micro: $7 |
| **NoSQL** | DynamoDB: $25 | Cosmos DB: $24 | Firestore: $20 |
| **Queue** | SQS: $0.40 | Service Bus: $0.05 | Pub/Sub: $0.60 |
| **API** | API Gateway: $3.50 | API Mgmt: $50* | API Gateway: $3.00 |
| **Storage** | S3: $2.30 | Blob: $1.80 | Cloud Storage: $2.00 |
| **Total (Serverless)** | ~$31 | ~$31 | ~$33 |
| **Total (with API Mgmt)** | ~$31 | ~$81* | ~$33 |

*Azure API Management minimum tier is more expensive but includes more features

---

## Migration Strategies

### Phase 1: Data Layer

```
1. Set up target cloud database
2. Enable replication from source to target
3. Monitor replication lag
4. Test read queries on target
5. Switch read traffic to target
6. Switch write traffic to target
7. Decommission source
```

### Phase 2: Application Layer

```
1. Deploy app to target cloud (dark launch)
2. Route 1% of traffic to new deployment
3. Monitor errors and performance
4. Gradually increase to 10%, 25%, 50%, 100%
5. Decommission old deployment
```

### Phase 3: Integration Layer

```
1. Migrate message queues
2. Update webhook endpoints
3. Migrate scheduled jobs
4. Update monitoring and alerts
```

## Summary

### When to Use Each Cloud

**AWS:**
- ✅ Most mature services
- ✅ Largest ecosystem
- ✅ Best Lambda performance
- ❌ Can be complex

**Azure:**
- ✅ Best .NET integration
- ✅ Hybrid cloud (Azure Arc)
- ✅ Enterprise features
- ❌ API Management expensive

**GCP:**
- ✅ Best for data/ML
- ✅ Simpler pricing
- ✅ Global network
- ❌ Smaller service catalog

**Multi-Cloud:**
- ✅ Avoid vendor lock-in
- ✅ Best of each platform
- ❌ Operational complexity
- ❌ Higher costs

## Next Steps

- [Cloud-Native AWS](09-CLOUD-NATIVE-AWS.md)
- [CQRS and Event Sourcing](11-CQRS-EVENT-SOURCING.md)
- [Design Patterns](07-DESIGN-PATTERNS.md)
