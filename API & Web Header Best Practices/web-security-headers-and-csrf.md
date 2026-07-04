# Web Security Headers & CSRF/XSRF Controls

Yes — after JWT/OAuth/OIDC, CSP, and CORS, I’d add a **secure web headers baseline** plus **CSRF/XSRF controls**.

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

OWASP’s HTTP Headers Cheat Sheet lists these headers as defense-in-depth controls for issues such as XSS, clickjacking, information disclosure, and browser-side abuse. ([OWASP Cheat Sheet Series][4])

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

[1]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security "Strict-Transport-Security header - HTTP - MDN Web Docs"
[2]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies "Using HTTP cookies - MDN Web Docs - Mozilla"
[3]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html "Cross-Site Request Forgery Prevention Cheat Sheet"
[4]: https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html "HTTP Headers - OWASP Cheat Sheet Series"

---

## Related

- [Content Security Policy (CSP) Best Practices](content-security-policy.md)
- [CORS Security Best Practices](cors-security-best-practices.md)
- [API Security Headers & Controls](api-security-headers-and-controls.md)

See [References](references.md) for the full citation registry.
