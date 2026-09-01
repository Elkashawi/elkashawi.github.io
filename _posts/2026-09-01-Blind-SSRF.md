---
title: Blind SSRF via MCP OAuth Discovery — Redirect-Based Bypass of a Private-IP Guard
date: 2026-09-01 00:00:00 +0300
categories: [Bug Bounty]
tags: [SSRF, Cloud, API-Security, Business-Logic]
---

Sometimes the interesting part of a bug isn't finding it, it's proving it. This one is an SSRF, which by itself isn't rare. What made it worth a writeup is that the target actually had a guard against SSRF, and the guard looked reasonable at first glance. It just didn't hold up once redirects came into play.

## Where it lives

The target ships an MCP (Model Context Protocol) integration, and part of that flow is an OAuth discovery step. To support arbitrary MCP servers, the app exposes an endpoint that takes a server URL from the client and does the discovery handshake on its behalf:

```
POST /api/mcp/oauth/discover
```

Present on both the production host and a staging mirror, same behavior on both. The endpoint takes a JSON body with an `mcp_server_url` field, and the server goes and fetches whatever's in it, server-side, to look for OAuth support.

That's the shape of almost every SSRF I go looking for: a parameter that clearly wants a hostname or a URL, followed by a server-side fetch. Worth a poke every time.

## Step 1: Prove the server-side fetch is real

Before touching anything internal, I wanted an undeniable, out-of-band confirmation that the app was actually reaching out to arbitrary hosts. Pointed `mcp_server_url` at an Interactsh listener:

```http
POST /api/mcp/oauth/discover HTTP/2
Host: stage.[REDACTED]
Content-Type: application/json
Content-Length: 92

{"mcp_server_url":"https://[REDACTED-OOB-DOMAIN]/mcp"}
```

```http
HTTP/2 400
Content-Type: text/plain

Bad request: OAuth discovery failed: No authorization support detected
```

That 400 is fine, expected even, since my listener obviously doesn't speak OAuth discovery. What mattered was the DNS and HTTP interaction that showed up on the listener within seconds, originating from an AWS egress IP belonging to the target. No authentication, no user interaction, just a raw POST. The server really was fetching arbitrary URLs on my behalf.

## Step 2: Find the guard

Next obvious move: point it at cloud instance metadata directly.

```http
POST /api/mcp/oauth/discover HTTP/2
Host: stage.[REDACTED]
Content-Type: application/json
Content-Length: 79

{"mcp_server_url":"http://169.254.169.254/latest/meta-data/"}
```

```http
HTTP/2 400
Content-Type: text/plain

Bad request: URL targets a private/internal IP address
```

So there's a guard, and it does exactly what it says: rejects a request where `mcp_server_url` is literally a private or link-local address. This is where a lot of people would stop and move on. But a guard that only checks the _literal string you supplied_ has a pretty obvious hole in it: it never re-checks anything the server follows _after_ that.

## Step 3: Bypass it with a redirect

Instead of pointing straight at the metadata IP, I pointed at a public open redirector that 302s to it:

```http
POST /api/mcp/oauth/discover HTTP/2
Host: stage.[REDACTED]
Content-Type: application/json
Content-Length: 155

{"mcp_server_url":"http://httpbin.org/redirect-to?url=http%3A%2F%2F169.254.169.254%2Flatest%2Fmeta-data%2F&status_code=302"}
```

```http
HTTP/2 400
Content-Type: text/plain

Bad request: OAuth discovery failed: No authorization support detected
```

Look closely and that's the exact same "no authorization support detected" error from Step 1, the one I only ever got when the server successfully reached and got a reply from _something_. The literal hostname in the request (`httpbin.org`) is public, so the guard waved it through. Then the server followed the 302, landed on `169.254.169.254`, and got a response from it, guard never re-checked because it only runs once, on the value the client sent.

The response also came back in about a second.

## Step 4: Turn the timing into an oracle

A ~1 second response on its own is suggestive but not conclusive, so I built a control case: the same request structure, same redirect chain, but pointed at a dead internal host instead (`10.255.255.1:12345`, unreachable).

- Redirected to a live internal target (metadata service): ~1s response.
- Redirected to a dead/unreachable internal target: ~31s response (TCP connect timeout).

That gap is consistent and reliable enough to use as a port scanner. From an unauthenticated, external position, I can ask "is anything listening at this internal IP:port" and get a real answer, just by watching the clock. That turns a blind SSRF into something with actual internal-recon value even without response bodies being reflected anywhere.

## Step 5: Confirm production, not just staging

Same four steps, same payloads, just swapped the `Host` header from `stage.[REDACTED]` to `[REDACTED]`. Identical behavior, identical guard, identical bypass. Production was affected the whole time.

## Impact

- Unauthenticated SSRF, reachable from the public internet, no auth and no user interaction required.
- Confirmed on production, not only staging.
- The app's only SSRF mitigation, a private-IP blocklist, only inspects the literal hostname supplied in the request. It never re-validates the destination after a redirect, so any public 302 redirector bypasses it completely.
- The response-time differential (~1s live vs. ~31s dead) is a reliable oracle for scanning and fingerprinting the internal network and cloud metadata surface from the outside, with no credentials needed.

Worth being precise about what this _isn't_: it's a blind SSRF. Neither PoC reflects the internal target's response body or headers back to me, so I haven't pulled metadata content out of it. If the target's metadata service still speaks IMDSv1 (no token required), this class of bug is a well-trodden path to credential theft, but that's a separate step I haven't demonstrated, and I'm not claiming it here.

## Remediation

Wrote this up for the report, repeating it here since it's the actual fix, not just "add more regex":

- Re-validate the _final_ destination IP/host after following redirects, not just the URL the client originally supplied. Or simpler: disable automatic redirect-following on this fetch entirely and make the client resolve redirects itself.
- Resolve the hostname to an IP and check that IP, and every IP in a redirect chain, against private/internal/link-local ranges (`169.254.0.0/16`, RFC1918, loopback, and their IPv6-mapped equivalents) before _and_ after each hop.
- If the feature allows it, enforce a strict allowlist of permitted destination hosts/schemes for `mcp_server_url` instead of a denylist.
- Normalize timeouts and error responses for all outbound fetch failures. A consistent, generic failure response closes off the timing side-channel used for internal recon.

![alt text](/assets/images/Blind-SSRF.png)
