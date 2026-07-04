# OAuth 2.0 Security Best Practices

For OAuth, think **secure protocol profile**, not just token signing. RS256 matters if your access tokens are JWTs, but OAuth security is mostly about **flow selection, redirect handling, client authentication, token handling, and replay prevention**.

Top 10 OAuth security practices:

1. **Use Authorization Code Flow with PKCE.**
   For user-facing apps, this should be your default. PKCE was designed to mitigate authorization-code interception attacks, and modern OAuth guidance pushes the ecosystem toward code flow + PKCE rather than older browser-token flows. ([IETF Datatracker][1])

2. **Avoid Implicit Flow and Resource Owner Password Credentials.**
   Do not put access tokens directly in browser redirects, and do not have apps collect user passwords. RFC 9700 deprecates less secure OAuth modes and updates the original OAuth 2.0 security guidance. ([RFC Editor][2])

3. **Use exact redirect URI matching.**
   Register precise redirect URIs and reject wildcards or partial matching. Redirect URI weaknesses are one of the most common ways OAuth implementations leak codes or tokens.

4. **Use `state`, and use `nonce` for OpenID Connect.**
   `state` protects the OAuth redirect flow from CSRF and mix-up style issues. If you are doing login with OIDC, validate `nonce` in the ID token to bind the authentication response to the browser session.

5. **Remember: OAuth is authorization, not authentication.**
   For login, use **OpenID Connect** on top of OAuth. Do not treat a raw OAuth access token as proof that the user authenticated unless you are following an OIDC profile and validating the ID token correctly.

6. **Authenticate confidential clients strongly.**
   Use `private_key_jwt`, mutual TLS, or strong client secrets stored server-side. Never put a client secret in a SPA, mobile app, desktop app, or other public client.

7. **Use short-lived access tokens and rotate refresh tokens.**
   RFC 9700 describes refresh-token rotation, where the authorization server issues a new refresh token on each refresh and invalidates the previous one, allowing reuse detection if a stolen refresh token is used. ([RFC Editor][3])

8. **Use sender-constrained tokens for higher-risk systems.**
   Bearer tokens are dangerous because possession is enough. Use DPoP or mTLS where appropriate so a stolen token cannot be replayed without the associated private key or client certificate. DPoP is standardized in RFC 9449 for sender-constraining OAuth access and refresh tokens. ([RFC Editor][4])

9. **Validate access tokens at the resource server.**
   Every API should validate issuer, audience, expiration, scopes/permissions, token type, signature or introspection result, and any tenant or client constraints. Do not accept an access token minted for another API.

10. **Use least privilege scopes and explicit resource indicators.**
    Avoid broad scopes like `admin`, `*`, or long-lived offline access unless truly required. Issue tokens for a specific API/resource and purpose, not for “everything the user can do.”

For more sensitive environments, add **Pushed Authorization Requests (PAR)** so authorization request details are sent directly from the client to the authorization server instead of being exposed and tampered with through the browser URL. PAR is defined in RFC 9126. ([IETF Datatracker][5])

A strong baseline looks like this:

```text
Flow: Authorization Code + PKCE
Login: OpenID Connect, not raw OAuth
Redirects: exact-match only
Client auth: private_key_jwt or mTLS for confidential clients
Access tokens: short-lived, audience-bound, scope-limited
Refresh tokens: rotated or sender-constrained
Replay protection: DPoP or mTLS for high-risk apps
Browser storage: avoid long-lived tokens in localStorage
Resource server: validate iss, aud, exp, scope, token type
High assurance: PAR/JAR, strong consent, audit logging
```

For a modern enterprise API, I’d start with: **OIDC + Authorization Code + PKCE + exact redirects + short-lived JWT access tokens signed with RS256/ES256 + refresh-token rotation + strict audience validation + DPoP or mTLS for high-risk clients.**

[1]: https://datatracker.ietf.org/doc/html/rfc7636 "RFC 7636 - Proof Key for Code Exchange by OAuth Public ..."
[2]: https://www.rfc-editor.org/rfc/rfc9700.html "RFC 9700: Best Current Practice for OAuth 2.0 Security"
[3]: https://www.rfc-editor.org/info/rfc9700/ "RFC 9700: Best Current Practice for OAuth 2.0 Security"
[4]: https://www.rfc-editor.org/info/rfc9449/ "OAuth 2.0 Demonstrating Proof of Possession (DPoP)"
[5]: https://datatracker.ietf.org/doc/html/rfc9126 "RFC 9126 - OAuth 2.0 Pushed Authorization Requests"

---

## Related

- [OpenID Connect (OIDC) Security Best Practices](oidc-security-best-practices.md)
- [Token Security (refresh rotation, sender-constrained tokens)](token-security-guide.md)
- [TLS & mTLS for APIs](tls-and-mtls-for-apis.md)

See [References](references.md) for the full citation registry.
