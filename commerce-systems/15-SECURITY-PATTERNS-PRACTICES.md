# Security Patterns and Best Practices for Payment Systems

## Overview

Comprehensive guide to security patterns, best practices, and compliance requirements for building secure payment systems. Covers PCI DSS compliance, encryption, authentication, input validation, fraud prevention, and security monitoring.

> **Note**: This document consolidates all security guidance. Previously separate content from the earlier "Security Best Practices" guide has been merged here.

## Table of Contents

1. [PCI DSS Compliance](#pci-dss-compliance)
2. [Encryption Patterns](#encryption-patterns)
3. [Authentication & Authorization](#authentication-authorization)
4. [Secure Communication](#secure-communication)
5. [Data Protection](#data-protection)
6. [Fraud Prevention](#fraud-prevention)
7. [API Security](#api-security)
8. [Security Monitoring](#security-monitoring)
9. [Security Checklist](#security-checklist)

---

## PCI DSS Compliance

### PCI DSS Requirements Overview

```
PCI DSS v4.0 - 12 Requirements

Build and Maintain Secure Network and Systems:
1. Install and maintain network security controls
2. Apply secure configurations to all system components

Protect Cardholder Data:
3. Protect stored account data
4. Protect cardholder data with strong cryptography during transmission

Maintain Vulnerability Management Program:
5. Protect all systems and networks from malicious software
6. Develop and maintain secure systems and software

Implement Strong Access Control Measures:
7. Restrict access to system components and cardholder data by business need to know
8. Identify users and authenticate access to system components
9. Restrict physical access to cardholder data

Regularly Monitor and Test Networks:
10. Log and monitor all access to system components and cardholder data
11. Test security of systems and networks regularly

Maintain Information Security Policy:
12. Support information security with organizational policies and programs
```

### SAQ Levels

```csharp
public enum PCIComplianceLevel
{
    // Never store, process, or transmit cardholder data
    // Use hosted payment pages (Stripe Checkout, PayPal)
    SAQA,           // Easiest - Merchants fully outsourcing
    
    // Store card data on payment provider's servers
    // Use tokenization/vaulting services
    SAQA_EP,        // E-commerce using payment page redirect
    
    // Process cards via terminal/POS
    SAQB,           // Imprint-only or standalone terminal
    
    // Store card data in your environment
    SAQC,           // Payment application systems connected to internet
    
    // Full assessment required
    SAQD_Merchant,  // All other merchants - most stringent
    SAQD_ServiceProvider  // Service providers
}
```

### Achieving SAQ-A Compliance

```csharp
/// <summary>
/// SAQ-A is the easiest compliance level - never touch card data
/// </summary>
public class SAQACompliantPaymentService
{
    private readonly IConfiguration _configuration;
    
    // ✅ CORRECT: Never handle raw card data
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Redirect to Stripe Checkout (hosted payment page)
        var sessionOptions = new SessionCreateOptions
        {
            PaymentMethodTypes = new List<string> { "card" },
            LineItems = new List<SessionLineItemOptions>
            {
                new SessionLineItemOptions
                {
                    PriceData = new SessionLineItemPriceDataOptions
                    {
                        Currency = "usd",
                        UnitAmount = (long)(request.Amount * 100),
                        ProductData = new SessionLineItemPriceDataProductDataOptions
                        {
                            Name = request.ProductName
                        }
                    },
                    Quantity = 1
                }
            },
            Mode = "payment",
            SuccessUrl = "https://yoursite.com/success",
            CancelUrl = "https://yoursite.com/cancel"
        };
        
        var service = new SessionService();
        var session = await service.CreateAsync(sessionOptions);
        
        // Return redirect URL - card data never touches your server
        return new PaymentResult
        {
            CheckoutUrl = session.Url,
            SessionId = session.Id
        };
    }
    
    // ❌ WRONG: Never do this - stores card data
    public class WrongPaymentRequest
    {
        public string CardNumber { get; set; }     // ❌ NEVER
        public string CVV { get; set; }            // ❌ NEVER
        public string ExpiryDate { get; set; }     // ❌ NEVER
    }
    
    // ✅ CORRECT: Only store tokens
    public class CorrectPaymentRequest
    {
        public string PaymentMethodId { get; set; }  // ✅ Token from provider
        public decimal Amount { get; set; }
        public string Currency { get; set; }
    }
}
```

### PCI DSS Requirement 3: Protect Stored Data

```csharp
/// <summary>
/// What you can and cannot store
/// </summary>
public class PCIDataStorage
{
    // ❌ NEVER STORE - Prohibited under any circumstances
    public class ProhibitedData
    {
        public string FullMagneticStripeData { get; set; }  // ❌ NEVER
        public string CVV { get; set; }                     // ❌ NEVER (CVC/CVV2/CID)
        public string PIN { get; set; }                     // ❌ NEVER
        public string PINBlock { get; set; }                // ❌ NEVER
    }
    
    // ⚠️ CAN STORE - But must be encrypted if stored
    public class AllowedData
    {
        // Primary Account Number (PAN) - must be encrypted
        public string TokenizedPAN { get; set; }            // ✅ Token from provider
        
        // Cardholder Name - can be stored
        public string CardholderName { get; set; }          // ✅ OK
        
        // Service Code - must be encrypted
        public string ServiceCode { get; set; }             // ⚠️ Encrypted
        
        // Expiration Date - must be encrypted
        public string ExpirationDate { get; set; }          // ⚠️ Encrypted
        
        // Last 4 digits - can be stored unencrypted
        public string Last4Digits { get; set; }             // ✅ OK
        
        // First 6 digits (BIN) - can be stored unencrypted
        public string BIN { get; set; }                     // ✅ OK
    }
    
    // ✅ BEST PRACTICE: Store only tokens
    public class RecommendedApproach
    {
        public string PaymentMethodId { get; set; }     // Token from Stripe/PayPal
        public string Last4 { get; set; }               // For display
        public string CardBrand { get; set; }           // Visa/MC/Amex
        public string ExpiryMonth { get; set; }         // For display
        public string ExpiryYear { get; set; }          // For display
        public string CardholderName { get; set; }      // For receipts
    }
}

/// <summary>
/// Database schema following PCI DSS
/// </summary>
public class PCICompliantDatabase
{
    public static void ConfigureDatabase(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Payment>(entity =>
        {
            entity.ToTable("Payments");
            
            // ✅ CORRECT: Store only tokens and safe data
            entity.Property(e => e.ProviderPaymentMethodId)
                .HasMaxLength(200)
                .IsRequired();
            
            entity.Property(e => e.Last4Digits)
                .HasMaxLength(4);
            
            entity.Property(e => e.CardBrand)
                .HasMaxLength(50);
            
            // ❌ NEVER add these columns:
            // CardNumber, CVV, MagneticStripeData, PIN
        });
    }
}
```

---

## Encryption Patterns

### Data at Rest Encryption

```csharp
using System.Security.Cryptography;
using Microsoft.EntityFrameworkCore;

/// <summary>
/// AES-256 encryption for sensitive data at rest
/// </summary>
public class DataEncryptionService
{
    private readonly byte[] _encryptionKey;
    
    public DataEncryptionService(IConfiguration configuration)
    {
        // Key should be stored in Azure Key Vault, AWS KMS, etc.
        var keyString = configuration["Encryption:MasterKey"];
        _encryptionKey = Convert.FromBase64String(keyString);
    }
    
    public string Encrypt(string plainText)
    {
        if (string.IsNullOrEmpty(plainText))
            return plainText;
        
        using var aes = Aes.Create();
        aes.Key = _encryptionKey;
        aes.Mode = CipherMode.CBC;
        aes.Padding = PaddingMode.PKCS7;
        aes.GenerateIV();
        
        using var encryptor = aes.CreateEncryptor();
        using var msEncrypt = new MemoryStream();
        
        // Prepend IV to ciphertext
        msEncrypt.Write(aes.IV, 0, aes.IV.Length);
        
        using (var csEncrypt = new CryptoStream(msEncrypt, encryptor, CryptoStreamMode.Write))
        using (var swEncrypt = new StreamWriter(csEncrypt))
        {
            swEncrypt.Write(plainText);
        }
        
        return Convert.ToBase64String(msEncrypt.ToArray());
    }
    
    public string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText))
            return cipherText;
        
        var fullCipher = Convert.FromBase64String(cipherText);
        
        using var aes = Aes.Create();
        aes.Key = _encryptionKey;
        aes.Mode = CipherMode.CBC;
        aes.Padding = PaddingMode.PKCS7;
        
        // Extract IV from beginning of ciphertext
        var iv = new byte[aes.IV.Length];
        var cipher = new byte[fullCipher.Length - iv.Length];
        
        Array.Copy(fullCipher, iv, iv.Length);
        Array.Copy(fullCipher, iv.Length, cipher, 0, cipher.Length);
        
        aes.IV = iv;
        
        using var decryptor = aes.CreateDecryptor();
        using var msDecrypt = new MemoryStream(cipher);
        using var csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read);
        using var srDecrypt = new StreamReader(csDecrypt);
        
        return srDecrypt.ReadToEnd();
    }
}

/// <summary>
/// Entity Framework encryption/decryption
/// </summary>
public class EncryptedValueConverter : ValueConverter<string, string>
{
    public EncryptedValueConverter(DataEncryptionService encryption)
        : base(
            v => encryption.Encrypt(v),
            v => encryption.Decrypt(v))
    {
    }
}

// Usage in DbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var encryptionService = new DataEncryptionService(_configuration);
    var converter = new EncryptedValueConverter(encryptionService);
    
    modelBuilder.Entity<Customer>()
        .Property(e => e.TaxId)
        .HasConversion(converter);
}
```

### Key Management

```csharp
using Azure.Security.KeyVault.Secrets;
using Azure.Identity;

/// <summary>
/// Azure Key Vault integration for key management
/// </summary>
public class KeyVaultService
{
    private readonly SecretClient _client;
    private readonly IMemoryCache _cache;
    
    public KeyVaultService(IConfiguration configuration, IMemoryCache cache)
    {
        var keyVaultUrl = configuration["KeyVault:Url"];
        _client = new SecretClient(
            new Uri(keyVaultUrl),
            new DefaultAzureCredential());
        _cache = cache;
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        // Check cache first
        if (_cache.TryGetValue(secretName, out string cachedValue))
        {
            return cachedValue;
        }
        
        // Retrieve from Key Vault
        var secret = await _client.GetSecretAsync(secretName);
        
        // Cache for 1 hour
        _cache.Set(secretName, secret.Value.Value, TimeSpan.FromHours(1));
        
        return secret.Value.Value;
    }
    
    public async Task<byte[]> GetEncryptionKeyAsync()
    {
        var keyBase64 = await GetSecretAsync("encryption-master-key");
        return Convert.FromBase64String(keyBase64);
    }
}

/// <summary>
/// AWS KMS integration
/// </summary>
public class KMSEncryptionService
{
    private readonly IAmazonKeyManagementService _kms;
    private readonly string _keyId;
    
    public KMSEncryptionService(IAmazonKeyManagementService kms, IConfiguration config)
    {
        _kms = kms;
        _keyId = config["AWS:KMS:KeyId"];
    }
    
    public async Task<string> EncryptAsync(string plaintext)
    {
        var request = new EncryptRequest
        {
            KeyId = _keyId,
            Plaintext = new MemoryStream(Encoding.UTF8.GetBytes(plaintext))
        };
        
        var response = await _kms.EncryptAsync(request);
        return Convert.ToBase64String(response.CiphertextBlob.ToArray());
    }
    
    public async Task<string> DecryptAsync(string ciphertext)
    {
        var request = new DecryptRequest
        {
            CiphertextBlob = new MemoryStream(Convert.FromBase64String(ciphertext))
        };
        
        var response = await _kms.DecryptAsync(request);
        return Encoding.UTF8.GetString(response.Plaintext.ToArray());
    }
}
```

### Field-Level Encryption

```csharp
/// <summary>
/// Attribute-based field-level encryption
/// </summary>
[AttributeUsage(AttributeTargets.Property)]
public class EncryptedAttribute : Attribute { }

public class Customer
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    
    [Encrypted]
    public string TaxId { get; set; }
    
    [Encrypted]
    public string BankAccount { get; set; }
    
    [Encrypted]
    public string DateOfBirth { get; set; }
}

/// <summary>
/// Interceptor for automatic encryption/decryption
/// </summary>
public class EncryptionInterceptor : SaveChangesInterceptor
{
    private readonly DataEncryptionService _encryption;
    
    public EncryptionInterceptor(DataEncryptionService encryption)
    {
        _encryption = encryption;
    }
    
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context is null)
            return base.SavingChangesAsync(eventData, result, cancellationToken);
        
        var entries = eventData.Context.ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified);
        
        foreach (var entry in entries)
        {
            var encryptedProperties = entry.Entity.GetType()
                .GetProperties()
                .Where(p => p.GetCustomAttribute<EncryptedAttribute>() != null);
            
            foreach (var property in encryptedProperties)
            {
                var value = property.GetValue(entry.Entity) as string;
                if (!string.IsNullOrEmpty(value))
                {
                    var encrypted = _encryption.Encrypt(value);
                    property.SetValue(entry.Entity, encrypted);
                }
            }
        }
        
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
    
    public override ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context is null)
            return base.SavedChangesAsync(eventData, result, cancellationToken);
        
        var entries = eventData.Context.ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Unchanged);
        
        foreach (var entry in entries)
        {
            var encryptedProperties = entry.Entity.GetType()
                .GetProperties()
                .Where(p => p.GetCustomAttribute<EncryptedAttribute>() != null);
            
            foreach (var property in encryptedProperties)
            {
                var value = property.GetValue(entry.Entity) as string;
                if (!string.IsNullOrEmpty(value))
                {
                    var decrypted = _encryption.Decrypt(value);
                    property.SetValue(entry.Entity, decrypted);
                }
            }
        }
        
        return base.SavedChangesAsync(eventData, result, cancellationToken);
    }
}
```

---

## Authentication & Authorization

### Multi-Factor Authentication (MFA)

```csharp
using OtpNet;
using QRCoder;

/// <summary>
/// TOTP-based 2FA implementation
/// </summary>
public class TwoFactorAuthService
{
    public (string Secret, string QRCodeUri) GenerateSecret(string userEmail)
    {
        // Generate random secret
        var secretKey = KeyGeneration.GenerateRandomKey(20);
        var base32Secret = Base32Encoding.ToString(secretKey);
        
        // Generate QR code URI
        var issuer = "YourPaymentApp";
        var label = Uri.EscapeDataString($"{issuer}:{userEmail}");
        var qrCodeUri = $"otpauth://totp/{label}?secret={base32Secret}&issuer={issuer}";
        
        return (base32Secret, qrCodeUri);
    }
    
    public byte[] GenerateQRCode(string qrCodeUri)
    {
        using var qrGenerator = new QRCodeGenerator();
        using var qrCodeData = qrGenerator.CreateQrCode(qrCodeUri, QRCodeGenerator.ECCLevel.Q);
        using var qrCode = new PngByteQRCode(qrCodeData);
        return qrCode.GetGraphic(20);
    }
    
    public bool ValidateCode(string secret, string code)
    {
        var secretBytes = Base32Encoding.ToBytes(secret);
        var totp = new Totp(secretBytes);
        
        // Allow 1 step before and after (30 second window each way)
        return totp.VerifyTotp(code, out _, new VerificationWindow(1, 1));
    }
}

/// <summary>
/// SMS-based 2FA implementation
/// </summary>
public class SMSTwoFactorService
{
    private readonly ITwilioService _twilio;
    private readonly IDistributedCache _cache;
    
    public async Task<string> SendCodeAsync(string phoneNumber)
    {
        // Generate 6-digit code
        var code = Random.Shared.Next(100000, 999999).ToString();
        
        // Store in cache with 5-minute expiration
        var cacheKey = $"2fa:{phoneNumber}";
        await _cache.SetStringAsync(
            cacheKey,
            code,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });
        
        // Send SMS
        await _twilio.SendSMSAsync(
            phoneNumber,
            $"Your verification code is: {code}. Valid for 5 minutes.");
        
        return code;
    }
    
    public async Task<bool> ValidateCodeAsync(string phoneNumber, string code)
    {
        var cacheKey = $"2fa:{phoneNumber}";
        var stored = await _cache.GetStringAsync(cacheKey);
        
        if (stored == null)
            return false;
        
        if (stored != code)
            return false;
        
        // Remove code after successful validation
        await _cache.RemoveAsync(cacheKey);
        
        return true;
    }
}
```

### Role-Based Access Control (RBAC)

```csharp
/// <summary>
/// Payment-specific roles and permissions
/// </summary>
public static class PaymentRoles
{
    public const string Admin = "PaymentAdmin";
    public const string Manager = "PaymentManager";
    public const string Processor = "PaymentProcessor";
    public const string Viewer = "PaymentViewer";
    public const string Auditor = "PaymentAuditor";
}

public static class PaymentPermissions
{
    // Read permissions
    public const string ViewPayments = "payments:view";
    public const string ViewCustomers = "customers:view";
    public const string ViewReports = "reports:view";
    
    // Write permissions
    public const string ProcessPayments = "payments:process";
    public const string RefundPayments = "payments:refund";
    public const string CancelPayments = "payments:cancel";
    
    // Admin permissions
    public const string ManageUsers = "users:manage";
    public const string ConfigureSettings = "settings:configure";
    public const string ViewAuditLogs = "audit:view";
    
    // Sensitive operations
    public const string ExportData = "data:export";
    public const string ViewPII = "pii:view";
}

/// <summary>
/// Policy-based authorization
/// </summary>
public static class AuthorizationPolicies
{
    public static void ConfigurePolicies(AuthorizationOptions options)
    {
        options.AddPolicy("CanProcessPayments", policy =>
            policy.RequireAssertion(context =>
                context.User.HasClaim(c => c.Type == "permission" && 
                    c.Value == PaymentPermissions.ProcessPayments)));
        
        options.AddPolicy("CanRefundPayments", policy =>
            policy.RequireAssertion(context =>
                context.User.HasClaim(c => c.Type == "permission" && 
                    c.Value == PaymentPermissions.RefundPayments) &&
                context.User.HasClaim(c => c.Type == "mfa_verified" && 
                    c.Value == "true")));
        
        options.AddPolicy("CanViewPII", policy =>
            policy.RequireAssertion(context =>
                context.User.HasClaim(c => c.Type == "permission" && 
                    c.Value == PaymentPermissions.ViewPII) &&
                context.User.HasClaim(c => c.Type == "mfa_verified" && 
                    c.Value == "true")));
    }
}

/// <summary>
/// Usage in controllers
/// </summary>
[Authorize(Policy = "CanProcessPayments")]
[HttpPost("api/payments")]
public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequest request)
{
    // Only users with ProcessPayments permission can access
    var result = await _paymentService.ProcessAsync(request);
    return Ok(result);
}

[Authorize(Policy = "CanRefundPayments")]
[RequireMFA] // Custom attribute requiring MFA
[HttpPost("api/refunds")]
public async Task<IActionResult> RefundPayment([FromBody] RefundRequest request)
{
    // Requires RefundPayments permission AND active MFA
    var result = await _refundService.ProcessAsync(request);
    return Ok(result);
}
```

### API Key Management

```csharp
/// <summary>
/// Secure API key generation and validation
/// </summary>
public class APIKeyService
{
    private readonly ApplicationDbContext _context;
    
    public async Task<APIKey> GenerateKeyAsync(string userId, string name, DateTime? expiresAt = null)
    {
        // Generate cryptographically secure key
        var keyBytes = new byte[32];
        using (var rng = RandomNumberGenerator.Create())
        {
            rng.GetBytes(keyBytes);
        }
        
        var key = Convert.ToBase64String(keyBytes)
            .Replace("+", "-")
            .Replace("/", "_")
            .Replace("=", "");
        
        var prefix = "pk_live_";
        var apiKey = new APIKey
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Name = name,
            KeyHash = HashKey(key),
            KeyPrefix = prefix,
            ExpiresAt = expiresAt,
            CreatedAt = DateTime.UtcNow,
            LastUsedAt = null
        };
        
        _context.APIKeys.Add(apiKey);
        await _context.SaveChangesAsync();
        
        // Return full key only once (can't be retrieved later)
        apiKey.FullKey = $"{prefix}{key}";
        return apiKey;
    }
    
    public async Task<APIKeyValidationResult> ValidateKeyAsync(string key)
    {
        if (string.IsNullOrEmpty(key))
            return APIKeyValidationResult.Invalid("No API key provided");
        
        // Extract prefix
        var parts = key.Split('_', 3);
        if (parts.Length != 3)
            return APIKeyValidationResult.Invalid("Invalid API key format");
        
        var prefix = $"{parts[0]}_{parts[1]}_";
        var keyHash = HashKey(parts[2]);
        
        // Find key in database
        var apiKey = await _context.APIKeys
            .FirstOrDefaultAsync(k => k.KeyPrefix == prefix && k.KeyHash == keyHash);
        
        if (apiKey == null)
            return APIKeyValidationResult.Invalid("API key not found");
        
        if (apiKey.RevokedAt != null)
            return APIKeyValidationResult.Invalid("API key has been revoked");
        
        if (apiKey.ExpiresAt != null && apiKey.ExpiresAt < DateTime.UtcNow)
            return APIKeyValidationResult.Invalid("API key has expired");
        
        // Update last used
        apiKey.LastUsedAt = DateTime.UtcNow;
        await _context.SaveChangesAsync();
        
        return APIKeyValidationResult.Valid(apiKey);
    }
    
    private string HashKey(string key)
    {
        using var sha256 = SHA256.Create();
        var hashBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(key));
        return Convert.ToBase64String(hashBytes);
    }
}

/// <summary>
/// API Key authentication middleware
/// </summary>
public class APIKeyAuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context, APIKeyService apiKeyService)
    {
        // Check for API key in header
        if (!context.Request.Headers.TryGetValue("X-API-Key", out var apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new { error = "API key required" });
            return;
        }
        
        // Validate key
        var validation = await apiKeyService.ValidateKeyAsync(apiKey);
        
        if (!validation.IsValid)
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new { error = validation.ErrorMessage });
            return;
        }
        
        // Add key info to context
        context.Items["APIKey"] = validation.APIKey;
        context.Items["UserId"] = validation.APIKey.UserId;
        
        await _next(context);
    }
}
```

---

## Secure Communication

### TLS/SSL Configuration

```csharp
/// <summary>
/// Enforce TLS 1.2+ and secure ciphers
/// </summary>
public static class SecureHttpClientConfiguration
{
    public static IHttpClientBuilder AddSecureHttpClient(
        this IServiceCollection services,
        string name)
    {
        return services.AddHttpClient(name)
            .ConfigurePrimaryHttpMessageHandler(() =>
            {
                var handler = new HttpClientHandler
                {
                    // Enforce TLS 1.2 or higher
                    SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13,
                    
                    // Validate server certificate
                    ServerCertificateCustomValidationCallback = (message, cert, chain, errors) =>
                    {
                        if (errors == SslPolicyErrors.None)
                            return true;
                        
                        // Log certificate errors
                        var logger = LoggerFactory.Create(builder => builder.AddConsole())
                            .CreateLogger("SSL");
                        logger.LogError($"SSL certificate error: {errors}");
                        
                        return false;
                    }
                };
                
                return handler;
            });
    }
}

/// <summary>
/// Kestrel HTTPS configuration
/// </summary>
public static class SecureKestrelConfiguration
{
    public static void ConfigureKestrel(WebApplicationBuilder builder)
    {
        builder.WebHost.ConfigureKestrel(options =>
        {
            options.ConfigureHttpsDefaults(httpsOptions =>
            {
                // Enforce TLS 1.2+
                httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
                
                // Certificate configuration
                httpsOptions.ServerCertificate = LoadCertificate();
                
                // Client certificate validation (for mutual TLS)
                httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
                httpsOptions.ClientCertificateValidation = (cert, chain, errors) =>
                {
                    // Custom validation logic
                    return errors == SslPolicyErrors.None;
                };
            });
        });
    }
    
    private static X509Certificate2 LoadCertificate()
    {
        // Load from Azure Key Vault, AWS Certificate Manager, etc.
        return new X509Certificate2("path/to/cert.pfx", "password");
    }
}
```

### Certificate Pinning

```csharp
/// <summary>
/// HTTP client with certificate pinning
/// </summary>
public class CertificatePinningHandler : HttpClientHandler
{
    private readonly HashSet<string> _trustedFingerprints;
    
    public CertificatePinningHandler(params string[] trustedFingerprints)
    {
        _trustedFingerprints = new HashSet<string>(trustedFingerprints);
        
        ServerCertificateCustomValidationCallback = ValidateCertificate;
    }
    
    private bool ValidateCertificate(
        HttpRequestMessage request,
        X509Certificate2 certificate,
        X509Chain chain,
        SslPolicyErrors errors)
    {
        // First, check standard validation
        if (errors != SslPolicyErrors.None &&
            errors != SslPolicyErrors.RemoteCertificateChainErrors)
        {
            return false;
        }
        
        // Then check certificate pinning
        var fingerprint = certificate.GetCertHashString(HashAlgorithmName.SHA256);
        
        if (!_trustedFingerprints.Contains(fingerprint))
        {
            var logger = LoggerFactory.Create(b => b.AddConsole())
                .CreateLogger("CertPin");
            logger.LogWarning($"Certificate pinning failed. Fingerprint: {fingerprint}");
            return false;
        }
        
        return true;
    }
}

// Usage
services.AddHttpClient("StripeAPI", client =>
{
    client.BaseAddress = new Uri("https://api.stripe.com");
})
.ConfigurePrimaryHttpMessageHandler(() => new CertificatePinningHandler(
    "ABC123DEF456...", // Stripe cert fingerprint
    "789XYZ012UVW..." // Backup cert fingerprint
));
```

---

## Data Protection

### Personal Identifiable Information (PII) Protection

```csharp
/// <summary>
/// PII detection and masking
/// </summary>
public class PIIProtectionService
{
    private static readonly Regex EmailRegex = new(@"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b");
    private static readonly Regex PhoneRegex = new(@"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b");
    private static readonly Regex SSNRegex = new(@"\b\d{3}-\d{2}-\d{4}\b");
    private static readonly Regex CreditCardRegex = new(@"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b");
    
    public string MaskPII(string text)
    {
        if (string.IsNullOrEmpty(text))
            return text;
        
        // Mask emails
        text = EmailRegex.Replace(text, m => MaskEmail(m.Value));
        
        // Mask phone numbers
        text = PhoneRegex.Replace(text, m => "***-***-" + m.Value.Substring(m.Value.Length - 4));
        
        // Mask SSN
        text = SSNRegex.Replace(text, "***-**-****");
        
        // Mask credit cards
        text = CreditCardRegex.Replace(text, "**** **** **** ****");
        
        return text;
    }
    
    private string MaskEmail(string email)
    {
        var parts = email.Split('@');
        if (parts.Length != 2)
            return email;
        
        var localPart = parts[0];
        var domain = parts[1];
        
        if (localPart.Length <= 2)
            return $"**@{domain}";
        
        return $"{localPart[0]}***{localPart[^1]}@{domain}";
    }
    
    public bool ContainsPII(string text)
    {
        if (string.IsNullOrEmpty(text))
            return false;
        
        return EmailRegex.IsMatch(text) ||
               PhoneRegex.IsMatch(text) ||
               SSNRegex.IsMatch(text) ||
               CreditCardRegex.IsMatch(text);
    }
}

/// <summary>
/// Logging filter to remove PII
/// </summary>
public class PIIFilteringLogger : ILogger
{
    private readonly ILogger _inner;
    private readonly PIIProtectionService _piiProtection;
    
    public void Log<TState>(
        LogLevel logLevel,
        EventId eventId,
        TState state,
        Exception exception,
        Func<TState, Exception, string> formatter)
    {
        var message = formatter(state, exception);
        var sanitized = _piiProtection.MaskPII(message);
        
        _inner.Log(logLevel, eventId, state, exception, (s, e) => sanitized);
    }
}
```

### Data Retention and Deletion

```csharp
/// <summary>
/// GDPR-compliant data retention service
/// </summary>
public class DataRetentionService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DataRetentionService> _logger;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DeleteExpiredDataAsync();
                await Task.Delay(TimeSpan.FromDays(1), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in data retention service");
            }
        }
    }
    
    private async Task DeleteExpiredDataAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        var retentionDate = DateTime.UtcNow.AddYears(-7); // 7 year retention
        
        // Delete old payments (keep metadata for auditing)
        var oldPayments = await context.Payments
            .Where(p => p.CreatedAt < retentionDate && p.Status == "Completed")
            .ToListAsync();
        
        foreach (var payment in oldPayments)
        {
            // Anonymize personal data
            payment.CustomerEmail = "deleted@anonymized.com";
            payment.CustomerName = "DELETED";
            payment.BillingAddress = null;
            payment.ShippingAddress = null;
        }
        
        await context.SaveChangesAsync();
        
        _logger.LogInformation($"Anonymized {oldPayments.Count} old payment records");
    }
}

/// <summary>
/// GDPR Right to Deletion (Right to be Forgotten)
/// </summary>
public class GDPRDeletionService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<GDPRDeletionService> _logger;
    
    public async Task<DeletionResult> DeleteCustomerDataAsync(string customerId)
    {
        _logger.LogInformation($"Processing deletion request for customer: {customerId}");
        
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // 1. Delete customer profile
            var customer = await _context.Customers.FindAsync(customerId);
            if (customer != null)
            {
                _context.Customers.Remove(customer);
            }
            
            // 2. Anonymize payment records (keep for financial auditing)
            var payments = await _context.Payments
                .Where(p => p.CustomerId == customerId)
                .ToListAsync();
            
            foreach (var payment in payments)
            {
                payment.CustomerEmail = $"deleted-{Guid.NewGuid()}@anonymized.com";
                payment.CustomerName = "DELETED USER";
                payment.CustomerId = "DELETED";
            }
            
            // 3. Delete saved payment methods
            var paymentMethods = await _context.PaymentMethods
                .Where(pm => pm.CustomerId == customerId)
                .ToListAsync();
            
            _context.PaymentMethods.RemoveRange(paymentMethods);
            
            // 4. Create audit log
            _context.AuditLogs.Add(new AuditLog
            {
                Action = "GDPR_DELETION",
                CustomerId = customerId,
                Details = $"Deleted {paymentMethods.Count} payment methods, " +
                         $"anonymized {payments.Count} payment records",
                Timestamp = DateTime.UtcNow
            });
            
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
            
            return DeletionResult.Success(customerId);
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync();
            _logger.LogError(ex, $"Failed to delete data for customer: {customerId}");
            throw;
        }
    }
}
```

---

**(Continuing in next response due to length...)**

Let me continue with the remaining critical sections:

---

## Fraud Prevention

### Velocity Checks

```csharp
/// <summary>
/// Prevent rapid-fire fraud attempts
/// </summary>
public class VelocityCheckService
{
    private readonly IDistributedCache _cache;
    
    public async Task<VelocityCheckResult> CheckVelocityAsync(
        string identifier,
        VelocityRule rule)
    {
        var key = $"velocity:{rule.Name}:{identifier}";
        var count = await GetCountAsync(key);
        
        if (count >= rule.MaxAttempts)
        {
            return VelocityCheckResult.Blocked(
                $"Too many attempts. Try again in {rule.WindowMinutes} minutes.");
        }
        
        await IncrementCountAsync(key, rule.WindowMinutes);
        
        return VelocityCheckResult.Allowed();
    }
    
    private async Task<int> GetCountAsync(string key)
    {
        var value = await _cache.GetStringAsync(key);
        return string.IsNullOrEmpty(value) ? 0 : int.Parse(value);
    }
    
    private async Task IncrementCountAsync(string key, int windowMinutes)
    {
        var count = await GetCountAsync(key);
        await _cache.SetStringAsync(
            key,
            (count + 1).ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(windowMinutes)
            });
    }
}

public class VelocityRule
{
    public string Name { get; set; }
    public int MaxAttempts { get; set; }
    public int WindowMinutes { get; set; }
}

public static class VelocityRules
{
    // Max 3 payment attempts per card per 5 minutes
    public static readonly VelocityRule PaymentPerCard = new()
    {
        Name = "payment_per_card",
        MaxAttempts = 3,
        WindowMinutes = 5
    };
    
    // Max 10 payments per IP per hour
    public static readonly VelocityRule PaymentPerIP = new()
    {
        Name = "payment_per_ip",
        MaxAttempts = 10,
        WindowMinutes = 60
    };
    
    // Max 5 failed login attempts per 15 minutes
    public static readonly VelocityRule FailedLogin = new()
    {
        Name = "failed_login",
        MaxAttempts = 5,
        WindowMinutes = 15
    };
}
```

### Risk Scoring

```csharp
/// <summary>
/// Calculate fraud risk score
/// </summary>
public class FraudRiskScorer
{
    public async Task<RiskScore> CalculateRiskAsync(PaymentRequest payment, HttpContext context)
    {
        var score = 0.0;
        var reasons = new List<string>();
        
        // IP Address checks
        var ipRisk = await CheckIPRiskAsync(context.Connection.RemoteIpAddress);
        score += ipRisk.Score;
        if (ipRisk.Score > 0) reasons.Add(ipRisk.Reason);
        
        // Amount checks
        if (payment.Amount > 1000)
        {
            score += 20;
            reasons.Add("High value transaction");
        }
        
        // Velocity checks
        var cardVelocity = await CheckCardVelocityAsync(payment.Last4);
        score += cardVelocity;
        if (cardVelocity > 0) reasons.Add("Multiple recent attempts with this card");
        
        // Device fingerprint
        var deviceRisk = CheckDeviceFingerprint(context.Request.Headers);
        score += deviceRisk.Score;
        if (deviceRisk.Score > 0) reasons.Add(deviceRisk.Reason);
        
        // Time of day
        var hour = DateTime.UtcNow.Hour;
        if (hour >= 2 && hour <= 5)
        {
            score += 10;
            reasons.Add("Transaction at unusual hour");
        }
        
        // Billing/shipping mismatch
        if (payment.BillingCountry != payment.ShippingCountry)
        {
            score += 15;
            reasons.Add("Billing and shipping countries don't match");
        }
        
        return new RiskScore
        {
            Score = Math.Min(score, 100),
            Level = score < 30 ? RiskLevel.Low :
                   score < 60 ? RiskLevel.Medium :
                   score < 80 ? RiskLevel.High : RiskLevel.Critical,
            Reasons = reasons
        };
    }
}
```

---

## API Security

### Input Validation

```csharp
public class PaymentRequestValidator : AbstractValidator<PaymentRequest>
{
    public PaymentRequestValidator()
    {
        RuleFor(x => x.Amount)
            .GreaterThan(0)
            .LessThan(1000000)
            .WithMessage("Amount must be between $0 and $1,000,000");

        RuleFor(x => x.Currency)
            .NotEmpty()
            .Length(3)
            .Matches("^[A-Z]{3}$")
            .WithMessage("Currency must be a valid ISO 4217 code");

        RuleFor(x => x.PaymentMethodId)
            .NotEmpty()
            .Matches("^(pm|card|ba)_[A-Za-z0-9]+$")
            .WithMessage("Invalid payment method ID");

        RuleFor(x => x.OrderId)
            .NotEmpty()
            .MaximumLength(100)
            .WithMessage("Order ID is required and must be less than 100 characters");
    }
}
```

### Prevent Amount Manipulation

```csharp
public class SecurePaymentService
{
    public async Task<PaymentResult> ProcessPaymentAsync(
        string orderId,
        string paymentMethodId)
    {
        // ❌ DON'T trust amount from client
        // var amount = request.Amount;

        // ✅ DO fetch amount from your database
        var order = await _orderRepository.GetByIdAsync(orderId);

        if (order == null)
            throw new InvalidOperationException("Order not found");

        if (order.PaymentStatus == PaymentStatus.Paid)
            throw new InvalidOperationException("Order already paid");

        // Use server-side amount
        var result = await _gateway.ProcessPaymentAsync(
            paymentMethodId,
            order.TotalAmount // From your database, not client
        );

        return result;
    }
}
```

### SQL Injection Prevention

```csharp
// ✅ Use parameterized queries (EF Core does this automatically)
var payments = await _context.Payments
    .Where(p => p.OrderId == orderId)
    .ToListAsync();

// ❌ NEVER concatenate SQL
// var query = $"SELECT * FROM Payments WHERE OrderId = '{orderId}'"; // VULNERABLE!
```

### API Key Storage & Rotation

```csharp
// 1. User Secrets (Development)
// dotnet user-secrets set "Stripe:SecretKey" "sk_test_..."

// 2. Environment Variables (Production)
// STRIPE__SECRETKEY=sk_live_...

// 3. Azure Key Vault (Production)
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((context, config) =>
        {
            if (context.HostingEnvironment.IsProduction())
            {
                var builtConfig = config.Build();
                config.AddAzureKeyVault(
                    new Uri(builtConfig["KeyVault:Endpoint"]),
                    new DefaultAzureCredential());
            }
        });

// 4. AWS Secrets Manager
public class SecretsService
{
    private readonly IAmazonSecretsManager _secretsManager;

    public async Task<string> GetSecretAsync(string secretName)
    {
        var response = await _secretsManager.GetSecretValueAsync(
            new GetSecretValueRequest { SecretId = secretName });
        return response.SecretString;
    }
}

// 5. Key Rotation
public class RotatingApiKeyProvider : IApiKeyProvider
{
    private string _cachedKey;
    private DateTime _lastRefresh;
    private readonly TimeSpan _refreshInterval = TimeSpan.FromHours(1);

    public async Task<string> GetApiKeyAsync()
    {
        if (_cachedKey == null || DateTime.UtcNow - _lastRefresh > _refreshInterval)
        {
            _cachedKey = await FetchKeyFromSecureStoreAsync();
            _lastRefresh = DateTime.UtcNow;
        }
        return _cachedKey;
    }
}
```

---

## Security Monitoring

### Audit Logging

```csharp
/// <summary>
/// Comprehensive audit logging
/// </summary>
public class AuditLogger
{
    private readonly ApplicationDbContext _context;
    
    public async Task LogAsync(AuditLog log)
    {
        _context.AuditLogs.Add(log);
        await _context.SaveChangesAsync();
    }
    
    public async Task LogPaymentActionAsync(
        string action,
        Guid paymentId,
        string userId,
        Dictionary<string, string> metadata = null)
    {
        await LogAsync(new AuditLog
        {
            Action = action,
            EntityType = "Payment",
            EntityId = paymentId.ToString(),
            UserId = userId,
            Timestamp = DateTime.UtcNow,
            Metadata = JsonSerializer.Serialize(metadata ?? new())
        });
    }
}

/// <summary>
/// Audit middleware
/// </summary>
public class AuditMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context, AuditLogger auditLogger)
    {
        var path = context.Request.Path.Value;
        var method = context.Request.Method;
        
        // Only audit sensitive operations
        if (IsSensitiveOperation(path, method))
        {
            var userId = context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            var ipAddress = context.Connection.RemoteIpAddress?.ToString();
            
            await auditLogger.LogAsync(new AuditLog
            {
                Action = $"{method} {path}",
                UserId = userId,
                IPAddress = ipAddress,
                UserAgent = context.Request.Headers["User-Agent"],
                Timestamp = DateTime.UtcNow
            });
        }
        
        await _next(context);
    }
    
    private bool IsSensitiveOperation(string path, string method)
    {
        return path.Contains("/api/payments") ||
               path.Contains("/api/refunds") ||
               path.Contains("/api/customers") ||
               (method == "DELETE" || method == "PUT");
    }
}
```

---

## Security Checklist

### Pre-Production Checklist

```yaml
Authentication & Authorization:
  ☐ MFA enabled for admin accounts
  ☐ Password policies enforced (min 12 chars, complexity)
  ☐ API keys rotated regularly
  ☐ Session timeouts configured (15 min)
  ☐ RBAC implemented with least privilege

Data Protection:
  ☐ All sensitive data encrypted at rest (AES-256)
  ☐ All communication over TLS 1.2+
  ☐ PII identified and protected
  ☐ Card data NEVER stored (using tokens only)
  ☐ Database backups encrypted
  ☐ Encryption keys in Key Vault/KMS

PCI DSS Compliance:
  ☐ SAQ level determined and documented
  ☐ Cardholder data environment (CDE) isolated
  ☐ Network security controls in place
  ☐ Quarterly vulnerability scans scheduled
  ☐ Annual penetration test completed
  ☐ Security awareness training completed

Application Security:
  ☐ Input validation on all endpoints
  ☐ SQL injection prevention (parameterized queries)
  ☐ XSS prevention (output encoding)
  ☐ CSRF tokens implemented
  ☐ Security headers configured (CSP, HSTS, etc.)
  ☐ Rate limiting implemented
  ☐ DDoS protection enabled

Monitoring & Response:
  ☐ Security monitoring enabled
  ☐ Intrusion detection configured
  ☐ Audit logging enabled for all sensitive operations
  ☐ Log retention policy defined (min 1 year)
  ☐ Incident response plan documented
  ☐ Security team contacts updated
  ☐ Alert thresholds configured

Code Security:
  ☐ Dependencies scanned for vulnerabilities
  ☐ SAST (static analysis) completed
  ☐ DAST (dynamic analysis) completed
  ☐ Secrets not in source code
  ☐ Security code review completed
  ☐ Penetration testing completed
```

---

## Summary

### Security Priorities

| Priority | Focus Area | Impact |
|----------|-----------|--------|
| **P0** | Never store card data | Critical |
| **P0** | Encrypt all sensitive data | Critical |
| **P0** | TLS 1.2+ everywhere | Critical |
| **P1** | Implement MFA | High |
| **P1** | Comprehensive audit logging | High |
| **P1** | Rate limiting & DDoS protection | High |
| **P2** | Fraud detection | Medium |
| **P2** | Security monitoring | Medium |

This completes the comprehensive security patterns and best practices documentation!
