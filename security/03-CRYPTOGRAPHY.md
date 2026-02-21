# Cryptography

A comprehensive guide to applied cryptography — covering encryption at rest and in transit, TLS, hashing, digital signatures, key management, public key infrastructure (PKI), and cryptographic best practices for modern applications.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Cryptography Fundamentals](#cryptography-fundamentals)
   - [Symmetric vs Asymmetric Encryption](#symmetric-vs-asymmetric-encryption)
   - [Hashing](#hashing)
   - [Digital Signatures](#digital-signatures)
3. [Encryption at Rest](#encryption-at-rest)
   - [AES Encryption](#aes-encryption)
   - [Database Encryption](#database-encryption)
   - [Disk Encryption](#disk-encryption)
   - [Application-Level Encryption](#application-level-encryption)
4. [Encryption in Transit](#encryption-in-transit)
   - [TLS 1.3](#tls-13)
   - [TLS Certificate Management](#tls-certificate-management)
   - [mTLS for Service-to-Service](#mtls-for-service-to-service)
   - [Certificate Transparency](#certificate-transparency)
5. [Key Management](#key-management)
   - [Key Lifecycle](#key-lifecycle)
   - [Key Management Services](#key-management-services)
   - [Envelope Encryption](#envelope-encryption)
   - [Key Rotation](#key-rotation)
6. [Public Key Infrastructure (PKI)](#public-key-infrastructure-pki)
   - [Certificate Authorities](#certificate-authorities)
   - [Certificate Validation](#certificate-validation)
   - [Internal PKI](#internal-pki)
7. [Cryptographic Hash Functions](#cryptographic-hash-functions)
   - [Hash Function Properties](#hash-function-properties)
   - [Common Hash Algorithms](#common-hash-algorithms)
   - [HMAC](#hmac)
8. [Common Cryptographic Mistakes](#common-cryptographic-mistakes)
9. [Next Steps](#next-steps)
10. [Version History](#version-history)

---

## Overview

Cryptography is the foundation of data protection in modern software systems. It provides the mathematical guarantees that underpin confidentiality (encryption), integrity (hashing and signatures), and authentication (certificates and signed tokens). Applied correctly, cryptography protects data at rest, in transit, and in use. Applied incorrectly, it creates a false sense of security while leaving data exposed.

This document covers practical, applied cryptography for software engineers — not the mathematical theory, but the tools, algorithms, and patterns needed to protect data in real systems.

### Target Audience

- **Developers** implementing encryption, hashing, and signature verification in application code
- **DevOps Engineers** configuring TLS, managing certificates, and integrating key management services
- **Security Engineers** auditing cryptographic implementations and recommending improvements
- **Architects** designing data protection strategies and key management architectures

### Scope

- Symmetric and asymmetric encryption fundamentals
- Encryption at rest: AES, database encryption, disk encryption, application-level encryption
- Encryption in transit: TLS 1.3, certificate management, mTLS
- Key management: lifecycle, KMS integration, envelope encryption, rotation
- Public key infrastructure: CAs, certificate validation, internal PKI
- Cryptographic hashing: SHA-256, HMAC, integrity verification
- Common cryptographic mistakes and how to avoid them

---

## Cryptography Fundamentals

### Symmetric vs Asymmetric Encryption

| Property | Symmetric | Asymmetric |
|----------|-----------|-----------|
| Keys | Single shared key | Public + private key pair |
| Speed | Fast | Slow (100–1000x slower) |
| Use case | Encrypting data (AES) | Key exchange, signatures (RSA, ECDSA) |
| Key distribution | Must share key securely | Public key can be shared openly |
| Algorithms | AES-256-GCM, ChaCha20-Poly1305 | RSA-2048+, ECDSA P-256, Ed25519 |

```
Symmetric Encryption
────────────────────
Plaintext ──[key]──► Ciphertext ──[same key]──► Plaintext

Asymmetric Encryption
─────────────────────
Plaintext ──[public key]──► Ciphertext ──[private key]──► Plaintext

In Practice (Hybrid Encryption)
───────────────────────────────
1. Generate a random symmetric key (session key)
2. Encrypt the data with the symmetric key (fast)
3. Encrypt the symmetric key with the recipient's public key (slow, but key is small)
4. Send both the encrypted data and the encrypted session key
```

### Hashing

A **cryptographic hash function** takes input of any size and produces a fixed-size output (digest) with the following properties:

- **Deterministic** — Same input always produces the same output
- **One-way** — Computationally infeasible to reverse (cannot recover input from output)
- **Collision-resistant** — Computationally infeasible to find two different inputs with the same output
- **Avalanche effect** — A small change in input produces a completely different output

```
Hashing Use Cases
─────────────────
Password storage     → bcrypt/Argon2 (slow, salted — see 01-AUTHENTICATION.md)
Data integrity       → SHA-256 hash of files, artifacts, messages
Digital signatures   → Hash the message, then sign the hash
HMAC                 → Keyed hash for message authentication codes
Content addressing   → Git commit hashes, Docker image digests
```

### Digital Signatures

A **digital signature** proves that a message was created by a known sender (authentication) and has not been altered (integrity). It uses asymmetric cryptography.

```
Digital Signature Flow
──────────────────────
Signing:
  message → hash(message) → encrypt(hash, private_key) → signature

Verification:
  message, signature → decrypt(signature, public_key) → hash₁
                       hash(message) → hash₂
                       if hash₁ == hash₂ → valid signature
```

**Use cases:**

- Code signing — Verify that software comes from a trusted publisher
- Artifact signing — Verify that container images and packages are authentic
- JWT signing — Verify that tokens were issued by a trusted authority
- Git commit signing — Verify that commits were authored by the claimed person

---

## Encryption at Rest

### AES Encryption

**AES (Advanced Encryption Standard)** is the most widely used symmetric encryption algorithm. Use **AES-256-GCM** (Galois/Counter Mode) for authenticated encryption — it provides both confidentiality and integrity.

```
AES-256-GCM
────────────
- Key size: 256 bits
- Block size: 128 bits
- Mode: GCM (authenticated encryption with associated data — AEAD)
- Output: ciphertext + authentication tag (16 bytes)
- IV/Nonce: 96 bits (12 bytes) — MUST be unique per encryption operation with the same key

Why GCM?
- Provides encryption AND integrity (authentication tag)
- Detects tampering — decryption fails if ciphertext or associated data is modified
- Hardware-accelerated on modern CPUs (AES-NI)
```

**Critical rule:** Never reuse a nonce (IV) with the same key. Nonce reuse with GCM completely breaks confidentiality and authenticity.

### Database Encryption

| Level | What is Encrypted | Protection Against |
|-------|------------------|-------------------|
| **Transparent Data Encryption (TDE)** | Entire database files on disk | Physical theft of storage media |
| **Column-level encryption** | Specific sensitive columns | Database admin access, SQL injection data exfil |
| **Application-level encryption** | Data encrypted before reaching the database | Database compromise, admin access, backups |

### Disk Encryption

- **Full disk encryption (FDE)** — Encrypts the entire disk (LUKS on Linux, BitLocker on Windows, FileVault on macOS)
- **Cloud volume encryption** — AWS EBS encryption, Azure Disk Encryption, GCP CMEK
- Protects against physical theft and unauthorized access to storage media
- Does **not** protect against attacks on a running system (OS-level access)

### Application-Level Encryption

Encrypt sensitive data in the application layer before storing it:

- Encrypt PII, financial data, and health records before writing to the database
- Use envelope encryption (encrypt data with a DEK, encrypt DEK with a KEK from KMS)
- Store the encrypted DEK alongside the ciphertext
- Application controls which users/services can decrypt the data

---

## Encryption in Transit

### TLS 1.3

**TLS 1.3** is the current recommended version of the Transport Layer Security protocol. It provides encryption, integrity, and authentication for network communication.

```
TLS 1.3 Improvements over TLS 1.2
──────────────────────────────────
- Faster handshake: 1-RTT (one round trip), 0-RTT for resumption
- Removed insecure algorithms: RC4, DES, 3DES, MD5, SHA-1, static RSA
- Simplified cipher suites: only AEAD ciphers (AES-GCM, ChaCha20-Poly1305)
- Forward secrecy: mandatory (ephemeral key exchange only)
- Encrypted handshake: server certificate is encrypted in TLS 1.3

Recommended Cipher Suites (TLS 1.3)
────────────────────────────────────
TLS_AES_256_GCM_SHA384
TLS_AES_128_GCM_SHA256
TLS_CHACHA20_POLY1305_SHA256
```

### TLS Certificate Management

| Task | Recommendation |
|------|---------------|
| **Certificate authority** | Use a trusted public CA (Let's Encrypt, DigiCert) or internal CA for private services |
| **Certificate lifetime** | 90 days (Let's Encrypt default) — short-lived certificates reduce exposure |
| **Automated renewal** | Use ACME protocol (certbot, cert-manager) for automated renewal |
| **Key size** | RSA 2048+ or ECDSA P-256 (preferred — smaller, faster) |
| **Certificate pinning** | Avoid in most cases — use Certificate Transparency instead |
| **Wildcard certificates** | Use sparingly — a compromised wildcard key exposes all subdomains |

### mTLS for Service-to-Service

**Mutual TLS** authenticates both the client and server, providing strong service-to-service identity in a zero-trust architecture.

- Use a service mesh (Istio, Linkerd) for automatic mTLS between services
- Or manage client certificates manually with an internal CA
- Rotate certificates automatically with short lifetimes (24–72 hours in service meshes)

### Certificate Transparency

**Certificate Transparency (CT)** is a public log system that records all publicly trusted certificates. It allows domain owners to monitor for unauthorized certificate issuance.

- Monitor CT logs for certificates issued for your domains
- Use tools like crt.sh, Facebook CT Monitor, or Cert Spotter
- Require CT compliance in your TLS configuration

---

## Key Management

### Key Lifecycle

```
Key Lifecycle
─────────────
Generation → Storage → Distribution → Use → Rotation → Revocation → Destruction

Generation:   Use a CSPRNG (cryptographically secure pseudo-random number generator)
              Never generate keys from predictable sources
Storage:      Store in HSM or KMS — never in source code, config files, or environment vars
Distribution: Use secure channels (TLS, key wrapping) to distribute keys
Use:          Enforce access controls on key usage
Rotation:     Rotate keys on a regular schedule and on compromise
Revocation:   Revoke compromised keys immediately; maintain a revocation list
Destruction:  Securely destroy keys when no longer needed (cryptographic erasure)
```

### Key Management Services

| Service | Provider | HSM-Backed | Features |
|---------|----------|-----------|----------|
| **AWS KMS** | AWS | Yes (FIPS 140-2 Level 3) | Envelope encryption, key policies, audit via CloudTrail |
| **Azure Key Vault** | Azure | Yes (FIPS 140-2 Level 2/3) | Keys, secrets, certificates in one service |
| **Google Cloud KMS** | GCP | Yes (FIPS 140-2 Level 3) | Symmetric/asymmetric, import your own keys |
| **HashiCorp Vault** | Open source / Enterprise | Optional (via HSM backend) | Secrets engine, dynamic secrets, transit encryption |

### Envelope Encryption

**Envelope encryption** is a pattern where data is encrypted with a data encryption key (DEK), and the DEK itself is encrypted with a key encryption key (KEK) managed by a KMS.

```
Envelope Encryption
───────────────────
Encrypt:
  1. Request a new DEK from KMS → receive plaintext DEK + encrypted DEK
  2. Encrypt data with plaintext DEK (AES-256-GCM)
  3. Store encrypted data + encrypted DEK together
  4. Discard the plaintext DEK from memory

Decrypt:
  1. Send the encrypted DEK to KMS → receive plaintext DEK
  2. Decrypt data with plaintext DEK
  3. Discard the plaintext DEK from memory
```

**Why envelope encryption?**

- The KEK never leaves the KMS (hardware security module)
- Key rotation only requires re-encrypting the DEK, not re-encrypting all data
- Access to the data requires both the encrypted DEK and KMS access

### Key Rotation

- **Symmetric keys (AES):** Rotate every 90 days or per policy; use envelope encryption to avoid re-encrypting all data
- **TLS certificates:** Rotate every 90 days or shorter; automate with ACME
- **Service account keys:** Rotate every 90 days; prefer short-lived tokens instead
- **Signing keys (JWT, artifacts):** Rotate periodically; support multiple active keys during transition
- **On compromise:** Rotate immediately and re-encrypt affected data

---

## Public Key Infrastructure (PKI)

### Certificate Authorities

A **Certificate Authority (CA)** issues digital certificates that bind a public key to an identity. The CA vouches for the identity of the certificate holder.

```
PKI Trust Chain
───────────────
Root CA (self-signed, offline)
  └── Intermediate CA (signed by Root)
        └── Leaf Certificate (signed by Intermediate)
              - Your server's TLS certificate
              - Your service's client certificate
```

### Certificate Validation

When validating a certificate, verify:

1. **Chain of trust** — The certificate chains to a trusted root CA
2. **Expiry** — The certificate is within its validity period
3. **Revocation** — The certificate has not been revoked (CRL or OCSP)
4. **Subject** — The certificate's subject (CN or SAN) matches the expected hostname
5. **Key usage** — The certificate is authorized for the intended purpose

### Internal PKI

For service-to-service communication within your infrastructure, run an internal CA:

- Use tools like **step-ca**, **cfssl**, or **HashiCorp Vault PKI** to manage an internal CA
- Issue short-lived certificates (24–72 hours) with automated renewal
- Do not add your internal CA to public trust stores — keep it isolated to your infrastructure
- Use a service mesh (Istio, Linkerd) to automate certificate issuance and rotation

---

## Cryptographic Hash Functions

### Hash Function Properties

| Property | Description |
|----------|-------------|
| **Pre-image resistance** | Given a hash, it is infeasible to find the original input |
| **Second pre-image resistance** | Given an input, it is infeasible to find a different input with the same hash |
| **Collision resistance** | It is infeasible to find any two different inputs with the same hash |

### Common Hash Algorithms

| Algorithm | Output Size | Status | Use Cases |
|-----------|------------|--------|-----------|
| **SHA-256** | 256 bits | ✅ Secure | Data integrity, digital signatures, HMAC |
| **SHA-384** | 384 bits | ✅ Secure | TLS cipher suites |
| **SHA-512** | 512 bits | ✅ Secure | High-security applications |
| **SHA-3** | 256/512 bits | ✅ Secure | Alternative to SHA-2, different construction |
| **BLAKE3** | 256 bits | ✅ Secure | High-performance hashing |
| **MD5** | 128 bits | ❌ Broken | Never use for security (collisions found) |
| **SHA-1** | 160 bits | ❌ Broken | Never use for security (collisions found) |

### HMAC

**HMAC (Hash-Based Message Authentication Code)** combines a hash function with a secret key to provide both integrity and authentication.

```
HMAC Computation
────────────────
HMAC(key, message) = hash((key ⊕ opad) || hash((key ⊕ ipad) || message))

Use Cases:
- API request signing (verify the request came from a trusted client)
- Webhook verification (verify the webhook payload is authentic)
- Token generation (CSRF tokens, signed cookies)
```

---

## Common Cryptographic Mistakes

| Mistake | Why It Is Dangerous | Correct Approach |
|---------|-------------------|-----------------|
| **Using ECB mode** | Identical plaintext blocks produce identical ciphertext blocks, revealing patterns | Use GCM or CTR mode with authentication |
| **Reusing nonces/IVs** | Nonce reuse in GCM completely breaks security | Generate a unique nonce for every encryption operation |
| **Hard-coding keys** | Keys in source code are easily extracted | Use a KMS or secrets manager |
| **Rolling your own crypto** | Custom algorithms are almost certainly flawed | Use well-tested libraries (libsodium, OpenSSL, platform APIs) |
| **Using MD5 or SHA-1** | Broken — known collision attacks | Use SHA-256 or better |
| **Encrypting without authenticating** | Ciphertext can be tampered with undetected | Use AEAD (AES-GCM, ChaCha20-Poly1305) |
| **Storing keys alongside data** | If storage is compromised, both data and keys are exposed | Store keys in KMS, separate from encrypted data |
| **Using RSA without padding** | Textbook RSA is deterministic and insecure | Use OAEP padding for encryption, PSS for signatures |
| **Ignoring key rotation** | Compromised keys remain valid indefinitely | Rotate keys on schedule and on compromise |
| **Insufficient key length** | Short keys can be brute-forced | AES-256 for symmetric, RSA-2048+ or ECDSA P-256 for asymmetric |

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [04-SUPPLY-CHAIN-SECURITY.md](04-SUPPLY-CHAIN-SECURITY.md) | Supply Chain Security | Dependency scanning, SBOM, artifact signing |
| [05-INFRASTRUCTURE-SECURITY.md](05-INFRASTRUCTURE-SECURITY.md) | Infrastructure Security | Secrets management, network policies, zero trust |
| [01-AUTHENTICATION.md](01-AUTHENTICATION.md) | Authentication | Password hashing, MFA, OAuth 2.0, JWT |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Cryptography documentation |
