# Payment Methods and Flows

## Overview

A detailed guide to payment methods, their characteristics, and implementation flows across multiple providers. This document covers credit/debit cards, digital wallets, bank transfers, Buy Now Pay Later (BNPL), and alternative payment methods — with practical .NET 10.0+ examples for Stripe, PayPal, Square, Adyen, and Braintree.

## Table of Contents

1. [Payment Method Categories](#payment-method-categories)
2. [Credit and Debit Cards](#credit-and-debit-cards)
3. [Digital Wallets](#digital-wallets)
4. [Bank Transfers and Direct Debit](#bank-transfers-and-direct-debit)
5. [Buy Now Pay Later (BNPL)](#buy-now-pay-later-bnpl)
6. [Alternative and Regional Payment Methods](#alternative-and-regional-payment-methods)
7. [Multi-Provider Payment Method Abstraction](#multi-provider-payment-method-abstraction)
8. [Payment Flow Diagrams](#payment-flow-diagrams)
9. [Provider Comparison Matrix](#provider-comparison-matrix)
10. [Implementation Checklist](#implementation-checklist)

---

## Payment Method Categories

| Category | Examples | Typical Flow | Settlement Time |
|----------|----------|-------------|-----------------|
| Cards | Visa, Mastercard, Amex, Discover | Authorize → Capture → Settle | 2–3 business days |
| Digital Wallets | Apple Pay, Google Pay, PayPal | Token → Authorize → Capture | 1–3 business days |
| Bank Transfers | ACH, SEPA, iDEAL, Bancontact | Initiate → Confirm → Settle | 1–5 business days |
| BNPL | Klarna, Afterpay, Affirm | Redirect → Authorize → Capture | 2–5 business days |
| Alternative | Boleto, OXXO, GrabPay, WeChat Pay | Generate voucher/QR → Await payment | Varies by method |

### Payment Method Lifecycle

```
Customer selects method → Collect payment details → Authorize
    ↓                                                   ↓
Method-specific UI                              Provider API call
(card form, wallet,                             (Stripe, PayPal, etc.)
 redirect, QR code)                                     ↓
                                                Capture / Confirm
                                                        ↓
                                                  Settlement
                                                        ↓
                                                  Reconciliation
```

---

## Credit and Debit Cards

Cards remain the most common payment method worldwide. Each provider has a slightly different API surface, but the underlying flow is the same: collect card details securely (via tokenization), authorize, then capture.

### Card Flow Overview

```
Customer enters card → Tokenized on client → Token sent to server
    ↓
Server calls provider API with token
    ↓
Provider returns: Authorized / Declined / Requires 3DS
    ↓
If 3DS required → Redirect/challenge → Confirm
    ↓
Capture (immediate or deferred)
    ↓
Settlement (2–3 business days)
```

### Stripe — Card Payment with Payment Intents

```csharp
public class StripeCardPaymentService
{
    private readonly IStripeClient _client;

    public async Task<PaymentResult> ProcessCardPaymentAsync(CardPaymentRequest request)
    {
        var service = new PaymentIntentService(_client);

        var options = new PaymentIntentCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
            Currency = request.Currency.ToLower(),
            PaymentMethodTypes = new List<string> { "card" },
            PaymentMethod = request.PaymentMethodToken,
            Confirm = true,
            ReturnUrl = request.ReturnUrl,
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId
            }
        };

        var intent = await service.CreateAsync(options, new RequestOptions
        {
            IdempotencyKey = request.IdempotencyKey
        });

        return new PaymentResult
        {
            TransactionId = intent.Id,
            Status = intent.Status switch
            {
                "succeeded" => PaymentStatus.Completed,
                "requires_action" => PaymentStatus.RequiresAction,
                "requires_capture" => PaymentStatus.Authorized,
                _ => PaymentStatus.Pending
            },
            RequiresAction = intent.Status == "requires_action",
            ClientSecret = intent.ClientSecret
        };
    }
}
```

### PayPal — Card Payment via Orders API

PayPal processes card payments through its Orders API. The card details are tokenized on the client side using the PayPal JavaScript SDK.

```csharp
public class PayPalCardPaymentService
{
    private readonly HttpClient _httpClient;
    private readonly PayPalSettings _settings;

    public async Task<PaymentResult> ProcessCardAsync(CardPaymentRequest request)
    {
        var order = new
        {
            intent = "CAPTURE",
            purchase_units = new[]
            {
                new
                {
                    reference_id = request.OrderId,
                    amount = new
                    {
                        currency_code = request.Currency,
                        value = request.Amount.ToString("F2")
                    }
                }
            },
            payment_source = new
            {
                card = new
                {
                    vault_id = request.PaymentMethodToken
                }
            }
        };

        var response = await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v2/checkout/orders", order);
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();

        return new PaymentResult
        {
            TransactionId = result.GetProperty("id").GetString(),
            Status = result.GetProperty("status").GetString() == "COMPLETED"
                ? PaymentStatus.Completed
                : PaymentStatus.Pending
        };
    }
}
```

### Square — Card Payment via Payments API

```csharp
public class SquareCardPaymentService
{
    private readonly ISquareClient _client;

    public async Task<PaymentResult> ProcessCardAsync(CardPaymentRequest request)
    {
        var body = new CreatePaymentRequest.Builder(
                sourceId: request.PaymentMethodToken,
                idempotencyKey: request.IdempotencyKey)
            .AmountMoney(new Money.Builder()
                .Amount(ConvertToSmallestUnit(request.Amount, request.Currency))
                .Currency(request.Currency)
                .Build())
            .ReferenceId(request.OrderId)
            .Note($"Payment for order {request.OrderId}")
            .Build();

        var result = await _client.PaymentsApi.CreatePaymentAsync(body);

        return new PaymentResult
        {
            TransactionId = result.Payment.Id,
            Status = result.Payment.Status == "COMPLETED"
                ? PaymentStatus.Completed
                : PaymentStatus.Pending
        };
    }
}
```

### Adyen — Card Payment via Checkout API

```csharp
public class AdyenCardPaymentService
{
    private readonly HttpClient _httpClient;
    private readonly AdyenSettings _settings;

    public async Task<PaymentResult> ProcessCardAsync(CardPaymentRequest request)
    {
        var paymentRequest = new
        {
            merchantAccount = _settings.MerchantAccount,
            amount = new
            {
                value = ConvertToSmallestUnit(request.Amount, request.Currency),
                currency = request.Currency
            },
            reference = request.OrderId,
            paymentMethod = new
            {
                type = "scheme",
                storedPaymentMethodId = request.PaymentMethodToken
            },
            returnUrl = request.ReturnUrl,
            shopperInteraction = "Ecommerce",
            recurringProcessingModel = "CardOnFile"
        };

        var response = await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v71/payments", paymentRequest);
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();

        var resultCode = result.GetProperty("resultCode").GetString();

        return new PaymentResult
        {
            TransactionId = result.GetProperty("pspReference").GetString(),
            Status = resultCode switch
            {
                "Authorised" => PaymentStatus.Completed,
                "RedirectShopper" => PaymentStatus.RequiresAction,
                "Refused" => PaymentStatus.Failed,
                _ => PaymentStatus.Pending
            },
            RequiresAction = resultCode == "RedirectShopper",
            ActionUrl = resultCode == "RedirectShopper"
                ? result.GetProperty("action").GetProperty("url").GetString()
                : null
        };
    }
}
```

### Braintree — Card Payment via Transaction API

```csharp
public class BraintreeCardPaymentService
{
    private readonly IBraintreeGateway _gateway;

    public async Task<PaymentResult> ProcessCardAsync(CardPaymentRequest request)
    {
        var transactionRequest = new TransactionRequest
        {
            Amount = request.Amount,
            PaymentMethodNonce = request.PaymentMethodToken,
            OrderId = request.OrderId,
            Options = new TransactionOptionsRequest
            {
                SubmitForSettlement = true
            }
        };

        var result = await _gateway.Transaction.SaleAsync(transactionRequest);

        return new PaymentResult
        {
            TransactionId = result.Target?.Id,
            Status = result.IsSuccess()
                ? PaymentStatus.Completed
                : PaymentStatus.Failed,
            ErrorMessage = result.IsSuccess() ? null : result.Message
        };
    }
}
```

### 3D Secure (SCA) Flow

3D Secure is required for Strong Customer Authentication (SCA) in the EU/UK. Each provider handles 3DS differently but the conceptual flow is the same:

```
1. Server creates payment → Provider returns "requires_action"
2. Client presents challenge (iframe / redirect)
3. Customer completes authentication
4. Client confirms payment
5. Server receives webhook confirming success/failure
```

```csharp
// Handling 3DS across providers
public interface IThreeDSecureHandler
{
    bool RequiresAuthentication(PaymentResult result);
    string GetAuthenticationUrl(PaymentResult result);
    Task<PaymentResult> ConfirmAuthenticationAsync(string transactionId);
}

public class StripeThreeDSecureHandler : IThreeDSecureHandler
{
    private readonly IStripeClient _client;

    public StripeThreeDSecureHandler(IStripeClient client)
    {
        _client = client;
    }

    public bool RequiresAuthentication(PaymentResult result)
        => result.Status == PaymentStatus.RequiresAction;

    public string GetAuthenticationUrl(PaymentResult result)
        => result.ActionUrl;

    public async Task<PaymentResult> ConfirmAuthenticationAsync(string paymentIntentId)
    {
        var service = new PaymentIntentService(_client);
        var intent = await service.GetAsync(paymentIntentId);

        return new PaymentResult
        {
            TransactionId = intent.Id,
            Status = intent.Status == "succeeded"
                ? PaymentStatus.Completed
                : PaymentStatus.Failed
        };
    }
}
```

---

## Digital Wallets

Digital wallets (Apple Pay, Google Pay, PayPal) provide a faster checkout experience by using pre-stored payment credentials. They typically produce a payment token that is processed like a card.

### Digital Wallet Flow

```
Customer taps wallet button → Device authentication (Face ID / fingerprint)
    ↓
Wallet generates encrypted token
    ↓
Token sent to your server
    ↓
Server sends token to provider → Authorize → Capture
    ↓
Settlement
```

### Apple Pay

#### Stripe — Apple Pay via Payment Request API

```csharp
// Server side — no special handling needed
// Apple Pay generates a PaymentMethod token that is used like a card token
public async Task<PaymentResult> ProcessApplePayAsync(WalletPaymentRequest request)
{
    var service = new PaymentIntentService(_client);

    var options = new PaymentIntentCreateOptions
    {
        Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
        Currency = request.Currency.ToLower(),
        PaymentMethod = request.WalletToken, // Apple Pay token
        Confirm = true,
        PaymentMethodTypes = new List<string> { "card" },
        Metadata = new Dictionary<string, string>
        {
            ["order_id"] = request.OrderId,
            ["wallet_type"] = "apple_pay"
        }
    };

    var intent = await service.CreateAsync(options);

    return new PaymentResult
    {
        TransactionId = intent.Id,
        Status = intent.Status == "succeeded"
            ? PaymentStatus.Completed
            : PaymentStatus.Pending
    };
}
```

#### Frontend — Apple Pay Button

```javascript
// Using Stripe Payment Request API for Apple Pay
const paymentRequest = stripe.paymentRequest({
  country: 'US',
  currency: 'usd',
  total: { label: 'My Store', amount: 2999 },
  requestPayerName: true,
  requestPayerEmail: true,
});

// Check if Apple Pay is available
const canMakePayment = await paymentRequest.canMakePayment();

if (canMakePayment?.applePay) {
  const prButton = elements.create('paymentRequestButton', {
    paymentRequest,
  });
  prButton.mount('#apple-pay-button');
}

paymentRequest.on('paymentmethod', async (ev) => {
  // Send ev.paymentMethod.id to your server
  const response = await fetch('/api/payments/wallet', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      paymentMethodId: ev.paymentMethod.id,
      amount: 2999,
      currency: 'usd',
    }),
  });

  const result = await response.json();
  ev.complete(result.success ? 'success' : 'fail');
});
```

### Google Pay

#### Adyen — Google Pay Integration

```csharp
public class AdyenGooglePayService
{
    private readonly HttpClient _httpClient;
    private readonly AdyenSettings _settings;

    public async Task<PaymentResult> ProcessGooglePayAsync(WalletPaymentRequest request)
    {
        var paymentRequest = new
        {
            merchantAccount = _settings.MerchantAccount,
            amount = new
            {
                value = ConvertToSmallestUnit(request.Amount, request.Currency),
                currency = request.Currency
            },
            reference = request.OrderId,
            paymentMethod = new
            {
                type = "googlepay",
                googlePayToken = request.WalletToken
            },
            returnUrl = request.ReturnUrl
        };

        var response = await _httpClient.PostAsJsonAsync(
            $"{_settings.BaseUrl}/v71/payments", paymentRequest);
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();

        return new PaymentResult
        {
            TransactionId = result.GetProperty("pspReference").GetString(),
            Status = result.GetProperty("resultCode").GetString() == "Authorised"
                ? PaymentStatus.Completed
                : PaymentStatus.Pending
        };
    }
}
```

### PayPal Wallet

PayPal as a wallet follows a redirect-based flow:

```
Customer clicks "Pay with PayPal" → Redirect to PayPal login
    ↓
Customer approves payment on PayPal
    ↓
Redirect back to merchant with approval token
    ↓
Server captures the order
    ↓
Settlement
```

```csharp
// See 17-PAYPAL-INTEGRATION.md for full PayPal wallet implementation
// The Orders API v2 handles PayPal wallet payments natively
```

---

## Bank Transfers and Direct Debit

Bank-based payment methods move funds directly between bank accounts. They typically have lower fees than cards but longer settlement times and different confirmation flows.

### ACH Direct Debit (US)

#### Stripe — ACH via Payment Intents

```csharp
public class StripeAchService
{
    private readonly IStripeClient _client;

    public async Task<PaymentResult> CreateAchPaymentAsync(BankPaymentRequest request)
    {
        var service = new PaymentIntentService(_client);

        var options = new PaymentIntentCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
            Currency = "usd", // ACH only supports USD
            PaymentMethodTypes = new List<string> { "us_bank_account" },
            PaymentMethod = request.PaymentMethodId,
            Confirm = true,
            MandateData = new PaymentIntentMandateDataOptions
            {
                CustomerAcceptance = new PaymentIntentMandateDataCustomerAcceptanceOptions
                {
                    Type = "online",
                    Online = new PaymentIntentMandateDataCustomerAcceptanceOnlineOptions
                    {
                        IpAddress = request.CustomerIpAddress,
                        UserAgent = request.CustomerUserAgent
                    }
                }
            },
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId
            }
        };

        var intent = await service.CreateAsync(options);

        // ACH payments are not instant — status will be "processing"
        return new PaymentResult
        {
            TransactionId = intent.Id,
            Status = intent.Status switch
            {
                "processing" => PaymentStatus.Processing,
                "succeeded" => PaymentStatus.Completed,
                "requires_action" => PaymentStatus.RequiresAction,
                _ => PaymentStatus.Pending
            }
        };
    }
}
```

### SEPA Direct Debit (EU)

#### Stripe — SEPA via Payment Intents

```csharp
public class StripeSepaService
{
    private readonly IStripeClient _client;

    public async Task<PaymentResult> CreateSepaPaymentAsync(BankPaymentRequest request)
    {
        var service = new PaymentIntentService(_client);

        var options = new PaymentIntentCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
            Currency = "eur", // SEPA only supports EUR
            PaymentMethodTypes = new List<string> { "sepa_debit" },
            PaymentMethod = request.PaymentMethodId,
            Confirm = true,
            MandateData = new PaymentIntentMandateDataOptions
            {
                CustomerAcceptance = new PaymentIntentMandateDataCustomerAcceptanceOptions
                {
                    Type = "online",
                    Online = new PaymentIntentMandateDataCustomerAcceptanceOnlineOptions
                    {
                        IpAddress = request.CustomerIpAddress,
                        UserAgent = request.CustomerUserAgent
                    }
                }
            },
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId
            }
        };

        var intent = await service.CreateAsync(options);

        return new PaymentResult
        {
            TransactionId = intent.Id,
            Status = PaymentStatus.Processing // SEPA takes 3–5 business days
        };
    }
}
```

### iDEAL (Netherlands)

#### Stripe — iDEAL via Payment Intents

```csharp
public async Task<PaymentResult> CreateIdealPaymentAsync(BankPaymentRequest request)
{
    var service = new PaymentIntentService(_client);

    var options = new PaymentIntentCreateOptions
    {
        Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
        Currency = "eur",
        PaymentMethodTypes = new List<string> { "ideal" },
        PaymentMethod = request.PaymentMethodId,
        Confirm = true,
        ReturnUrl = request.ReturnUrl
    };

    var intent = await service.CreateAsync(options);

    return new PaymentResult
    {
        TransactionId = intent.Id,
        Status = PaymentStatus.RequiresAction,
        RequiresAction = true,
        ActionUrl = intent.NextAction?.RedirectToUrl?.Url // Redirect to bank
    };
}
```

### Bank Transfer Flow Comparison

| Method | Region | Currency | Settlement | Reversible | Fees |
|--------|--------|----------|-----------|------------|------|
| ACH | US | USD | 3–5 days | Yes (60 days) | ~$0.80 flat |
| SEPA | EU | EUR | 3–5 days | Yes (8 weeks) | ~€0.35 flat |
| iDEAL | NL | EUR | Instant confirm, 1 day settle | No | ~€0.29 flat |
| Bancontact | BE | EUR | Instant confirm, 1 day settle | No | ~1.4% + €0.25 |
| BACS | UK | GBP | 3 days | Yes (indemnity claim) | ~£0.20–£0.50 |

---

## Buy Now Pay Later (BNPL)

BNPL methods allow customers to split their purchase into installments while merchants receive the full amount upfront. The BNPL provider assumes the credit risk.

### BNPL Flow

```
Customer selects BNPL at checkout → Redirect to BNPL provider
    ↓
Provider performs credit check / shows installment plan
    ↓
Customer approves → Redirect back to merchant
    ↓
Merchant captures payment → Receives full amount
    ↓
Customer pays installments to BNPL provider
```

### Klarna via Stripe

```csharp
public class StripeKlarnaService
{
    private readonly IStripeClient _client;

    public async Task<PaymentResult> CreateKlarnaPaymentAsync(BnplPaymentRequest request)
    {
        var service = new PaymentIntentService(_client);

        var options = new PaymentIntentCreateOptions
        {
            Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
            Currency = request.Currency.ToLower(),
            PaymentMethodTypes = new List<string> { "klarna" },
            PaymentMethodData = new PaymentIntentPaymentMethodDataOptions
            {
                Type = "klarna"
            },
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId
            },
            Confirm = true,
            ReturnUrl = request.ReturnUrl
        };

        var intent = await service.CreateAsync(options);

        return new PaymentResult
        {
            TransactionId = intent.Id,
            Status = PaymentStatus.RequiresAction,
            RequiresAction = true,
            ActionUrl = intent.NextAction?.RedirectToUrl?.Url
        };
    }
}
```

### Afterpay / Clearpay via Stripe

```csharp
public async Task<PaymentResult> CreateAfterpayPaymentAsync(BnplPaymentRequest request)
{
    var service = new PaymentIntentService(_client);

    var options = new PaymentIntentCreateOptions
    {
        Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
        Currency = request.Currency.ToLower(),
        PaymentMethodTypes = new List<string> { "afterpay_clearpay" },
        Shipping = new ChargeShippingOptions
        {
            Name = request.ShippingName,
            Address = new AddressOptions
            {
                Line1 = request.ShippingAddress.Line1,
                City = request.ShippingAddress.City,
                State = request.ShippingAddress.State,
                PostalCode = request.ShippingAddress.PostalCode,
                Country = request.ShippingAddress.Country
            }
        },
        Confirm = true,
        ReturnUrl = request.ReturnUrl
    };

    var intent = await service.CreateAsync(options);

    return new PaymentResult
    {
        TransactionId = intent.Id,
        Status = PaymentStatus.RequiresAction,
        RequiresAction = true,
        ActionUrl = intent.NextAction?.RedirectToUrl?.Url
    };
}
```

### Affirm via Stripe

```csharp
public async Task<PaymentResult> CreateAffirmPaymentAsync(BnplPaymentRequest request)
{
    var service = new PaymentIntentService(_client);

    var options = new PaymentIntentCreateOptions
    {
        Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
        Currency = "usd", // Affirm currently supports USD only
        PaymentMethodTypes = new List<string> { "affirm" },
        Confirm = true,
        ReturnUrl = request.ReturnUrl
    };

    var intent = await service.CreateAsync(options);

    return new PaymentResult
    {
        TransactionId = intent.Id,
        Status = PaymentStatus.RequiresAction,
        RequiresAction = true,
        ActionUrl = intent.NextAction?.RedirectToUrl?.Url
    };
}
```

### BNPL Comparison

| Provider | Regions | Installment Options | Merchant Fee | Min/Max Order |
|----------|---------|-------------------|-------------|---------------|
| Klarna | EU, US, UK, AU | Pay in 4, Pay later, Financing | 3.29% + $0.30 | $35–$10,000 |
| Afterpay/Clearpay | US, UK, AU, NZ | Pay in 4 (bi-weekly) | 4–6% + $0.30 | $1–$2,000 |
| Affirm | US, CA | 3, 6, 12 month plans | 5.99%+ | $50–$30,000 |
| PayPal Pay Later | US, UK, AU | Pay in 4, Monthly | Included in PayPal fees | $30–$1,500 |

---

## Alternative and Regional Payment Methods

Different regions have dominant payment methods that should be supported for maximum conversion.

### Regional Payment Methods Map

| Region | Popular Methods | Integration Via |
|--------|----------------|-----------------|
| 🇧🇷 Brazil | Boleto, Pix | Stripe, Adyen |
| 🇲🇽 Mexico | OXXO | Stripe, Adyen |
| 🇳🇱 Netherlands | iDEAL | Stripe, Adyen, Mollie |
| 🇧🇪 Belgium | Bancontact | Stripe, Adyen |
| 🇩🇪 Germany | Giropay, SOFORT | Stripe, Adyen |
| 🇵🇱 Poland | Przelewy24 (P24) | Stripe, Adyen |
| 🇨🇳 China | WeChat Pay, Alipay | Stripe, Adyen |
| 🇯🇵 Japan | Konbini | Stripe |
| 🇸🇬 🇲🇾 Southeast Asia | GrabPay | Stripe, Adyen |
| 🇮🇳 India | UPI, Netbanking | Razorpay, Cashfree, Stripe |

### Voucher-Based Methods (Boleto, OXXO, Konbini)

Voucher methods generate a payment slip or code that the customer pays at a physical location or via online banking.

```
Customer selects voucher method → Server creates voucher
    ↓
Provider returns voucher URL / barcode / code
    ↓
Customer pays at convenience store / bank / online
    ↓
Provider sends webhook when payment is confirmed (may take 1–3 days)
    ↓
Server processes order
```

#### Stripe — Boleto (Brazil)

```csharp
public async Task<PaymentResult> CreateBoletoPaymentAsync(VoucherPaymentRequest request)
{
    var service = new PaymentIntentService(_client);

    var options = new PaymentIntentCreateOptions
    {
        Amount = ConvertToSmallestUnit(request.Amount, "brl"),
        Currency = "brl",
        PaymentMethodTypes = new List<string> { "boleto" },
        PaymentMethodData = new PaymentIntentPaymentMethodDataOptions
        {
            Type = "boleto",
            BillingDetails = new PaymentIntentPaymentMethodDataBillingDetailsOptions
            {
                Name = request.CustomerName,
                Email = request.CustomerEmail
            }
        },
        Confirm = true
    };

    var intent = await service.CreateAsync(options);

    return new PaymentResult
    {
        TransactionId = intent.Id,
        Status = PaymentStatus.Pending,
        // Customer receives boleto URL to pay
        ActionUrl = intent.NextAction?.BoletoDisplayDetails?.HostedVoucherUrl
    };
}
```

### QR Code Methods (Pix, WeChat Pay, Alipay)

```
Customer selects QR method → Server creates payment
    ↓
Provider returns QR code data
    ↓
Customer scans QR with mobile app → Confirms in app
    ↓
Provider sends webhook (near instant for Pix/WeChat)
    ↓
Server processes order
```

#### Stripe — WeChat Pay

```csharp
public async Task<PaymentResult> CreateWeChatPayAsync(QrPaymentRequest request)
{
    var service = new PaymentIntentService(_client);

    var options = new PaymentIntentCreateOptions
    {
        Amount = ConvertToSmallestUnit(request.Amount, request.Currency),
        Currency = request.Currency.ToLower(),
        PaymentMethodTypes = new List<string> { "wechat_pay" },
        PaymentMethodData = new PaymentIntentPaymentMethodDataOptions
        {
            Type = "wechat_pay"
        },
        PaymentMethodOptions = new PaymentIntentPaymentMethodOptionsOptions
        {
            WechatPay = new PaymentIntentPaymentMethodOptionsWechatPayOptions
            {
                Client = "web"
            }
        },
        Confirm = true
    };

    var intent = await service.CreateAsync(options);

    return new PaymentResult
    {
        TransactionId = intent.Id,
        Status = PaymentStatus.RequiresAction,
        RequiresAction = true,
        // QR code URL for customer to scan
        QrCodeUrl = intent.NextAction?.WechatPayDisplayQrCode?.Data
    };
}
```

---

## Multi-Provider Payment Method Abstraction

When supporting multiple providers and payment methods, use a unified abstraction layer that routes to the correct provider based on the payment method and business rules.

### Unified Payment Method Model

```csharp
public enum PaymentMethodType
{
    Card,
    ApplePay,
    GooglePay,
    PayPalWallet,
    AchDirectDebit,
    SepaDirectDebit,
    Ideal,
    Bancontact,
    Klarna,
    Afterpay,
    Affirm,
    Boleto,
    Oxxo,
    WeChatPay,
    Alipay,
    Pix
}

public record PaymentMethodInfo
{
    public PaymentMethodType Type { get; init; }
    public string Token { get; init; }
    public string Provider { get; init; }
    public Dictionary<string, string> Metadata { get; init; } = new();
}
```

### Payment Method Router

```csharp
public class PaymentMethodRouter
{
    private readonly IReadOnlyDictionary<PaymentMethodType, string> _methodToProvider;
    private readonly IServiceProvider _serviceProvider;

    public PaymentMethodRouter(
        IOptions<PaymentRoutingConfig> config,
        IServiceProvider serviceProvider)
    {
        _methodToProvider = config.Value.MethodProviderMap;
        _serviceProvider = serviceProvider;
    }

    public IPaymentGateway GetGateway(PaymentMethodType method)
    {
        var providerName = _methodToProvider.TryGetValue(method, out var provider)
            ? provider
            : throw new NotSupportedException($"Payment method {method} is not configured");

        return _serviceProvider
            .GetKeyedService<IPaymentGateway>(providerName)
            ?? throw new InvalidOperationException($"Provider {providerName} not registered");
    }
}

// Configuration
public class PaymentRoutingConfig
{
    public Dictionary<PaymentMethodType, string> MethodProviderMap { get; set; } = new()
    {
        [PaymentMethodType.Card] = "Stripe",
        [PaymentMethodType.ApplePay] = "Stripe",
        [PaymentMethodType.GooglePay] = "Adyen",
        [PaymentMethodType.PayPalWallet] = "PayPal",
        [PaymentMethodType.AchDirectDebit] = "Stripe",
        [PaymentMethodType.SepaDirectDebit] = "Stripe",
        [PaymentMethodType.Ideal] = "Stripe",
        [PaymentMethodType.Klarna] = "Stripe",
        [PaymentMethodType.Afterpay] = "Stripe",
        [PaymentMethodType.Boleto] = "Stripe",
        [PaymentMethodType.WeChatPay] = "Adyen"
    };
}
```

### Unified Payment Service

```csharp
public class UnifiedPaymentService
{
    private readonly PaymentMethodRouter _router;
    private readonly IPaymentRepository _repository;
    private readonly ILogger<UnifiedPaymentService> _logger;

    public async Task<PaymentResult> ProcessPaymentAsync(UnifiedPaymentRequest request)
    {
        _logger.LogInformation(
            "Processing {Method} payment for order {OrderId} via {Provider}",
            request.Method.Type, request.OrderId, request.Method.Provider);

        var gateway = _router.GetGateway(request.Method.Type);

        var payment = Payment.Create(
            request.OrderId, request.Amount, request.Currency);
        await _repository.AddAsync(payment);

        try
        {
            var result = await gateway.ProcessPaymentAsync(new PaymentRequest
            {
                OrderId = request.OrderId,
                Amount = request.Amount,
                Currency = request.Currency,
                PaymentMethodToken = request.Method.Token,
                IdempotencyKey = request.IdempotencyKey,
                ReturnUrl = request.ReturnUrl
            });

            payment.MarkAsProcessing(result.TransactionId);

            if (result.Status == PaymentStatus.Completed)
            {
                payment.MarkAsCompleted();
            }

            await _repository.UpdateAsync(payment);
            return result;
        }
        catch (Exception ex)
        {
            payment.MarkAsFailed(ex.Message);
            await _repository.UpdateAsync(payment);
            throw;
        }
    }
}
```

---

## Payment Flow Diagrams

### Standard Card Flow (Stripe)

```
┌─────────┐    ┌──────────┐    ┌────────┐    ┌────────┐
│ Customer │    │  Client  │    │ Server │    │ Stripe │
└────┬─────┘    └────┬─────┘    └───┬────┘    └───┬────┘
     │               │              │              │
     │ Enter card    │              │              │
     │──────────────>│              │              │
     │               │ Create PaymentIntent        │
     │               │─────────────>│              │
     │               │              │ POST /v1/payment_intents
     │               │              │─────────────>│
     │               │              │  client_secret
     │               │  clientSecret│<─────────────│
     │               │<─────────────│              │
     │               │ confirmCardPayment          │
     │               │────────────────────────────>│
     │               │              │    Webhook: payment_intent.succeeded
     │               │              │<─────────────│
     │  Confirmation │              │              │
     │<──────────────│              │              │
```

### Redirect-Based Flow (PayPal, BNPL, iDEAL)

```
┌─────────┐    ┌──────────┐    ┌────────┐    ┌──────────┐
│ Customer │    │  Client  │    │ Server │    │ Provider │
└────┬─────┘    └────┬─────┘    └───┬────┘    └────┬─────┘
     │               │              │               │
     │ Click "Pay"   │              │               │
     │──────────────>│              │               │
     │               │ Create order │               │
     │               │─────────────>│               │
     │               │              │  POST /orders │
     │               │              │──────────────>│
     │               │              │  approval_url │
     │               │  redirect URL│<──────────────│
     │               │<─────────────│               │
     │   Redirect to provider       │               │
     │──────────────────────────────────────────────>│
     │   Approve payment             │               │
     │──────────────────────────────────────────────>│
     │   Redirect back with token    │               │
     │<──────────────────────────────────────────────│
     │               │  Capture     │               │
     │               │─────────────>│               │
     │               │              │ POST /capture │
     │               │              │──────────────>│
     │               │              │   Webhook     │
     │               │              │<──────────────│
     │  Confirmation │              │               │
     │<──────────────│              │               │
```

### Asynchronous / Delayed Flow (ACH, SEPA, Boleto)

```
┌─────────┐    ┌────────┐    ┌──────────┐
│ Customer │    │ Server │    │ Provider │
└────┬─────┘    └───┬────┘    └────┬─────┘
     │              │              │
     │ Submit bank  │              │
     │─────────────>│              │
     │              │ Create debit │
     │              │─────────────>│
     │              │  "processing"│
     │  Pending msg │<─────────────│
     │<─────────────│              │
     │              │              │
     │    (3–5 business days)      │
     │              │              │
     │              │  Webhook:    │
     │              │  succeeded   │
     │              │<─────────────│
     │  Email conf. │              │
     │<─────────────│              │
```

---

## Provider Comparison Matrix

### Payment Method Support by Provider

| Payment Method | Stripe | PayPal | Square | Adyen | Braintree |
|---------------|--------|--------|--------|-------|-----------|
| Visa/Mastercard | ✅ | ✅ | ✅ | ✅ | ✅ |
| Amex | ✅ | ✅ | ✅ | ✅ | ✅ |
| Apple Pay | ✅ | ✅ | ✅ | ✅ | ✅ |
| Google Pay | ✅ | ✅ | ✅ | ✅ | ✅ |
| PayPal Wallet | ✅ (Link) | ✅ | ❌ | ✅ | ✅ |
| ACH | ✅ | ❌ | ✅ | ❌ | ❌ |
| SEPA | ✅ | ❌ | ❌ | ✅ | ❌ |
| iDEAL | ✅ | ✅ | ❌ | ✅ | ❌ |
| Klarna | ✅ | ❌ | ❌ | ✅ | ❌ |
| Afterpay | ✅ | ❌ | ✅ | ✅ | ❌ |
| Affirm | ✅ | ❌ | ✅ | ❌ | ❌ |
| WeChat Pay | ✅ | ❌ | ❌ | ✅ | ❌ |
| Alipay | ✅ | ❌ | ❌ | ✅ | ❌ |
| Boleto | ✅ | ❌ | ❌ | ✅ | ❌ |
| Pix | ✅ | ❌ | ❌ | ✅ | ❌ |

### Provider Fee Comparison (Approximate)

| Provider | Card Fee | Processing | Wallet Fee | BNPL Fee |
|----------|---------|-----------|-----------|----------|
| Stripe | 2.9% + $0.30 | Included | Same as card | 3–6% + $0.30 |
| PayPal | 2.99% + $0.49 | Included | Same as card | Included |
| Square | 2.9% + $0.30 | Included | Same as card | 6% + $0.30 |
| Adyen | IC++ pricing | Included | IC++ | Varies |
| Braintree | 2.59% + $0.49 | Included | Same as card | N/A |

### Integration Complexity

| Provider | Setup Difficulty | SDK Quality | Documentation | Sandbox |
|----------|-----------------|------------|---------------|---------|
| Stripe | ⭐ Easy | Excellent | Excellent | Full sandbox |
| PayPal | ⭐⭐ Medium | Good | Good | Full sandbox |
| Square | ⭐⭐ Medium | Good | Good | Full sandbox |
| Adyen | ⭐⭐⭐ Complex | Good | Good | Test environment |
| Braintree | ⭐⭐ Medium | Good | Good | Full sandbox |

---

## Implementation Checklist

### Before Adding a New Payment Method

- [ ] Verify provider supports the method in your target regions
- [ ] Check fee structure and compare with alternatives
- [ ] Confirm currency and country restrictions
- [ ] Review settlement timelines and fund availability
- [ ] Understand refund/chargeback policies for the method
- [ ] Verify PCI compliance requirements (some methods reduce PCI scope)

### Integration Steps

- [ ] Add the payment method type to your `PaymentMethodType` enum
- [ ] Configure provider routing in `PaymentRoutingConfig`
- [ ] Implement the provider-specific payment flow
- [ ] Handle redirect/async flows if required
- [ ] Add webhook handling for the new method's events
- [ ] Implement refund support for the method
- [ ] Add test scenarios using provider sandbox
- [ ] Update frontend to display the new payment option
- [ ] Configure method availability by region/currency
- [ ] Test full end-to-end flow in sandbox before go-live

### Monitoring

- [ ] Track conversion rate per payment method
- [ ] Monitor failure rates by method and provider
- [ ] Set up alerts for settlement delays
- [ ] Track refund/dispute rates per method
- [ ] Monitor regional availability and provider uptime

## Next Steps

- [Stripe Integration](16-STRIPE-INTEGRATION.md) — deep dive into Stripe APIs
- [PayPal Integration](17-PAYPAL-INTEGRATION.md) — deep dive into PayPal APIs
- [Common Scenarios](18-COMMON-SCENARIOS.md) — end-to-end payment flow implementations
- [Strategy Pattern](03-STRATEGY-PATTERN.md) — provider routing and fallback patterns
- [Security Patterns](15-SECURITY-PATTERNS-PRACTICES.md) — PCI compliance and fraud prevention
