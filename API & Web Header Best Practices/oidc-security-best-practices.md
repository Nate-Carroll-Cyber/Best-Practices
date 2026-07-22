# OpenID Connect (OIDC) Security Best Practices

OIDC is the **authentication layer on top of OAuth 2.0**. OAuth answers "can this client access this resource?" OIDC answers "who is the user?" by issuing an **ID Token** and standard identity claims. OpenID Connect Core defines it as an identity layer on top of OAuth 2.0. ([OpenID Foundation][1])

Top 10 OIDC security things to do:

1. **Use Authorization Code Flow + PKCE.**
   Do not use Implicit or Hybrid Flow for modern apps. Use `response_type=code` with PKCE for all clients, including confidential clients. RFC 9700 recommends PKCE as current OAuth security best practice. ([IETF Datatracker][2])

2. **Validate the ID Token fully.**
   Check signature, `iss`, `aud`, `exp`, `iat`, `nonce`, and `azp` when applicable (`azp` matters when the token has multiple audiences). Do not just decode the JWT and trust the claims. OIDC Core requires clients to validate the ID Token and verify the `nonce` when one was sent. If you requested `max_age` or a specific `acr`, also validate `auth_time` and `acr` in the token. ([OpenID Foundation][1])

3. **Use `nonce` for login replay protection.**
   `state` protects the OAuth redirect transaction; `nonce` binds the ID Token to the authentication request. Use both. OIDC Core mandates `nonce` in Implicit/Hybrid flows; in code flow it is optional per spec but should be treated as required — it is your ID-token replay defense and (per RFC 9700) can also serve the CSRF role.

4. **Use `state` for CSRF and redirect-flow integrity.**
   Generate a high-entropy `state`, bind it to the browser session, and verify it on callback. If your provider supports it, also validate the `iss` authorization-response parameter (RFC 9207) — that, not `state`, is the defense against mix-up attacks when a client talks to multiple providers. OWASP highlights OAuth/OIDC redirect and authorization-server weaknesses, including redirect URI validation issues. ([OWASP Foundation][3])

5. **Use exact redirect URI matching.**
   No wildcards, partial string matching, open redirects, or user-controlled redirect destinations. A weak `redirect_uri` can send authorization codes or tokens to an attacker-controlled endpoint. ([OWASP Foundation][3])

6. **Validate the issuer and JWKS source, and pin accepted algorithms.**
   Fetch signing keys only from the trusted issuer's discovery metadata / JWKS endpoint, and verify the discovery document's `issuer` matches the URL you resolved it from. Do not let the token header's `kid`, `jku`, or `x5u` point you to arbitrary attacker-controlled keys. Configure an explicit algorithm allowlist (e.g. RS256 or ES256 only) and reject `alg: none` and unexpected algorithm switches.

7. **Separate ID Tokens from Access Tokens.**
   ID Tokens are for the client to authenticate the user. Access tokens are for APIs. Your API should not accept an ID Token as an access token, and your app should not treat an access token as proof of login. If you do handle tokens from Hybrid or Implicit responses, validate `at_hash`/`c_hash` to bind them to the ID Token — with pure code flow this doesn't arise.

8. **Do not put authorization decisions solely in the ID Token.**
   Use ID Token claims for authentication context. For API authorization, validate access-token scopes, audience, tenant, roles, or permissions at the resource server.

9. **Require stronger authentication context for sensitive actions.**
   Use claims such as `acr` and `amr` when you need MFA, phishing-resistant auth, step-up authentication, or assurance-level checks, and request them via `acr_values` or `max_age`. Do not assume every successful OIDC login has the same assurance.

10. **Handle logout and sessions deliberately.**
    OIDC login creates both an application session and an identity-provider session — logging out of your app does not log the user out of the IdP. Set secure session cookies, short lifetimes, and re-authentication rules for sensitive actions; clear local app sessions on logout, and use RP-Initiated Logout (and Front-/Back-Channel Logout where the provider supports them) when the IdP session must end too.

A strong OIDC baseline:

```text
Flow: Authorization Code + PKCE
Response type: code
Scopes: openid profile email only when needed
Redirect URI: exact match only
State: required, high entropy, session-bound
Nonce: required for login
Response validation: check iss parameter (RFC 9207) if supported
ID Token: validate signature, iss, aud, exp, iat, nonce, azp
Signing: RS256/ES256/EdDSA from trusted JWKS, explicit alg allowlist
Access token: audience-bound, scope-limited, API-validated
Storage: secure HttpOnly SameSite cookies for web sessions
```

**On signing algorithms:** RS256 is fine for OIDC ID Tokens, especially when many clients need to verify tokens using a public key; ES256/EdDSA are equally good choices. But the security win comes from **strict validation**, not the algorithm. Configure the client to accept only the expected algorithm, issuer, audience, and JWKS for that specific provider.

[1]: https://openid.net/specs/openid-connect-core-1_0.html "OpenID Connect Core 1.0 incorporating errata set 2"
[2]: https://datatracker.ietf.org/doc/rfc9700/ "RFC 9700 - Best Current Practice for OAuth 2.0 Security"
[3]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/05.1-Testing_for_OAuth_Authorization_Server_Weaknesses "Testing for OAuth Authorization Server Weaknesses"

---

## Related

- [OAuth 2.0 Security Best Practices](oauth-security-best-practices.md)
- [JWT Security Best Practices](jwt-security-best-practices.md)
- [Token Security (ID vs access token separation)](token-security-guide.md)

See [References](references.md) for the full citation registry.
