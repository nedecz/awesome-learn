# Test Cards, Sandbox Environments & Registration Guide

## Overview

Comprehensive reference for test card numbers, sandbox configuration, account registration, and testing strategies for all major payment providers. Essential for development and testing without processing real payments.

> **Note**: This document consolidates test cards, sandbox features, and provider registration into a single reference.

## Table of Contents

1. [Provider Registration Quick Start](#provider-registration-quick-start)
2. [Stripe Test Cards](#stripe-test-cards)
3. [PayPal Sandbox](#paypal-sandbox)
4. [Square Sandbox](#square-sandbox)
5. [Adyen Test Cards](#adyen-test-cards)
6. [Braintree Sandbox](#braintree-sandbox)
7. [Authorize.Net Test Mode](#authorizenet-test-mode)
8. [Regional Providers](#regional-providers)
9. [Testing Scenarios](#testing-scenarios)
10. [Webhook Testing](#webhook-testing)
11. [Configuration Templates](#configuration-templates)
12. [Troubleshooting](#troubleshooting)
13. [Best Practices](#best-practices)

---

## Provider Registration Quick Start

| Provider | Signup URL | Time to Start | Key Format |
|----------|-----------|---------------|-----------|
| **Stripe** | [dashboard.stripe.com/register](https://dashboard.stripe.com/register) | 5 min | `sk_test_...` / `pk_test_...` |
| **PayPal** | [developer.paypal.com](https://developer.paypal.com/home) | 10–15 min | Client ID + Secret |
| **Square** | [developer.squareup.com](https://developer.squareup.com/) | 5 min | `sandbox-sq0...` |
| **Adyen** | [adyen.com](https://www.adyen.com/signup) | 1–3 days* | API Key + Merchant Account |
| **Braintree** | [braintreepayments.com/sandbox](https://www.braintreepayments.com/sandbox) | 10 min | Merchant ID + Public/Private Key |
| **Authorize.Net** | [developer.authorize.net](https://developer.authorize.net/) | 5 min | API Login ID + Transaction Key |
| **Razorpay** | [dashboard.razorpay.com](https://dashboard.razorpay.com/) | 5 min | `rzp_test_...` |
| **Mercado Pago** | [mercadopago.com/developers](https://www.mercadopago.com/developers/) | 10–15 min | `TEST-...` |

*Adyen requires sales contact and approval

---

## Stripe Test Cards

### Basic Test Cards

```csharp
public static class StripeTestCards
{
    // SUCCESS SCENARIOS
    public const string Visa = "4242424242424242";
    public const string VisaDebit = "4000056655665556";
    public const string Mastercard = "5555555555554444";
    public const string MastercardDebit = "5200828282828210";
    public const string MastercardPrepaid = "5105105105105100";
    public const string Amex = "378282246310005";
    public const string Discover = "6011111111111117";
    public const string DinersClub = "3056930009020004";
    public const string DinersClub14Digit = "36227206271667";
    public const string JCB = "3566002020360505";
    public const string UnionPay = "6200000000000005";
    
    // FAILURE SCENARIOS
    public const string DeclinedCard = "4000000000000002";
    public const string InsufficientFunds = "4000000000009995";
    public const string LostCard = "4000000000009987";
    public const string StolenCard = "4000000000009979";
    public const string ExpiredCard = "4000000000000069";
    public const string IncorrectCVC = "4000000000000127";
    public const string ProcessingError = "4000000000000119";
    public const string DeclinedFraudulent = "4100000000000019";
    
    // 3D SECURE AUTHENTICATION
    public const string Require3DS = "4000002500003155";
    public const string Require3DSSuccess = "4000002760003184"; // Auth succeeds
    public const string Require3DSFailure = "4000008400001629"; // Auth fails
    public const string Require3DSNotSupported = "4000002760003184";
    
    // SPECIAL SCENARIOS
    public const string AttachableToCustomer = "4000000000000341";
    public const string ChargeableInCurrency = "4000000000000010"; // Any currency
    public const string SuccessWithReview = "4000000000009235"; // Creates review
    public const string AlwaysBlock = "4000000000000036";
    public const string BlockIfCVCFail = "4000000000000101";
    
    // INTERNATIONAL CARDS
    public const string BrazilCard = "4000000760000002";
    public const string MexicoCard = "4000004840008001";
    public const string IndiaCard = "4000003560000008";
    
    // METADATA
    public static Dictionary<string, CardInfo> GetCardInfo()
    {
        return new Dictionary<string, CardInfo>
        {
            [Visa] = new CardInfo
            {
                Number = Visa,
                Brand = "Visa",
                ExpMonth = 12,
                ExpYear = DateTime.Now.Year + 2,
                CVC = "123",
                Country = "US",
                Funding = "credit",
                Description = "Always succeeds"
            },
            [DeclinedCard] = new CardInfo
            {
                Number = DeclinedCard,
                Brand = "Visa",
                ExpMonth = 12,
                ExpYear = DateTime.Now.Year + 2,
                CVC = "123",
                Country = "US",
                Funding = "credit",
                Description = "Always declines with generic decline code"
            },
            [InsufficientFunds] = new CardInfo
            {
                Number = InsufficientFunds,
                Brand = "Visa",
                ExpMonth = 12,
                ExpYear = DateTime.Now.Year + 2,
                CVC = "123",
                Country = "US",
                Funding = "credit",
                Description = "Declined - insufficient funds"
            }
        };
    }
}

public class CardInfo
{
    public string Number { get; set; }
    public string Brand { get; set; }
    public int ExpMonth { get; set; }
    public int ExpYear { get; set; }
    public string CVC { get; set; }
    public string Country { get; set; }
    public string Funding { get; set; }
    public string Description { get; set; }
}
```

### Stripe Testing Usage

```csharp
using Stripe;

public class StripeTestingService
{
    private readonly StripeClient _client;
    
    public StripeTestingService()
    {
        // Use test API key
        StripeConfiguration.ApiKey = "sk_test_...";
        _client = new StripeClient("sk_test_...");
    }
    
    public async Task<PaymentIntent> CreateTestPaymentAsync()
    {
        var options = new PaymentIntentCreateOptions
        {
            Amount = 2000, // $20.00
            Currency = "usd",
            PaymentMethodTypes = new List<string> { "card" },
            PaymentMethod = await CreateTestPaymentMethodAsync(StripeTestCards.Visa)
        };
        
        var service = new PaymentIntentService(_client);
        return await service.CreateAsync(options);
    }
    
    private async Task<string> CreateTestPaymentMethodAsync(string cardNumber)
    {
        var options = new PaymentMethodCreateOptions
        {
            Type = "card",
            Card = new PaymentMethodCardOptions
            {
                Number = cardNumber,
                ExpMonth = 12,
                ExpYear = DateTime.Now.Year + 2,
                Cvc = "123"
            }
        };
        
        var service = new PaymentMethodService(_client);
        var paymentMethod = await service.CreateAsync(options);
        return paymentMethod.Id;
    }
    
    // Test webhook events
    public async Task TriggerTestWebhookAsync(string eventType)
    {
        var options = new EventCreateOptions
        {
            Type = eventType
        };
        
        var service = new EventService(_client);
        await service.CreateAsync(options);
    }
}
```

### Stripe Test Mode Features

```yaml
Test Mode Features:
  - All API keys prefixed with "sk_test_" or "pk_test_"
  - No real money processed
  - Full API functionality
  - Test webhooks
  - Test customer creation
  - Test subscription management
  - Dashboard test mode toggle

CLI Testing:
  # Install Stripe CLI
  stripe login
  
  # Listen to webhooks locally
  stripe listen --forward-to localhost:5000/api/webhooks/stripe
  
  # Trigger test events
  stripe trigger payment_intent.succeeded
  stripe trigger payment_intent.payment_failed
  stripe trigger charge.refunded
```

---

## PayPal Sandbox

### PayPal Sandbox Accounts

```csharp
public static class PayPalSandboxAccounts
{
    // Create sandbox accounts at: https://developer.paypal.com/dashboard/accounts
    
    public static class Personal
    {
        public const string Email = "sb-buyer@personal.example.com";
        public const string Password = "12345678";
        public const string FirstName = "John";
        public const string LastName = "Doe";
    }
    
    public static class Business
    {
        public const string Email = "sb-merchant@business.example.com";
        public const string Password = "12345678";
        public const string CompanyName = "Test Merchant";
    }
    
    // Pre-configured test cards in sandbox account
    public static class TestCards
    {
        public const string Visa = "4032039658382527";
        public const string Mastercard = "5425233430109903";
        public const string Amex = "374245455400126";
        public const string Discover = "6011000991001201";
        
        // These cards will have different behaviors
        public const string DeclinedVisa = "4563456345634563";
        public const string InsufficientFunds = "4000000000000010";
    }
}

public class PayPalSandboxService
{
    private readonly APIContext _apiContext;
    
    public PayPalSandboxService()
    {
        var config = new Dictionary<string, string>
        {
            ["mode"] = "sandbox", // Important!
            ["clientId"] = "YOUR_SANDBOX_CLIENT_ID",
            ["clientSecret"] = "YOUR_SANDBOX_CLIENT_SECRET"
        };
        
        var accessToken = new OAuthTokenCredential(config).GetAccessToken();
        _apiContext = new APIContext(accessToken) { Config = config };
    }
    
    public async Task<Payment> CreateSandboxPaymentAsync()
    {
        var payment = new Payment
        {
            Intent = "sale",
            Payer = new Payer { PaymentMethod = "paypal" },
            Transactions = new List<Transaction>
            {
                new Transaction
                {
                    Amount = new Amount
                    {
                        Total = "100.00",
                        Currency = "USD"
                    }
                }
            },
            RedirectUrls = new RedirectUrls
            {
                ReturnUrl = "https://localhost:5001/payment/success",
                CancelUrl = "https://localhost:5001/payment/cancel"
            }
        };
        
        return await Task.Run(() => payment.Create(_apiContext));
    }
}
```

### PayPal Sandbox Configuration

```json
{
  "PayPal": {
    "Mode": "sandbox",
    "ClientId": "YOUR_SANDBOX_CLIENT_ID",
    "ClientSecret": "YOUR_SANDBOX_CLIENT_SECRET",
    "SandboxUrl": "https://api.sandbox.paypal.com",
    "SandboxWebUrl": "https://www.sandbox.paypal.com"
  }
}
```

### PayPal Sandbox Features

```yaml
Sandbox Features:
  - Separate sandbox accounts (buyer and seller)
  - Virtual money transactions
  - Full checkout flow testing
  - Webhook testing
  - Refund simulation
  - Dispute simulation
  
How to Use:
  1. Create sandbox account at developer.paypal.com
  2. Create test buyer accounts
  3. Create test business accounts
  4. Fund accounts with test money
  5. Use sandbox API credentials
  
Testing Checklist:
  - Create payment
  - Complete checkout with test buyer account
  - Capture payment
  - Issue refund
  - Test webhooks
  - Test disputes
```

---

## Square Sandbox

### Square Test Cards

```csharp
public static class SquareTestCards
{
    // SUCCESS CARDS
    public static class Success
    {
        public const string Visa = "4111 1111 1111 1111";
        public const string Mastercard = "5105 1051 0510 5100";
        public const string Amex = "3782 822463 10005";
        public const string Discover = "6011 0000 0000 0004";
        public const string DiscoverDiners = "3056 9309 0259 04";
        public const string JCB = "3569 9900 1009 5841";
        
        // All above cards:
        // - CVV: Any 3 digits (4 for Amex)
        // - Postal Code: Any valid postal code
        // - Expiration: Any future date
    }
    
    // DECLINED CARDS
    public static class Declined
    {
        public const string Generic = "4000 0000 0000 0002";
        public const string InsufficientFunds = "4000 0000 0000 0010";
        public const string CardDeclined = "4000 0000 0000 0028";
        public const string ExpiredCard = "4000 0000 0000 0069";
        public const string InvalidCVV = "4000 0000 0000 0127";
        public const string InvalidPIN = "4000 0000 0000 0135";
        public const string InvalidPostalCode = "4000 0000 0000 0143";
    }
    
    // SCA (3D Secure) TESTING
    public static class SCA
    {
        public const string AuthenticationRequired = "4000 0025 0000 0003";
        public const string AuthenticationUnavailable = "4000 0027 6000 0016";
    }
    
    // SPECIAL SCENARIOS
    public static class Special
    {
        public const string AlwaysInReview = "4000 0000 0000 0259";
        public const string ProcessingError = "4000 0000 0000 0119";
    }
}

public class SquareSandboxService
{
    private readonly ISquareClient _client;
    
    public SquareSandboxService()
    {
        _client = new SquareClient.Builder()
            .Environment(Square.Environment.Sandbox) // Sandbox mode
            .AccessToken("YOUR_SANDBOX_ACCESS_TOKEN")
            .Build();
    }
    
    public async Task<CreatePaymentResponse> CreateSandboxPaymentAsync()
    {
        // First, generate a card nonce from Square.js in sandbox mode
        // Or use a test nonce: "cnon:card-nonce-ok" (always succeeds)
        
        var paymentsApi = _client.PaymentsApi;
        
        var body = new CreatePaymentRequest.Builder(
            sourceId: "cnon:card-nonce-ok", // Test nonce
            idempotencyKey: Guid.NewGuid().ToString())
            .AmountMoney(new Money.Builder()
                .Amount(1000L) // $10.00
                .Currency("USD")
                .Build())
            .Build();
        
        return await paymentsApi.CreatePaymentAsync(body);
    }
}
```

### Square Sandbox Configuration

```csharp
public static class SquareSandboxConfig
{
    // Sandbox credentials from developer.squareup.com
    public const string SandboxAccessToken = "YOUR_SANDBOX_ACCESS_TOKEN";
    public const string SandboxApplicationId = "YOUR_SANDBOX_APP_ID";
    public const string SandboxLocationId = "YOUR_SANDBOX_LOCATION_ID";
    
    // Test card nonces (use instead of real card nonces in sandbox)
    public static class TestNonces
    {
        public const string AlwaysSucceed = "cnon:card-nonce-ok";
        public const string AlwaysDecline = "cnon:card-nonce-declined";
        public const string AlwaysDeclineCVV = "cnon:card-nonce-declined-cvv";
        public const string AlwaysDeclinePostalCode = "cnon:card-nonce-declined-postalcode";
        public const string AlwaysDeclineExpiration = "cnon:card-nonce-declined-expiration";
    }
}
```

### Square Sandbox Features

```yaml
Sandbox Environment:
  URL: https://connect.squareupsandbox.com
  Dashboard: https://developer.squareup.com/apps
  
Features:
  - Test card processing
  - Customer management
  - Order creation
  - Refunds
  - Webhooks
  - Terminal integration (simulated)
  
Setup:
  1. Create Square developer account
  2. Create application
  3. Get sandbox credentials
  4. Use sandbox environment in SDK
  5. Test with provided test cards
  
Important Notes:
  - Sandbox and production are separate
  - Can't use production cards in sandbox
  - Virtual test money only
  - All features available except real terminal hardware
```

---

## Adyen Test Cards

### Adyen Test Card Numbers

```csharp
public static class AdyenTestCards
{
    // 3D SECURE 2 AUTHENTICATION
    public static class ThreeDSecure2
    {
        // Successful authentication
        public const string Success = "4917 6100 0000 0000";
        
        // Failed authentication
        public const string Failure = "4917 6100 0000 0001";
        
        // Card not enrolled for 3DS
        public const string NotEnrolled = "4917 6100 0000 0002";
        
        // Authentication unavailable
        public const string Unavailable = "4917 6100 0000 0003";
        
        // Frictionless flow
        public const string Frictionless = "4917 6100 0000 0004";
    }
    
    // AUTHORIZATION RESULTS
    public static class Results
    {
        public const string Authorised = "4111 1111 4555 1142";
        public const string Refused = "4111 1111 1111 1111";
        public const string Error = "4111 1111 1111 1112";
        public const string Pending = "4111 1111 1111 1113";
        public const string Cancelled = "4111 1111 1111 1114";
    }
    
    // CVC CHECKS
    public static class CVC
    {
        public const string Match = "4166 6766 6766 6746"; // Use CVC: 737
        public const string NoMatch = "4166 6766 6766 6746"; // Use CVC: 123
        public const string NotProvided = "4166 6766 6766 6746"; // Don't send CVC
    }
    
    // AVS CHECKS
    public static class AVS
    {
        public const string Match = "4444 3333 2222 1111"; // Use postal: 12345
        public const string NoMatch = "4444 3333 2222 1111"; // Use postal: 99999
    }
    
    // REGIONAL CARDS
    public static class Regional
    {
        public const string Maestro = "6759 6498 2643 8453";
        public const string Visa = "4111 1111 1111 1111";
        public const string Mastercard = "5555 4444 3333 1111";
        public const string Amex = "3700 0000 0000 002";
        public const string Diners = "3600 6666 3333 44";
        public const string Discover = "6011 6011 6011 6611";
        public const string JCB = "3569 9900 1009 5841";
        public const string UnionPay = "6250 9470 0000 0014";
    }
}

public class AdyenTestingService
{
    private readonly Client _client;
    private readonly Checkout _checkout;
    
    public AdyenTestingService()
    {
        var config = new Config
        {
            XApiKey = "YOUR_TEST_API_KEY",
            Environment = Adyen.Model.Enum.Environment.Test // Test environment
        };
        
        _client = new Client(config);
        _checkout = new Checkout(_client);
    }
    
    public async Task<PaymentResponse> CreateTestPaymentAsync(string testCard)
    {
        var paymentRequest = new PaymentRequest
        {
            Amount = new Amount { Currency = "EUR", Value = 1000 }, // €10.00
            Reference = $"TEST-{Guid.NewGuid()}",
            MerchantAccount = "YOUR_TEST_MERCHANT_ACCOUNT",
            PaymentMethod = new CheckoutPaymentMethod
            {
                Type = "scheme",
                Number = testCard.Replace(" ", ""),
                ExpiryMonth = "03",
                ExpiryYear = "2030",
                Cvc = "737"
            }
        };
        
        return await _checkout.PaymentsAsync(paymentRequest);
    }
}
```

### Adyen Test Environment

```yaml
Test Environment:
  URL: https://checkout-test.adyen.com
  API: https://pal-test.adyen.com
  
Credentials:
  Company Account: Create at https://ca-test.adyen.com
  Merchant Account: Auto-created test account
  API Key: Generate in Customer Area
  
Card Details for Testing:
  - Card Number: Use test cards above
  - Expiry: Any future date (MM/YYYY)
  - CVC: 737 (for most scenarios)
  - Cardholder Name: Any name
  
Testing Features:
  - 3D Secure 2 flows
  - Payment methods by region
  - Webhooks
  - Refunds
  - Captures
  - Cancellations
  - Risk management
```

---

## Braintree Sandbox

### Braintree Test Cards

```csharp
public static class BraintreeTestCards
{
    // SUCCESS CARDS
    public static class Success
    {
        public const string Visa = "4111111111111111";
        public const string Mastercard = "5555555555554444";
        public const string Amex = "378282246310005";
        public const string Discover = "6011111111111117";
        public const string JCB = "3530111333300000";
        public const string Maestro = "6304000000000000";
    }
    
    // PROCESSOR DECLINED
    public static class Declined
    {
        public const string ProcessorDeclined = "4000111111111115";
        public const string InsufficientFunds = "4000111111111125";
        public const string TransactionNotAllowed = "4000111111111127";
        public const string InvalidCVV = "4000111111111129";
    }
    
    // GATEWAY REJECTED
    public static class Rejected
    {
        public const string CVV = "4000111111111103"; // CVV doesn't match
        public const string AVS = "4000111111111101"; // AVS doesn't match
        public const string AVSAndCVV = "4000111111111102"; // Both don't match
        public const string Fraud = "4000111111111140"; // Fraud detected
    }
    
    // CARD VERIFICATION
    public static class Verification
    {
        public const string ProcessorDeclined = "4000111111111010";
        public const string Failed = "4000111111111019";
    }
    
    // All test cards use:
    // - Expiry: Any future date
    // - CVV: 123 (or any 3 digits)
    // - Postal Code: Any valid code
}

public class BraintreeSandboxService
{
    private readonly BraintreeGateway _gateway;
    
    public BraintreeSandboxService()
    {
        _gateway = new BraintreeGateway
        {
            Environment = Braintree.Environment.SANDBOX, // Sandbox!
            MerchantId = "YOUR_SANDBOX_MERCHANT_ID",
            PublicKey = "YOUR_SANDBOX_PUBLIC_KEY",
            PrivateKey = "YOUR_SANDBOX_PRIVATE_KEY"
        };
    }
    
    public async Task<Result<Transaction>> CreateTestTransactionAsync(string testCard)
    {
        var request = new TransactionRequest
        {
            Amount = 10.00M,
            CreditCard = new TransactionCreditCardRequest
            {
                Number = testCard,
                ExpirationDate = "12/2025",
                CVV = "123"
            },
            Options = new TransactionOptionsRequest
            {
                SubmitForSettlement = true
            }
        };
        
        return await Task.Run(() => _gateway.Transaction.Sale(request));
    }
    
    // Generate client token for Drop-in UI
    public string GenerateClientToken()
    {
        return _gateway.ClientToken.Generate();
    }
}
```

### Braintree Test Nonces

```csharp
public static class BraintreeTestNonces
{
    // Instead of real payment method nonces, use these in sandbox
    public const string Visa = "fake-valid-visa-nonce";
    public const string Mastercard = "fake-valid-mastercard-nonce";
    public const string Amex = "fake-valid-amex-nonce";
    public const string Discover = "fake-valid-discover-nonce";
    
    // PayPal
    public const string PayPalOneTime = "fake-paypal-one-time-nonce";
    public const string PayPalBilling = "fake-paypal-billing-agreement-nonce";
    
    // Special scenarios
    public const string ProcessorDeclined = "fake-processor-declined-visa-nonce";
    public const string GatewayRejected = "fake-gateway-rejected-nonce";
    public const string Consumed = "fake-consumed-nonce"; // Already used
}
```

---

## Authorize.Net Test Mode

### Authorize.Net Test Cards

```csharp
public static class AuthorizeNetTestCards
{
    // SUCCESS CARDS
    public static class Success
    {
        public const string Visa = "4007000000027";
        public const string Visa2 = "4012888818888";
        public const string Mastercard = "5424000000000015";
        public const string Amex = "370000000000002";
        public const string Discover = "6011000000000012";
        public const string DinersClub = "38000000000006";
        public const string JCB = "3088000000000017";
    }
    
    // DECLINED CARDS
    public static class Declined
    {
        public const string Generic = "4222222222222";
    }
    
    // All test cards use:
    // - Expiry: Any future date
    // - CVV: Any 3-4 digits
    // - Any billing information
}

public class AuthorizeNetTestService
{
    private readonly string _apiLoginId;
    private readonly string _transactionKey;
    
    public AuthorizeNetTestService()
    {
        _apiLoginId = "YOUR_TEST_API_LOGIN_ID";
        _transactionKey = "YOUR_TEST_TRANSACTION_KEY";
        
        // Set to sandbox environment
        ApiOperationBase<ANetApiRequest, ANetApiResponse>.RunEnvironment = 
            AuthorizeNet.Environment.SANDBOX;
    }
    
    public async Task<createTransactionResponse> CreateTestTransactionAsync(string testCard)
    {
        var merchantAuth = new merchantAuthenticationType
        {
            name = _apiLoginId,
            ItemElementName = ItemChoiceType.transactionKey,
            Item = _transactionKey
        };
        
        var creditCard = new creditCardType
        {
            cardNumber = testCard,
            expirationDate = "1230", // MMYY
            cardCode = "123"
        };
        
        var payment = new paymentType { Item = creditCard };
        
        var transactionRequest = new transactionRequestType
        {
            transactionType = transactionTypeEnum.authCaptureTransaction.ToString(),
            amount = 10.00M,
            payment = payment
        };
        
        var request = new createTransactionRequest
        {
            merchantAuthentication = merchantAuth,
            transactionRequest = transactionRequest
        };
        
        var controller = new createTransactionController(request);
        controller.Execute();
        
        return controller.GetApiResponse();
    }
}
```

### Authorize.Net Test Environment

```yaml
Sandbox Environment:
  URL: https://test.authorize.net
  API: https://apitest.authorize.net/xml/v1/request.api
  
Test Credentials:
  API Login ID: Get from sandbox account
  Transaction Key: Generate in sandbox
  
Test Features:
  - Card processing
  - eCheck processing
  - Customer profiles
  - Recurring billing
  - Transaction reporting
  
Test Amounts:
  - $0.01 to $99.99: Approved
  - $1.00 (exact): Declined
  - $2.00 (exact): Expired card error
```

---

## Regional Providers

### Razorpay Test Cards (India)

```csharp
public static class RazorpayTestCards
{
    // SUCCESS CARDS
    public const string Domestic = "4111111111111111";
    public const string International = "5104015555555558";
    
    // FAILURE SCENARIOS
    public const string InsufficientFunds = "4000000000000002";
    public const string DeclinedByNetwork = "4000000000000010";
    public const string InvalidCVV = "4000000000000036";
    
    // 3D SECURE
    public const string Success3DS = "5267318506248573"; // OTP: 1234
    public const string Failed3DS = "4000000000000002";
    
    // Test Details:
    // - Name: Any
    // - CVV: 123
    // - Expiry: Any future date
    // - For 3DS: Use OTP 1234
}

public class RazorpayTestService
{
    private readonly RazorpayClient _client;
    
    public RazorpayTestService()
    {
        // Use test key (starts with "rzp_test_")
        _client = new RazorpayClient("rzp_test_...", "YOUR_TEST_SECRET");
    }
    
    public async Task<Dictionary<string, object>> CreateTestOrderAsync()
    {
        var options = new Dictionary<string, object>
        {
            ["amount"] = 100000, // Amount in paise (1000.00 INR)
            ["currency"] = "INR",
            ["receipt"] = $"TEST_{Guid.NewGuid()}"
        };
        
        return await Task.Run(() => _client.Order.Create(options));
    }
}
```

### Mercado Pago Test Cards (Latin America)

```csharp
public static class MercadoPagoTestCards
{
    // Test cards vary by country
    public static class Brazil
    {
        public const string Approved = "5031 4332 1540 6351";
        public const string Rejected = "5031 7557 3453 0604";
        public const string InsufficientFunds = "5031 5571 3604 0756";
        
        // CPF for testing: 12345678909
    }
    
    public static class Argentina
    {
        public const string Approved = "5031 7557 3453 0604";
        public const string Rejected = "5031 4332 1540 6351";
        
        // DNI for testing: 12345678
    }
    
    public static class Mexico
    {
        public const string Approved = "5474 9254 3267 0366";
        public const string Rejected = "5474 0253 6932 3763";
        
        // RFC for testing: XAXX010101000
    }
    
    // All cards use:
    // - Name: APRO (approved) or OTHE (rejected)
    // - CVV: 123
    // - Expiry: Any future date
}
```

---

## Testing Scenarios

### Comprehensive Test Suite

```csharp
public class PaymentTestSuite
{
    private readonly IPaymentGateway _gateway;
    
    [Theory]
    [InlineData(StripeTestCards.Visa, true, "Successful payment")]
    [InlineData(StripeTestCards.DeclinedCard, false, "Card declined")]
    [InlineData(StripeTestCards.InsufficientFunds, false, "Insufficient funds")]
    [InlineData(StripeTestCards.LostCard, false, "Lost card")]
    [InlineData(StripeTestCards.StolenCard, false, "Stolen card")]
    [InlineData(StripeTestCards.ExpiredCard, false, "Expired card")]
    [InlineData(StripeTestCards.IncorrectCVC, false, "Incorrect CVC")]
    public async Task TestPaymentScenarios(
        string cardNumber, 
        bool shouldSucceed, 
        string scenario)
    {
        // Arrange
        var request = new PaymentRequest
        {
            Amount = 100.00m,
            Currency = "USD",
            CardNumber = cardNumber,
            ExpiryMonth = 12,
            ExpiryYear = DateTime.Now.Year + 2,
            CVV = "123"
        };
        
        // Act
        var result = await _gateway.ProcessPaymentAsync(request);
        
        // Assert
        Assert.Equal(shouldSucceed, result.IsSuccess);
        
        if (!shouldSucceed)
        {
            Assert.NotNull(result.ErrorMessage);
            _output.WriteLine($"Scenario: {scenario}");
            _output.WriteLine($"Error: {result.ErrorMessage}");
        }
    }
    
    [Fact]
    public async Task Test_3DSecureFlow()
    {
        // Arrange
        var request = new PaymentRequest
        {
            Amount = 100.00m,
            Currency = "USD",
            CardNumber = StripeTestCards.Require3DS
        };
        
        // Act
        var result = await _gateway.ProcessPaymentAsync(request);
        
        // Assert
        Assert.True(result.Requires3DSecure);
        Assert.NotNull(result.AuthenticationUrl);
        
        // Simulate 3DS authentication
        var authenticated = await Authenticate3DSecureAsync(result.AuthenticationUrl);
        Assert.True(authenticated);
        
        // Complete payment
        var completedResult = await _gateway.Complete3DSecurePaymentAsync(
            result.TransactionId);
        Assert.True(completedResult.IsSuccess);
    }
    
    [Fact]
    public async Task Test_RefundFlow()
    {
        // Arrange - Create successful payment
        var payment = await CreateSuccessfulPaymentAsync();
        
        // Act - Refund the payment
        var refundResult = await _gateway.RefundPaymentAsync(
            payment.TransactionId, 
            payment.Amount);
        
        // Assert
        Assert.True(refundResult.IsSuccess);
        Assert.NotNull(refundResult.RefundId);
    }
    
    [Fact]
    public async Task Test_PartialRefund()
    {
        // Arrange
        var payment = await CreateSuccessfulPaymentAsync(100.00m);
        
        // Act - Partial refund
        var refundResult = await _gateway.RefundPaymentAsync(
            payment.TransactionId, 
            50.00m); // Refund half
        
        // Assert
        Assert.True(refundResult.IsSuccess);
        Assert.Equal(50.00m, refundResult.RefundedAmount);
    }
}
```

---

## Webhook Testing

### Testing Webhook Delivery

```csharp
public class WebhookTestingService
{
    private readonly IWebhookHandler _webhookHandler;
    
    // Stripe webhook testing
    [Fact]
    public async Task Test_StripeWebhook()
    {
        // Use Stripe CLI to forward webhooks
        // stripe listen --forward-to localhost:5000/api/webhooks/stripe
        
        // Or trigger test event
        // stripe trigger payment_intent.succeeded
        
        var testPayload = @"{
            'id': 'evt_test_webhook',
            'type': 'payment_intent.succeeded',
            'data': {
                'object': {
                    'id': 'pi_test_123',
                    'amount': 2000,
                    'currency': 'usd',
                    'status': 'succeeded'
                }
            }
        }";
        
        var signature = GenerateStripeSignature(testPayload);
        
        var result = await _webhookHandler.HandleStripeWebhookAsync(
            testPayload, 
            signature);
        
        Assert.True(result.IsSuccess);
    }
    
    // PayPal webhook testing
    [Fact]
    public async Task Test_PayPalWebhook()
    {
        var testPayload = @"{
            'event_type': 'PAYMENT.CAPTURE.COMPLETED',
            'resource': {
                'id': 'CAPTURE-123',
                'amount': {
                    'value': '100.00',
                    'currency_code': 'USD'
                },
                'status': 'COMPLETED'
            }
        }";
        
        // PayPal sends verification headers
        var headers = new Dictionary<string, string>
        {
            ["PAYPAL-TRANSMISSION-ID"] = "test-id",
            ["PAYPAL-TRANSMISSION-TIME"] = DateTime.UtcNow.ToString("o"),
            ["PAYPAL-TRANSMISSION-SIG"] = "test-signature"
        };
        
        var result = await _webhookHandler.HandlePayPalWebhookAsync(
            testPayload, 
            headers);
        
        Assert.True(result.IsSuccess);
    }
}
```

### Local Webhook Testing Tools

```bash
# Stripe CLI
stripe login
stripe listen --forward-to localhost:5000/api/webhooks/stripe
stripe trigger payment_intent.succeeded

# ngrok (for any provider)
ngrok http 5000
# Use ngrok URL as webhook endpoint

# PayPal Webhook Simulator
# Available in PayPal Developer Dashboard

# Square Webhook Testing
# Test webhooks in Square Dashboard -> Webhooks -> Test
```

---

## Configuration Templates

### .env Template (All Providers)

```bash
# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# PayPal
PAYPAL_MODE=sandbox
PAYPAL_CLIENT_ID=...
PAYPAL_CLIENT_SECRET=...

# Square
SQUARE_ENVIRONMENT=sandbox
SQUARE_ACCESS_TOKEN=...
SQUARE_APPLICATION_ID=...
SQUARE_LOCATION_ID=...

# Adyen
ADYEN_API_KEY=...
ADYEN_MERCHANT_ACCOUNT=...
ADYEN_ENVIRONMENT=TEST

# Braintree
BRAINTREE_ENVIRONMENT=sandbox
BRAINTREE_MERCHANT_ID=...
BRAINTREE_PUBLIC_KEY=...
BRAINTREE_PRIVATE_KEY=...

# Authorize.Net
AUTHORIZENET_API_LOGIN_ID=...
AUTHORIZENET_TRANSACTION_KEY=...
AUTHORIZENET_ENVIRONMENT=SANDBOX

# Razorpay
RAZORPAY_KEY_ID=rzp_test_...
RAZORPAY_KEY_SECRET=...

# Mercado Pago
MERCADOPAGO_ACCESS_TOKEN=TEST-...
MERCADOPAGO_PUBLIC_KEY=TEST-...
```

### appsettings.Development.json Template

```json
{
  "PaymentProviders": {
    "Stripe": {
      "SecretKey": "",
      "PublishableKey": "",
      "WebhookSecret": ""
    },
    "PayPal": {
      "Mode": "sandbox",
      "ClientId": "",
      "ClientSecret": ""
    },
    "Square": {
      "Environment": "sandbox",
      "AccessToken": "",
      "ApplicationId": "",
      "LocationId": ""
    },
    "Adyen": {
      "Environment": "Test",
      "ApiKey": "",
      "MerchantAccount": ""
    },
    "Braintree": {
      "Environment": "sandbox",
      "MerchantId": "",
      "PublicKey": "",
      "PrivateKey": ""
    },
    "AuthorizeNet": {
      "Environment": "SANDBOX",
      "ApiLoginId": "",
      "TransactionKey": ""
    }
  }
}
```

---

## Troubleshooting

### API Keys Not Working

- Using correct environment (test vs. production)?
- Copied full key (including prefixes like `sk_test_`, `rzp_test_`)?
- No extra spaces or line breaks?
- Key has required permissions?
- Account fully activated?

### Test Cards Being Declined

- Using production API keys with test cards?
- Using test API keys with real cards?
- Incorrect CVV or expiry date format?
- Provider-specific card requirements not met?

**Fix:** Verify you're in test mode, double-check card numbers from provider docs.

### Webhooks Not Receiving Events

1. Use ngrok for local testing: `ngrok http 5000`
2. Use provider CLI tools: `stripe listen --forward-to localhost:5000/webhook`
3. Check webhook logs in provider dashboard

### Sandbox Account Limitations

Some providers limit test transactions, API call rate, webhook delivery attempts, or expire sandbox data. Check provider documentation for specifics and clean up old test data regularly.

---

## Best Practices

### Testing Checklist

```yaml
Core Scenarios:
  - ✅ Successful payment
  - ✅ Declined payment
  - ✅ Insufficient funds
  - ✅ Invalid card details
  - ✅ Expired card
  - ✅ Incorrect CVV/CVC
  - ✅ Failed AVS check

3D Secure:
  - ✅ Authentication required
  - ✅ Authentication succeeded
  - ✅ Authentication failed
  - ✅ Card not enrolled

Refunds:
  - ✅ Full refund
  - ✅ Partial refund
  - ✅ Multiple partial refunds
  - ✅ Refund declined

Edge Cases:
  - ✅ Duplicate transaction
  - ✅ Network timeout
  - ✅ Processing error
  - ✅ Concurrent requests
  - ✅ Idempotency

Webhooks:
  - ✅ Signature verification
  - ✅ Event processing
  - ✅ Duplicate events
  - ✅ Event replay
  - ✅ Failed delivery retry
```

### Environment Configuration

```csharp
public class PaymentConfiguration
{
    public static IPaymentGateway ConfigureGateway(IConfiguration config)
    {
        var environment = config["Environment"]; // "test" or "production"
        var provider = config["PaymentProvider"];
        
        return provider.ToLower() switch
        {
            "stripe" => new StripePaymentGateway(
                apiKey: environment == "test" 
                    ? config["Stripe:TestSecretKey"]
                    : config["Stripe:LiveSecretKey"]),
            
            "paypal" => new PayPalPaymentGateway(
                clientId: config[$"PayPal:{environment}:ClientId"],
                clientSecret: config[$"PayPal:{environment}:ClientSecret"],
                mode: environment == "test" ? "sandbox" : "live"),
            
            "square" => new SquarePaymentGateway(
                accessToken: config[$"Square:{environment}:AccessToken"],
                environment: environment == "test" ? "sandbox" : "production"),
            
            _ => throw new NotSupportedException()
        };
    }
}

// appsettings.Development.json
{
  "Environment": "test",
  "PaymentProvider": "stripe",
  "Stripe": {
    "TestSecretKey": "sk_test_...",
    "TestPublishableKey": "pk_test_..."
  },
  "PayPal": {
    "test": {
      "ClientId": "...",
      "ClientSecret": "..."
    }
  }
}
```

---

## Summary

### Quick Reference

| Provider | Test Mode | Test Cards | Webhook Testing |
|----------|-----------|------------|-----------------|
| **Stripe** | API key: sk_test_* | 4242424242424242 | Stripe CLI |
| **PayPal** | Mode: sandbox | Sandbox accounts | Dashboard |
| **Square** | Environment.Sandbox | 4111111111111111 | Dashboard |
| **Adyen** | Environment.Test | 4917610000000000 | Test webhooks |
| **Braintree** | Environment.SANDBOX | 4111111111111111 | Test panel |
| **Authorize.Net** | SANDBOX environment | 4007000000027 | Test mode |

### Key Takeaways

✅ Always use test mode/sandbox for development  
✅ Never use real cards in test environments  
✅ Test all success and failure scenarios  
✅ Verify webhook signature validation  
✅ Test 3D Secure authentication flows  
✅ Validate refund functionality  
✅ Use provider CLI tools for webhook testing  
✅ Keep test credentials separate from production  

## Next Steps

- [Security Patterns & Practices](15-SECURITY-PATTERNS-PRACTICES.md)
- [Testing Guide](14-E2E-INTEGRATION-TESTING.md)
- [Webhook Patterns](04-WEBHOOK-PATTERNS.md)
