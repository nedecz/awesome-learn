# Stripe Integration Guide

## Overview

Comprehensive guide for integrating Stripe into .NET 10.0+ payment systems. Covers Payment Intents, Checkout Sessions, Customer management, Subscriptions, Connect (marketplace payments), and production best practices.

## Table of Contents

1. [Setup and Configuration](#setup-and-configuration)
2. [Payment Intents API](#payment-intents-api)
3. [Checkout Sessions](#checkout-sessions)
4. [Customer Management](#customer-management)
5. [Subscriptions and Recurring Billing](#subscriptions-and-recurring-billing)
6. [Stripe Connect (Marketplace)](#stripe-connect-marketplace)
7. [Refunds](#refunds)
8. [Webhook Handling](#webhook-handling)
9. [Stripe Elements (Frontend)](#stripe-elements-frontend)
10. [Production Checklist](#production-checklist)

---

## Setup and Configuration

### NuGet Package

```bash
dotnet add package Stripe.net
```

### Configuration

```json
{
  "Stripe": {
    "SecretKey": "sk_test_...",
    "PublishableKey": "pk_test_...",
    "WebhookSecret": "whsec_...",
    "ApiVersion": "2024-04-10"
  }
}
```

```csharp
public class StripeSettings
{
    public string SecretKey { get; set; }
    public string PublishableKey { get; set; }
    public string WebhookSecret { get; set; }
    public string ApiVersion { get; set; }
}

// Program.cs
builder.Services.Configure<StripeSettings>(builder.Configuration.GetSection("Stripe"));
builder.Services.AddSingleton<IStripeClient>(sp =>
{
    var settings = sp.GetRequiredService<IOptions<StripeSettings>>().Value;
    return new StripeClient(settings.SecretKey);
});

// Register Stripe services
builder.Services.AddScoped<PaymentIntentService>();
builder.Services.AddScoped<CustomerService>();
builder.Services.AddScoped<RefundService>();
builder.Services.AddScoped<SubscriptionService>();
```

---

## Payment Intents API

The recommended approach for one-time payments with SCA (Strong Customer Authentication) support.

### Create Payment Intent

```csharp
public class StripePaymentGateway : IPaymentGateway
{
    private readonly IStripeClient _client;
    private readonly ILogger<StripePaymentGateway> _logger;

    public string ProviderName => "Stripe";

    public StripePaymentGateway(IStripeClient client, ILogger<StripePaymentGateway> logger)
    {
        _client = client;
        _logger = logger;
    }

    public async Task<PaymentResult> CreatePaymentIntentAsync(PaymentRequest request)
    {
        var service = new PaymentIntentService(_client);

        var options = new PaymentIntentCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
            Currency = request.Currency.ToLower(),
            PaymentMethodTypes = new List<string> { "card" },
            Customer = request.StripeCustomerId,
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId,
                ["customer_email"] = request.CustomerEmail
            },
            Description = $"Payment for order {request.OrderId}",
            StatementDescriptor = "MYSTORE",
            IdempotencyKey = request.IdempotencyKey
        };

        // Optionally confirm immediately (server-side)
        if (!string.IsNullOrEmpty(request.PaymentMethodId))
        {
            options.PaymentMethod = request.PaymentMethodId;
            options.Confirm = true;
            options.ReturnUrl = request.ReturnUrl;
        }

        try
        {
            var intent = await service.CreateAsync(options);

            return new PaymentResult
            {
                IsSuccess = intent.Status is "succeeded" or "requires_capture",
                TransactionId = intent.Id,
                Status = MapStatus(intent.Status),
                ClientSecret = intent.ClientSecret,    // For frontend confirmation
                RequiresAction = intent.Status == "requires_action",
                ActionUrl = intent.NextAction?.RedirectToUrl?.Url
            };
        }
        catch (StripeException ex)
        {
            _logger.LogError(ex, "Stripe payment failed for order {OrderId}", request.OrderId);
            throw StripeExceptionTranslator.Translate(ex);
        }
    }

    // Confirm after 3D Secure
    public async Task<PaymentResult> ConfirmPaymentIntentAsync(string paymentIntentId)
    {
        var service = new PaymentIntentService(_client);
        var intent = await service.ConfirmAsync(paymentIntentId);

        return new PaymentResult
        {
            IsSuccess = intent.Status == "succeeded",
            TransactionId = intent.Id,
            Status = MapStatus(intent.Status)
        };
    }

    // Capture a pre-authorized payment
    public async Task<PaymentResult> CapturePaymentIntentAsync(string paymentIntentId, long? amountToCapture = null)
    {
        var service = new PaymentIntentService(_client);
        var options = new PaymentIntentCaptureOptions();

        if (amountToCapture.HasValue)
            options.AmountToCapture = amountToCapture.Value;

        var intent = await service.CaptureAsync(paymentIntentId, options);

        return new PaymentResult
        {
            IsSuccess = intent.Status == "succeeded",
            TransactionId = intent.Id,
            Status = PaymentStatus.Completed
        };
    }

    private static long ConvertToSmallestUnit(decimal amount, string currency)
    {
        // Zero-decimal currencies (JPY, KRW, etc.)
        var zeroDecimal = new HashSet<string> { "bif", "clp", "djf", "gnf", "jpy", "kmf", "krw", "mga", "pyg", "rwf", "ugx", "vnd", "vuv", "xaf", "xof", "xpf" };
        return zeroDecimal.Contains(currency.ToLower())
            ? (long)amount
            : (long)(amount * 100);
    }

    private static PaymentStatus MapStatus(string stripeStatus) => stripeStatus switch
    {
        "succeeded" => PaymentStatus.Completed,
        "requires_payment_method" => PaymentStatus.Failed,
        "requires_confirmation" => PaymentStatus.Pending,
        "requires_action" => PaymentStatus.RequiresAction,
        "processing" => PaymentStatus.Processing,
        "requires_capture" => PaymentStatus.Authorized,
        "canceled" => PaymentStatus.Cancelled,
        _ => PaymentStatus.Unknown
    };
}
```

---

## Checkout Sessions

Use Stripe-hosted checkout for a faster integration path.

```csharp
public class StripeCheckoutService
{
    private readonly IStripeClient _client;

    public async Task<CheckoutSessionResult> CreateCheckoutSessionAsync(CheckoutRequest request)
    {
        var service = new SessionService(_client);

        var options = new SessionCreateOptions
        {
            Mode = request.IsSubscription ? "subscription" : "payment",
            SuccessUrl = $"{request.BaseUrl}/checkout/success?session_id={{CHECKOUT_SESSION_ID}}",
            CancelUrl = $"{request.BaseUrl}/checkout/cancel",
            Customer = request.StripeCustomerId,
            CustomerEmail = request.StripeCustomerId == null ? request.CustomerEmail : null,
            LineItems = request.Items.Select(item => new SessionLineItemOptions
            {
                PriceData = new SessionLineItemPriceDataOptions
                {
                    Currency = request.Currency.ToLower(),
                    UnitAmount = ConvertToSmallestUnit(item.UnitPrice, request.Currency),
                    ProductData = new SessionLineItemPriceDataProductDataOptions
                    {
                        Name = item.Name,
                        Description = item.Description,
                        Images = item.ImageUrls
                    }
                },
                Quantity = item.Quantity
            }).ToList(),
            PaymentIntentData = request.IsSubscription ? null : new SessionPaymentIntentDataOptions
            {
                Metadata = new Dictionary<string, string>
                {
                    ["order_id"] = request.OrderId
                }
            },
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId
            },
            // Collect billing address
            BillingAddressCollection = "required",
            // Collect shipping address
            ShippingAddressCollection = request.RequiresShipping
                ? new SessionShippingAddressCollectionOptions
                {
                    AllowedCountries = new List<string> { "US", "CA", "GB", "DE", "FR" }
                }
                : null,
            // Automatic tax calculation
            AutomaticTax = new SessionAutomaticTaxOptions { Enabled = true }
        };

        var session = await service.CreateAsync(options);

        return new CheckoutSessionResult
        {
            SessionId = session.Id,
            CheckoutUrl = session.Url
        };
    }

    // Retrieve session after success redirect
    public async Task<CheckoutSessionDetails> GetSessionDetailsAsync(string sessionId)
    {
        var service = new SessionService(_client);
        var session = await service.GetAsync(sessionId, new SessionGetOptions
        {
            Expand = new List<string> { "line_items", "payment_intent", "customer" }
        });

        return new CheckoutSessionDetails
        {
            PaymentIntentId = session.PaymentIntentId,
            CustomerId = session.CustomerId,
            CustomerEmail = session.CustomerDetails?.Email,
            PaymentStatus = session.PaymentStatus,
            AmountTotal = session.AmountTotal,
            Currency = session.Currency
        };
    }
}
```

---

## Customer Management

```csharp
public class StripeCustomerService
{
    private readonly IStripeClient _client;

    public async Task<StripeCustomer> CreateCustomerAsync(CustomerCreateRequest request)
    {
        var service = new CustomerService(_client);

        var options = new CustomerCreateOptions
        {
            Email = request.Email,
            Name = request.Name,
            Phone = request.Phone,
            Address = new AddressOptions
            {
                Line1 = request.Address.Line1,
                City = request.Address.City,
                State = request.Address.State,
                PostalCode = request.Address.PostalCode,
                Country = request.Address.Country
            },
            Metadata = new Dictionary<string, string>
            {
                ["internal_customer_id"] = request.InternalId
            }
        };

        var customer = await service.CreateAsync(options);
        return MapToCustomer(customer);
    }

    // Attach payment method to customer
    public async Task AttachPaymentMethodAsync(string customerId, string paymentMethodId)
    {
        var service = new PaymentMethodService(_client);
        await service.AttachAsync(paymentMethodId, new PaymentMethodAttachOptions
        {
            Customer = customerId
        });

        // Set as default payment method
        var customerService = new CustomerService(_client);
        await customerService.UpdateAsync(customerId, new CustomerUpdateOptions
        {
            InvoiceSettings = new CustomerInvoiceSettingsOptions
            {
                DefaultPaymentMethod = paymentMethodId
            }
        });
    }

    // List payment methods
    public async Task<List<PaymentMethodInfo>> ListPaymentMethodsAsync(string customerId)
    {
        var service = new PaymentMethodService(_client);
        var methods = await service.ListAsync(new PaymentMethodListOptions
        {
            Customer = customerId,
            Type = "card"
        });

        return methods.Data.Select(m => new PaymentMethodInfo
        {
            Id = m.Id,
            Brand = m.Card.Brand,
            Last4 = m.Card.Last4,
            ExpMonth = m.Card.ExpMonth,
            ExpYear = m.Card.ExpYear,
            IsDefault = m.Id == methods.Data.First().Id
        }).ToList();
    }
}
```

---

## Subscriptions and Recurring Billing

```csharp
public class StripeSubscriptionService
{
    private readonly IStripeClient _client;

    public async Task<SubscriptionResult> CreateSubscriptionAsync(SubscriptionCreateRequest request)
    {
        var service = new SubscriptionService(_client);

        var options = new SubscriptionCreateOptions
        {
            Customer = request.StripeCustomerId,
            Items = new List<SubscriptionItemOptions>
            {
                new() { Price = request.StripePriceId }
            },
            DefaultPaymentMethod = request.PaymentMethodId,
            PaymentBehavior = "default_incomplete",
            PaymentSettings = new SubscriptionPaymentSettingsOptions
            {
                SaveDefaultPaymentMethod = "on_subscription"
            },
            Expand = new List<string> { "latest_invoice.payment_intent" },
            Metadata = new Dictionary<string, string>
            {
                ["plan_name"] = request.PlanName,
                ["internal_user_id"] = request.UserId
            },
            // Trial period
            TrialPeriodDays = request.TrialDays > 0 ? request.TrialDays : null,
            // Proration behavior
            ProrationBehavior = "create_prorations"
        };

        var subscription = await service.CreateAsync(options);

        return new SubscriptionResult
        {
            SubscriptionId = subscription.Id,
            Status = subscription.Status,
            ClientSecret = subscription.LatestInvoice?.PaymentIntent?.ClientSecret,
            CurrentPeriodEnd = subscription.CurrentPeriodEnd
        };
    }

    // Upgrade/downgrade subscription
    public async Task<SubscriptionResult> UpdateSubscriptionAsync(string subscriptionId, string newPriceId)
    {
        var service = new SubscriptionService(_client);
        var subscription = await service.GetAsync(subscriptionId);

        var options = new SubscriptionUpdateOptions
        {
            Items = new List<SubscriptionItemOptions>
            {
                new()
                {
                    Id = subscription.Items.Data[0].Id,
                    Price = newPriceId
                }
            },
            ProrationBehavior = "always_invoice"
        };

        var updated = await service.UpdateAsync(subscriptionId, options);

        return new SubscriptionResult
        {
            SubscriptionId = updated.Id,
            Status = updated.Status,
            CurrentPeriodEnd = updated.CurrentPeriodEnd
        };
    }

    // Cancel subscription
    public async Task CancelSubscriptionAsync(string subscriptionId, bool cancelImmediately = false)
    {
        var service = new SubscriptionService(_client);

        if (cancelImmediately)
        {
            await service.CancelAsync(subscriptionId);
        }
        else
        {
            // Cancel at end of billing period
            await service.UpdateAsync(subscriptionId, new SubscriptionUpdateOptions
            {
                CancelAtPeriodEnd = true
            });
        }
    }
}
```

---

## Stripe Connect (Marketplace)

For marketplace or platform payments where funds are split between the platform and sellers.

```csharp
public class StripeConnectService
{
    private readonly IStripeClient _client;

    // Create Connected Account
    public async Task<ConnectedAccountResult> CreateConnectedAccountAsync(SellerOnboardingRequest request)
    {
        var service = new AccountService(_client);

        var options = new AccountCreateOptions
        {
            Type = "express",
            Email = request.Email,
            Country = request.Country,
            Capabilities = new AccountCapabilitiesOptions
            {
                CardPayments = new AccountCapabilitiesCardPaymentsOptions { Requested = true },
                Transfers = new AccountCapabilitiesTransfersOptions { Requested = true }
            },
            BusinessProfile = new AccountBusinessProfileOptions
            {
                Url = request.WebsiteUrl
            }
        };

        var account = await service.CreateAsync(options);

        // Generate onboarding link
        var linkService = new AccountLinkService(_client);
        var link = await linkService.CreateAsync(new AccountLinkCreateOptions
        {
            Account = account.Id,
            RefreshUrl = $"{request.BaseUrl}/connect/refresh",
            ReturnUrl = $"{request.BaseUrl}/connect/return",
            Type = "account_onboarding"
        });

        return new ConnectedAccountResult
        {
            AccountId = account.Id,
            OnboardingUrl = link.Url
        };
    }

    // Direct charge with application fee
    public async Task<PaymentResult> CreateDirectChargeAsync(MarketplacePaymentRequest request)
    {
        var service = new PaymentIntentService(_client);

        var options = new PaymentIntentCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
            Currency = request.Currency.ToLower(),
            ApplicationFeeAmount = ConvertToSmallestUnit(request.PlatformFee, request.Currency),
            PaymentMethod = request.PaymentMethodId,
            Confirm = true,
            TransferData = new PaymentIntentTransferDataOptions
            {
                Destination = request.SellerStripeAccountId
            }
        };

        var requestOptions = new RequestOptions
        {
            IdempotencyKey = request.IdempotencyKey
        };

        var intent = await service.CreateAsync(options, requestOptions);

        return new PaymentResult
        {
            IsSuccess = intent.Status == "succeeded",
            TransactionId = intent.Id,
            Status = MapStatus(intent.Status)
        };
    }

    // Separate charges and transfers (more control)
    public async Task<PaymentResult> CreateSeparateChargeAsync(MarketplacePaymentRequest request)
    {
        // 1. Charge the customer on the platform
        var intentService = new PaymentIntentService(_client);
        var intent = await intentService.CreateAsync(new PaymentIntentCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
            Currency = request.Currency.ToLower(),
            PaymentMethod = request.PaymentMethodId,
            Confirm = true
        });

        // 2. Transfer to connected account
        var transferService = new TransferService(_client);
        await transferService.CreateAsync(new TransferCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount - request.PlatformFee, request.Currency),
            Currency = request.Currency.ToLower(),
            Destination = request.SellerStripeAccountId,
            SourceTransaction = intent.LatestChargeId
        });

        return new PaymentResult
        {
            IsSuccess = true,
            TransactionId = intent.Id
        };
    }
}
```

---

## Refunds

```csharp
public class StripeRefundService
{
    private readonly IStripeClient _client;

    public async Task<RefundResult> CreateRefundAsync(RefundRequest request)
    {
        var service = new Stripe.RefundService(_client);

        var options = new RefundCreateOptions
        {
            PaymentIntent = request.PaymentIntentId,
            Amount = request.IsPartial
                ? ConvertToSmallestUnit(request.Amount, request.Currency)
                : null, // null = full refund
            Reason = request.Reason switch
            {
                RefundReason.Duplicate => "duplicate",
                RefundReason.Fraudulent => "fraudulent",
                RefundReason.CustomerRequest => "requested_by_customer",
                _ => null
            },
            Metadata = new Dictionary<string, string>
            {
                ["refund_reason"] = request.ReasonDescription,
                ["refunded_by"] = request.InitiatedBy
            }
        };

        try
        {
            var refund = await service.CreateAsync(options);

            return new RefundResult
            {
                IsSuccess = refund.Status == "succeeded",
                RefundId = refund.Id,
                Amount = refund.Amount / 100m,
                Status = refund.Status
            };
        }
        catch (StripeException ex) when (ex.StripeError.Code == "charge_already_refunded")
        {
            return new RefundResult
            {
                IsSuccess = false,
                ErrorMessage = "This payment has already been fully refunded."
            };
        }
    }
}
```

---

## Webhook Handling

```csharp
[ApiController]
[Route("api/webhooks/stripe")]
public class StripeWebhookController : ControllerBase
{
    private readonly StripeSettings _settings;
    private readonly IPaymentEventHandler _eventHandler;
    private readonly ILogger<StripeWebhookController> _logger;

    [HttpPost]
    public async Task<IActionResult> HandleWebhook()
    {
        var json = await new StreamReader(HttpContext.Request.Body).ReadToEndAsync();

        Event stripeEvent;
        try
        {
            stripeEvent = EventUtility.ConstructEvent(
                json,
                Request.Headers["Stripe-Signature"],
                _settings.WebhookSecret);
        }
        catch (StripeException ex)
        {
            _logger.LogWarning(ex, "Stripe webhook signature verification failed");
            return BadRequest("Invalid signature");
        }

        _logger.LogInformation("Stripe webhook received: {EventType} ({EventId})", stripeEvent.Type, stripeEvent.Id);

        try
        {
            await ProcessEvent(stripeEvent);
            return Ok();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing Stripe webhook {EventType}", stripeEvent.Type);
            return StatusCode(500);
        }
    }

    private async Task ProcessEvent(Event stripeEvent)
    {
        switch (stripeEvent.Type)
        {
            case "payment_intent.succeeded":
                var intent = stripeEvent.Data.Object as PaymentIntent;
                await _eventHandler.HandlePaymentSucceededAsync(intent);
                break;

            case "payment_intent.payment_failed":
                var failedIntent = stripeEvent.Data.Object as PaymentIntent;
                await _eventHandler.HandlePaymentFailedAsync(failedIntent);
                break;

            case "charge.refunded":
                var charge = stripeEvent.Data.Object as Charge;
                await _eventHandler.HandleRefundAsync(charge);
                break;

            case "invoice.payment_succeeded":
                var invoice = stripeEvent.Data.Object as Invoice;
                await _eventHandler.HandleInvoicePaidAsync(invoice);
                break;

            case "invoice.payment_failed":
                var failedInvoice = stripeEvent.Data.Object as Invoice;
                await _eventHandler.HandleInvoiceFailedAsync(failedInvoice);
                break;

            case "customer.subscription.updated":
                var subscription = stripeEvent.Data.Object as Subscription;
                await _eventHandler.HandleSubscriptionUpdatedAsync(subscription);
                break;

            case "customer.subscription.deleted":
                var cancelledSub = stripeEvent.Data.Object as Subscription;
                await _eventHandler.HandleSubscriptionCancelledAsync(cancelledSub);
                break;

            case "charge.dispute.created":
                var dispute = stripeEvent.Data.Object as Dispute;
                await _eventHandler.HandleDisputeCreatedAsync(dispute);
                break;

            default:
                _logger.LogInformation("Unhandled Stripe event type: {EventType}", stripeEvent.Type);
                break;
        }
    }
}
```

---

## Stripe Elements (Frontend)

### React Integration

```tsx
import { loadStripe } from '@stripe/stripe-js';
import { Elements, CardElement, useStripe, useElements } from '@stripe/react-stripe-js';

const stripePromise = loadStripe('pk_test_...');

function CheckoutForm() {
  const stripe = useStripe();
  const elements = useElements();

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();

    if (!stripe || !elements) return;

    // 1. Get client secret from your server
    const response = await fetch('/api/payments/create-intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount: 2999, currency: 'usd', orderId: 'order-123' }),
    });
    const { clientSecret } = await response.json();

    // 2. Confirm payment on the client
    const result = await stripe.confirmCardPayment(clientSecret, {
      payment_method: {
        card: elements.getElement(CardElement)!,
        billing_details: { name: 'Jenny Rosen' },
      },
    });

    if (result.error) {
      console.error(result.error.message);
    } else if (result.paymentIntent?.status === 'succeeded') {
      // Payment succeeded — redirect or show confirmation
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <CardElement options={{
        style: {
          base: { fontSize: '16px', color: '#424770' },
          invalid: { color: '#9e2146' },
        },
      }} />
      <button type="submit" disabled={!stripe}>Pay</button>
    </form>
  );
}

// Wrap in Elements provider
function App() {
  return (
    <Elements stripe={stripePromise}>
      <CheckoutForm />
    </Elements>
  );
}
```

---

## Production Checklist

### Before Go-Live

- [ ] Switch from `sk_test_` to `sk_live_` keys (stored in Azure Key Vault or AWS Secrets Manager)
- [ ] Enable webhook signature verification with `whsec_` secret
- [ ] Configure idempotency keys on all payment creation endpoints
- [ ] Set up Stripe Radar for fraud detection
- [ ] Enable 3D Secure for SCA compliance (EU/UK)
- [ ] Set `statement_descriptor` for recognizable charges
- [ ] Configure automatic retry on failed invoices (subscription billing)
- [ ] Set up Stripe webhook retry policy with dead letter queue
- [ ] Test all webhook event types in Stripe CLI before deployment
- [ ] Configure PCI-compliant logging (never log full card numbers)

### Monitoring

- [ ] Alert on payment failure rate > 5%
- [ ] Alert on webhook processing failures
- [ ] Monitor Stripe dashboard for disputes
- [ ] Track conversion rates by payment method
- [ ] Set up Stripe Sigma for custom reporting

### Testing

- [ ] Use [Test Cards](13-TEST-CARDS-SANDBOX.md) for all scenarios (success, decline, 3DS)
- [ ] Test webhook handling with Stripe CLI: `stripe listen --forward-to localhost:5000/api/webhooks/stripe`
- [ ] Test subscription lifecycle: create → upgrade → downgrade → cancel
- [ ] Test refund flows: full, partial, dispute
- [ ] Load test payment endpoints

## Next Steps

- [PayPal Integration](17-PAYPAL-INTEGRATION.md)
- [Test Cards & Sandbox](13-TEST-CARDS-SANDBOX.md)
- [Webhook Patterns](04-WEBHOOK-PATTERNS.md)
- [Error Handling](05-ERROR-HANDLING.md)
