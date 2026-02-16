# PayPal Integration Guide

## Overview

Complete guide for integrating PayPal into .NET 8.0+ payment systems. Covers REST API v2 (Orders), Checkout SDK, Subscriptions, Payouts, and Disputes.

## Table of Contents

1. [Setup and Configuration](#setup-and-configuration)
2. [Orders API v2 (Recommended)](#orders-api-v2-recommended)
3. [Checkout Integration](#checkout-integration)
4. [Subscriptions (Billing Agreements)](#subscriptions-billing-agreements)
5. [Payouts](#payouts)
6. [Refunds](#refunds)
7. [Disputes and Chargebacks](#disputes-and-chargebacks)
8. [Webhook Handling](#webhook-handling)
9. [Frontend Integration](#frontend-integration)
10. [Production Checklist](#production-checklist)

---

## Setup and Configuration

### NuGet Packages

```bash
dotnet add package PayPalCheckoutSdk
dotnet add package PayPalHttp
```

### Configuration

```json
{
  "PayPal": {
    "ClientId": "AY...",
    "ClientSecret": "EL...",
    "Mode": "sandbox",
    "WebhookId": "WH-..."
  }
}
```

```csharp
public class PayPalSettings
{
    public string ClientId { get; set; }
    public string ClientSecret { get; set; }
    public string Mode { get; set; } // "sandbox" or "live"
    public string WebhookId { get; set; }

    public string BaseUrl => Mode == "live"
        ? "https://api-m.paypal.com"
        : "https://api-m.sandbox.paypal.com";
}

// PayPal HttpClient setup
public class PayPalClientFactory
{
    private readonly PayPalSettings _settings;

    public PayPalHttpClient Create()
    {
        PayPalEnvironment environment = _settings.Mode == "live"
            ? new LiveEnvironment(_settings.ClientId, _settings.ClientSecret)
            : new SandboxEnvironment(_settings.ClientId, _settings.ClientSecret);

        return new PayPalHttpClient(environment);
    }
}

// DI Registration
builder.Services.Configure<PayPalSettings>(builder.Configuration.GetSection("PayPal"));
builder.Services.AddSingleton<PayPalClientFactory>();
```

---

## Orders API v2 (Recommended)

The Orders API v2 is the recommended integration for one-time payments.

### Create Order

```csharp
public class PayPalGateway : IPaymentGateway
{
    private readonly PayPalClientFactory _clientFactory;
    private readonly ILogger<PayPalGateway> _logger;

    public string ProviderName => "PayPal";

    public async Task<PaymentResult> CreateOrderAsync(PaymentRequest request)
    {
        var client = _clientFactory.Create();

        var orderRequest = new OrdersCreateRequest();
        orderRequest.Prefer("return=representation");
        orderRequest.RequestBody(BuildOrderRequestBody(request));

        try
        {
            var response = await client.Execute(orderRequest);
            var order = response.Result<Order>();

            var approvalUrl = order.Links
                .FirstOrDefault(l => l.Rel == "approve")?.Href;

            return new PaymentResult
            {
                IsSuccess = true,
                TransactionId = order.Id,
                Status = MapStatus(order.Status),
                RequiresAction = true,
                ActionUrl = approvalUrl // Redirect buyer here
            };
        }
        catch (HttpException ex)
        {
            _logger.LogError(ex, "PayPal order creation failed for {OrderId}", request.OrderId);
            throw new PaymentGatewayException("PayPal order creation failed", "PayPal", ex.StatusCode, ex);
        }
    }

    private OrderRequest BuildOrderRequestBody(PaymentRequest request)
    {
        return new OrderRequest
        {
            CheckoutPaymentIntent = "CAPTURE",
            ApplicationContext = new ApplicationContext
            {
                BrandName = "My Store",
                LandingPage = "BILLING",
                UserAction = "PAY_NOW",
                ReturnUrl = request.ReturnUrl,
                CancelUrl = request.CancelUrl,
                ShippingPreference = request.RequiresShipping
                    ? "SET_PROVIDED_ADDRESS"
                    : "NO_SHIPPING"
            },
            PurchaseUnits = new List<PurchaseUnitRequest>
            {
                new()
                {
                    ReferenceId = request.OrderId,
                    Description = $"Order {request.OrderId}",
                    CustomId = request.OrderId,
                    AmountWithBreakdown = new AmountWithBreakdown
                    {
                        CurrencyCode = request.Currency,
                        Value = request.Amount.ToString("F2"),
                        AmountBreakdown = new AmountBreakdown
                        {
                            ItemTotal = new Money
                            {
                                CurrencyCode = request.Currency,
                                Value = request.SubTotal.ToString("F2")
                            },
                            TaxTotal = new Money
                            {
                                CurrencyCode = request.Currency,
                                Value = request.TaxAmount.ToString("F2")
                            },
                            Shipping = request.ShippingAmount > 0
                                ? new Money { CurrencyCode = request.Currency, Value = request.ShippingAmount.ToString("F2") }
                                : null
                        }
                    },
                    Items = request.Items.Select(item => new Item
                    {
                        Name = item.Name,
                        Description = item.Description,
                        UnitAmount = new Money
                        {
                            CurrencyCode = request.Currency,
                            Value = item.UnitPrice.ToString("F2")
                        },
                        Quantity = item.Quantity.ToString(),
                        Category = item.IsDigital ? "DIGITAL_GOODS" : "PHYSICAL_GOODS",
                        Sku = item.Sku
                    }).ToList()
                }
            }
        };
    }

    private PaymentStatus MapStatus(string paypalStatus) => paypalStatus switch
    {
        "COMPLETED" => PaymentStatus.Completed,
        "APPROVED" => PaymentStatus.Authorized,
        "CREATED" => PaymentStatus.Pending,
        "VOIDED" => PaymentStatus.Cancelled,
        "PAYER_ACTION_REQUIRED" => PaymentStatus.RequiresAction,
        _ => PaymentStatus.Unknown
    };
}
```

### Capture Order (After Buyer Approval)

```csharp
public async Task<PaymentResult> CaptureOrderAsync(string orderId)
{
    var client = _clientFactory.Create();
    var captureRequest = new OrdersCaptureRequest(orderId);
    captureRequest.Prefer("return=representation");
    captureRequest.RequestBody(new OrderActionRequest());

    try
    {
        var response = await client.Execute(captureRequest);
        var order = response.Result<Order>();

        var capture = order.PurchaseUnits[0].Payments.Captures[0];

        return new PaymentResult
        {
            IsSuccess = order.Status == "COMPLETED",
            TransactionId = capture.Id,
            Status = PaymentStatus.Completed
        };
    }
    catch (HttpException ex) when (ex.StatusCode == 422)
    {
        _logger.LogWarning("PayPal order {OrderId} already captured or invalid", orderId);
        return PaymentResult.Failure(new PaymentError
        {
            Code = "ORDER_ALREADY_CAPTURED",
            Message = "This order has already been captured.",
            IsRetryable = false
        });
    }
}
```

### Authorize and Capture Separately

```csharp
// 1. Create order with AUTHORIZE intent
public async Task<PaymentResult> AuthorizeOrderAsync(PaymentRequest request)
{
    var body = BuildOrderRequestBody(request);
    body.CheckoutPaymentIntent = "AUTHORIZE"; // Instead of CAPTURE

    var orderRequest = new OrdersCreateRequest();
    orderRequest.RequestBody(body);

    var response = await _clientFactory.Create().Execute(orderRequest);
    var order = response.Result<Order>();

    return new PaymentResult
    {
        TransactionId = order.Id,
        Status = PaymentStatus.Authorized
    };
}

// 2. Authorize after buyer approval
public async Task<string> ConfirmAuthorizationAsync(string orderId)
{
    var client = _clientFactory.Create();
    var request = new OrdersAuthorizeRequest(orderId);
    request.RequestBody(new AuthorizeRequest());

    var response = await client.Execute(request);
    var order = response.Result<Order>();

    return order.PurchaseUnits[0].Payments.Authorizations[0].Id;
}

// 3. Capture the authorized amount (within 3 days for PayPal, 29 days for cards)
public async Task<PaymentResult> CaptureAuthorizationAsync(string authorizationId, decimal? amount = null)
{
    var client = _clientFactory.Create();
    var request = new AuthorizationsCaptureRequest(authorizationId);

    var body = new CaptureRequest();
    if (amount.HasValue)
    {
        body.Amount = new Money { CurrencyCode = "USD", Value = amount.Value.ToString("F2") };
    }
    request.RequestBody(body);

    var response = await client.Execute(request);
    var capture = response.Result<Capture>();

    return new PaymentResult
    {
        IsSuccess = capture.Status == "COMPLETED",
        TransactionId = capture.Id,
        Status = PaymentStatus.Completed
    };
}
```

---

## Checkout Integration

### Server-Side Order Creation Pattern

```csharp
[ApiController]
[Route("api/paypal")]
public class PayPalController : ControllerBase
{
    private readonly PayPalGateway _gateway;

    [HttpPost("orders")]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        var result = await _gateway.CreateOrderAsync(new PaymentRequest
        {
            OrderId = request.OrderId,
            Amount = request.Amount,
            Currency = request.Currency,
            ReturnUrl = $"{Request.Scheme}://{Request.Host}/api/paypal/success",
            CancelUrl = $"{Request.Scheme}://{Request.Host}/api/paypal/cancel"
        });

        return Ok(new { orderId = result.TransactionId });
    }

    [HttpPost("orders/{orderId}/capture")]
    public async Task<IActionResult> CaptureOrder(string orderId)
    {
        var result = await _gateway.CaptureOrderAsync(orderId);

        if (result.IsSuccess)
        {
            // Fulfill the order
            return Ok(new { transactionId = result.TransactionId });
        }

        return BadRequest(new { error = result.Error.Message });
    }
}
```

---

## Subscriptions (Billing Agreements)

```csharp
public class PayPalSubscriptionService
{
    private readonly HttpClient _httpClient;
    private readonly PayPalSettings _settings;

    // Create a billing plan (do once per plan)
    public async Task<string> CreatePlanAsync(PlanCreateRequest request)
    {
        var token = await GetAccessTokenAsync();

        var plan = new
        {
            product_id = request.ProductId,
            name = request.PlanName,
            description = request.Description,
            billing_cycles = new[]
            {
                new
                {
                    frequency = new { interval_unit = "MONTH", interval_count = 1 },
                    tenure_type = "TRIAL",
                    sequence = 1,
                    total_cycles = request.TrialMonths > 0 ? request.TrialMonths : 0,
                    pricing_scheme = new
                    {
                        fixed_price = new { value = "0.00", currency_code = request.Currency }
                    }
                },
                new
                {
                    frequency = new { interval_unit = "MONTH", interval_count = 1 },
                    tenure_type = "REGULAR",
                    sequence = 2,
                    total_cycles = 0, // 0 = infinite
                    pricing_scheme = new
                    {
                        fixed_price = new { value = request.Price.ToString("F2"), currency_code = request.Currency }
                    }
                }
            },
            payment_preferences = new
            {
                auto_bill_outstanding = true,
                payment_failure_threshold = 3
            }
        };

        var response = await _httpClient.PostAsJsonAsync($"{_settings.BaseUrl}/v1/billing/plans", plan);
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();
        return result.GetProperty("id").GetString();
    }

    // Create subscription
    public async Task<SubscriptionResult> CreateSubscriptionAsync(string planId, string returnUrl, string cancelUrl)
    {
        var subscription = new
        {
            plan_id = planId,
            application_context = new
            {
                brand_name = "My Store",
                user_action = "SUBSCRIBE_NOW",
                return_url = returnUrl,
                cancel_url = cancelUrl
            }
        };

        var response = await _httpClient.PostAsJsonAsync($"{_settings.BaseUrl}/v1/billing/subscriptions", subscription);
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();

        var approveLink = result.GetProperty("links")
            .EnumerateArray()
            .First(l => l.GetProperty("rel").GetString() == "approve")
            .GetProperty("href").GetString();

        return new SubscriptionResult
        {
            SubscriptionId = result.GetProperty("id").GetString(),
            Status = result.GetProperty("status").GetString(),
            ApprovalUrl = approveLink
        };
    }

    // Cancel subscription
    public async Task CancelSubscriptionAsync(string subscriptionId, string reason)
    {
        var body = new { reason };
        await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v1/billing/subscriptions/{subscriptionId}/cancel", body);
    }

    private async Task<string> GetAccessTokenAsync()
    {
        var credentials = Convert.ToBase64String(
            Encoding.UTF8.GetBytes($"{_settings.ClientId}:{_settings.ClientSecret}"));

        var request = new HttpRequestMessage(HttpMethod.Post, $"{_settings.BaseUrl}/v1/oauth2/token");
        request.Headers.Authorization = new AuthenticationHeaderValue("Basic", credentials);
        request.Content = new FormUrlEncodedContent(new[] { new KeyValuePair<string, string>("grant_type", "client_credentials") });

        var response = await _httpClient.SendAsync(request);
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();
        return result.GetProperty("access_token").GetString();
    }
}
```

---

## Payouts

Send money to sellers, affiliates, or customers.

```csharp
public class PayPalPayoutService
{
    public async Task<PayoutResult> SendPayoutAsync(PayoutRequest request)
    {
        var payout = new
        {
            sender_batch_header = new
            {
                sender_batch_id = Guid.NewGuid().ToString(),
                email_subject = request.EmailSubject ?? "You have a payment",
                email_message = request.EmailMessage ?? "You received a payment."
            },
            items = request.Recipients.Select(r => new
            {
                recipient_type = "EMAIL",
                amount = new { value = r.Amount.ToString("F2"), currency = request.Currency },
                receiver = r.Email,
                note = r.Note,
                sender_item_id = r.ReferenceId
            }).ToArray()
        };

        var response = await _httpClient.PostAsJsonAsync($"{_settings.BaseUrl}/v1/payments/payouts", payout);
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();

        return new PayoutResult
        {
            BatchId = result.GetProperty("batch_header").GetProperty("payout_batch_id").GetString(),
            Status = result.GetProperty("batch_header").GetProperty("batch_status").GetString()
        };
    }
}
```

---

## Refunds

```csharp
public class PayPalRefundService
{
    public async Task<RefundResult> RefundCaptureAsync(string captureId, decimal? amount = null, string currency = "USD")
    {
        var body = amount.HasValue
            ? new { amount = new { value = amount.Value.ToString("F2"), currency_code = currency } }
            : null;

        var response = await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v2/payments/captures/{captureId}/refund",
            body);

        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadAsStringAsync();
            _logger.LogError("PayPal refund failed: {Error}", error);
            return RefundResult.Failure("Refund failed");
        }

        var result = await response.Content.ReadFromJsonAsync<JsonElement>();

        return new RefundResult
        {
            IsSuccess = true,
            RefundId = result.GetProperty("id").GetString(),
            Status = result.GetProperty("status").GetString()
        };
    }
}
```

---

## Disputes and Chargebacks

```csharp
public class PayPalDisputeService
{
    // List open disputes
    public async Task<List<DisputeInfo>> GetOpenDisputesAsync()
    {
        var response = await _httpClient.GetFromJsonAsync<JsonElement>(
            $"{_settings.BaseUrl}/v1/customer/disputes?status=WAITING_FOR_SELLER_RESPONSE");

        return response.GetProperty("items").EnumerateArray()
            .Select(d => new DisputeInfo
            {
                DisputeId = d.GetProperty("dispute_id").GetString(),
                Reason = d.GetProperty("reason").GetString(),
                Amount = decimal.Parse(d.GetProperty("dispute_amount").GetProperty("value").GetString()),
                Status = d.GetProperty("status").GetString(),
                CreatedAt = d.GetProperty("create_time").GetDateTime()
            }).ToList();
    }

    // Accept claim (issue refund)
    public async Task AcceptDisputeAsync(string disputeId, string note)
    {
        var body = new { note };
        await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v1/customer/disputes/{disputeId}/accept-claim", body);
    }

    // Provide evidence to contest
    public async Task ProvideEvidenceAsync(string disputeId, DisputeEvidence evidence)
    {
        var body = new
        {
            evidence = new[]
            {
                new
                {
                    evidence_type = "PROOF_OF_FULFILLMENT",
                    evidence_info = new
                    {
                        tracking_info = new[]
                        {
                            new
                            {
                                carrier_name = evidence.CarrierName,
                                tracking_number = evidence.TrackingNumber
                            }
                        }
                    },
                    notes = evidence.Notes
                }
            }
        };

        await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v1/customer/disputes/{disputeId}/provide-evidence", body);
    }
}
```

---

## Webhook Handling

```csharp
[ApiController]
[Route("api/webhooks/paypal")]
public class PayPalWebhookController : ControllerBase
{
    private readonly PayPalSettings _settings;
    private readonly ILogger<PayPalWebhookController> _logger;

    [HttpPost]
    public async Task<IActionResult> HandleWebhook()
    {
        var body = await new StreamReader(Request.Body).ReadToEndAsync();

        // Verify webhook signature
        if (!await VerifyWebhookSignature(body))
        {
            _logger.LogWarning("PayPal webhook signature verification failed");
            return BadRequest("Invalid signature");
        }

        var payload = JsonSerializer.Deserialize<JsonElement>(body);
        var eventType = payload.GetProperty("event_type").GetString();

        _logger.LogInformation("PayPal webhook: {EventType}", eventType);

        switch (eventType)
        {
            case "CHECKOUT.ORDER.APPROVED":
                await HandleOrderApproved(payload);
                break;
            case "PAYMENT.CAPTURE.COMPLETED":
                await HandleCaptureCompleted(payload);
                break;
            case "PAYMENT.CAPTURE.DENIED":
                await HandleCaptureDenied(payload);
                break;
            case "PAYMENT.CAPTURE.REFUNDED":
                await HandleRefund(payload);
                break;
            case "BILLING.SUBSCRIPTION.ACTIVATED":
                await HandleSubscriptionActivated(payload);
                break;
            case "BILLING.SUBSCRIPTION.CANCELLED":
                await HandleSubscriptionCancelled(payload);
                break;
            case "CUSTOMER.DISPUTE.CREATED":
                await HandleDisputeCreated(payload);
                break;
            default:
                _logger.LogInformation("Unhandled PayPal event: {EventType}", eventType);
                break;
        }

        return Ok();
    }

    private async Task<bool> VerifyWebhookSignature(string body)
    {
        var verifyRequest = new
        {
            auth_algo = Request.Headers["PAYPAL-AUTH-ALGO"].ToString(),
            cert_url = Request.Headers["PAYPAL-CERT-URL"].ToString(),
            transmission_id = Request.Headers["PAYPAL-TRANSMISSION-ID"].ToString(),
            transmission_sig = Request.Headers["PAYPAL-TRANSMISSION-SIG"].ToString(),
            transmission_time = Request.Headers["PAYPAL-TRANSMISSION-TIME"].ToString(),
            webhook_id = _settings.WebhookId,
            webhook_event = JsonSerializer.Deserialize<object>(body)
        };

        var response = await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v1/notifications/verify-webhook-signature", verifyRequest);

        var result = await response.Content.ReadFromJsonAsync<JsonElement>();
        return result.GetProperty("verification_status").GetString() == "SUCCESS";
    }
}
```

---

## Frontend Integration

### PayPal JavaScript SDK

```html
<!-- Load PayPal JS SDK -->
<script src="https://www.paypal.com/sdk/js?client-id=YOUR_CLIENT_ID&currency=USD"></script>

<div id="paypal-button-container"></div>

<script>
  paypal.Buttons({
    createOrder: async () => {
      const response = await fetch('/api/paypal/orders', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ orderId: 'order-123', amount: 29.99, currency: 'USD' }),
      });
      const { orderId } = await response.json();
      return orderId;
    },

    onApprove: async (data) => {
      const response = await fetch(`/api/paypal/orders/${data.orderID}/capture`, {
        method: 'POST',
      });
      const result = await response.json();

      if (result.transactionId) {
        window.location.href = `/order/confirmation?txn=${result.transactionId}`;
      }
    },

    onError: (err) => {
      console.error('PayPal error:', err);
    },

    onCancel: () => {
      console.log('Payment cancelled by user');
    },

    style: {
      layout: 'vertical',
      color: 'blue',
      shape: 'rect',
      label: 'paypal',
    },
  }).render('#paypal-button-container');
</script>
```

---

## Production Checklist

### Before Go-Live

- [ ] Switch from sandbox to live credentials
- [ ] Store credentials in Azure Key Vault / AWS Secrets Manager
- [ ] Enable webhook signature verification
- [ ] Configure webhook endpoints in PayPal Developer Portal
- [ ] Set up IPN (Instant Payment Notification) as backup to webhooks
- [ ] Test all payment flows end-to-end in sandbox
- [ ] Configure automatic retry for failed webhook deliveries
- [ ] Implement idempotency on capture and refund endpoints
- [ ] Handle currency-specific decimal rules
- [ ] Set up dispute monitoring and auto-response rules

### PayPal vs Stripe Feature Comparison

| Feature | PayPal | Stripe |
|---------|--------|--------|
| Checkout UX | Redirect / popup | Embedded elements |
| Subscription billing | Billing Plans API | Subscriptions API |
| Marketplace | Commerce Platform | Connect |
| Payouts to sellers | Payouts API | Connect payouts |
| Dispute handling | Disputes API | Disputes API |
| 3D Secure | Automatic for cards | Configurable |
| Zero-code checkout | PayPal Buttons | Payment Links |
| PCI scope | SAQ-A (redirect) | SAQ-A (Elements) |

## Next Steps

- [Stripe Integration](16-STRIPE-INTEGRATION.md)
- [Common Scenarios](18-COMMON-SCENARIOS.md)
- [Test Cards & Sandbox](13-TEST-CARDS-SANDBOX.md)
