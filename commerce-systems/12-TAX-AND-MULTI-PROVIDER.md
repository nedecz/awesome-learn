# Tax Handling and Multi-Provider Payment Integration

## Overview

This document covers tax calculation, compliance, and integration with multiple payment providers including Stripe, PayPal, Square, Adyen, Braintree, Authorize.net, and specialized regional providers.

## Table of Contents

1. [Tax Handling](#tax-handling)
2. [Payment Provider Comparison](#payment-provider-comparison)
3. [PayPal Integration](#paypal-integration)
4. [Square Integration](#square-integration)
5. [Adyen Integration](#adyen-integration)
6. [Braintree Integration](#braintree-integration)
7. [Authorize.Net Integration](#authorizenet-integration)
8. [Regional Providers](#regional-providers)
9. [Multi-Provider Architecture](#multi-provider-architecture)
10. [Provider Selection Strategy](#provider-selection-strategy)

---

## Tax Handling

### Tax Calculation Services

#### Stripe Tax Integration

```csharp
using Stripe;
using Stripe.Tax;

public class StripeTaxCalculator : ITaxCalculator
{
    private readonly CalculationService _calculationService;
    
    public StripeTaxCalculator()
    {
        _calculationService = new CalculationService();
    }
    
    public async Task<TaxCalculation> CalculateTaxAsync(TaxCalculationRequest request)
    {
        var options = new CalculationCreateOptions
        {
            Currency = request.Currency,
            LineItems = request.Items.Select(item => new CalculationLineItemOptions
            {
                Amount = (long)(item.Amount * 100),
                Reference = item.ProductId,
                TaxCode = item.TaxCode // Stripe tax code
            }).ToList(),
            CustomerDetails = new CalculationCustomerDetailsOptions
            {
                Address = new AddressOptions
                {
                    Line1 = request.CustomerAddress.Line1,
                    City = request.CustomerAddress.City,
                    State = request.CustomerAddress.State,
                    PostalCode = request.CustomerAddress.PostalCode,
                    Country = request.CustomerAddress.Country
                },
                AddressSource = "shipping"
            },
            ShippingCost = new CalculationShippingCostOptions
            {
                Amount = (long)(request.ShippingAmount * 100)
            }
        };
        
        var calculation = await _calculationService.CreateAsync(options);
        
        return new TaxCalculation
        {
            TaxAmount = calculation.TaxAmountExclusive / 100m,
            TaxRate = calculation.TaxAmountExclusive / (decimal)calculation.AmountTotal,
            TaxBreakdown = calculation.TaxBreakdown.Select(b => new TaxBreakdownItem
            {
                JurisdictionName = b.Jurisdiction,
                TaxAmount = b.TaxAmountExclusive / 100m,
                TaxRate = b.TaxRatePercentageDecimal,
                TaxType = b.Jurisdiction == "US" ? "Sales Tax" : "VAT"
            }).ToList(),
            CalculationId = calculation.Id
        };
    }
}
```

#### Avalara (TaxJar) Integration

```csharp
using Avalara.AvaTax.RestClient;

public class AvalaraTaxCalculator : ITaxCalculator
{
    private readonly AvaTaxClient _client;
    
    public AvalaraTaxCalculator(string accountId, string licenseKey)
    {
        _client = new AvaTaxClient("PaymentApp", "1.0", Environment.MachineName, AvaTaxEnvironment.Production)
            .WithSecurity(accountId, licenseKey);
    }
    
    public async Task<TaxCalculation> CalculateTaxAsync(TaxCalculationRequest request)
    {
        var transaction = new CreateTransactionModel
        {
            Type = DocumentType.SalesOrder,
            CompanyCode = "DEFAULT",
            Date = DateTime.UtcNow,
            CustomerCode = request.CustomerId,
            
            Addresses = new AddressesModel
            {
                ShipFrom = new AddressLocationInfo
                {
                    Line1 = request.OriginAddress.Line1,
                    City = request.OriginAddress.City,
                    Region = request.OriginAddress.State,
                    PostalCode = request.OriginAddress.PostalCode,
                    Country = request.OriginAddress.Country
                },
                ShipTo = new AddressLocationInfo
                {
                    Line1 = request.CustomerAddress.Line1,
                    City = request.CustomerAddress.City,
                    Region = request.CustomerAddress.State,
                    PostalCode = request.CustomerAddress.PostalCode,
                    Country = request.CustomerAddress.Country
                }
            },
            
            Lines = request.Items.Select((item, index) => new LineItemModel
            {
                Number = (index + 1).ToString(),
                Quantity = item.Quantity,
                Amount = item.Amount,
                TaxCode = item.TaxCode,
                ItemCode = item.ProductId,
                Description = item.Description
            }).ToList()
        };
        
        var result = await _client.CreateTransactionAsync(null, transaction);
        
        return new TaxCalculation
        {
            TaxAmount = result.TotalTax ?? 0,
            TaxRate = result.TotalTaxCalculated > 0 
                ? (result.TotalTax ?? 0) / result.TotalTaxCalculated 
                : 0,
            TaxBreakdown = result.Lines.SelectMany(l => l.Details.Select(d => new TaxBreakdownItem
            {
                JurisdictionName = d.JurisName,
                TaxAmount = d.Tax ?? 0,
                TaxRate = d.Rate ?? 0,
                TaxType = d.TaxType.ToString()
            })).ToList(),
            CalculationId = result.Code
        };
    }
    
    public async Task CommitTransactionAsync(string calculationId)
    {
        var commitRequest = new CommitTransactionModel
        {
            Commit = true
        };
        
        await _client.CommitTransactionAsync("DEFAULT", calculationId, null, commitRequest);
    }
}
```

#### TaxJar Integration

```csharp
using Taxjar;

public class TaxJarCalculator : ITaxCalculator
{
    private readonly TaxjarApi _client;
    
    public TaxJarCalculator(string apiToken)
    {
        _client = new TaxjarApi(apiToken);
    }
    
    public async Task<TaxCalculation> CalculateTaxAsync(TaxCalculationRequest request)
    {
        var taxRequest = new
        {
            from_country = request.OriginAddress.Country,
            from_zip = request.OriginAddress.PostalCode,
            from_state = request.OriginAddress.State,
            from_city = request.OriginAddress.City,
            from_street = request.OriginAddress.Line1,
            
            to_country = request.CustomerAddress.Country,
            to_zip = request.CustomerAddress.PostalCode,
            to_state = request.CustomerAddress.State,
            to_city = request.CustomerAddress.City,
            to_street = request.CustomerAddress.Line1,
            
            amount = request.Items.Sum(i => i.Amount),
            shipping = request.ShippingAmount,
            
            line_items = request.Items.Select(item => new
            {
                id = item.ProductId,
                quantity = item.Quantity,
                product_tax_code = item.TaxCode,
                unit_price = item.Amount / item.Quantity,
                discount = item.Discount
            }).ToArray()
        };
        
        var tax = await Task.Run(() => _client.TaxForOrder(taxRequest));
        
        return new TaxCalculation
        {
            TaxAmount = (decimal)tax.AmountToCollect,
            TaxRate = (decimal)tax.Rate,
            TaxBreakdown = tax.Breakdown.LineItems.Select(item => new TaxBreakdownItem
            {
                JurisdictionName = tax.Breakdown.State,
                TaxAmount = (decimal)item.TaxCollectable,
                TaxRate = (decimal)item.CombinedTaxRate,
                TaxType = "Sales Tax"
            }).ToList()
        };
    }
}
```

### VAT Handling (Europe)

```csharp
public class VatCalculator : ITaxCalculator
{
    private static readonly Dictionary<string, decimal> VatRates = new()
    {
        ["AT"] = 0.20m,  // Austria
        ["BE"] = 0.21m,  // Belgium
        ["DE"] = 0.19m,  // Germany
        ["FR"] = 0.20m,  // France
        ["GB"] = 0.20m,  // UK
        ["IT"] = 0.22m,  // Italy
        ["ES"] = 0.21m,  // Spain
        ["NL"] = 0.21m,  // Netherlands
        ["PL"] = 0.23m,  // Poland
        ["IE"] = 0.23m   // Ireland
    };
    
    public async Task<TaxCalculation> CalculateTaxAsync(TaxCalculationRequest request)
    {
        var country = request.CustomerAddress.Country;
        
        // Check if B2B (reverse charge)
        if (request.IsBusinessCustomer && !string.IsNullOrEmpty(request.VatNumber))
        {
            var isValid = await ValidateVatNumberAsync(request.VatNumber, country);
            
            if (isValid && country != request.OriginAddress.Country)
            {
                // Reverse charge - no VAT
                return new TaxCalculation
                {
                    TaxAmount = 0,
                    TaxRate = 0,
                    ReverseCharge = true,
                    VatNumber = request.VatNumber
                };
            }
        }
        
        // B2C or domestic B2B
        if (VatRates.TryGetValue(country, out var rate))
        {
            var subtotal = request.Items.Sum(i => i.Amount) + request.ShippingAmount;
            var taxAmount = subtotal * rate;
            
            return new TaxCalculation
            {
                TaxAmount = taxAmount,
                TaxRate = rate,
                TaxType = "VAT",
                TaxBreakdown = new List<TaxBreakdownItem>
                {
                    new TaxBreakdownItem
                    {
                        JurisdictionName = country,
                        TaxAmount = taxAmount,
                        TaxRate = rate,
                        TaxType = "VAT"
                    }
                }
            };
        }
        
        return new TaxCalculation { TaxAmount = 0, TaxRate = 0 };
    }
    
    private async Task<bool> ValidateVatNumberAsync(string vatNumber, string country)
    {
        // Call VIES (VAT Information Exchange System)
        using var client = new HttpClient();
        var response = await client.GetAsync(
            $"https://ec.europa.eu/taxation_customs/vies/rest-api/ms/{country}/vat/{vatNumber}");
        
        if (!response.IsSuccessStatusCode)
            return false;
        
        var result = await response.Content.ReadFromJsonAsync<ViesResponse>();
        return result?.IsValid ?? false;
    }
}

public class ViesResponse
{
    public bool IsValid { get; set; }
    public string Name { get; set; }
    public string Address { get; set; }
}
```

### Tax Abstractions

```csharp
public interface ITaxCalculator
{
    Task<TaxCalculation> CalculateTaxAsync(TaxCalculationRequest request);
}

public class TaxCalculationRequest
{
    public string CustomerId { get; set; }
    public Address OriginAddress { get; set; }
    public Address CustomerAddress { get; set; }
    public List<LineItem> Items { get; set; }
    public decimal ShippingAmount { get; set; }
    public string Currency { get; set; }
    
    // VAT specific
    public bool IsBusinessCustomer { get; set; }
    public string VatNumber { get; set; }
}

public class TaxCalculation
{
    public decimal TaxAmount { get; set; }
    public decimal TaxRate { get; set; }
    public string TaxType { get; set; }
    public List<TaxBreakdownItem> TaxBreakdown { get; set; }
    public string CalculationId { get; set; }
    
    // VAT specific
    public bool ReverseCharge { get; set; }
    public string VatNumber { get; set; }
}

public class TaxBreakdownItem
{
    public string JurisdictionName { get; set; }
    public decimal TaxAmount { get; set; }
    public decimal TaxRate { get; set; }
    public string TaxType { get; set; }
}

public class LineItem
{
    public string ProductId { get; set; }
    public string Description { get; set; }
    public int Quantity { get; set; }
    public decimal Amount { get; set; }
    public decimal Discount { get; set; }
    public string TaxCode { get; set; }
}
```

### Tax-Inclusive Payment Processing

```csharp
public class TaxAwarePaymentService
{
    private readonly ITaxCalculator _taxCalculator;
    private readonly IPaymentService _paymentService;
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Calculate tax
        var taxCalc = await _taxCalculator.CalculateTaxAsync(new TaxCalculationRequest
        {
            CustomerId = request.CustomerId,
            OriginAddress = request.MerchantAddress,
            CustomerAddress = request.CustomerAddress,
            Items = request.Items,
            ShippingAmount = request.ShippingAmount,
            Currency = request.Currency,
            IsBusinessCustomer = request.IsBusinessCustomer,
            VatNumber = request.VatNumber
        });
        
        // Calculate total with tax
        var subtotal = request.Items.Sum(i => i.Amount) + request.ShippingAmount;
        var total = subtotal + taxCalc.TaxAmount;
        
        // Process payment
        request.Amount = total;
        request.TaxAmount = taxCalc.TaxAmount;
        request.TaxCalculationId = taxCalc.CalculationId;
        
        var result = await _paymentService.ProcessPaymentAsync(request);
        
        // Commit tax calculation if payment succeeded
        if (result.IsSuccess && !string.IsNullOrEmpty(taxCalc.CalculationId))
        {
            await CommitTaxCalculationAsync(taxCalc.CalculationId);
        }
        
        return result;
    }
}
```

---

## Payment Provider Comparison

### Feature Comparison Matrix

| Feature | Stripe | PayPal | Square | Adyen | Braintree | Authorize.Net |
|---------|--------|--------|--------|-------|-----------|---------------|
| **Transaction Fee** | 2.9% + 30¢ | 2.9% + 30¢ | 2.6% + 10¢ | Custom | 2.9% + 30¢ | 2.9% + 30¢ |
| **International** | ✅ 135+ countries | ✅ 200+ countries | ✅ Limited | ✅ Global | ✅ 45+ countries | ✅ US, CA, EU, AU |
| **Currencies** | 135+ | 25+ | Limited | 150+ | 130+ | Multiple |
| **ACH/Bank Transfer** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Apple Pay** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Google Pay** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Subscriptions** | ✅ Excellent | ✅ Good | ✅ Good | ✅ Excellent | ✅ Good | ✅ Basic |
| **Fraud Detection** | ✅ Radar | ✅ Basic | ✅ Basic | ✅ Advanced | ✅ Basic | ✅ Basic |
| **PCI Compliance** | ✅ SAQ A | ✅ SAQ A | ✅ SAQ A | ✅ SAQ A | ✅ SAQ A | ✅ SAQ A |
| **3D Secure** | ✅ SCA | ✅ Yes | ✅ Yes | ✅ Advanced | ✅ Yes | ✅ Yes |
| **Marketplace** | ✅ Connect | ✅ Adaptive | ❌ | ✅ Platforms | ❌ | ❌ |
| **Settlement Time** | 2-7 days | 1-3 days | 1-2 days | 1-3 days | 2-4 days | 2-3 days |

---

## PayPal Integration

For complete PayPal integration including Orders API v2, Subscriptions, Payouts, Refunds, Disputes, Webhooks, and frontend integration, see [22-PAYPAL-INTEGRATION](17-PAYPAL-INTEGRATION.md).

> The PayPal integration follows the same `IPaymentGateway` interface used by other providers in this document. See the dedicated guide for the recommended Checkout SDK v2 implementation.

---

## Square Integration

```csharp
using Square;
using Square.Models;

public class SquarePaymentGateway : IPaymentGateway
{
    private readonly ISquareClient _client;
    
    public SquarePaymentGateway(string accessToken, string environment)
    {
        _client = new SquareClient.Builder()
            .Environment(environment == "production" 
                ? Square.Environment.Production 
                : Square.Environment.Sandbox)
            .AccessToken(accessToken)
            .Build();
    }
    
    public async Task<GatewayResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var paymentsApi = _client.PaymentsApi;
        
        var amountMoney = new Money.Builder()
            .Amount(Convert.ToInt64(request.Amount * 100))
            .Currency(request.Currency)
            .Build();
        
        var body = new CreatePaymentRequest.Builder(
            sourceId: request.PaymentMethodId, // Card nonce from Square.js
            idempotencyKey: request.IdempotencyKey ?? Guid.NewGuid().ToString())
            .AmountMoney(amountMoney)
            .LocationId(request.LocationId)
            .ReferenceId(request.OrderId)
            .Note(request.Description)
            .Build();
        
        try
        {
            var result = await paymentsApi.CreatePaymentAsync(body);
            var payment = result.Payment;
            
            return new GatewayResult
            {
                IsSuccess = payment.Status == "COMPLETED",
                TransactionId = payment.Id,
                Status = payment.Status,
                ReceiptUrl = payment.ReceiptUrl
            };
        }
        catch (ApiException ex)
        {
            return new GatewayResult
            {
                IsSuccess = false,
                ErrorMessage = ex.Errors[0].Detail,
                ErrorCode = ex.Errors[0].Code
            };
        }
    }
    
    public async Task<GatewayResult> RefundPaymentAsync(string paymentId, decimal amount, string currency)
    {
        var refundsApi = _client.RefundsApi;
        
        var amountMoney = new Money.Builder()
            .Amount(Convert.ToInt64(amount * 100))
            .Currency(currency)
            .Build();
        
        var body = new RefundPaymentRequest.Builder(
            idempotencyKey: Guid.NewGuid().ToString(),
            amountMoney: amountMoney,
            paymentId: paymentId)
            .Build();
        
        try
        {
            var result = await refundsApi.RefundPaymentAsync(body);
            var refund = result.Refund;
            
            return new GatewayResult
            {
                IsSuccess = refund.Status == "COMPLETED",
                TransactionId = refund.Id,
                Status = refund.Status
            };
        }
        catch (ApiException ex)
        {
            return new GatewayResult
            {
                IsSuccess = false,
                ErrorMessage = ex.Errors[0].Detail
            };
        }
    }
    
    public async Task<List<Payment>> ListPaymentsAsync(DateTime beginTime, DateTime endTime)
    {
        var paymentsApi = _client.PaymentsApi;
        
        var result = await paymentsApi.ListPaymentsAsync(
            beginTime: beginTime.ToString("yyyy-MM-dd'T'HH:mm:ss'Z'"),
            endTime: endTime.ToString("yyyy-MM-dd'T'HH:mm:ss'Z'"));
        
        return result.Payments;
    }
}
```

---

## Adyen Integration

```csharp
using Adyen;
using Adyen.Model.Checkout;
using Adyen.Service;

public class AdyenPaymentGateway : IPaymentGateway
{
    private readonly Client _client;
    private readonly Checkout _checkout;
    
    public AdyenPaymentGateway(string apiKey, string merchantAccount, string environment)
    {
        var config = new Config
        {
            XApiKey = apiKey,
            MerchantAccount = merchantAccount,
            Environment = environment == "live" 
                ? Adyen.Model.Enum.Environment.Live 
                : Adyen.Model.Enum.Environment.Test
        };
        
        _client = new Client(config);
        _checkout = new Checkout(_client);
    }
    
    public async Task<GatewayResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var paymentRequest = new PaymentRequest
        {
            Amount = new Amount
            {
                Currency = request.Currency,
                Value = Convert.ToInt64(request.Amount * 100)
            },
            Reference = request.OrderId,
            MerchantAccount = _client.Config.MerchantAccount,
            PaymentMethod = new CheckoutPaymentMethod
            {
                Type = request.PaymentMethodType, // "scheme", "ideal", etc.
                EncryptedCardNumber = request.EncryptedCardNumber,
                EncryptedExpiryMonth = request.EncryptedExpiryMonth,
                EncryptedExpiryYear = request.EncryptedExpiryYear,
                EncryptedSecurityCode = request.EncryptedSecurityCode
            },
            ReturnUrl = request.ReturnUrl,
            ShopperReference = request.CustomerId,
            ShopperEmail = request.CustomerEmail,
            ShopperIP = request.IpAddress,
            BillingAddress = new Address
            {
                Street = request.BillingAddress.Line1,
                City = request.BillingAddress.City,
                StateOrProvince = request.BillingAddress.State,
                PostalCode = request.BillingAddress.PostalCode,
                Country = request.BillingAddress.Country
            },
            LineItems = request.Items.Select(item => new LineItem
            {
                Id = item.ProductId,
                Description = item.Description,
                Quantity = item.Quantity,
                AmountIncludingTax = Convert.ToInt64(item.Amount * 100),
                AmountExcludingTax = Convert.ToInt64((item.Amount - item.TaxAmount) * 100)
            }).ToList()
        };
        
        try
        {
            var response = await _checkout.PaymentsAsync(paymentRequest);
            
            return new GatewayResult
            {
                IsSuccess = response.ResultCode == PaymentResponse.ResultCodeEnum.Authorised,
                TransactionId = response.PspReference,
                Status = response.ResultCode.ToString(),
                AuthCode = response.AuthCode,
                Action = response.Action // For 3D Secure
            };
        }
        catch (Exception ex)
        {
            return new GatewayResult
            {
                IsSuccess = false,
                ErrorMessage = ex.Message
            };
        }
    }
    
    public async Task<GatewayResult> CapturePaymentAsync(string pspReference, decimal amount, string currency)
    {
        var captureRequest = new CaptureRequest
        {
            Amount = new Amount
            {
                Currency = currency,
                Value = Convert.ToInt64(amount * 100)
            },
            MerchantAccount = _client.Config.MerchantAccount,
            Reference = Guid.NewGuid().ToString()
        };
        
        var modification = new Modification(_client);
        var response = await modification.CaptureAsync(pspReference, captureRequest);
        
        return new GatewayResult
        {
            IsSuccess = response.Response == "[capture-received]",
            TransactionId = response.PspReference,
            Status = response.Response
        };
    }
    
    public async Task<GatewayResult> RefundPaymentAsync(string pspReference, decimal amount, string currency)
    {
        var refundRequest = new RefundRequest
        {
            Amount = new Amount
            {
                Currency = currency,
                Value = Convert.ToInt64(amount * 100)
            },
            MerchantAccount = _client.Config.MerchantAccount,
            Reference = Guid.NewGuid().ToString()
        };
        
        var modification = new Modification(_client);
        var response = await modification.RefundAsync(pspReference, refundRequest);
        
        return new GatewayResult
        {
            IsSuccess = response.Response == "[refund-received]",
            TransactionId = response.PspReference,
            Status = response.Response
        };
    }
}
```

---

## Braintree Integration

```csharp
using Braintree;

public class BraintreePaymentGateway : IPaymentGateway
{
    private readonly BraintreeGateway _gateway;
    
    public BraintreePaymentGateway(string merchantId, string publicKey, string privateKey, string environment)
    {
        var env = environment == "production" 
            ? Braintree.Environment.PRODUCTION 
            : Braintree.Environment.SANDBOX;
        
        _gateway = new BraintreeGateway
        {
            Environment = env,
            MerchantId = merchantId,
            PublicKey = publicKey,
            PrivateKey = privateKey
        };
    }
    
    public async Task<GatewayResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var transactionRequest = new TransactionRequest
        {
            Amount = request.Amount,
            PaymentMethodNonce = request.PaymentMethodNonce,
            OrderId = request.OrderId,
            Options = new TransactionOptionsRequest
            {
                SubmitForSettlement = true
            },
            Customer = new CustomerRequest
            {
                Email = request.CustomerEmail,
                FirstName = request.CustomerFirstName,
                LastName = request.CustomerLastName
            },
            BillingAddress = new AddressRequest
            {
                StreetAddress = request.BillingAddress.Line1,
                Locality = request.BillingAddress.City,
                Region = request.BillingAddress.State,
                PostalCode = request.BillingAddress.PostalCode,
                CountryCodeAlpha2 = request.BillingAddress.Country
            },
            LineItems = request.Items.Select(item => new TransactionLineItemRequest
            {
                Name = item.Name,
                Kind = TransactionLineItemKind.DEBIT,
                Quantity = item.Quantity,
                UnitAmount = item.Price,
                TotalAmount = item.Amount
            }).ToArray()
        };
        
        var result = await Task.Run(() => _gateway.Transaction.Sale(transactionRequest));
        
        if (result.IsSuccess())
        {
            return new GatewayResult
            {
                IsSuccess = true,
                TransactionId = result.Target.Id,
                Status = result.Target.Status.ToString(),
                AuthCode = result.Target.ProcessorAuthorizationCode
            };
        }
        else
        {
            return new GatewayResult
            {
                IsSuccess = false,
                ErrorMessage = result.Message,
                ErrorCode = result.Transaction?.ProcessorResponseCode
            };
        }
    }
    
    public async Task<GatewayResult> RefundPaymentAsync(string transactionId, decimal? amount = null)
    {
        var result = await Task.Run(() => 
            amount.HasValue 
                ? _gateway.Transaction.Refund(transactionId, amount.Value)
                : _gateway.Transaction.Refund(transactionId));
        
        if (result.IsSuccess())
        {
            return new GatewayResult
            {
                IsSuccess = true,
                TransactionId = result.Target.Id,
                Status = result.Target.Status.ToString()
            };
        }
        else
        {
            return new GatewayResult
            {
                IsSuccess = false,
                ErrorMessage = result.Message
            };
        }
    }
    
    public async Task<string> GenerateClientTokenAsync(string customerId = null)
    {
        var request = new ClientTokenRequest();
        
        if (!string.IsNullOrEmpty(customerId))
        {
            request.CustomerId = customerId;
        }
        
        return await Task.Run(() => _gateway.ClientToken.Generate(request));
    }
}
```

---

## Authorize.Net Integration

```csharp
using AuthorizeNet.Api.Contracts.V1;
using AuthorizeNet.Api.Controllers;

public class AuthorizeNetGateway : IPaymentGateway
{
    private readonly string _apiLoginId;
    private readonly string _transactionKey;
    private readonly bool _isProduction;
    
    public AuthorizeNetGateway(string apiLoginId, string transactionKey, bool isProduction)
    {
        _apiLoginId = apiLoginId;
        _transactionKey = transactionKey;
        _isProduction = isProduction;
        
        ApiOperationBase<ANetApiRequest, ANetApiResponse>.RunEnvironment = 
            isProduction ? AuthorizeNet.Environment.PRODUCTION : AuthorizeNet.Environment.SANDBOX;
    }
    
    public async Task<GatewayResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var merchantAuth = new merchantAuthenticationType
        {
            name = _apiLoginId,
            ItemElementName = ItemChoiceType.transactionKey,
            Item = _transactionKey
        };
        
        var creditCard = new creditCardType
        {
            cardNumber = request.CardNumber,
            expirationDate = request.ExpirationDate, // MMYY
            cardCode = request.Cvv
        };
        
        var payment = new paymentType { Item = creditCard };
        
        var billTo = new customerAddressType
        {
            firstName = request.CustomerFirstName,
            lastName = request.CustomerLastName,
            address = request.BillingAddress.Line1,
            city = request.BillingAddress.City,
            state = request.BillingAddress.State,
            zip = request.BillingAddress.PostalCode,
            country = request.BillingAddress.Country
        };
        
        var lineItems = request.Items.Select(item => new lineItemType
        {
            itemId = item.ProductId,
            name = item.Name,
            quantity = item.Quantity,
            unitPrice = item.Price
        }).ToArray();
        
        var transactionRequest = new transactionRequestType
        {
            transactionType = transactionTypeEnum.authCaptureTransaction.ToString(),
            amount = request.Amount,
            payment = payment,
            billTo = billTo,
            lineItems = lineItems,
            order = new orderType
            {
                invoiceNumber = request.OrderId,
                description = request.Description
            }
        };
        
        var apiRequest = new createTransactionRequest
        {
            merchantAuthentication = merchantAuth,
            transactionRequest = transactionRequest
        };
        
        var controller = new createTransactionController(apiRequest);
        controller.Execute();
        
        var response = controller.GetApiResponse();
        
        if (response != null)
        {
            if (response.messages.resultCode == messageTypeEnum.Ok)
            {
                var transResponse = response.transactionResponse;
                
                return new GatewayResult
                {
                    IsSuccess = true,
                    TransactionId = transResponse.transId,
                    AuthCode = transResponse.authCode,
                    Status = "Approved"
                };
            }
            else
            {
                var errors = response.transactionResponse?.errors ?? response.messages.message;
                
                return new GatewayResult
                {
                    IsSuccess = false,
                    ErrorMessage = errors[0].text,
                    ErrorCode = errors[0].code
                };
            }
        }
        
        return new GatewayResult
        {
            IsSuccess = false,
            ErrorMessage = "No response from gateway"
        };
    }
    
    public async Task<GatewayResult> RefundPaymentAsync(string transactionId, decimal amount, string last4Digits)
    {
        var merchantAuth = new merchantAuthenticationType
        {
            name = _apiLoginId,
            ItemElementName = ItemChoiceType.transactionKey,
            Item = _transactionKey
        };
        
        var creditCard = new creditCardType
        {
            cardNumber = last4Digits,
            expirationDate = "XXXX"
        };
        
        var payment = new paymentType { Item = creditCard };
        
        var transactionRequest = new transactionRequestType
        {
            transactionType = transactionTypeEnum.refundTransaction.ToString(),
            amount = amount,
            payment = payment,
            refTransId = transactionId
        };
        
        var apiRequest = new createTransactionRequest
        {
            merchantAuthentication = merchantAuth,
            transactionRequest = transactionRequest
        };
        
        var controller = new createTransactionController(apiRequest);
        controller.Execute();
        
        var response = controller.GetApiResponse();
        
        if (response?.messages.resultCode == messageTypeEnum.Ok)
        {
            return new GatewayResult
            {
                IsSuccess = true,
                TransactionId = response.transactionResponse.transId,
                Status = "Refunded"
            };
        }
        
        return new GatewayResult
        {
            IsSuccess = false,
            ErrorMessage = response?.messages.message[0].text
        };
    }
}
```

---

## Regional Providers

### Razorpay (India)

```csharp
using Razorpay.Api;

public class RazorpayGateway : IPaymentGateway
{
    private readonly RazorpayClient _client;
    
    public RazorpayGateway(string keyId, string keySecret)
    {
        _client = new RazorpayClient(keyId, keySecret);
    }
    
    public async Task<GatewayResult> CreateOrderAsync(PaymentRequest request)
    {
        var options = new Dictionary<string, object>
        {
            ["amount"] = Convert.ToInt32(request.Amount * 100), // paise
            ["currency"] = request.Currency,
            ["receipt"] = request.OrderId,
            ["notes"] = new Dictionary<string, string>
            {
                ["customer_id"] = request.CustomerId,
                ["order_id"] = request.OrderId
            }
        };
        
        var order = await Task.Run(() => _client.Order.Create(options));
        
        return new GatewayResult
        {
            IsSuccess = true,
            TransactionId = order["id"].ToString(),
            Status = order["status"].ToString()
        };
    }
    
    public async Task<GatewayResult> VerifyPaymentAsync(string orderId, string paymentId, string signature)
    {
        var attributes = new Dictionary<string, string>
        {
            ["razorpay_order_id"] = orderId,
            ["razorpay_payment_id"] = paymentId,
            ["razorpay_signature"] = signature
        };
        
        try
        {
            Utils.verifyPaymentSignature(attributes);
            
            return new GatewayResult
            {
                IsSuccess = true,
                TransactionId = paymentId,
                Status = "Verified"
            };
        }
        catch
        {
            return new GatewayResult
            {
                IsSuccess = false,
                ErrorMessage = "Signature verification failed"
            };
        }
    }
}
```

### Mercado Pago (Latin America)

```csharp
using MercadoPago.Config;
using MercadoPago.Client.Payment;
using MercadoPago.Resource.Payment;

public class MercadoPagoGateway : IPaymentGateway
{
    public MercadoPagoGateway(string accessToken)
    {
        MercadoPagoConfig.AccessToken = accessToken;
    }
    
    public async Task<GatewayResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var client = new PaymentClient();
        
        var paymentRequest = new PaymentCreateRequest
        {
            TransactionAmount = request.Amount,
            Token = request.CardToken,
            Description = request.Description,
            Installments = request.Installments,
            PaymentMethodId = request.PaymentMethodId,
            Payer = new PaymentPayerRequest
            {
                Email = request.CustomerEmail,
                FirstName = request.CustomerFirstName,
                LastName = request.CustomerLastName,
                Identification = new PaymentIdentificationRequest
                {
                    Type = request.IdentificationType,
                    Number = request.IdentificationNumber
                }
            },
            ExternalReference = request.OrderId
        };
        
        var payment = await client.CreateAsync(paymentRequest);
        
        return new GatewayResult
        {
            IsSuccess = payment.Status == "approved",
            TransactionId = payment.Id.ToString(),
            Status = payment.Status,
            StatusDetail = payment.StatusDetail
        };
    }
}
```

---

## Multi-Provider Architecture

### Provider Factory

```csharp
public interface IPaymentGatewayFactory
{
    IPaymentGateway CreateGateway(string provider);
}

public class PaymentGatewayFactory : IPaymentGatewayFactory
{
    private readonly IConfiguration _configuration;
    private readonly IServiceProvider _serviceProvider;
    
    public IPaymentGateway CreateGateway(string provider)
    {
        return provider.ToLower() switch
        {
            "stripe" => new StripePaymentGateway(
                _configuration["Stripe:SecretKey"]),
            
            "paypal" => new PayPalPaymentGateway(
                _configuration["PayPal:ClientId"],
                _configuration["PayPal:ClientSecret"],
                _configuration["PayPal:Mode"]),
            
            "square" => new SquarePaymentGateway(
                _configuration["Square:AccessToken"],
                _configuration["Square:Environment"]),
            
            "adyen" => new AdyenPaymentGateway(
                _configuration["Adyen:ApiKey"],
                _configuration["Adyen:MerchantAccount"],
                _configuration["Adyen:Environment"]),
            
            "braintree" => new BraintreePaymentGateway(
                _configuration["Braintree:MerchantId"],
                _configuration["Braintree:PublicKey"],
                _configuration["Braintree:PrivateKey"],
                _configuration["Braintree:Environment"]),
            
            "authorizenet" => new AuthorizeNetGateway(
                _configuration["AuthorizeNet:ApiLoginId"],
                _configuration["AuthorizeNet:TransactionKey"],
                bool.Parse(_configuration["AuthorizeNet:IsProduction"])),
            
            "razorpay" => new RazorpayGateway(
                _configuration["Razorpay:KeyId"],
                _configuration["Razorpay:KeySecret"]),
            
            _ => throw new NotSupportedException($"Payment provider '{provider}' is not supported")
        };
    }
}
```

### Multi-Provider Service

```csharp
public class MultiProviderPaymentService : IPaymentService
{
    private readonly IPaymentGatewayFactory _gatewayFactory;
    private readonly IProviderSelector _providerSelector;
    private readonly IPaymentRepository _repository;
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Select best provider
        var provider = await _providerSelector.SelectProviderAsync(request);
        
        // Get gateway instance
        var gateway = _gatewayFactory.CreateGateway(provider);
        
        // Process payment
        var result = await gateway.ProcessPaymentAsync(request);
        
        // Save payment record
        var payment = new Payment
        {
            Provider = provider,
            ProviderTransactionId = result.TransactionId,
            Amount = request.Amount,
            Status = result.IsSuccess ? "Completed" : "Failed"
        };
        
        await _repository.SaveAsync(payment);
        
        return result;
    }
}
```

---

## Provider Selection Strategy

### Smart Routing

```csharp
public class SmartProviderSelector : IProviderSelector
{
    private readonly IProviderCostCalculator _costCalculator;
    private readonly IProviderPerformanceTracker _performanceTracker;
    
    public async Task<string> SelectProviderAsync(PaymentRequest request)
    {
        var eligibleProviders = GetEligibleProviders(request);
        
        // Calculate scores for each provider
        var scores = new Dictionary<string, double>();
        
        foreach (var provider in eligibleProviders)
        {
            var score = await CalculateProviderScoreAsync(provider, request);
            scores[provider] = score;
        }
        
        // Select provider with highest score
        return scores.OrderByDescending(s => s.Value).First().Key;
    }
    
    private async Task<double> CalculateProviderScoreAsync(string provider, PaymentRequest request)
    {
        // Factors: cost, success rate, speed, features
        var cost = await _costCalculator.CalculateCostAsync(provider, request);
        var performance = await _performanceTracker.GetPerformanceAsync(provider);
        
        var costScore = 1.0 - (cost / 100); // Lower cost = higher score
        var successScore = performance.SuccessRate;
        var speedScore = 1.0 - (performance.AverageLatency / 5000); // Under 5s
        
        // Weighted average
        return (costScore * 0.3) + (successScore * 0.5) + (speedScore * 0.2);
    }
    
    private List<string> GetEligibleProviders(PaymentRequest request)
    {
        var providers = new List<string>();
        
        // Regional eligibility
        if (request.Country == "IN")
        {
            providers.Add("razorpay");
        }
        else if (request.Country == "BR")
        {
            providers.Add("mercadopago");
        }
        else
        {
            providers.Add("stripe");
            providers.Add("paypal");
            providers.Add("adyen");
        }
        
        // Amount-based eligibility
        if (request.Amount < 1000)
        {
            providers.Add("square");
        }
        
        return providers;
    }
}
```

### Cascading Fallback

```csharp
public class CascadingPaymentService : IPaymentService
{
    private readonly IPaymentGatewayFactory _gatewayFactory;
    private readonly List<string> _providerPriority = new() 
    { 
        "stripe", 
        "adyen", 
        "paypal" 
    };
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        foreach (var provider in _providerPriority)
        {
            try
            {
                var gateway = _gatewayFactory.CreateGateway(provider);
                var result = await gateway.ProcessPaymentAsync(request);
                
                if (result.IsSuccess)
                {
                    return result;
                }
                
                // Try next provider if failed
            }
            catch (Exception ex)
            {
                // Log and try next provider
                _logger.LogWarning(ex, "Provider {Provider} failed", provider);
            }
        }
        
        return PaymentResult.Failed("All providers failed");
    }
}
```

---

## Summary

This documentation covers:

✅ **Tax Calculation** - Stripe Tax, Avalara, TaxJar, VAT  
✅ **Multi-Provider** - Stripe, PayPal, Square, Adyen, Braintree, Authorize.Net  
✅ **Regional Providers** - Razorpay (India), Mercado Pago (LATAM)  
✅ **Smart Routing** - Cost optimization, performance-based selection  
✅ **Fallback Strategies** - Cascading providers for resilience

## Next Steps

- [Security Patterns & Practices](15-SECURITY-PATTERNS-PRACTICES.md)
- [Multi-Cloud Integration](10-MULTI-CLOUD-ALTERNATIVES.md)
- [Testing Guide](14-E2E-INTEGRATION-TESTING.md)
