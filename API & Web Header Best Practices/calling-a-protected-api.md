# Calling a Protected API with a Token

The solid pattern for calling a protected API with a token:

```http
POST /v1/payments HTTP/1.1
Host: api.example.com
Authorization: Bearer <short-lived-access-token>
Content-Type: application/json
Accept: application/json
Idempotency-Key: 8f7b4d37-2f4e-4c32-8b2d-3a0d2d6d0d91
X-Request-ID: 3d3d3f8e-91ff-4f92-9b18-9a7c3df42a12

{
  "account_id": "acct_12345",
  "amount": 12500,
  "currency": "USD",
  "description": "Invoice 1042"
}
```

The token should go in the **`Authorization: Bearer` header**, not in the URL, query string, or JSON body. RFC 6750 defines bearer-token usage for OAuth-protected resources with the authorization header as the primary method — it technically also permits a form-encoded body parameter and a query parameter, but explicitly discourages the query form because URLs leak through logs and history. It also requires TLS for bearer-token use. ([IETF Datatracker][1]) OWASP's REST guidance also recommends HTTPS-only APIs because it protects credentials such as API keys and JWTs in transit. ([OWASP Cheat Sheet Series][2])

For a higher-assurance machine-to-machine API, use the same POST but add **mTLS at the transport layer**. The client certificate is not sent as an application header; it is negotiated during TLS, and the API gateway maps the certificate identity to the client/service identity. See [Implementing mTLS at the Gateway](mtls-at-the-gateway.md).

```bash
curl --tlsv1.3 \
  --cert ./client.crt \
  --key ./client.key \
  -X POST "https://api.example.com/v1/payments" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Idempotency-Key: 8f7b4d37-2f4e-4c32-8b2d-3a0d2d6d0d91" \
  -H "X-Request-ID: 3d3d3f8e-91ff-4f92-9b18-9a7c3df42a12" \
  --data '{
    "account_id": "acct_12345",
    "amount": 12500,
    "currency": "USD",
    "description": "Invoice 1042"
  }'
```

A good API response avoids caching sensitive data and returns a minimal body:

```http
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Cache-Control: no-store
X-Content-Type-Options: nosniff
X-Request-ID: 3d3d3f8e-91ff-4f92-9b18-9a7c3df42a12

{
  "id": "pay_789",
  "status": "created"
}
```

## Error responses

RFC 6750 also defines how the API should reject bad tokens: a `401` with a `WWW-Authenticate` challenge and a machine-readable error code — `invalid_token` for expired/malformed/revoked tokens, `insufficient_scope` (with `403`) when the token is valid but lacks the required scope. ([IETF Datatracker][1])

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api", error="invalid_token",
                  error_description="The access token expired"
```

Clients should treat `401 invalid_token` as "refresh or re-authenticate, then retry once" and `403 insufficient_scope` as "do not retry with the same token."

## Server-side validation

```text
TLS is valid
mTLS client certificate is valid, if required
Token signature or introspection result is valid
Algorithm is explicitly allowed
Issuer is trusted
Audience matches this API
Token is not expired
Token type is access_token, not id_token
Scope/permission allows this action
Client identity matches the certificate, if using mTLS-bound tokens
Subject and client are not revoked
Object-level authorization is enforced for account_id
Request body matches schema
Idempotency key prevents duplicate state-changing requests
```

## API keys vs. bearer tokens

For an **API key** rather than an OAuth access token, still prefer an authorization-style header:

```http
Authorization: ApiKey <api-key-value>
```

or, if your platform standardizes it:

```http
X-API-Key: <api-key-value>
```

Note that `ApiKey` is not an IANA-registered HTTP authentication scheme (unlike `Bearer`, `Basic`, or `DPoP`) — it works, and it keeps credentials out of URLs, but treat it as a platform convention and document it as such. For OAuth/JWT access tokens, use the standard form:

```http
Authorization: Bearer <access-token>
```

(If you adopt DPoP sender-constrained tokens, the scheme changes to `Authorization: DPoP <token>` plus a `DPoP` proof header — see [Token Security](token-security-guide.md).)

Never log the raw token. Log only safe metadata such as `jti`, issuer, subject hash, client ID, certificate thumbprint, request ID, decision, and failure reason.

[1]: https://datatracker.ietf.org/doc/html/rfc6750 "RFC 6750 - The OAuth 2.0 Authorization Framework: Bearer Token Usage"
[2]: https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html "REST Security Cheat Sheet"

---

## Related

- [API Security Headers & Controls](api-security-headers-and-controls.md)
- [Implementing mTLS at the Gateway](mtls-at-the-gateway.md)
- [Idempotency-Key & X-Request-ID Headers](idempotency-key-and-request-id.md)
- [Token Security (validation checklist)](token-security-guide.md)
- [API Keys](api-keys.md)

See [References](references.md) for the full citation registry.
