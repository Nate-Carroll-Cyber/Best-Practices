# OAuth 2.0 Security Best Practices

For OAuth, think **secure protocol profile**, not just token signing. RS256 matters if your access tokens are JWTs, but OAuth security is mostly about **flow selection, redirect handling, client authentication, token handling, and replay prevention**.

Top 10 OAuth security practices:

1. **Use Authorization Code Flow with PKCE.**
   For user-facing apps, this should be your default. PKCE was designed to mitigate authorization-code interception attacks, and modern OAuth guidance pushes the ecosystem toward code flow + PKCE rather than older browser-token flows. ([IETF Datatracker][1])

   Flow choice is the first security decision, and it's driven by three questions: is a user involved, can the client keep a secret, and how does the app interact with APIs. No-user backend jobs and automated services use the **Client Credentials Flow**; web apps with a secure backend use the **Authorization Code Flow** so tokens stay on the server and out of the browser; SPAs and native/mobile apps use **Authorization Code with PKCE**, driving redirects through the system browser. The **Resource Owner Password Credentials Flow** is a last resort, appropriate only when redirects are impossible and the client is fully trusted with the user's credentials. Kiosks and IoT devices that can't run a browser login use **Client-Initiated Backchannel Authentication (CIBA)** instead, and an app needing tokens for multiple APIs repeats the authorization request with a different audience per resource server.

   **How PKCE works:** public clients can't safely store a secret, so PKCE substitutes dynamic values. The client generates a random *code verifier* and sends its SHA-256 hash — the *code challenge* — during authorization. After user approval, the client redeems the authorization code together with the original verifier, and the server issues tokens only if the verifier matches the challenge. This defeats code-interception attacks (only the legitimate client knows the verifier) and adds a CSRF binding to the client instance. OAuth 2.1 mandates PKCE for all Authorization Code flows and replaces the Implicit flow for public clients.

2. **Avoid Implicit Flow and Resource Owner Password Credentials.**
   Do not put access tokens directly in browser redirects, and do not have apps collect user passwords. RFC 9700 deprecates less secure OAuth modes and updates the original OAuth 2.0 security guidance. ([RFC Editor][2])

3. **Use exact redirect URI matching.**
   Register precise redirect URIs and reject wildcards or partial matching. Redirect URI weaknesses are one of the most common ways OAuth implementations leak codes or tokens.

   Exact matching is a primary defense against **COAT (Cross-App Auth Account Takeover).** On workflow-automation and virtual-assistant platforms, OAuth roles can invert: an untrusted third-party app acts as the Authorization Server while the platform acts as the OAuth client, creating **Client ID Confusion**. In a COAT attack, a malicious app intercepts an authorization code intended for a legitimate app, the platform redeems that code at the attacker's endpoint, and the attacker gains access to linked services such as email or cloud storage. The attacker captures the victim's authorization code and injects it into a connection they control.

4. **Use `state`, and use `nonce` for OpenID Connect.**
   `state` protects the OAuth redirect flow from CSRF and mix-up style issues. If you are doing login with OIDC, validate `nonce` in the ID token to bind the authentication response to the browser session.

   `state` also blocks **CORF (Cross-app Auth Request Forgery):** an attacker initiates an auth flow, lures the victim into completing it, and forges a connection between the attacker's tool and the victim's account. CORF exploits platforms that store auth context in the redirect URI rather than the user session — which bypasses CSRF protections including PKCE. Unlike COAT (a connection/context mix-up), CORF targets user-session confusion. Defenses: bind each auth session to the initiating user session, put unique context identifiers in the `state` parameter, and avoid sharing auth providers across tools or tenants.

   The same session-binding discipline addresses broader **cross-user vulnerabilities**. *Session fixation* lets an attacker plant a known session ID and take over the session after the victim logs in; *redirect manipulation* alters redirect URIs or parameters to leak tokens or gain unauthorized access. Weak session binding and improper `state` use are the shared root cause behind both CORF and COAT. Mitigate with unique session IDs, app- and tenant-specific `state` binding, and session-aware PKCE.

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

   Rigorous audience and caller-identity validation is what stops the **Confused Deputy problem:** a trusted agent tricked into misusing its privileges on an attacker's behalf. It arises precisely when servers fail to enforce token audience, scopes, or caller identity, letting attackers exploit loosely scoped or impersonatable tokens to reach unauthorized resources. In multi-hop AI/agent workflows the impact is account compromise and privilege escalation. Beyond the validations above, defend it by binding user identity to specific agents, limiting agent permissions, and adding context-aware access control, just-in-time credentials, verifiable credentials, mutual TLS, and behavioral monitoring.

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
