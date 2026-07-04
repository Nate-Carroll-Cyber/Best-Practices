# CORS Security Best Practices

Yes — for a modern web/API stack, CORS should be **explicit, narrow, and boring**. CORS controls which browser-based origins are allowed to read responses from your API; it is **not** an authentication or CSRF defense by itself. PortSwigger notes that CORS relaxes the same-origin policy but does not protect against CSRF. ([PortSwigger][1])

Top CORS security practices:

1. **Allow only trusted origins.**
   Do not reflect any `Origin` header back automatically. Maintain a strict allowlist such as:

```text
https://app.example.com
https://admin.example.com
```

2. **Avoid `Access-Control-Allow-Origin: *` for sensitive APIs.**
   Wildcard CORS allows any website to read the response for non-credentialed requests. For public static assets, it can be fine; for authenticated APIs, avoid it. MDN defines `Access-Control-Allow-Origin` as the header that indicates whether the response can be shared with requesting code from a given origin. ([MDN Web Docs][2])

3. **Never combine wildcard origin with credentials.**
   Browsers reject `Access-Control-Allow-Origin: *` when credentials are included, and OWASP notes that credentialed CORS requires specific trusted origins rather than wildcard access. ([OWASP Cheat Sheet Series][3])

4. **Use `Access-Control-Allow-Credentials: true` only when required.**
   If you do not need cookies, client certs, or browser-managed credentials, omit this header. MDN states that `true` is the only valid value and recommends omitting the header when credentials are not needed. ([MDN Web Docs][4])

5. **Restrict methods.**
   Do not return every method by default. Use only what the API needs:

```http
Access-Control-Allow-Methods: GET, POST
```

6. **Restrict headers.**
   Allow only required request headers, such as:

```http
Access-Control-Allow-Headers: Authorization, Content-Type
```

MDN describes `Access-Control-Allow-Headers` as the response header used to indicate which custom request headers are supported in CORS requests. ([MDN Web Docs][5])

7. **Do not allow `null` origins.**
   Treat `Origin: null` as untrusted unless you have a very specific use case. It can appear from sandboxed documents, local files, or unusual browser contexts.

8. **Validate exact origins, not suffixes.**
   Avoid checks like “ends with `example.com`,” because `evil-example.com` or malformed origins can slip through. Parse the origin and match exact scheme, host, and port.

9. **Keep preflight caching reasonable.**
   `Access-Control-Max-Age` can reduce preflight noise, but overly long values can make policy mistakes persist in browsers.

10. **Pair CORS with CSRF controls.**
    If cookies are used, also use SameSite cookies, CSRF tokens, origin/referer checks, and proper session security. OWASP specifically warns that enabling credentialed CORS still requires allowing only select origins you control. ([OWASP Cheat Sheet Series][3])

A good baseline for an authenticated API:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type, X-CSRF-Token
Access-Control-Max-Age: 600
Vary: Origin
```

For a public, unauthenticated API:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET
Access-Control-Allow-Headers: Content-Type
```

For your security checklist, I’d phrase it as:

> Implement a restrictive CORS policy using exact trusted-origin allowlists, no wildcard access for authenticated APIs, no automatic origin reflection, minimal allowed methods and headers, credentialed CORS only when required, and CSRF protections for cookie-based sessions.

[1]: https://portswigger.net/web-security/cors "What is CORS (cross-origin resource sharing)?"
[2]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Origin "Access-Control-Allow-Origin header - HTTP - MDN Web Docs"
[3]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html "Cross-Site Request Forgery Prevention Cheat Sheet"
[4]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Credentials "Access-Control-Allow-Credentials header - MDN Web Docs"
[5]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Headers "Access-Control-Allow-Headers header - HTTP - MDN Web Docs"

---

## Related

- [Web Security Headers & CSRF/XSRF Controls](web-security-headers-and-csrf.md)
- [API Security Headers & Controls](api-security-headers-and-controls.md)

See [References](references.md) for the full citation registry.
