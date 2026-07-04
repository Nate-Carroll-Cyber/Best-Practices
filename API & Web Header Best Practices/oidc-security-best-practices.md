# OpenID Connect (OIDC) Security Best Practices

I assume you mean **OIDC — OpenID Connect**.

OIDC is the **authentication layer on top of OAuth 2.0**. OAuth answers “can this client access this resource?” OIDC answers “who is the user?” by issuing an **ID Token** and standard identity claims. OpenID Connect Core defines it as an identity layer on top of OAuth 2.0. ([OpenID Foundation][1])

Top 10 OIDC security things to do:

1. **Use Authorization Code Flow + PKCE.**
   Do not use Implicit Flow for modern apps. Use `response_type=code` with PKCE, including confidential clients where practical. RFC 9700 recommends PKCE as current OAuth security best practice. ([IETF Datatracker][2])

2. **Validate the ID Token fully.**
   Check signature, `iss`, `aud`, `exp`, `iat`, `nonce`, and `azp` when applicable. Do not just decode the JWT and trust the claims. OIDC Core requires clients to validate the ID Token and verify the `nonce` when used. ([OpenID Foundation][1])

3. **Use `nonce` for login replay protection.**
   `state` protects the OAuth redirect transaction; `nonce` binds the ID Token to the authentication request. Use both.

4. **Use `state` for CSRF and redirect-flow integrity.**
   Generate a high-entropy `state`, bind it to the browser session, and verify it on callback. OWASP highlights OAuth/OIDC redirect and authorization-server weaknesses, including redirect URI validation issues. ([OWASP Foundation][3])

5. **Use exact redirect URI matching.**
   No wildcards, partial string matching, open redirects, or user-controlled redirect destinations. A weak `redirect_uri` can send authorization codes or tokens to an attacker-controlled endpoint. ([OWASP Foundation][3])

6. **Validate the issuer and JWKS source.**
   Fetch signing keys only from the trusted issuer’s discovery metadata / JWKS endpoint. Do not let the token header’s `kid`, `jku`, or `x5u` point you to arbitrary attacker-controlled keys.

7. **Separate ID Tokens from Access Tokens.**
   ID Tokens are for the client to authenticate the user. Access tokens are for APIs. Your API should not accept an ID Token as an access token, and your app should not treat an access token as proof of login.

8. **Do not put authorization decisions solely in the ID Token.**
   Use ID Token claims for authentication context. For API authorization, validate access-token scopes, audience, tenant, roles, or permissions at the resource server.

9. **Require stronger authentication context for sensitive actions.**
   Use claims such as `acr` and `amr` when you need MFA, phishing-resistant auth, step-up authentication, or assurance-level checks. Do not assume every successful OIDC login has the same assurance.

10. **Handle logout and sessions deliberately.**
    OIDC login creates both an application session and an identity-provider session. Set secure session cookies, short lifetimes, re-authentication rules for sensitive actions, and clear local app sessions on logout.

A strong OIDC baseline:

```text
Flow: Authorization Code + PKCE
Response type: code
Scopes: openid profile email only when needed
Redirect URI: exact match only
State: required, high entropy, session-bound
Nonce: required for login
ID Token: validate signature, iss, aud, exp, iat, nonce, azp
Signing: RS256/ES256/EdDSA from trusted JWKS
Access token: audience-bound, scope-limited, API-validated
Storage: secure HttpOnly SameSite cookies for web sessions
```

For your assumption: **RS256 is fine for OIDC ID Tokens**, especially when many clients need to verify tokens using a public key. But the security win comes from **strict validation**, not just the algorithm. Configure the client to accept only the expected algorithm, issuer, audience, and JWKS for that specific provider.

[1]: https://openid.net/specs/openid-connect-core-1_0.html "OpenID Connect Core 1.0 incorporating errata set 2"
[2]: https://datatracker.ietf.org/doc/rfc9700/ "RFC 9700 - Best Current Practice for OAuth 2.0 Security"
[3]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/05.1-Testing_for_OAuth_Authorization_Server_Weaknesses "Testing for OAuth Authorization Server Weaknesses"

---

## Related

- [OAuth 2.0 Security Best Practices](oauth-security-best-practices.md)
- [JWT Security Best Practices](jwt-security-best-practices.md)
- [Token Security (ID vs access token separation)](token-security-guide.md)

See [References](references.md) for the full citation registry.
