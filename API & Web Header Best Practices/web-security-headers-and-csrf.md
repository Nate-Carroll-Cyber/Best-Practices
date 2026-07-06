# Web Security Headers & CSRF/XSRF Controls

Yes — after JWT/OAuth/OIDC, CSP, and CORS, I'd add a **secure web headers baseline** plus **CSRF/XSRF controls**.

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

For **HSTS**, use it only after the entire site and subdomains are ready for HTTPS. MDN notes that HSTS tells browsers to access the site only over HTTPS, and preload eligibility requires at least a one-year `max-age` with `includeSubDomains`. ([MDN Web Docs][1])

For **cookies**, set authentication/session cookies like this:

```http
Set-Cookie: __Host-session=<value>; Path=/; Secure; HttpOnly; SameSite=Lax
```

For highly sensitive apps, use `SameSite=Strict` where it does not break the workflow. For cross-site OIDC/OAuth redirects, you may need `SameSite=Lax` or narrowly scoped `SameSite=None; Secure` for specific cookies. MDN recommends `Secure` so cookies are only sent over HTTPS and `HttpOnly` to block JavaScript access to cookie values. ([MDN Web Docs][2])

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

OWASP recommends synchronizer tokens or signed double-submit cookies for CSRF prevention, and notes that custom request headers are especially useful for AJAX/API endpoints because they force a CORS preflight and are harder for simple cross-site form posts to abuse. ([OWASP Cheat Sheet Series][3])

A good CSRF token pattern:

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

And reject unsafe methods like `POST`, `PUT`, `PATCH`, and `DELETE` if the origin or token is missing.

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

OWASP's HTTP Headers Cheat Sheet lists these headers as defense-in-depth controls for issues such as XSS, clickjacking, information disclosure, and browser-side abuse. ([OWASP Cheat Sheet Series][4])

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

The short version for your checklist:

> Implement HSTS, secure cookie attributes, CSRF/XSRF protections, CSP, restrictive CORS, clickjacking protection, MIME-sniffing protection, referrer controls, browser permission restrictions, cross-origin isolation headers where appropriate, and no-store caching for sensitive pages.

---

## Insecure Direct Object References (IDOR) & Access Control

**Authentication vs Authorization:** Authentication verifies identity ("who are you"); authorization verifies permissions ("what can you do?"). IDOR is an authorization failure—authentication can be perfect, but if authorization checks are missing or bypassable, IDOR exists.

### Prevention Controls

**Backend Authorization (Required):**
- All authorization checks must occur on the backend; frontend or gateway-only checks are insufficient
- Every request that references a resource must verify the authenticated user owns or is permitted to access that resource
- Never rely solely on the resource ID being "hard to guess" (UUIDs leak in responses, JS files, metadata, referrer headers)

**Authorization Placement:**
```text
1. Authenticate the user (verify identity via session/token)
2. Extract the requested resource ID from the request
3. Verify the user owns or has permission for that resource
4. Only then return or modify the resource
```

**Multi-Tenant Isolation:**
- Tenant context (subdomain, header `X-Tenant-ID`, JWT claim `tenant_id`) must be:
  - Verified from a trusted source (not user-controlled input alone)
  - Matched against the authenticated user's tenant before any cross-tenant access is allowed
- Test that modifying tenant identifiers cannot access other organizations' data

### Detection Methodology

**Setup:**
- Maintain at least two test accounts with different roles or tenant associations
- Test both horizontal access (peer data) and vertical access (higher-privilege resources)

**Where Identifiers Hide (Test All):**
- URL path: `/users/123`
- Query strings: `/profile?user_id=456`
- Request body (JSON/form): `{"account_id": 789}`
- Headers: `X-User-ID: 999`
- Cookies: Session identifiers and custom tracking IDs
- Real-world PII: SSN, phone, email, VIN (already known to attackers)

**Bypass Techniques to Test:**

| Technique | Example | Why It Works |
| --- | --- | --- |
| **Parameter Pollution** | Send duplicate params; frameworks handle them differently (PHP takes last, ASP.NET concatenates, Node/Express arrays, Flask takes first) | Authorization check may only validate first param |
| **HTTP Method Swap** | If `GET /resource/123` is blocked, try `POST`, `PUT`, `PATCH`, `DELETE` | Different method handlers may have inconsistent checks |
| **Override Headers** | `X-HTTP-Method-Override: DELETE` | Application may honor override before authorization |
| **Content-Type Swap** | Change `application/json` to `application/x-www-form-urlencoded` or `text/xml` | Different parsers may normalize or parse IDs differently |
| **Encoded Identifiers** | Base64-encode, decode, modify, re-encode; try double/triple encoding | WAFs may decode once; authorization may skip validation |
| **API Version Downgrade** | Try `/api/v1/resource/123` if current is `/api/v3/` | Older versions may lack authorization patches |
| **JSON Array Injection** | Send `{"ids": [123, 456]}` instead of single ID | API authorizes first element, processes all |
| **Path Traversal** | `/users/123/files/../../../files/456` | Directory traversal logic may bypass authorization scope |

**Blind IDOR (No Visible Data):**
- Authorization may be missing even if you cannot see returned data
- Test by modifying a resource ID and checking if an action succeeds (email changed, password reset sent, setting updated)
- Check application logs, email inboxes, or side effects for confirmation

**Second-Order IDOR:**
- Injected reference is used later (e.g., add victim's email to referral, download report containing injected data)
- May require multiple requests to exploit

---

## Related

- [Content Security Policy (CSP) Best Practices](content-security-policy.md)
- [CORS Security Best Practices](cors-security-best-practices.md)
- [API Security Headers & Controls](api-security-headers-and-controls.md)

See [References](references.md) for the full citation registry.

[1]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security "Strict-Transport-Security header - HTTP - MDN Web Docs"
[2]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies "Using HTTP cookies - MDN Web Docs - Mozilla"
[3]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html "Cross-Site Request Forgery Prevention Cheat Sheet"
[4]: https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html "HTTP Headers - OWASP Cheat Sheet Series"
