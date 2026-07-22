# Content Security Policy (CSP) Best Practices

> **Scope note for this repository.** The guidance below is generic **browser-document** hardening. This repo's shared ingress is an **API-only gateway** (FastMCP, a non-browser MCP API) that also fronts Keycloak, so it deliberately does **not** apply most of this. See [Deployment position for this repository](#deployment-position-for-this-repository) at the end before changing any ingress header. CSP resource directives (`script-src`, `style-src`, `object-src`, `eval`, …) govern how a browser executes an HTML document; they do nothing for JSON/MCP API responses and can break a browser UI (like Keycloak's) that shares the same ingress.

**Content Security Policy (CSP)** is one of the most important browser-side controls for reducing XSS impact. It tells the browser which scripts, styles, images, frames, connections, and other resources are allowed to load or execute. MDN describes CSP as a response header that lets site administrators control what resources a page may load, helping defend against cross-site scripting. ([MDN Web Docs][1])

Top CSP security practices:

1. **Send CSP as an HTTP response header**, not only a `<meta>` tag. OWASP recommends the `Content-Security-Policy` header and setting it on all responses, not just the index page. Note that a `<meta>`-delivered policy cannot use `frame-ancestors`, `report-uri`/`report-to`, or sandbox directives — another reason to prefer the header. ([OWASP Cheat Sheet Series][2])

2. **Start in report-only mode.** Use `Content-Security-Policy-Report-Only` first so you can collect violations without breaking production. You can send both the enforced and report-only headers at once — useful for testing a stricter policy against live traffic while a looser one is enforced.

3. **Avoid `unsafe-inline`.** Inline scripts are a major XSS risk. MDN recommends using nonces or hashes instead of allowing all inline scripts. ([GitHub][3])

4. **Avoid `unsafe-eval`.** It enables risky JavaScript execution patterns such as `eval()` and string-based code execution.

5. **Use nonces or hashes for scripts.** A nonce-based or hash-based CSP is stronger than broad host allowlists, which web.dev notes are often bypassable in real-world configurations. `'strict-dynamic'` is a complement, not a substitute: it only **propagates** trust from a script that was already authorized by a nonce or hash to the scripts *that* script loads — you must still generate the nonce/hash for the bootstrap script. In supporting browsers, `'strict-dynamic'` also causes host allowlists and `'self'` to be **ignored** for scripts. ([MDN Web Docs][1], [web.dev][4])

6. **Use `default-src 'self'` as a fallback baseline.** Then explicitly define `script-src`, `style-src`, `img-src`, `connect-src`, `frame-src`, and other directives. Remember that many directives (e.g. `form-action`, `base-uri`, `frame-ancestors`) do **not** fall back to `default-src` and must be set explicitly.

7. **Use `object-src 'none'`.** This blocks legacy plugin content such as Flash-style objects.

8. **Use `base-uri 'self'` or `base-uri 'none'`.** This prevents attackers from injecting a `<base>` tag that rewrites relative URLs (which can defeat nonce-based script loading).

9. **Use `frame-ancestors 'none'` or trusted origins only.** This helps prevent clickjacking and supersedes the older `X-Frame-Options` header for modern browsers.

10. **Report and monitor violations.** CSP is only effective if you tune it. Use `report-to` (with its companion `Reporting-Endpoints` header) to collect blocked script, frame, and connection attempts, and keep the deprecated `report-uri` alongside it for browsers that don't yet support `report-to`.

A strong starting point for a traditional server-rendered app. Note the `Reporting-Endpoints` header that defines the endpoint named by `report-to`:

```http
Reporting-Endpoints: csp-endpoint="https://example.com/csp-reports"
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM_PER_REQUEST}' 'strict-dynamic';
  style-src 'self' 'nonce-{RANDOM_PER_REQUEST}';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self';
  frame-ancestors 'none';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests;   /* optional — see note below */
  report-uri https://example.com/csp-reports;
  report-to csp-endpoint;
```

A few things to know about this policy:

- The nonce must be **cryptographically random and regenerated per response** — a static or predictable nonce provides no protection.
- With `'strict-dynamic'` in `script-src`, browsers that support it **ignore** host allowlists and `'self'` for scripts, relying on the nonce plus propagated trust instead. Older browsers ignore `'strict-dynamic'` and fall back to the nonce/host list, so the policy degrades safely.
- `img-src ... https:` is a broad allowlist for images; it's low-risk for images but don't carry that pattern over to `script-src`.
- `upgrade-insecure-requests` is **optional**. It's useful when migrating legacy pages that still contain `http://` sub-resource URLs, but a new HTTPS-only app doesn't need it, and it can complicate local development over `http://localhost`. Drop it unless you have mixed-content to migrate.
- **Report-only is a migration tool, not a destination.** `Content-Security-Policy-Report-Only` is right while tuning, but it enforces nothing — a policy that never graduates to the enforcing `Content-Security-Policy` header provides no protection.

For SPAs or apps with third-party analytics, tag managers, CDNs, or payment widgets, you'll need a more tailored policy. The important part is to **avoid broad allowlists like `script-src *` or `script-src https:`**, and to avoid using CSP as your only XSS defense — it's a mitigation that reduces impact, not a substitute for output encoding and input validation.

Good baseline:

```text
CSP header: required (HTTP header, not meta)
Mode: report-only first, enforce after tuning
Scripts: nonce/hash-based + strict-dynamic
Inline JS: blocked unless nonce/hash approved
Eval: blocked
Objects: blocked
Frames: restricted (frame-ancestors)
Forms: restricted (form-action)
Base URI: locked down
Reports: monitored (report-to + report-uri)
```

For a modern web app, include **CSP alongside secure cookies, CSRF protection, OIDC/OAuth hardening, JWT validation, input/output encoding, and dependency security**.

---

## Deployment position for this repository

The policy above is written for a browser application that emits HTML and controls its own response pipeline. This repository is **not** that: the shared ingress is an API-only gateway (FastMCP, a non-browser MCP API) that also fronts Keycloak. The current, deliberate policy is:

```http
Content-Security-Policy: frame-ancestors 'none'
```

That gives clickjacking protection (the one CSP directive that's meaningful without an HTML document of our own) and nothing more. It does **not** restrict scripts, styles, objects, forms, connections, or `eval()` — which is correct here, for these reasons:

- **A strict browser CSP doesn't fit an API.** `script-src`/`style-src`/`object-src`/etc. govern how a browser executes an HTML document. FastMCP returns JSON/MCP, not HTML, so those directives protect nothing on our own responses.
- **A global strict policy would break Keycloak.** Keycloak's login and admin UI are real browser documents served through the same ingress. A `default-src`/`script-src` policy applied at the shared ingress could break them, and the ingress can't tell Keycloak HTML apart from FastMCP JSON.
- **A nonce-based CSP can't be done in static ingress `addHeaders`.** The nonce must be cryptographically random, regenerated per HTML response, and injected into the matching `<script>`/`<style>` elements of that response — MDN confirms the per-response requirement. Static header configuration has no way to generate or inject a per-response nonce, so a nonce policy has to live in the application that renders the HTML.

**Recommended position:**

1. **Keep** `frame-ancestors 'none'` on the shared ingress.
2. **Do not** add a global `default-src`/`script-src` policy while Keycloak shares that ingress.
3. **Add** a complete nonce/hash-based CSP only when a dedicated browser UI exists with its own response pipeline (which can generate per-response nonces) and its own CSP reporting endpoint — scoped to that UI's routes, not the shared ingress.
4. **For FastMCP JSON/MCP responses**, spend the effort where it matters for an API: TLS, authentication, request/body-size limits, absence of permissive CORS, and cache-control headers — not browser resource directives. See [API Security Headers & Controls](api-security-headers-and-controls.md) and [Token Security](token-security-guide.md).

In short: the generic guidance above is sound for browser documents, but this repo serves an API-only gateway, and the two must be treated differently.

[1]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy "Content-Security-Policy (CSP) header - HTTP - MDN Web Docs"
[2]: https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html "Content Security Policy Cheat Sheet"
[3]: https://github.com/mdn/content/blob/main/files/en-us/web/http/reference/headers/content-security-policy/script-src/index.md?plain=1 "http - reference - headers - content-security-policy - script-src"
[4]: https://web.dev/articles/strict-csp "Mitigate cross-site scripting (XSS) with a strict Content Security Policy"

---

## Related

- [Web Security Headers & CSRF/XSRF Controls](web-security-headers-and-csrf.md)
- [CORS Security Best Practices](cors-security-best-practices.md)

See [References](references.md) for the full citation registry.
