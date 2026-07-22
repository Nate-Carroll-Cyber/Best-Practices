# Web Security Headers, CSRF, IDOR & SSRF

A secure web headers baseline plus CSRF/XSRF controls, followed by two access-and-request-layer topics that belong alongside them: IDOR (broken object-level authorization) and SSRF. This complements the CSP and CORS guides — see Related.

## Recommended web security headers

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-site
Cache-Control: no-store
```

For **HSTS**, enable it only after the entire site and its subdomains are ready for HTTPS — `includeSubDomains` + `preload` is effectively irreversible on the timescale of the preload list, so a subdomain that still needs HTTP will break. MDN notes that HSTS tells browsers to access the site only over HTTPS, and preload eligibility requires a `max-age` of at least one year (31536000) together with `includeSubDomains` and `preload`. ([MDN Web Docs][1])

For **cookies**, set authentication/session cookies like this:

```http
Set-Cookie: __Host-session=<value>; Path=/; Secure; HttpOnly; SameSite=Lax
```

The `__Host-` prefix is enforced by the browser: it requires `Secure`, `Path=/`, and no `Domain` attribute, which prevents a subdomain from writing the cookie. For highly sensitive apps use `SameSite=Strict` where it doesn't break the workflow. Cross-site OIDC/OAuth redirects may need `SameSite=Lax` (or narrowly scoped `SameSite=None; Secure` for specific cookies). MDN recommends `Secure` so cookies are only sent over HTTPS and `HttpOnly` to block JavaScript access to cookie values. ([MDN Web Docs][2])

## CSRF / XSRF controls

CSRF is not solved by one header. Use a combination of:

```text
SameSite cookies
CSRF token
Origin/Referer validation
Custom request headers for APIs
No state-changing GET requests
Re-authentication for sensitive actions
```

OWASP recommends synchronizer tokens or **signed** double-submit cookies for CSRF prevention — plain (unsigned) double-submit is weakened by subdomain cookie injection, so the token should be HMAC-bound to the session. Custom request headers are especially useful for AJAX/API endpoints because they force a CORS preflight and can't be set by a simple cross-site form post. ([OWASP Cheat Sheet Series][3])

A good CSRF token pattern (note: the CSRF cookie is intentionally **not** `HttpOnly`, since client JS must read it to echo into the header):

```http
Set-Cookie: __Host-csrf=<random>; Path=/; Secure; SameSite=Lax
X-CSRF-Token: <signed-or-session-bound-token>
```

For APIs using cookie-based auth, require:

```text
Origin: https://app.example.com
X-CSRF-Token: valid
Content-Type: application/json
```

And reject unsafe methods (`POST`, `PUT`, `PATCH`, `DELETE`) if the origin or token is missing. Note that `SameSite=Lax` already blocks cross-site sends of these methods in modern browsers — treat it as the primary layer and the token as defense-in-depth for older clients and same-site-subdomain attacks.

## Header-by-header purpose

| Control                           | Purpose                                |
| --------------------------------- | -------------------------------------- |
| `Strict-Transport-Security`       | Forces HTTPS after first trusted visit |
| `Content-Security-Policy`         | Reduces XSS and injection impact       |
| `X-Content-Type-Options: nosniff` | Prevents MIME sniffing                 |
| `Referrer-Policy`                 | Limits sensitive URL leakage           |
| `Permissions-Policy`              | Disables unnecessary browser features  |
| `frame-ancestors` in CSP          | Prevents clickjacking                  |
| `Cross-Origin-Opener-Policy`      | Isolates browsing context              |
| `Cross-Origin-Resource-Policy`    | Controls cross-origin resource loading |
| `Cache-Control: no-store`         | Prevents sensitive response caching    |
| Secure cookie attributes          | Protect session tokens                 |

Note on clickjacking: CSP `frame-ancestors 'none'` is the modern control and supersedes the legacy `X-Frame-Options: DENY`; send both only if you need to support older browsers that don't honor `frame-ancestors`. OWASP's HTTP Headers Cheat Sheet lists these headers as defense-in-depth controls for XSS, clickjacking, information disclosure, and browser-side abuse. ([OWASP Cheat Sheet Series][4])

## Practical baseline for an authenticated web app

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{random}'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-site
Cache-Control: no-store
Vary: Origin
```

And for cookies:

```http
Set-Cookie: __Host-session=<opaque-session-id>; Path=/; Secure; HttpOnly; SameSite=Lax
Set-Cookie: __Host-csrf=<csrf-value>; Path=/; Secure; SameSite=Lax
```

The short version:

> Implement HSTS, secure cookie attributes, CSRF/XSRF protections, CSP, restrictive CORS, clickjacking protection, MIME-sniffing protection, referrer controls, browser permission restrictions, cross-origin isolation headers where appropriate, and no-store caching for sensitive pages.

---

## Insecure Direct Object References (IDOR) & Access Control

IDOR is the web/testing name for what the OWASP API Security Top 10 calls **Broken Object Level Authorization (API1:2023)** — the single most common and impactful API vulnerability. ([OWASP Foundation][5])

**Authentication vs authorization:** authentication verifies identity ("who are you?"); authorization verifies permissions ("what can you do?"). IDOR is an authorization failure — authentication can be perfect, but if the per-object authorization check is missing or bypassable, IDOR exists.

### Prevention controls

**Backend authorization (required):**
- All authorization checks must occur on the backend; frontend or gateway-only checks are insufficient.
- Every request that references a resource must verify the authenticated user owns or is permitted to access that resource.
- Never rely solely on the resource ID being "hard to guess" — UUIDs leak in responses, JS bundles, metadata, and referrer headers. Unguessability is not authorization.

**Authorization placement:**
```text
1. Authenticate the user (verify identity via session/token)
2. Extract the requested resource ID from the request
3. Verify the user owns or has permission for that resource
4. Only then return or modify the resource
```

**Multi-tenant isolation:**
- Derive tenant context from a trusted source (a JWT `tenant_id` claim or server-side session), **not** from user-controlled input alone such as a request body or an `X-Tenant-ID` header the client can set.
- Match it against the authenticated user's tenant before any cross-tenant access is allowed.
- Test that modifying tenant identifiers cannot reach another organization's data.

### Detection methodology

**Setup:**
- Maintain at least two test accounts with different roles or tenant associations.
- Test both horizontal access (a peer's data) and vertical access (higher-privilege resources).

**Where identifiers hide (test all):**
- URL path: `/users/123`
- Query strings: `/profile?user_id=456`
- Request body (JSON/form): `{"account_id": 789}`
- Headers: `X-User-ID: 999`
- Cookies: session identifiers and custom tracking IDs
- Real-world PII used as keys: SSN, phone, email, VIN (already known to attackers)

**Bypass techniques to test:**

| Technique | Example | Why it works |
| --- | --- | --- |
| Parameter pollution | Send duplicate params; frameworks differ (PHP/last, ASP.NET concatenates, Express arrays, Flask/first) | The authorization check may validate only one occurrence |
| HTTP method swap | If `GET /resource/123` is blocked, try `POST`/`PUT`/`PATCH`/`DELETE` | Different method handlers may have inconsistent checks |
| Override headers | `X-HTTP-Method-Override: DELETE` | App may honor the override before authorization runs |
| Content-Type swap | Change `application/json` to `x-www-form-urlencoded` or `text/xml` | Different parsers normalize IDs differently |
| Encoded identifiers | Base64/double/triple-encode the ID | A WAF may decode once; authorization may skip re-validation |
| API version downgrade | Try `/api/v1/resource/123` when current is `/api/v3/` | Older versions may lack authorization patches |
| Array/wrapping injection | Send `{"ids": [123, 456]}` instead of one ID | API authorizes the first element, processes all |
| Path traversal | `/users/123/files/../../files/456` | Traversal logic may escape the authorization scope |

**Blind IDOR (no visible data):**
- Authorization may be missing even when no data is returned. Modify a resource ID and check whether an *action* succeeds (email changed, reset sent, setting updated). Confirm via side effects: logs, email inboxes, downstream state.

**Second-order IDOR:**
- An injected reference is used later (e.g., add a victim's email to a referral, then download a report containing the injected data). May take multiple requests to exploit.

---

## Server-Side Request Forgery (SSRF) Protections

SSRF occurs when an attacker can influence a server-side component to make HTTP requests to unintended destinations — internal services, cloud metadata endpoints, or external attacker-controlled servers. Any component that accepts a URL, hostname, or IP as input and makes a backend request is a potential SSRF vector. SSRF is **API7:2023** in the OWASP API Security Top 10. ([OWASP Foundation][6])

### Common SSRF entry points

```text
Webhook URLs and callback configurations
PDF/image renderers that fetch remote resources
URL preview or link unfurling features
File import from URL (CSV, XML, ICAL)
API integrations that proxy or forward requests
SSO/OAuth redirect and token exchange endpoints
AI/RAG pipelines fetching external documents or tool endpoints
```

### Prevention controls

**Input validation (required):**
- Parse and validate URLs before making any request; reject schemes other than `https` (and `http` only where explicitly required).
- Resolve the hostname to an IP *before* making the request, then validate the resolved IP against a denylist.
- Block RFC 1918 private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), loopback (`127.0.0.0/8`), link-local (`169.254.0.0/16`), and IPv6 equivalents (`::1`, `fc00::/7`, `fe80::/10`).
- Block cloud metadata endpoints explicitly: `169.254.169.254`, `fd00:ec2::254`, `metadata.google.internal`.
- Prefer an **allowlist** of permitted destinations over a denylist wherever the set of legitimate targets is known — denylists are routinely bypassed by the encoding tricks below.

**DNS rebinding defense:**
- Resolve DNS once, pin the resulting IP, validate *that* IP, and connect to it — do not let the HTTP client re-resolve between validation and connection (the classic TOCTOU that defeats naive validation).
- Set short DNS TTLs on internal infrastructure, but do not trust TTLs from external domains.

**Network-layer controls:**
- Place components that make outbound requests in a segment with egress filtering (firewall or security group).
- Allowlist specific outbound destinations where possible; default-deny all other egress.
- Route outbound HTTP through a forward proxy with destination logging and policy enforcement.

**Response handling:**
- Do not return raw responses from backend-fetched URLs to the client; parse and extract only the expected data (prevents blind-SSRF exfiltration).
- Limit response size, and follow redirects cautiously — re-validate each redirect target against the same denylist (a `302` to `169.254.169.254` is the most common bypass).
- Set aggressive timeouts to blunt internal port scanning.

### Cloud metadata hardening

| Cloud | Mitigation |
| --- | --- |
| AWS | Enforce IMDSv2 (`HttpTokens: required`): a session token must first be obtained via a `PUT` carrying `X-aws-ec2-metadata-token-ttl-seconds`, which a `GET`-based SSRF cannot satisfy |
| GCP | Metadata server requires the `Metadata-Flavor: Google` request header (and rejects requests with `X-Forwarded-For`); block `metadata.google.internal` at the network level |
| Azure | IMDS requires the `Metadata: true` header and rejects requests with an `X-Forwarded-For` header; restrict access via network policy where supported |

### Detection & testing

**Test cases:**
- Submit `http://127.0.0.1`, `http://[::1]`, `http://169.254.169.254/latest/meta-data/` in any URL input.
- Try alternate IP encodings: decimal (`http://2130706433`), octal (`http://0177.0.0.1`), hex (`http://0x7f000001`) — all resolve to `127.0.0.1`.
- DNS rebinding: a domain that alternates between an external IP and `127.0.0.1`.
- Redirect chains: an external URL that `302`-redirects to an internal address.
- Non-HTTP schemes: `file:///etc/passwd`, `gopher://`, `dict://`.

**Logging:**
- Log every outbound request from server-side components: destination IP, resolved hostname, port, scheme, response code, and the originating feature/user.
- Alert on any outbound connection to private IP ranges, metadata endpoints, or unexpected ports.

---

## Related

- [Content Security Policy (CSP) Best Practices](content-security-policy.md)
- [CORS Security Best Practices](cors-security-best-practices.md)
- [API Security Headers & Controls](api-security-headers-and-controls.md)
- [Token Security (audience, scope, tenant validation)](token-security-guide.md)

[1]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security "Strict-Transport-Security header - HTTP - MDN Web Docs"
[2]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies "Using HTTP cookies - MDN Web Docs"
[3]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html "Cross-Site Request Forgery Prevention Cheat Sheet"
[4]: https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html "HTTP Headers - OWASP Cheat Sheet Series"
[5]: https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/ "API1:2023 Broken Object Level Authorization"
[6]: https://owasp.org/API-Security/editions/2023/en/0xa7-server-side-request-forgery/ "API7:2023 Server Side Request Forgery"

See [References](references.md) for the full citation registry.
