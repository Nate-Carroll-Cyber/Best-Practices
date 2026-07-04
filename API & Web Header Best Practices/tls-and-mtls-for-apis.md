# TLS & mTLS for APIs

Yes — for an API, **TLS is mandatory**, and **mTLS is recommended for higher-trust or machine-to-machine paths**.

## TLS baseline

Use TLS everywhere: browser to frontend, frontend to API, API to API, API to database, and service to service. OWASP’s REST guidance recommends HTTPS/TLS for REST services, and NIST SP 800-52 provides TLS configuration guidance for protecting data in transit. ([OWASP Cheat Sheet Series][1])

Recommended baseline:

```text
TLS versions:
- Prefer TLS 1.3
- Allow TLS 1.2 only if needed
- Disable SSL, TLS 1.0, and TLS 1.1

Certificates:
- Use publicly trusted certs for public services
- Use private CA or service mesh CA for internal services
- Automate issuance and rotation
- Monitor expiration

Configuration:
- Strong cipher suites only
- Forward secrecy
- HSTS for browser-facing HTTPS
- No weak renegotiation or compression
- Redirect HTTP to HTTPS
- Validate cert chains and hostnames
```

For public APIs, TLS gives **server authentication and encrypted transport**. It proves the client is talking to your API endpoint and protects the request and response from passive interception or tampering.

## When to use mTLS

Use **mutual TLS** when the server must also authenticate the client with a certificate.

Good use cases:

```text
- Service-to-service APIs
- Partner/B2B APIs
- Internal administrative APIs
- Payment, financial, health, or sensitive data APIs
- Zero-trust internal networks
- OAuth client authentication
- Certificate-bound access tokens
- Kubernetes/service mesh workloads
```

mTLS gives you two-way authentication:

```text
TLS:
Client verifies server certificate.

mTLS:
Client verifies server certificate.
Server verifies client certificate.
```

For OAuth, RFC 8705 defines mutual-TLS client authentication and certificate-bound access tokens, which helps prevent stolen bearer tokens from being replayed without the client certificate. ([RFC Editor][2])

## Practical mTLS API pattern

```text
External user/browser traffic:
Browser → TLS → Web/API gateway

Internal service traffic:
Service A → mTLS → Service B

High-risk OAuth clients:
Client cert + OAuth token binding

Partner APIs:
Partner cert allowlist + mTLS + OAuth/JWT scopes
```

A good control stack for sensitive APIs:

```text
TLS 1.3
mTLS for machine clients
OAuth/OIDC for identity and authorization
JWT audience/scope validation
Certificate-bound tokens where appropriate
Strict CORS for browser clients
Rate limiting
API gateway policy enforcement
Centralized certificate lifecycle management
```

## mTLS implementation checklist

```text
- Issue client certificates from a trusted CA
- Use short-lived certificates where possible
- Rotate certificates automatically
- Validate certificate chain, expiry, SAN, EKU, and revocation status
- Map certificate identity to service/client identity
- Do not rely only on CN; use SAN/SPIFFE/service identity
- Log certificate subject, issuer, serial, thumbprint, and mapped identity
- Block expired, revoked, unknown, or wrong-purpose certificates
- Use separate CAs or trust bundles by environment
- Do not reuse production client certificates in dev/test
```

## Common mistakes

| Mistake                                    | Why it matters                                                                       |
| ------------------------------------------ | ------------------------------------------------------------------------------------ |
| Using TLS only at the public load balancer | Internal traffic may still be plaintext                                              |
| Accepting any valid client certificate     | mTLS must validate identity, not just “has a cert”                                   |
| Long-lived client certs                    | Harder to revoke and rotate safely                                                   |
| No certificate inventory                   | Expiration becomes an outage risk                                                    |
| No revocation/rotation plan                | Compromised certs stay useful                                                        |
| Using mTLS instead of authorization        | mTLS proves client identity; it does not replace scopes, roles, or object-level auth |
| Browser mTLS for normal users              | Usually poor UX and operationally heavy                                              |
| No hostname verification                   | Opens the door to impersonation                                                      |

## Kubernetes / service mesh angle

In Kubernetes, mTLS is commonly enforced through a service mesh such as Istio, Linkerd, or Consul, or through sidecars/API gateways. The goal is to make service identity and encryption automatic while still enforcing authorization policies separately.

For Kubernetes APIs and internal services:

```text
- Use mTLS between workloads
- Use workload identity rather than static shared secrets
- Deny plaintext service-to-service traffic where possible
- Enforce namespace/service-level authorization
- Rotate workload certificates automatically
- Log and alert on failed mTLS handshakes
```

## Simple rule

Use **TLS for everything**.

Use **mTLS when the client is a system, service, workload, partner, or high-risk OAuth client that must be strongly authenticated at the transport layer**.

For most modern enterprise APIs, the strongest pattern is:

```text
TLS 1.3 externally
mTLS internally or for partner/service clients
OAuth/OIDC for user and client authorization
JWT validation at the resource server
Short-lived tokens
Certificate-bound tokens for high-risk clients
```

[1]: https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html "REST Security Cheat Sheet"
[2]: https://www.rfc-editor.org/info/rfc8705/ "RFC 8705: OAuth 2.0 Mutual-TLS Client Authentication ..."

---

## Related

- [Implementing mTLS at the Gateway](implementing-mtls-at-the-gateway.md)
- [Token Security (certificate-bound tokens, RFC 8705)](token-security-guide.md)

See [References](references.md) for the full citation registry.
