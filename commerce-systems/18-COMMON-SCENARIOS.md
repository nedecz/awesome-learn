# Common Payment Scenarios

## Overview

Step-by-step implementation guides for the most common real-world payment scenarios in .NET 10.0+ commerce systems: subscription billing, marketplace/split payments, pre-authorization and capture, cart checkout, invoice payments, and more.

## Table of Contents

1. [E-Commerce Cart Checkout](#e-commerce-cart-checkout)
2. [Subscription Billing](#subscription-billing)
3. [Marketplace / Split Payments](#marketplace-split-payments)
4. [Pre-Authorization and Capture](#pre-authorization-and-capture)
5. [Invoice Payments](#invoice-payments)
6. [Pay-What-You-Want / Donations](#pay-what-you-want-donations)
7. [Multi-Currency Checkout](#multi-currency-checkout)
8. [Saved Payment Methods](#saved-payment-methods)
9. [Partial Payments and Installments](#partial-payments-and-installments)
10. [Handling Failed Payments](#handling-failed-payments)

---

## E-Commerce Cart Checkout

The standard flow: customer adds items to cart, proceeds to checkout, pays.

```csharp
public class CheckoutService
{
    private readonly IPaymentGateway _gateway;
    private readonly IOrderRepository _orderRepository;
    private readonly ITaxCalculator _taxCalculator;
    private readonly IInventoryService _inventoryService;
    private readonly ILogger<CheckoutService> _logger;

    public async Task<CheckoutResult> ProcessCheckoutAsync(CheckoutRequest request)
    {
        // 1. Validate cart / inventory
        var cartValidation = await _inventoryService.ValidateCartAsync(request.CartItems);
        if (!cartValidation.IsValid)
        {
            return CheckoutResult.Failure($"Items unavailable: {string.Join(", ", cartValidation.UnavailableItems)}");
        }

        // 2. Calculate totals
        var subtotal = request.CartItems.Sum(i => i.Price * i.Quantity);
        var tax = await _taxCalculator.CalculateTaxAsync(subtotal, request.ShippingAddress);
        var shipping = await CalculateShippingAsync(request.CartItems, request.ShippingAddress);
        var total = subtotal + tax + shipping;

        // 3. Create order
        var order = Order.Create(
            customerId: request.CustomerId,
            items: request.CartItems,
            subtotal: subtotal,
            tax: tax,
            shipping: shipping,
            total: total,
            currency: request.Currency);

        await _orderRepository.AddAsync(order);

        // 4. Process payment
        var paymentResult = await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            OrderId = order.Id.ToString(),
            Amount = total,
            Currency = request.Currency,
            PaymentMethodId = request.PaymentMethodId,
            CustomerEmail = request.CustomerEmail,
            IdempotencyKey = $"checkout-{order.Id}"
        });

        // 5. Update order status
        if (paymentResult.IsSuccess)
        {
            order.MarkAsPaid(paymentResult.TransactionId);
            await _inventoryService.ReserveItemsAsync(request.CartItems);
            _logger.LogInformation("Order {OrderId} paid successfully", order.Id);
        }
        else if (paymentResult.RequiresAction)
        {
            order.MarkAsPendingAction();
        }
        else
        {
            order.MarkAsFailed(paymentResult.Error?.Message);
            _logger.LogWarning("Payment failed for order {OrderId}: {Error}", order.Id, paymentResult.Error?.Message);
        }

        await _orderRepository.UpdateAsync(order);

        return new CheckoutResult
        {
            IsSuccess = paymentResult.IsSuccess,
            OrderId = order.Id,
            TransactionId = paymentResult.TransactionId,
            RequiresAction = paymentResult.RequiresAction,
            ActionUrl = paymentResult.ActionUrl,
            Error = paymentResult.Error?.UserFacingMessage
        };
    }
}
```

---

## Subscription Billing

Recurring payments with trial periods, upgrades, and cancellations.

```csharp
public class SubscriptionManager
{
    private readonly IPaymentGateway _gateway;
    private readonly ISubscriptionRepository _subscriptionRepo;
    private readonly INotificationService _notifications;

    // Create new subscription
    public async Task<SubscriptionResult> CreateSubscriptionAsync(SubscriptionCreateRequest request)
    {
        // 1. Create or retrieve customer in payment provider
        var customerId = await _gateway.GetOrCreateCustomerAsync(
            request.CustomerEmail, request.PaymentMethodId);

        // 2. Create subscription with payment provider
        var providerResult = await _gateway.CreateSubscriptionAsync(new ProviderSubscriptionRequest
        {
            CustomerId = customerId,
            PriceId = request.PlanPriceId,
            PaymentMethodId = request.PaymentMethodId,
            TrialDays = request.TrialDays,
            Metadata = new Dictionary<string, string>
            {
                ["user_id"] = request.UserId,
                ["plan"] = request.PlanName
            }
        });

        // 3. Save subscription locally
        var subscription = Subscription.Create(
            userId: request.UserId,
            planId: request.PlanId,
            providerSubscriptionId: providerResult.SubscriptionId,
            trialEndsAt: request.TrialDays > 0
                ? DateTime.UtcNow.AddDays(request.TrialDays)
                : null,
            currentPeriodEnd: providerResult.CurrentPeriodEnd);

        await _subscriptionRepo.AddAsync(subscription);

        return new SubscriptionResult
        {
            SubscriptionId = subscription.Id,
            Status = subscription.Status,
            TrialEndsAt = subscription.TrialEndsAt
        };
    }

    // Upgrade / downgrade
    public async Task<SubscriptionResult> ChangePlanAsync(Guid subscriptionId, string newPlanId)
    {
        var subscription = await _subscriptionRepo.GetByIdAsync(subscriptionId);
        var newPlan = await _planRepository.GetByIdAsync(newPlanId);

        // Prorate: charge/credit the difference
        await _gateway.UpdateSubscriptionAsync(
            subscription.ProviderSubscriptionId, newPlan.ProviderPriceId);

        subscription.ChangePlan(newPlanId, newPlan.Name);
        await _subscriptionRepo.UpdateAsync(subscription);

        await _notifications.SendAsync(subscription.UserId,
            $"Your plan has been changed to {newPlan.Name}.");

        return new SubscriptionResult { SubscriptionId = subscription.Id, Status = "active" };
    }

    // Cancel at end of billing period
    public async Task CancelSubscriptionAsync(Guid subscriptionId, string reason)
    {
        var subscription = await _subscriptionRepo.GetByIdAsync(subscriptionId);

        await _gateway.CancelSubscriptionAsync(
            subscription.ProviderSubscriptionId, cancelImmediately: false);

        subscription.MarkForCancellation(reason);
        await _subscriptionRepo.UpdateAsync(subscription);
    }

    // Handle renewal webhook
    public async Task HandleRenewalAsync(string providerSubscriptionId, DateTime newPeriodEnd, string invoiceId)
    {
        var subscription = await _subscriptionRepo.GetByProviderIdAsync(providerSubscriptionId);

        subscription.Renew(newPeriodEnd);
        await _subscriptionRepo.UpdateAsync(subscription);

        _logger.LogInformation("Subscription {Id} renewed until {PeriodEnd}", subscription.Id, newPeriodEnd);
    }

    // Handle failed renewal (dunning)
    public async Task HandleRenewalFailureAsync(string providerSubscriptionId, int attemptCount)
    {
        var subscription = await _subscriptionRepo.GetByProviderIdAsync(providerSubscriptionId);

        if (attemptCount >= 3)
        {
            subscription.Suspend();
            await _notifications.SendAsync(subscription.UserId,
                "Your subscription has been suspended due to payment failure. Please update your payment method.");
        }
        else
        {
            await _notifications.SendAsync(subscription.UserId,
                $"Payment attempt {attemptCount} failed. We'll retry automatically. Please check your payment method.");
        }

        await _subscriptionRepo.UpdateAsync(subscription);
    }
}
```

---

## Marketplace / Split Payments

Split a single payment between a platform and one or more sellers.

```csharp
public class MarketplacePaymentService
{
    private readonly IPaymentGateway _gateway;
    private readonly ISellerRepository _sellerRepo;

    // Single seller: direct charge with platform fee
    public async Task<PaymentResult> ProcessDirectChargeAsync(MarketplaceOrderRequest request)
    {
        var seller = await _sellerRepo.GetByIdAsync(request.SellerId);
        var platformFee = CalculatePlatformFee(request.Amount, seller.CommissionRate);

        return await _gateway.CreateDirectChargeAsync(new MarketplacePaymentRequest
        {
            Amount = request.Amount,
            Currency = request.Currency,
            PaymentMethodId = request.PaymentMethodId,
            SellerStripeAccountId = seller.StripeConnectAccountId,
            PlatformFee = platformFee,
            IdempotencyKey = $"marketplace-{request.OrderId}"
        });
    }

    // Multi-seller: split payment across sellers
    public async Task<MultiSellerPaymentResult> ProcessMultiSellerPaymentAsync(MultiSellerOrderRequest request)
    {
        // 1. Charge the full amount on the platform
        var totalAmount = request.SellerItems.Sum(s => s.Amount);
        var chargeResult = await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            OrderId = request.OrderId,
            Amount = totalAmount,
            Currency = request.Currency,
            PaymentMethodId = request.PaymentMethodId
        });

        if (!chargeResult.IsSuccess)
            return MultiSellerPaymentResult.Failure(chargeResult.Error);

        // 2. Create transfers to each seller
        var transfers = new List<TransferResult>();
        foreach (var sellerItem in request.SellerItems)
        {
            var seller = await _sellerRepo.GetByIdAsync(sellerItem.SellerId);
            var platformFee = CalculatePlatformFee(sellerItem.Amount, seller.CommissionRate);
            var sellerAmount = sellerItem.Amount - platformFee;

            var transfer = await _gateway.CreateTransferAsync(new TransferRequest
            {
                Amount = sellerAmount,
                Currency = request.Currency,
                DestinationAccountId = seller.StripeConnectAccountId,
                SourceTransactionId = chargeResult.TransactionId,
                Metadata = new Dictionary<string, string>
                {
                    ["order_id"] = request.OrderId,
                    ["seller_id"] = sellerItem.SellerId.ToString()
                }
            });

            transfers.Add(transfer);
        }

        return new MultiSellerPaymentResult
        {
            IsSuccess = true,
            ChargeTransactionId = chargeResult.TransactionId,
            Transfers = transfers
        };
    }

    private decimal CalculatePlatformFee(decimal amount, decimal commissionRate)
    {
        return Math.Round(amount * commissionRate, 2);
    }
}
```

---

## Pre-Authorization and Capture

Authorize now, capture later (hotels, car rentals, pre-orders).

```csharp
public class PreAuthService
{
    private readonly IPaymentGateway _gateway;
    private readonly IAuthorizationRepository _authRepo;

    // Step 1: Authorize (hold funds)
    public async Task<AuthorizationResult> AuthorizeAsync(AuthorizationRequest request)
    {
        var result = await _gateway.AuthorizePaymentAsync(new PaymentRequest
        {
            OrderId = request.OrderId,
            Amount = request.Amount,
            Currency = request.Currency,
            PaymentMethodId = request.PaymentMethodId,
            CaptureMethod = "manual" // Don't capture yet
        });

        if (result.IsSuccess)
        {
            var auth = new Authorization
            {
                TransactionId = result.TransactionId,
                OrderId = request.OrderId,
                AuthorizedAmount = request.Amount,
                Currency = request.Currency,
                ExpiresAt = DateTime.UtcNow.AddDays(7), // Stripe: 7 days for most cards
                Status = AuthorizationStatus.Active
            };
            await _authRepo.AddAsync(auth);
        }

        return new AuthorizationResult
        {
            IsSuccess = result.IsSuccess,
            TransactionId = result.TransactionId,
            AuthorizedAmount = request.Amount,
            ExpiresAt = DateTime.UtcNow.AddDays(7)
        };
    }

    // Step 2: Capture (full or partial)
    public async Task<CaptureResult> CaptureAsync(string transactionId, decimal? captureAmount = null)
    {
        var auth = await _authRepo.GetByTransactionIdAsync(transactionId);

        if (auth.Status != AuthorizationStatus.Active)
            return CaptureResult.Failure("Authorization is not active");

        if (auth.ExpiresAt < DateTime.UtcNow)
            return CaptureResult.Failure("Authorization has expired");

        if (captureAmount.HasValue && captureAmount.Value > auth.AuthorizedAmount)
            return CaptureResult.Failure("Capture amount exceeds authorized amount");

        var result = await _gateway.CapturePaymentAsync(transactionId, captureAmount);

        auth.MarkAsCaptured(captureAmount ?? auth.AuthorizedAmount);
        await _authRepo.UpdateAsync(auth);

        return new CaptureResult
        {
            IsSuccess = result.IsSuccess,
            CapturedAmount = captureAmount ?? auth.AuthorizedAmount
        };
    }

    // Void (cancel the authorization)
    public async Task<VoidResult> VoidAuthorizationAsync(string transactionId)
    {
        var auth = await _authRepo.GetByTransactionIdAsync(transactionId);
        await _gateway.VoidPaymentAsync(transactionId);

        auth.MarkAsVoided();
        await _authRepo.UpdateAsync(auth);

        return VoidResult.Success();
    }

    // Background job: expire stale authorizations
    public async Task ExpireStaleAuthorizationsAsync()
    {
        var staleAuths = await _authRepo.GetExpiredAuthorizationsAsync();

        foreach (var auth in staleAuths)
        {
            try
            {
                await _gateway.VoidPaymentAsync(auth.TransactionId);
                auth.MarkAsExpired();
                await _authRepo.UpdateAsync(auth);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to void expired auth {TransactionId}", auth.TransactionId);
            }
        }
    }
}
```

---

## Invoice Payments

Generate invoices and collect payment.

```csharp
public class InvoiceService
{
    public async Task<InvoiceResult> CreateAndSendInvoiceAsync(InvoiceCreateRequest request)
    {
        // 1. Create invoice
        var invoice = Invoice.Create(
            customerId: request.CustomerId,
            dueDate: request.DueDate ?? DateTime.UtcNow.AddDays(30),
            items: request.LineItems,
            currency: request.Currency);

        // 2. Generate payment link
        var paymentLink = await _gateway.CreatePaymentLinkAsync(new PaymentLinkRequest
        {
            Amount = invoice.Total,
            Currency = invoice.Currency,
            Description = $"Invoice #{invoice.Number}",
            Metadata = new Dictionary<string, string>
            {
                ["invoice_id"] = invoice.Id.ToString()
            },
            ExpiresAt = invoice.DueDate.AddDays(7)
        });

        invoice.SetPaymentLink(paymentLink.Url);
        await _invoiceRepository.AddAsync(invoice);

        // 3. Send invoice email
        await _emailService.SendInvoiceAsync(new InvoiceEmail
        {
            To = request.CustomerEmail,
            InvoiceNumber = invoice.Number,
            Amount = invoice.Total,
            DueDate = invoice.DueDate,
            PaymentUrl = paymentLink.Url,
            LineItems = invoice.Items
        });

        return new InvoiceResult
        {
            InvoiceId = invoice.Id,
            InvoiceNumber = invoice.Number,
            PaymentUrl = paymentLink.Url
        };
    }

    // Background job: send reminders for overdue invoices
    public async Task SendOverdueRemindersAsync()
    {
        var overdueInvoices = await _invoiceRepository.GetOverdueAsync();

        foreach (var invoice in overdueInvoices)
        {
            var daysPastDue = (DateTime.UtcNow - invoice.DueDate).Days;

            if (daysPastDue == 1 || daysPastDue == 7 || daysPastDue == 14)
            {
                await _emailService.SendOverdueReminderAsync(invoice, daysPastDue);
                _logger.LogInformation("Overdue reminder sent for invoice {InvoiceNumber}, {Days} days past due",
                    invoice.Number, daysPastDue);
            }
        }
    }
}
```

---

## Pay-What-You-Want / Donations

```csharp
public class DonationService
{
    public async Task<PaymentResult> ProcessDonationAsync(DonationRequest request)
    {
        // Validate minimum amount
        if (request.Amount < 1.00m)
            return PaymentResult.Failure(new PaymentError { Code = "MIN_AMOUNT", Message = "Minimum donation is $1.00" });

        // Validate maximum
        if (request.Amount > 100_000m)
            return PaymentResult.Failure(new PaymentError { Code = "MAX_AMOUNT", Message = "Maximum donation is $100,000" });

        var result = await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            Amount = request.Amount,
            Currency = request.Currency ?? "USD",
            PaymentMethodId = request.PaymentMethodId,
            CustomerEmail = request.DonorEmail,
            Metadata = new Dictionary<string, string>
            {
                ["type"] = "donation",
                ["campaign"] = request.CampaignId,
                ["donor_name"] = request.DonorName,
                ["is_anonymous"] = request.IsAnonymous.ToString()
            },
            StatementDescriptor = "CHARITY DONATION"
        });

        if (result.IsSuccess)
        {
            // Generate tax receipt
            await _receiptService.GenerateDonationReceiptAsync(
                request.DonorEmail, request.Amount, request.Currency, result.TransactionId);
        }

        return result;
    }
}
```

---

## Multi-Currency Checkout

```csharp
public class MultiCurrencyCheckoutService
{
    private readonly ICurrencyConverter _currencyConverter;

    public async Task<CheckoutResult> ProcessMultiCurrencyCheckoutAsync(CheckoutRequest request)
    {
        // 1. Determine customer's preferred currency
        var customerCurrency = request.PreferredCurrency
            ?? await _geoService.DetectCurrencyAsync(request.IpAddress)
            ?? "USD";

        // 2. Convert prices if needed
        var baseTotal = request.CartItems.Sum(i => i.Price * i.Quantity);
        decimal displayTotal;
        decimal exchangeRate = 1m;

        if (customerCurrency != request.BaseCurrency)
        {
            var conversion = await _currencyConverter.ConvertAsync(
                baseTotal, request.BaseCurrency, customerCurrency);
            displayTotal = conversion.ConvertedAmount;
            exchangeRate = conversion.ExchangeRate;
        }
        else
        {
            displayTotal = baseTotal;
        }

        // 3. Process in customer's currency
        var result = await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            Amount = displayTotal,
            Currency = customerCurrency,
            PaymentMethodId = request.PaymentMethodId,
            Metadata = new Dictionary<string, string>
            {
                ["base_currency"] = request.BaseCurrency,
                ["base_amount"] = baseTotal.ToString("F2"),
                ["exchange_rate"] = exchangeRate.ToString("F6"),
                ["conversion_timestamp"] = DateTime.UtcNow.ToString("O")
            }
        });

        return new CheckoutResult
        {
            IsSuccess = result.IsSuccess,
            ChargedAmount = displayTotal,
            ChargedCurrency = customerCurrency,
            ExchangeRate = exchangeRate
        };
    }
}
```

---

## Saved Payment Methods

```csharp
public class SavedPaymentMethodService
{
    // Save card for future use (with customer consent)
    public async Task<SavedMethodResult> SavePaymentMethodAsync(SaveMethodRequest request)
    {
        // 1. Create/retrieve customer in provider
        var customerId = await _gateway.GetOrCreateCustomerAsync(request.CustomerEmail);

        // 2. Attach payment method
        await _gateway.AttachPaymentMethodAsync(customerId, request.PaymentMethodId);

        // 3. Verify via $0 auth (optional but recommended)
        var verifyResult = await _gateway.VerifyPaymentMethodAsync(customerId, request.PaymentMethodId);

        if (!verifyResult.IsSuccess)
            return SavedMethodResult.Failure("Card verification failed");

        // 4. Store reference locally (not the card data — that's in the provider)
        var savedMethod = new SavedPaymentMethod
        {
            CustomerId = request.InternalCustomerId,
            ProviderCustomerId = customerId,
            ProviderPaymentMethodId = request.PaymentMethodId,
            CardBrand = verifyResult.CardBrand,
            Last4 = verifyResult.Last4,
            ExpMonth = verifyResult.ExpMonth,
            ExpYear = verifyResult.ExpYear,
            IsDefault = request.SetAsDefault
        };
        await _savedMethodRepo.AddAsync(savedMethod);

        return SavedMethodResult.Success(savedMethod);
    }

    // Pay with saved method (one-click checkout)
    public async Task<PaymentResult> PayWithSavedMethodAsync(string internalCustomerId, string orderId, decimal amount, string currency)
    {
        var savedMethod = await _savedMethodRepo.GetDefaultAsync(internalCustomerId);

        if (savedMethod == null)
            return PaymentResult.Failure(new PaymentError { Code = "NO_SAVED_METHOD", Message = "No saved payment method found" });

        return await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            OrderId = orderId,
            Amount = amount,
            Currency = currency,
            PaymentMethodId = savedMethod.ProviderPaymentMethodId,
            CustomerId = savedMethod.ProviderCustomerId,
            OffSession = true // Customer is not present — must handle auth failures
        });
    }
}
```

---

## Partial Payments and Installments

```csharp
public class InstallmentService
{
    public async Task<InstallmentPlanResult> CreateInstallmentPlanAsync(InstallmentRequest request)
    {
        var installmentAmount = Math.Round(request.TotalAmount / request.NumberOfInstallments, 2);
        var adjustmentOnLast = request.TotalAmount - (installmentAmount * request.NumberOfInstallments);

        var plan = new InstallmentPlan
        {
            OrderId = request.OrderId,
            TotalAmount = request.TotalAmount,
            Currency = request.Currency,
            NumberOfInstallments = request.NumberOfInstallments,
            Installments = Enumerable.Range(1, request.NumberOfInstallments)
                .Select(i => new Installment
                {
                    Number = i,
                    Amount = i == request.NumberOfInstallments
                        ? installmentAmount + adjustmentOnLast
                        : installmentAmount,
                    DueDate = DateTime.UtcNow.AddMonths(i - 1),
                    Status = i == 1 ? InstallmentStatus.Due : InstallmentStatus.Scheduled
                }).ToList()
        };

        await _installmentRepo.AddAsync(plan);

        // Process first installment immediately
        var firstInstallment = plan.Installments.First();
        var result = await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            OrderId = $"{request.OrderId}-inst-1",
            Amount = firstInstallment.Amount,
            Currency = request.Currency,
            PaymentMethodId = request.PaymentMethodId
        });

        if (result.IsSuccess)
        {
            firstInstallment.MarkAsPaid(result.TransactionId);
            await _installmentRepo.UpdateAsync(plan);
        }

        return new InstallmentPlanResult
        {
            PlanId = plan.Id,
            FirstPaymentSuccess = result.IsSuccess,
            Schedule = plan.Installments.Select(i => new { i.Number, i.Amount, i.DueDate }).ToList()
        };
    }

    // Background job: process due installments
    public async Task ProcessDueInstallmentsAsync()
    {
        var dueInstallments = await _installmentRepo.GetDueInstallmentsAsync();

        foreach (var (plan, installment) in dueInstallments)
        {
            var savedMethod = await _savedMethodRepo.GetDefaultAsync(plan.CustomerId);
            var result = await _gateway.ProcessPaymentAsync(new PaymentRequest
            {
                OrderId = $"{plan.OrderId}-inst-{installment.Number}",
                Amount = installment.Amount,
                Currency = plan.Currency,
                PaymentMethodId = savedMethod.ProviderPaymentMethodId,
                OffSession = true
            });

            if (result.IsSuccess)
            {
                installment.MarkAsPaid(result.TransactionId);
            }
            else
            {
                installment.MarkAsFailed(result.Error?.Message);
                await _notifications.SendPaymentFailedAsync(plan.CustomerId, installment);
            }

            await _installmentRepo.UpdateAsync(plan);
        }
    }
}
```

---

## Handling Failed Payments

### Retry and Dunning Flow

```csharp
public class DunningService
{
    private readonly int[] _retryScheduleDays = { 1, 3, 7, 14 };

    public async Task HandleFailedPaymentAsync(string paymentId, int attemptNumber)
    {
        var payment = await _paymentRepo.GetByIdAsync(paymentId);

        if (attemptNumber > _retryScheduleDays.Length)
        {
            // Final failure — suspend service
            await SuspendServiceAsync(payment);
            return;
        }

        // Schedule retry
        var retryDelay = TimeSpan.FromDays(_retryScheduleDays[attemptNumber - 1]);

        await _backgroundJobs.ScheduleAsync(
            () => RetryPaymentAsync(paymentId, attemptNumber + 1),
            retryDelay);

        // Notify customer
        await _notifications.SendAsync(payment.CustomerId, new FailedPaymentNotification
        {
            AttemptNumber = attemptNumber,
            NextRetryDate = DateTime.UtcNow.Add(retryDelay),
            UpdatePaymentUrl = $"{_settings.BaseUrl}/account/payment-methods",
            AmountDue = payment.Amount,
            Currency = payment.Currency
        });
    }

    private async Task SuspendServiceAsync(Payment payment)
    {
        await _subscriptionService.SuspendAsync(payment.SubscriptionId);

        await _notifications.SendAsync(payment.CustomerId, new ServiceSuspendedNotification
        {
            Reason = "Payment failed after multiple attempts",
            ReactivateUrl = $"{_settings.BaseUrl}/account/reactivate"
        });

        _logger.LogWarning("Service suspended for customer {CustomerId} due to payment failure",
            payment.CustomerId);
    }
}
```

---

## Scenario Quick Reference

| Scenario | Key Pattern | Provider Feature |
|----------|------------|-----------------|
| Cart checkout | Single charge | Payment Intent / Order |
| Subscription | Recurring billing | Stripe Subscriptions / PayPal Plans |
| Marketplace | Split payments | Stripe Connect / PayPal Commerce Platform |
| Pre-auth + capture | Two-step capture | `capture_method: manual` |
| Invoicing | Payment links | Stripe Invoices / PayPal Invoicing |
| Donations | Flexible amounts | Custom amount + metadata |
| Multi-currency | FX conversion | Presentment currency |
| One-click pay | Saved methods | Customer + PaymentMethod |
| Installments | Scheduled charges | Off-session payments |
| Failed payments | Dunning | Smart retries + notifications |

## Next Steps

- [Stripe Integration](16-STRIPE-INTEGRATION.md)
- [PayPal Integration](17-PAYPAL-INTEGRATION.md)
- [Error Handling](05-ERROR-HANDLING.md)
- [Webhook Patterns](04-WEBHOOK-PATTERNS.md)
