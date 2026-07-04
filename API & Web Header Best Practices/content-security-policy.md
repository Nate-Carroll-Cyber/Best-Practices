# Content Security Policy (CSP) Best Practices

Yes — **Content Security Policy (CSP)** is one of the most important browser-side controls for reducing XSS impact. It tells the browser which scripts, styles, images, frames, connections, and other resources are allowed to load or execute. MDN describes CSP as a response header that lets site administrators control what resources a page may load, helping defend against cross-site scripting. ([MDN Web Docs][1])

Top CSP security practices:

1. **Send CSP as an HTTP response header**, not only a `<meta>` tag. OWASP recommends the `Content-Security-Policy` header and setting it on all responses, not just the index page. ([OWASP Cheat Sheet Series][2])

2. **Start in report-only mode.** Use `Content-Security-Policy-Report-Only` first so you can collect violations without breaking production.

3. **Avoid `unsafe-inline`.** Inline scripts are a major XSS risk. MDN recommends using nonces or hashes instead of allowing all inline scripts. ([GitHub][3])

4. **Avoid `unsafe-eval`.** It enables risky JavaScript execution patterns such as `eval()` and string-based code execution.

5. **Use nonces or hashes for scripts.** A nonce-based or hash-based CSP is stronger than broad host allowlists, which web.dev notes are often bypassable in real-world configurations. ([web.dev][4])

6. **Use `default-src 'self'` as a fallback baseline.** Then explicitly define `script-src`, `style-src`, `img-src`, `connect-src`, `frame-src`, and other directives.

7. **Use `object-src 'none'`.** This blocks legacy plugin content such as Flash-style objects.

8. **Use `base-uri 'self'` or `base-uri 'none'`.** This prevents attackers from injecting a `<base>` tag that rewrites relative URLs.

9. **Use `frame-ancestors 'none'` or trusted origins only.** This helps prevent clickjacking and replaces older `X-Frame-Options` patterns for modern browsers.

10. **Report and monitor violations.** CSP is only effective if you tune it. Use `report-to` or `report-uri` to collect blocked script, frame, and connection attempts.

A strong starting point for a traditional server-rendered app:

```http
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
  upgrade-insecure-requests;
  report-to csp-endpoint;
```

For SPAs or apps with third-party analytics, tag managers, CDNs, or payment widgets, you will need a more tailored policy. The important part is to **avoid broad allowlists like `script-src *` or `https:`**, and to avoid using CSP as your only XSS defense.

Good baseline:

```text
CSP header: required
Mode: report-only first, enforce after tuning
Scripts: nonce/hash-based
Inline JS: blocked unless nonce/hash approved
Eval: blocked
Objects: blocked
Frames: restricted
Forms: restricted
Reports: monitored
```

So yes: for a modern web app, I would include **CSP alongside secure cookies, CSRF protection, OIDC/OAuth hardening, JWT validation, input/output encoding, and dependency security**.

[1]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy "Content-Security-Policy (CSP) header - HTTP - MDN Web Docs"
[2]: https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html "Content Security Policy Cheat Sheet"
[3]: https://github.com/mdn/content/blob/main/files/en-us/web/http/reference/headers/content-security-policy/script-src/index.md?plain=1 "http - reference - headers - content-security-policy - script-src"
[4]: https://web.dev/articles/strict-csp "Mitigate cross-site scripting (XSS) with a strict Content ..."

---

## Related

- [Web Security Headers & CSRF/XSRF Controls](web-security-headers-and-csrf.md)
- [CORS Security Best Practices](cors-security-best-practices.md)

See [References](references.md) for the full citation registry.
