# API Security Headers & Controls

For an **API**, use web headers plus API-specific controls. Some browser headers still matter, but API risk is more about **authn/authz, object-level authorization, rate limiting, schema validation, and token handling**.

## API security header baseline

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Type: application/json; charset=utf-8
X-Content-Type-Options: nosniff
Cache-Control: no-store
Pragma: no-cache
Referrer-Policy: no-referrer
Access-Control-Allow-Origin: https://app.example.com
Vary: Origin
```

Use **HSTS** at the API gateway or edge so clients always use HTTPS. Use `Content-Type: application/json` and `X-Content-Type-Options: nosniff` so browsers do not MIME-sniff API responses. For sensitive API responses, use `Cache-Control: no-store` so tokens, account data, and PII are not cached. OWASP’s REST guidance recommends HTTPS-only REST endpoints and appropriate CORS headers when browser-based cross-origin access is expected. ([OWASP Cheat Sheet Series][1])

## CORS for APIs

For browser-consumed APIs:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type, X-CSRF-Token
Access-Control-Max-Age: 600
Vary: Origin
```

Do not use wildcard CORS for authenticated APIs. OWASP recommends being as specific as possible with allowed origins and disabling CORS headers entirely when cross-domain browser calls are not expected. ([OWASP Cheat Sheet Series][1])

## CSRF / XSRF for APIs

CSRF matters **only when the browser automatically sends credentials**, such as cookies, client certificates, or HTTP auth. If the API uses `Authorization: Bearer <token>` and the token is not automatically attached by the browser, CSRF risk is much lower.

For cookie-authenticated APIs, require:

```http
Origin: https://app.example.com
X-CSRF-Token: <valid-session-bound-token>
Content-Type: application/json
```

OWASP recommends CSRF tokens, custom request headers, and origin/referer validation for CSRF protection, especially for AJAX/API endpoints. ([OWASP Cheat Sheet Series][2])

## API-specific controls I would prioritize

1. **Object-level authorization on every resource access.**
   Do not trust object IDs in paths, query strings, headers, or JSON bodies. OWASP ranks Broken Object Level Authorization as API1:2023 and says object-level authorization checks should be considered in every function that accesses a data source using a user-supplied ID. ([OWASP Foundation][3])

2. **Strong authentication and token validation.**
   Validate `iss`, `aud`, `exp`, scopes, token type, tenant, and signature or introspection result. Do not accept tokens minted for another API.

3. **Function-level authorization.**
   Separate “user can access this object” from “user can perform this action.” For example, reading an invoice is different from refunding it.

4. **Rate limits and resource limits.**
   Apply per-user, per-client, per-IP, per-route, and per-tenant limits. OWASP API4:2023 covers unrestricted resource consumption, and OWASP’s broken authentication examples highlight rate-limit bypass risks such as GraphQL batching. ([OWASP Foundation][4])

5. **Strict input validation.**
   Use schema validation for JSON bodies, query parameters, path parameters, and headers. Reject unknown fields where possible.

6. **Output filtering.**
   Return only fields the caller is authorized to see. This prevents broken object property-level authorization, where users can access or modify fields they should not control.

7. **Safe error handling.**
   Return consistent errors. Do not leak stack traces, SQL errors, internal hostnames, token validation details, or authorization policy internals.

8. **Request size, depth, and complexity limits.**
   Especially for GraphQL, search APIs, file uploads, batch endpoints, and report generation.

9. **Inventory and version management.**
   Know every API, route, method, owner, data classification, auth requirement, and deprecation date. OWASP API9:2023 calls out improper API inventory management as a risk area. ([API Security News][5])

10. **Logging and detection.**
    Log authentication failures, authorization denials, object ID tampering, rate-limit hits, suspicious query patterns, token failures, and admin actions. Avoid logging tokens or secrets.

## Practical API checklist

```text
HTTPS only
HSTS at edge
Strict CORS allowlist
CSRF tokens if cookie-authenticated
JWT/OAuth validation at every API
BOLA checks on every object access
Function-level authorization
Schema validation
Rate limits and quotas
Request body size limits
No sensitive caching
No secrets in logs
Centralized audit logging
API inventory and ownership
```

For an API, the most important distinction is this: **headers help, but authorization logic is the control that usually matters most.** OWASP’s 2023 API Top 10 puts Broken Object Level Authorization first for exactly that reason. ([OWASP Foundation][3])

[1]: https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html "REST Security Cheat Sheet"
[2]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html "Cross-Site Request Forgery Prevention Cheat Sheet"
[3]: https://owasp.org/API-Security/editions/2023/en/0x11-t10/ "OWASP Top 10 API Security Risks – 2023"
[4]: https://owasp.org/API-Security/editions/2023/en/0xa2-broken-authentication/ "API2:2023 Broken Authentication"
[5]: https://apisecurity.io/owasp-api-security-top-10/ "OWASP API Security Top 10 Vulnerabilities: 2023"

---

## Related

- [CORS Security Best Practices](cors-security-best-practices.md)
- [Web Security Headers & CSRF/XSRF Controls](web-security-headers-and-csrf.md)
- [Token Security (validation checklist)](token-security-guide.md)

See [References](references.md) for the full citation registry.
