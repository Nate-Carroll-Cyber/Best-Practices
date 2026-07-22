# JWT Security Best Practices

**RS256 is usually a good default** when one trusted issuer signs tokens and multiple APIs/services verify them with the public key. But the bigger rule is: **do not let the token choose the algorithm.** Your verifier should explicitly allow only the expected algorithm, issuer, audience, and key set. RFC 8725 calls out classic JWT attacks where `alg` is changed to `none` or from `RS256` to `HS256`, causing vulnerable libraries to validate forged tokens. ([RFC Editor][1])

## Top 10 JWT security practices

1. **Use asymmetric signing for distributed systems.**
   RS256 is fine; ES256 or EdDSA can also be good depending on library support and key-management maturity. Avoid HS256 unless the issuer and verifier are tightly controlled, because every verifier needs the shared secret — and any verifier holding it can also mint tokens.

2. **Hard-code the accepted algorithm.**
   Do not dynamically trust the `alg` header, and reject `none` outright. Configure your JWT library to accept only the expected algorithm, such as `RS256`, and reject everything else. RFC 8725 says libraries must let callers specify supported algorithms and must not use others. ([RFC Editor][1])

3. **Validate the signature every time.**
   Never decode-only and trust claims. Reject the entire token if signature validation fails. RFC 8725 states that all cryptographic operations used in the JWT must validate or the JWT must be rejected. ([RFC Editor][1])

4. **Validate `iss`, `aud`, `exp`, `nbf`, and `iat`.**
   The common mistake is verifying the signature but not checking whether the token was meant for your API. RFC 7519 defines `aud` as the intended recipient and says a recipient not listed in `aud` must reject the token; it also defines `exp`, `nbf`, `iat`, and `jti`. ([IETF Datatracker][2]) Allow only a small clock skew (a minute or two) when checking time claims.

5. **Use short-lived access tokens.**
   Think minutes, not days. For browser or API access tokens, 5–15 minutes is a common pattern. Use refresh tokens or re-authentication for longer sessions.

6. **Have a revocation strategy.**
   JWTs are stateless by design, which makes revocation harder. Use short TTLs, refresh-token rotation, and a denylist keyed by `jti` plus `iss` when immediate revocation matters. OWASP shows this style of revocation table using `jti`, `iss`, and `exp`. ([OWASP Cheat Sheet Series][3])

7. **Do not put secrets or sensitive data in JWT claims.**
   A signed JWT is integrity-protected, not confidential. Anyone who has the token can base64url-decode the claims. Store identifiers, scopes, roles, tenant IDs, and authorization context—not passwords, API keys, secrets, PII, or internal-only data. Use JWE/encryption only when you truly need confidential claims.

8. **Bind tokens to context where practical.**
   For browser sessions, consider a hardened cookie-bound fingerprint or sender-constrained token pattern (DPoP, mTLS) so a stolen bearer token is less useful. OWASP describes using a random hardened cookie value and storing only its SHA-256 hash in the JWT to detect replay. ([OWASP Cheat Sheet Series][3])

9. **Store and transmit tokens carefully.**
   Always use HTTPS. OWASP's REST guidance says secure REST services must only provide HTTPS endpoints because this protects credentials such as JWTs in transit. For browser apps, prefer secure, HttpOnly, SameSite cookies for refresh tokens; avoid long-lived tokens in localStorage. ([OWASP Cheat Sheet Series][4])

10. **Separate token types and validation rules.**
    Do not let an ID token, access token, refresh token, email-verification token, or password-reset token be accepted in the wrong place. Use different `aud`, `typ`, issuer, required claims, keys, or validation profiles. RFC 8725 recommends mutually exclusive validation rules to prevent cross-JWT confusion and token substitution. ([RFC Editor][1]) For OAuth access tokens specifically, RFC 9068 defines the `at+jwt` `typ` header so resource servers can refuse anything that isn't an access token. ([RFC Editor][5])

Two practices that round out the list:

- **Resolve keys only from trusted sources, and rotate them.** Look up `kid` exclusively in the JWKS you fetched from the trusted issuer; never honor `jku`, `x5u`, or embedded `jwk` headers from the token itself (RFC 8725 covers these header-based attacks). Rotate signing keys on a schedule and publish old + new keys in the JWKS during the overlap window so verification never breaks.

A strong baseline:

```text
alg: RS256 only (explicit allowlist; none rejected)
iss: exact trusted issuer
aud: exact API/resource identifier
exp: required, short-lived
nbf/iat: validated with small clock skew
jti: present if revocation/replay tracking matters
typ: explicit token type (at+jwt for OAuth access tokens)
kid: resolved only from trusted JWKS for that issuer; jku/x5u/jwk ignored
claims: minimal, non-sensitive
transport: HTTPS only
```

The secure design is not just RS256 — it is **fixed algorithm + strict claim validation + short TTL + revocation plan + safe storage + key rotation + token-type separation**.

[1]: https://www.rfc-editor.org/rfc/rfc8725.html "RFC 8725: JSON Web Token Best Current Practices"
[2]: https://datatracker.ietf.org/doc/html/rfc7519 "RFC 7519 - JSON Web Token (JWT)"
[3]: https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html "JSON Web Token for Java - OWASP Cheat Sheet Series"
[4]: https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html "REST Security - OWASP Cheat Sheet Series"
[5]: https://www.rfc-editor.org/rfc/rfc9068.html "RFC 9068: JWT Profile for OAuth 2.0 Access Tokens"

---

## Related

- [OAuth 2.0 Security Best Practices](oauth-security-best-practices.md)
- [OpenID Connect (OIDC) Security Best Practices](oidc-security-best-practices.md)
- [Token Security (canonical JWT validation + lifetimes)](token-security-guide.md)

See [References](references.md) for the full citation registry.
