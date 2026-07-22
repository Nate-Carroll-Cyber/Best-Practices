# Token Security: Types, Usage & Failure Modes

Tokens deserve their own section. Break them into **token types**, **where they are used**, and **how they fail**.

## Token types to distinguish

| Token                        | Purpose                                          | Common mistake                                        |
| ---------------------------- | ------------------------------------------------ | ----------------------------------------------------- |
| **Access token**             | Authorizes API access                            | Treated as login proof or accepted by the wrong API   |
| **Refresh token**            | Gets new access tokens                           | Stored like a normal access token or not rotated      |
| **ID token**                 | Proves user authentication to the client in OIDC | Sent to APIs as if it were an access token            |
| **Session token / cookie**   | Maintains web app session                        | Missing `Secure`, `HttpOnly`, `SameSite`              |
| **CSRF token**               | Proves browser request came from the app flow    | Confused with authentication                          |
| **API key**                  | Identifies app/client/integration                | Used as user authorization or placed in frontend code |
| **Reset / magic-link token** | One-time account action                          | Long-lived, reusable, or leaked in logs               |

## Core token security rules

**1. Do not mix token types.**
An **ID token** is for the client to authenticate the user. An **access token** is for the API. A **refresh token** is for the authorization server. Accepting the wrong token in the wrong place is a classic token-confusion problem. OIDC requires clients to validate ID token claims such as nonce when present, while APIs should validate access-token audience, scope, issuer, and expiry. ([OpenID Foundation][1])

**2. Validate access tokens completely.**
For JWT access or ID tokens, validate signature, allowed algorithm, issuer, audience, expiration, not-before time, token type, key ID, and required claims. RFC 8725 specifically warns against algorithm confusion and requires callers to control the accepted algorithms rather than trusting the token header. ([RFC Editor][2]) Not every access token is a JWT — for opaque tokens, validate via Token Introspection (RFC 7662) at the authorization server and apply the same issuer/audience/scope/expiry checks to the introspection response. ([RFC Editor][7])

**3. Prefer short-lived access tokens.**
Access tokens should usually be short-lived because bearer tokens are usable by whoever possesses them. OWASP's REST guidance notes that HTTPS protects credentials such as API keys and JWTs in transit, but short lifetimes reduce impact if tokens are still leaked through logs, browser storage, proxies, or compromised clients. ([OWASP Cheat Sheet Series][3])

**4. Rotate refresh tokens.**
Refresh tokens are high-value because they mint new access tokens. OAuth 2.0 Security Best Current Practice (RFC 9700) says refresh tokens for public clients must be sender-constrained or use refresh-token rotation — and rotation gives you reuse detection: a replayed old refresh token signals theft and should revoke the whole token family. ([IETF Datatracker][4])

**5. Use sender-constrained tokens for high-risk APIs.**
Bearer tokens are "bearer" because possession is enough. For machine-to-machine, partner, financial, healthcare, admin, or sensitive APIs, consider mTLS-bound tokens or DPoP. RFC 8705 defines OAuth mutual-TLS client authentication and certificate-bound access/refresh tokens, and RFC 9449 defines DPoP to sender-constrain OAuth tokens at the application layer. ([RFC Editor][5], [RFC Editor][8])

**6. Store browser tokens carefully.**
For browser apps, avoid long-lived tokens in `localStorage`. Prefer secure, HttpOnly, SameSite cookies for web sessions or refresh tokens, with CSRF controls when cookies are used. If using bearer access tokens in the browser, keep them short-lived and minimize exposure — or keep tokens out of the browser entirely with a backend-for-frontend (BFF) that holds them server-side behind a session cookie.

**7. Do not put sensitive data in tokens.**
A signed JWT is not encrypted. Anyone with the token can decode the claims. Store only what the recipient needs: subject, issuer, audience, expiration, scopes, tenant, roles, authorization context, and token ID. Avoid secrets, API keys, credentials, sensitive PII, or internal-only data.

**8. Use revocation where needed.**
JWTs are often stateless, which makes immediate revocation harder. Use short TTLs as the first control, then add revocation lists or deny lists using `jti` for high-risk sessions, terminated users, compromised devices, or refresh-token reuse events. OWASP describes blocklist-based approaches for JWT revocation. ([OWASP Cheat Sheet Series][6])

**9. Protect tokens from logs and URLs.**
Never put access tokens, refresh tokens, reset tokens, or session tokens in URLs. URLs leak through browser history, referrers, proxies, analytics, screenshots, and logs. Use authorization headers, secure cookies, or POST bodies for sensitive one-time flows.

**10. Separate token signing keys and rotate them.**
Use separate keys by environment, issuer, and token type where practical. Publish only public keys through trusted JWKS endpoints. Do not let `kid`, `jku`, or `x5u` headers cause the verifier to fetch arbitrary attacker-controlled keys.

## Practical lifetime guidance

A reasonable baseline:

```text
Access token:       5–15 minutes
ID token:           short-lived; tied to login session
Refresh token:      rotated; idle and absolute lifetime enforced
Session cookie:     idle timeout + absolute timeout
CSRF token:         session-bound or request-bound
Password reset:     one-time use; 10–30 minutes
Magic link:         one-time use; 5–15 minutes
API key:            scoped, monitored, rotated
```

## API validation checklist

For every protected API request:

```text
- Token is present in the expected location
- Signature or introspection result is valid
- Algorithm is explicitly allowed
- Issuer is trusted
- Audience matches this API
- Token has not expired
- Token type is access token, not ID token
- Scope/role/permission allows the action
- Tenant/client binding is correct
- Subject is active and not revoked
- Object-level authorization is enforced
```

## Strong pattern for modern apps

```text
Browser login:
OIDC Authorization Code + PKCE

Frontend session:
Secure, HttpOnly, SameSite cookie

API access:
Short-lived access token with exact audience and scopes

Refresh:
Rotating refresh token or server-side session

High-risk machine clients:
mTLS-bound or DPoP sender-constrained tokens

Storage:
No long-lived bearer tokens in localStorage

Logging:
Never log tokens; log token IDs, issuer, subject hash, client ID, and decision outcome instead
```

In one sentence:

> Use short-lived, audience-bound, scope-limited tokens; separate access, ID, refresh, session, CSRF, and reset-token use cases; validate JWTs fully; rotate refresh tokens; protect browser storage; never log or place tokens in URLs; use revocation for high-risk events; and apply sender-constrained tokens such as mTLS or DPoP for sensitive APIs.

[1]: https://openid.net/specs/openid-connect-core-1_0.html "OpenID Connect Core 1.0 incorporating errata set 2"
[2]: https://www.rfc-editor.org/info/rfc8725/ "RFC 8725: JSON Web Token Best Current Practices"
[3]: https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html "REST Security Cheat Sheet"
[4]: https://datatracker.ietf.org/doc/rfc9700/ "RFC 9700 - Best Current Practice for OAuth 2.0 Security"
[5]: https://www.rfc-editor.org/info/rfc8705/ "RFC 8705: OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens"
[6]: https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html "JSON Web Token for Java - OWASP Cheat Sheet Series"
[7]: https://www.rfc-editor.org/info/rfc7662/ "RFC 7662: OAuth 2.0 Token Introspection"
[8]: https://www.rfc-editor.org/info/rfc9449/ "RFC 9449: OAuth 2.0 Demonstrating Proof of Possession (DPoP)"

---

## Related

- [JWT Security Best Practices](jwt-security-best-practices.md)
- [OAuth 2.0 Security Best Practices](oauth-security-best-practices.md)
- [OpenID Connect (OIDC) Security Best Practices](oidc-security-best-practices.md)
- [API Keys](api-keys.md)

See [References](references.md) for the full citation registry.

