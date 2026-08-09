---
title: Bypassing SMS & CAPTCHA Registration Controls Through a Native Magento REST API
date: 2026-08-09
categories: [Bug Bounty]
tags:
  [
    Magento,
    Broken-Authentication,
    Business-Logic,
    API-Security,
    Registration-Bypass,
  ]
---

While I was testing the registration flow of a Magento-based website, I ran into something pretty interesting: I could completely bypass all the frontend security controls just by talking directly to a native Magento REST API.

What made it interesting was that the frontend and the backend had two completely different ideas of what it took to create an account.

## What the "normal" registration looked like

When I went through the website like a regular user, creating an account meant jumping through several hoops:

- Entering a valid Chinese mobile phone number
- Passing CAPTCHA validation
- Verifying an SMS code
- Only then finishing registration through the frontend

All of that told me the developers clearly cared about stopping bots and abuse here. Fair enough, that's a reasonable thing to protect.

But it also made me curious about one thing: were these checks actually happening on the backend, or were they just a frontend formality that a determined attacker could skip entirely?

## Poking around past the frontend

Since I knew I was dealing with Magento, I went looking at what REST endpoints the platform exposes by default. One in particular caught my eye:

```http
POST /rest/default/V1/customers
```

This is a native, built-in Magento endpoint for creating customers. So I tried hitting it directly, with no authentication and without touching the site's actual registration page at all:

```http
POST /rest/default/V1/customers HTTP/2
Host: [REDACTED]
Content-Type: application/json
Accept: application/json

{
  "customer": {
    "email": "[REDACTED]",
    "firstname": "[REDACTED]",
    "lastname": "[REDACTED]"
  },
  "password": "[REDACTED]"
}
```

And it just... worked. The account was created instantly. No CAPTCHA, no SMS code, no phone number, no CSRF token, no login required. Nothing. The backend was quietly offering a second door into account creation, and that door didn't have any of the locks the front door had.

## Okay, but can this account actually log in?

Creating an account through a raw API call is fun, but I needed to know if it actually meant anything. Would this account behave like a real customer, or would it just sit there half-broken in the database?

Magento also ships a customer token endpoint, so I tried authenticating with the credentials I'd just created:

```http
POST /rest/default/V1/integration/customer/token HTTP/2
Host: [REDACTED]
Content-Type: application/json

{
  "username": "[REDACTED]",
  "password": "[REDACTED]"
}
```

Back came a fully valid JWT. That was the moment this stopped being a curiosity and started being a real problem. This wasn't some dead, half-created record. It was a fully functional customer account that could authenticate through Magento's normal auth mechanism, no different from someone who had gone through SMS verification the "proper" way.

## Making sure the rest of the site trusted it too

Next I wanted to check whether the site's actual authenticated APIs, the ones the frontend uses, would accept this token as legitimate. So I threw it at a customer info endpoint:

```http
POST /rest/cn/V1/customer/customerInfo HTTP/2
Host: [REDACTED]
Authorization: Bearer <JWT>
Token: <Frontend Token>
Content-Type: application/json

{}
```

It worked. It handed back the customer's profile data without hesitation. So I pushed a bit further and hit the address book endpoint too:

```http
POST /rest/cn/V1/applet/addressList HTTP/2
Host: [REDACTED]
Authorization: Bearer <JWT>
Token: <Frontend Token>
Content-Type: application/json

{}
```

Same result. It returned the address information for the account, no questions asked. At this point it was clear: this wasn't a weird edge-case account, it was treated exactly like a normal, verified customer everywhere on the site.

## So what was actually broken here?

The real issue wasn't a flaw hiding inside the Magento API itself. It was that the site had two completely different definitions of "how do you create an account," and only one of them was actually enforced.

Go through the website normally, and you need a phone number, a CAPTCHA, and an SMS code before you get an account. Hit the native Magento API instead, and all you need is an email, a name, and a password, and you walk out with a working JWT that opens the same authenticated endpoints as everyone else.

The second path skipped every single control the first path was built around.

## Why this actually matters

At first this might sound harmless. Plenty of sites let anyone sign up. But that's exactly the point: this site had deliberately added extra friction to registration. CAPTCHA and SMS verification were there for a reason, presumably to keep bots and abuse out.

The problem is that those protections only existed on paper if a second, unguarded path could get you the same outcome. Once I had this working, it opened the door to things like:

- Creating accounts automatically at scale
- Skipping SMS verification entirely
- Skipping CAPTCHA entirely
- Bulk account creation for whatever comes next
- Abusing any functionality that's gated behind "verified customer" status
- Undermining the whole anti-abuse setup the team had built

How bad this actually gets in practice depends on what a freshly created customer account can do elsewhere on the platform, but the core issue is the same regardless: the controls weren't really controls, they were just speed bumps you could drive around.
