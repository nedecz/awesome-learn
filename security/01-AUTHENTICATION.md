# Authentication

A comprehensive guide to authentication — covering identity verification, password hashing, multi-factor authentication (MFA), single sign-on (SSO), OAuth 2.0, OpenID Connect, session management, token-based authentication, and passwordless approaches.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Authentication Fundamentals](#authentication-fundamentals)
   - [Authentication Factors](#authentication-factors)
   - [Authentication vs Authorization](#authentication-vs-authorization)
3. [Password Security](#password-security)
   - [Password Hashing](#password-hashing)
   - [Hashing Algorithm Comparison](#hashing-algorithm-comparison)
   - [Password Policies](#password-policies)
   - [Credential Storage](#credential-storage)
4. [Multi-Factor Authentication](#multi-factor-authentication)
   - [MFA Methods](#mfa-methods)
   - [TOTP Implementation](#totp-implementation)
   - [WebAuthn and FIDO2](#webauthn-and-fido2)
   - [Recovery Codes](#recovery-codes)
5. [OAuth 2.0 and OpenID Connect](#oauth-20-and-openid-connect)
   - [OAuth 2.0 Grant Types](#oauth-20-grant-types)
   - [Authorization Code Flow with PKCE](#authorization-code-flow-with-pkce)
   - [OpenID Connect](#openid-connect)
   - [Token Types](#token-types)
6. [Session Management](#session-management)
   - [Server-Side Sessions](#server-side-sessions)
   - [Session Security](#session-security)
   - [Session Fixation Prevention](#session-fixation-prevention)
7. [Token-Based Authentication](#token-based-authentication)
   - [JSON Web Tokens (JWT)](#json-web-tokens-jwt)
   - [JWT Security Considerations](#jwt-security-considerations)
   - [Token Refresh Strategies](#token-refresh-strategies)
   - [Token Revocation](#token-revocation)
8. [Single Sign-On (SSO)](#single-sign-on-sso)
   - [SAML 2.0](#saml-20)
   - [SSO with OpenID Connect](#sso-with-openid-connect)
   - [Federation](#federation)
9. [Passwordless Authentication](#passwordless-authentication)
   - [Magic Links](#magic-links)
   - [Passkeys](#passkeys)
   - [Biometric Authentication](#biometric-authentication)
10. [Service-to-Service Authentication](#service-to-service-authentication)
    - [Mutual TLS (mTLS)](#mutual-tls-mtls)
    - [API Keys](#api-keys)
    - [Service Tokens](#service-tokens)
11. [Account Security](#account-security)
    - [Account Lockout](#account-lockout)
    - [Rate Limiting Login Attempts](#rate-limiting-login-attempts)
    - [Credential Stuffing Protection](#credential-stuffing-protection)
    - [Secure Password Reset](#secure-password-reset)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Authentication is the process of verifying the identity of a user, service, or system. It answers the question: **"Who are you?"** Authentication is the first gate in the security model — if an attacker can bypass authentication, they gain access to the system and can potentially exploit any downstream vulnerability.

This document covers the full spectrum of authentication mechanisms: from password hashing and session management to OAuth 2.0, MFA, SSO, and modern passwordless approaches. Each mechanism is presented with its security properties, implementation considerations, and common pitfalls.

### Target Audience

- **Developers** implementing login flows, token management, and identity integration
- **Security Engineers** auditing authentication mechanisms and recommending improvements
- **Architects** designing identity and access management (IAM) systems
- **DevOps Engineers** configuring identity providers and service-to-service authentication

### Scope

- Authentication fundamentals and factors of authentication
- Password hashing algorithms (bcrypt, Argon2, scrypt) and secure credential storage
- Multi-factor authentication (TOTP, WebAuthn, FIDO2)
- OAuth 2.0 grant types and OpenID Connect
- Session management and token-based authentication (JWT)
- Single sign-on (SSO) with SAML and OIDC
- Passwordless authentication (passkeys, magic links)
- Service-to-service authentication (mTLS, API keys)
- Account security (lockout, rate limiting, credential stuffing protection)

---

## Authentication Fundamentals

### Authentication Factors

Authentication factors are categories of evidence used to verify identity. Stronger authentication combines multiple factors.

| Factor | Description | Examples |
|--------|-------------|---------|
| **Something you know** | Knowledge-based | Password, PIN, security question |
| **Something you have** | Possession-based | Phone (TOTP app), hardware key (YubiKey), smart card |
| **Something you are** | Biometric | Fingerprint, face recognition, retina scan |
| **Somewhere you are** | Location-based | IP geolocation, GPS, network proximity |
| **Something you do** | Behavioral | Typing patterns, mouse movements, gait analysis |

```
Authentication Strength
───────────────────────
Single Factor (1FA)    → Password only — weakest, most common
Two-Factor (2FA)       → Password + TOTP code — significantly stronger
Multi-Factor (MFA)     → Two or more different factors — recommended minimum
Passwordless           → Hardware key or biometric — strongest user experience
```

### Authentication vs Authorization

| Aspect | Authentication | Authorization |
|--------|---------------|---------------|
| Question | Who are you? | What can you do? |
| Timing | Happens first | Happens after authentication |
| Mechanism | Credentials, tokens, biometrics | Roles, permissions, policies |
| Failure | 401 Unauthorized | 403 Forbidden |
| Example | Login with username/password | Check if user can delete a resource |

---

## Password Security

### Password Hashing

Passwords must **never** be stored in plain text or with reversible encryption. They must be hashed using a slow, salted, one-way hashing algorithm designed specifically for password storage.

**Why "slow" hashing?**

General-purpose hash functions (SHA-256, MD5) are designed to be fast. An attacker with a GPU can compute billions of SHA-256 hashes per second, making brute-force attacks trivial. Password hashing algorithms are intentionally slow (configurable work factor) to make brute-force attacks computationally expensive.

```
Password Hashing Flow
─────────────────────
User registers:
  password → salt + hash(password, salt, work_factor) → store hash + salt

User logs in:
  password → hash(password, stored_salt, work_factor) → compare with stored hash
```

### Hashing Algorithm Comparison

| Algorithm | Recommended | Work Factor | Memory-Hard | Notes |
|-----------|------------|-------------|-------------|-------|
| **Argon2id** | ✅ Best choice | Time + memory + parallelism | Yes | Winner of the Password Hashing Competition (PHC). Resists GPU and ASIC attacks. |
| **bcrypt** | ✅ Good choice | Cost factor (log rounds) | No | Mature, widely supported. 72-byte password limit. |
| **scrypt** | ✅ Acceptable | N, r, p parameters | Yes | Memory-hard, but harder to tune than Argon2. |
| **PBKDF2** | ⚠️ Legacy | Iteration count | No | NIST approved but not memory-hard. Use only when Argon2/bcrypt are unavailable. |
| **SHA-256** | ❌ Never | N/A | No | Not a password hash — too fast, no salt by default. |
| **MD5** | ❌ Never | N/A | No | Broken — collisions found, trivially fast. |

### Password Policies

Modern password policies prioritize length over complexity and avoid counterproductive rules:

```
Recommended Password Policy
────────────────────────────
Minimum length          → 12 characters (16+ for admin accounts)
Maximum length          → At least 128 characters (do not truncate)
Character requirements  → None — let users choose any characters including Unicode
Breached password check → Reject passwords found in known breach databases
                          (Have I Been Pwned API, NIST SP 800-63B)
No periodic rotation    → Do not force regular password changes
                          (NIST SP 800-63B recommends against it)
No security questions   → They are effectively weaker passwords
```

### Credential Storage

- Hash passwords with Argon2id (preferred) or bcrypt
- Generate a unique random salt per password (most libraries do this automatically)
- Store the hash, salt, and algorithm identifier together
- Never log passwords or include them in error messages
- Never transmit passwords over unencrypted channels
- Use constant-time comparison to prevent timing attacks

---

## Multi-Factor Authentication

### MFA Methods

| Method | Security | User Experience | Phishing Resistant | Notes |
|--------|----------|-----------------|-------------------|-------|
| **Hardware security key** (FIDO2) | Highest | Good (tap key) | ✅ Yes | YubiKey, Titan Key. Strongest option. |
| **Passkeys** (FIDO2) | Highest | Best (biometric) | ✅ Yes | Device-bound or synced credentials. |
| **TOTP app** (Authenticator) | High | Good (enter code) | ❌ No | Google Authenticator, Authy. Phishable. |
| **Push notification** | High | Good (tap approve) | ⚠️ Partial | Can be susceptible to MFA fatigue attacks. |
| **SMS OTP** | Low | Good (auto-fill) | ❌ No | Vulnerable to SIM swapping and SS7 attacks. |
| **Email OTP** | Low | Fair (check email) | ❌ No | Dependent on email account security. |

### TOTP Implementation

**Time-Based One-Time Password (TOTP)** generates a 6-digit code that changes every 30 seconds, derived from a shared secret and the current time.

```
TOTP Flow
─────────
1. Server generates a random secret key (base32 encoded)
2. Server shares secret with user via QR code (otpauth:// URI)
3. User scans QR code with authenticator app
4. On login, user enters the current 6-digit code from the app
5. Server computes TOTP using the shared secret and current time
6. Server compares the codes (allowing ±1 time step for clock drift)
```

### WebAuthn and FIDO2

**WebAuthn** is a W3C standard for passwordless and phishing-resistant authentication using public key cryptography. **FIDO2** is the umbrella term for WebAuthn + CTAP2 (Client to Authenticator Protocol).

```
WebAuthn Registration Flow
──────────────────────────
1. Server sends a challenge to the browser
2. Browser prompts user to authenticate with their device
   (fingerprint, face, hardware key)
3. Authenticator creates a public/private key pair
4. Public key + attestation sent to server
5. Server stores the public key for future authentication

WebAuthn Authentication Flow
────────────────────────────
1. Server sends a challenge
2. Browser prompts user to authenticate
3. Authenticator signs the challenge with the private key
4. Server verifies the signature with the stored public key
```

### Recovery Codes

When users enable MFA, provide a set of one-time recovery codes in case they lose access to their second factor:

- Generate 8–10 single-use recovery codes
- Each code should be at least 8 characters, alphanumeric
- Hash recovery codes before storing them (like passwords)
- Allow each code to be used exactly once
- Provide an option to regenerate codes (which invalidates existing ones)
- Warn users to store codes securely offline

---

## OAuth 2.0 and OpenID Connect

### OAuth 2.0 Grant Types

| Grant Type | Use Case | Confidential Client | Notes |
|------------|----------|-------------------|-------|
| **Authorization Code + PKCE** | Web apps, mobile apps, SPAs | Both | ✅ Recommended for all new applications |
| **Client Credentials** | Service-to-service | Yes | Machine-to-machine, no user context |
| **Device Authorization** | TVs, CLI tools, IoT | No | For devices with limited input capability |
| **Authorization Code** (no PKCE) | Legacy server-side apps | Yes | ⚠️ Use PKCE instead |
| **Implicit** | Deprecated | N/A | ❌ Removed in OAuth 2.1 — use Auth Code + PKCE |
| **Resource Owner Password** | Deprecated | N/A | ❌ Removed in OAuth 2.1 — never use |

### Authorization Code Flow with PKCE

The **Authorization Code flow with PKCE** (Proof Key for Code Exchange) is the recommended flow for all applications, including single-page applications and mobile apps.

```
Authorization Code Flow with PKCE
──────────────────────────────────
┌────────┐                          ┌──────────────┐
│ Client │                          │ Auth Server  │
│ (App)  │                          │ (IdP)        │
└───┬────┘                          └──────┬───────┘
    │                                      │
    │ 1. Generate code_verifier            │
    │    (random string)                   │
    │    code_challenge =                  │
    │    SHA256(code_verifier)             │
    │                                      │
    │ 2. Redirect user with                │
    │    code_challenge ──────────────────►│
    │                                      │
    │                    3. User logs in    │
    │                       and consents   │
    │                                      │
    │◄──────────────── 4. Authorization    │
    │                     code (one-time)  │
    │                                      │
    │ 5. Exchange code +                   │
    │    code_verifier ──────────────────►│
    │                                      │
    │◄──────────────── 6. Access token     │
    │                     + Refresh token  │
    │                     (+ ID token      │
    │                      if OIDC)        │
    │                                      │
```

### OpenID Connect

**OpenID Connect (OIDC)** is an identity layer built on top of OAuth 2.0. While OAuth 2.0 is an authorization framework (what can you access?), OIDC adds authentication (who are you?).

OIDC adds:

- **ID Token** — A JWT containing user identity claims (sub, email, name)
- **UserInfo endpoint** — An API to fetch additional user profile data
- **Standard scopes** — `openid`, `profile`, `email`, `address`, `phone`
- **Discovery** — `.well-known/openid-configuration` endpoint for auto-configuration

### Token Types

| Token | Purpose | Format | Lifetime | Storage |
|-------|---------|--------|----------|---------|
| **Access Token** | Authorize API requests | JWT or opaque | Short (5–60 min) | Memory only |
| **Refresh Token** | Obtain new access tokens | Opaque | Long (hours–days) | Secure, httpOnly cookie or secure storage |
| **ID Token** | User identity claims | JWT | Short (5–60 min) | Memory only |
| **Authorization Code** | Exchange for tokens | Opaque | Very short (seconds) | URL parameter (one-time use) |

---

## Session Management

### Server-Side Sessions

Server-side sessions store session state on the server, with only a session identifier sent to the client (typically in a cookie).

```
Server-Side Session Flow
────────────────────────
1. User authenticates successfully
2. Server creates a session object (user ID, roles, expiry)
3. Server stores session in a session store (Redis, database)
4. Server sends session ID to client in a Set-Cookie header
5. Client sends session ID cookie on every subsequent request
6. Server looks up session data from the store on each request
7. On logout, server deletes the session from the store
```

### Session Security

| Control | Implementation |
|---------|---------------|
| **Session ID entropy** | Use a CSPRNG to generate at least 128 bits of randomness |
| **Cookie flags** | `Secure; HttpOnly; SameSite=Lax` (or `Strict`) |
| **Session timeout** | Idle timeout (15–30 min) and absolute timeout (8–24 hours) |
| **Session regeneration** | Generate a new session ID after login and privilege escalation |
| **Session invalidation** | Destroy session on logout; invalidate all sessions on password change |
| **Concurrent sessions** | Consider limiting concurrent sessions per user |

### Session Fixation Prevention

**Session fixation** is an attack where an attacker sets the session ID before the victim authenticates, then uses that known session ID to hijack the session.

**Prevention:** Always regenerate the session ID after successful authentication. Never accept session IDs from URL parameters or form data.

---

## Token-Based Authentication

### JSON Web Tokens (JWT)

A **JWT** is a compact, URL-safe token format that consists of three base64url-encoded parts separated by dots: `header.payload.signature`.

```
JWT Structure
─────────────
Header (algorithm, type):
  { "alg": "RS256", "typ": "JWT" }

Payload (claims):
  {
    "sub": "user-123",
    "iss": "https://auth.example.com",
    "aud": "https://api.example.com",
    "exp": 1700000000,
    "iat": 1699999000,
    "roles": ["user", "editor"]
  }

Signature:
  RS256(base64url(header) + "." + base64url(payload), private_key)
```

### JWT Security Considerations

| Risk | Mitigation |
|------|------------|
| **Algorithm confusion** | Always validate the `alg` header; reject `none` and unexpected algorithms |
| **Token theft** | Use short expiry (5–15 min); store in memory, not localStorage |
| **No revocation** | Maintain a deny list for revoked tokens or use short-lived tokens with refresh |
| **Payload exposure** | JWTs are encoded, not encrypted — do not put sensitive data in the payload |
| **Key management** | Use asymmetric keys (RS256/ES256) for distributed systems; rotate keys regularly |
| **Clock skew** | Allow a small leeway (30–60 seconds) when validating `exp` and `nbf` |

### Token Refresh Strategies

```
Silent Refresh with Refresh Tokens
───────────────────────────────────
1. Access token expires (short-lived, 5–15 min)
2. Client sends refresh token to token endpoint
3. Server validates refresh token and issues new access + refresh tokens
4. Old refresh token is invalidated (rotation)
5. Client uses new access token for API requests

Refresh Token Rotation
──────────────────────
- Issue a new refresh token with every access token refresh
- Invalidate the old refresh token immediately
- If an old refresh token is reused, invalidate ALL tokens for that session
  (indicates token theft)
```

### Token Revocation

Since JWTs are stateless, they cannot be revoked without additional infrastructure:

- **Short-lived tokens** — Set access token expiry to 5–15 minutes; the window of exposure is limited
- **Token deny list** — Maintain a list of revoked token IDs (`jti` claim) checked on each request
- **Refresh token revocation** — Revoke the refresh token so no new access tokens can be issued
- **Session-based revocation** — Include a session ID in the token; revoke the session server-side

---

## Single Sign-On (SSO)

### SAML 2.0

**Security Assertion Markup Language (SAML)** is an XML-based framework for exchanging authentication and authorization data between an Identity Provider (IdP) and a Service Provider (SP).

```
SAML SSO Flow (SP-Initiated)
─────────────────────────────
1. User visits Service Provider (SP)
2. SP redirects user to Identity Provider (IdP) with SAML AuthnRequest
3. IdP authenticates user (login page, MFA)
4. IdP creates a SAML Assertion (signed XML document with user attributes)
5. IdP redirects user back to SP with the SAML Response
6. SP validates the SAML Assertion (signature, audience, timestamps)
7. SP creates a local session for the user
```

### SSO with OpenID Connect

For new applications, OIDC-based SSO is preferred over SAML due to its simpler JSON-based format and better support for modern architectures (SPAs, mobile apps, APIs).

### Federation

**Identity federation** allows organizations to trust each other's identity providers, enabling users from one organization to access resources in another without creating separate accounts.

---

## Passwordless Authentication

### Magic Links

A **magic link** is a one-time, time-limited URL sent to the user's email. Clicking the link authenticates the user without requiring a password.

**Security considerations:**

- Links must expire quickly (5–15 minutes)
- Links must be single-use
- Links must be generated with sufficient randomness (128+ bits)
- The email channel becomes the authentication factor — protect it accordingly

### Passkeys

**Passkeys** are FIDO2 credentials that replace passwords entirely. They can be device-bound (stored on a single device) or synced across devices (via iCloud Keychain, Google Password Manager).

**Advantages:**

- Phishing-resistant — credentials are bound to the origin (domain)
- No passwords to forget, reuse, or have stolen
- Biometric authentication (fingerprint, face) provides a smooth user experience
- No server-side secrets — only public keys are stored

### Biometric Authentication

Biometric authentication uses physical characteristics (fingerprint, face, iris) to verify identity. In modern implementations, biometrics are used as a local unlock mechanism (e.g., to access a stored private key) rather than being transmitted to a server.

---

## Service-to-Service Authentication

### Mutual TLS (mTLS)

**Mutual TLS** authenticates both the client and server using X.509 certificates. It is the standard for zero-trust service mesh communication.

```
mTLS Handshake
──────────────
1. Client connects to server
2. Server presents its certificate; client validates it
3. Server requests client certificate
4. Client presents its certificate; server validates it
5. Both sides have verified each other's identity
6. Encrypted channel established
```

### API Keys

API keys are simple tokens used to identify and authenticate a calling application. They should be treated as secrets.

**Best practices:**

- Scope API keys to specific services and permissions
- Rotate API keys on a regular schedule
- Never embed API keys in source code or client-side applications
- Transmit API keys in headers (not URL parameters) to avoid logging
- Implement rate limiting per API key

### Service Tokens

For microservices, use short-lived tokens (JWT) issued by a central identity service. Each service authenticates using its own credentials (client certificate or client credentials grant) and receives a scoped token for downstream calls.

---

## Account Security

### Account Lockout

Temporarily lock accounts after repeated failed login attempts:

- Lock after 5–10 failed attempts
- Lock duration: 15–30 minutes (or until manual unlock)
- Notify the account owner of the lockout via email
- Log all lockout events for security monitoring

### Rate Limiting Login Attempts

Rate limiting is the first line of defense against brute-force and credential stuffing attacks:

- Limit by IP address (e.g., 10 attempts per minute per IP)
- Limit by account (e.g., 5 attempts per minute per account)
- Use exponential backoff for repeated failures
- Consider CAPTCHA after multiple failed attempts

### Credential Stuffing Protection

**Credential stuffing** uses breached username/password pairs from other sites to attempt login. Defend against it with:

- Require MFA (the strongest defense)
- Check passwords against known breach databases (Have I Been Pwned)
- Detect anomalous login patterns (new device, unusual location, high velocity)
- Use bot detection and CAPTCHA for suspicious login attempts

### Secure Password Reset

- Send a password reset link to the registered email (never reveal if the email exists)
- Reset tokens must be single-use and expire quickly (15–60 minutes)
- Hash reset tokens before storing them
- Invalidate all existing sessions after a password reset
- Require the current password for password change (not reset)
- Log password reset events and notify the user

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [02-AUTHORIZATION.md](02-AUTHORIZATION.md) | Authorization | RBAC, ABAC, policy engines, least privilege |
| [03-CRYPTOGRAPHY.md](03-CRYPTOGRAPHY.md) | Cryptography | Encryption, TLS, key management |
| [00-OVERVIEW.md](00-OVERVIEW.md) | Security Fundamentals | CIA triad, threat modeling, OWASP Top 10 |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Authentication documentation |
