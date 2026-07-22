# Implementing mTLS at the Gateway

Add mTLS **where TLS is terminated**, not inside the JSON body or POST payload. The client certificate is presented during the TLS handshake, before the HTTP request reaches your API. For APIs, the usual place is an **API gateway / reverse proxy / ingress controller**, with the backend only receiving traffic from that trusted layer. NGINX describes mTLS as using client certificates to verify both server and client during communication. ([NGINX Docs][1])

## 1. Basic architecture

```text
Client / service
  └── TLS handshake with client certificate
        ↓
API Gateway / NGINX / Envoy / Ingress
  └── validates client certificate against trusted client CA
        ↓
Backend API
  └── receives request only if mTLS succeeded
```

The backend should not trust a random `X-Client-Cert` header from the internet. The gateway must validate the certificate itself and set identity headers from its own TLS variables when forwarding — never pass inbound client-identity headers through.

## 2. NGINX mTLS example

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    # Server certificate presented to clients
    ssl_certificate     /etc/nginx/tls/server.crt;
    ssl_certificate_key /etc/nginx/tls/server.key;

    # Client CA used to verify client certificates
    ssl_client_certificate /etc/nginx/ca/client-ca.pem;
    ssl_verify_client on;
    ssl_verify_depth 2;

    # Revocation list for the client CA (nginx does not do OCSP
    # checks on client certs; use a CRL or short-lived certs)
    ssl_crl /etc/nginx/ca/client-ca.crl;

    ssl_protocols TLSv1.3 TLSv1.2;

    location / {
        # proxy_set_header REPLACES any same-named header from the
        # client, so setting these from TLS variables both strips
        # spoofed inbound values and injects the trusted ones.
        proxy_set_header X-Client-Cert-Verify      $ssl_client_verify;
        proxy_set_header X-Client-Cert-Subject     $ssl_client_s_dn;
        proxy_set_header X-Client-Cert-Fingerprint $ssl_client_fingerprint;

        # Full cert (URL-encoded PEM) if the backend must check
        # certificate-bound tokens (RFC 8705 x5t#S256):
        proxy_set_header X-Client-Cert             $ssl_client_escaped_cert;

        proxy_pass http://backend_api;
    }
}
```

The key directives are `ssl_client_certificate`, which defines the trusted CA certificates used to verify clients, and `ssl_verify_client`, which controls whether client certificate verification is required. ([Nginx][2]) For a gradual rollout, `ssl_verify_client optional` lets the handshake succeed without a cert while `$ssl_client_verify` tells the backend whether to accept the request — switch to `on` once all clients have certs.

Note: `$ssl_client_fingerprint` is the SHA-1 fingerprint. It's fine as a log/correlation field, but for token binding compute the SHA-256 thumbprint from the forwarded certificate instead.

## 3. Client-side POST with mTLS

```bash
curl --tlsv1.3 \
  --cert ./client.crt \
  --key ./client.key \
  --cacert ./server-ca.crt \
  -X POST "https://api.example.com/v1/payments" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 8f7b4d37-2f4e-4c32-8b2d-3a0d2d6d0d91" \
  -H "X-Request-ID: 3d3d3f8e-91ff-4f92-9b18-9a7c3df42a12" \
  --data '{
    "account_id": "acct_12345",
    "amount": 12500,
    "currency": "USD"
  }'
```

Here, the OAuth token still goes in the HTTP `Authorization` header, while the client certificate is used at the TLS layer. See [Calling a Protected API with a Token](calling-a-protected-api.md) for the full request/response pattern and server-side validation steps.

## 4. Kubernetes service-to-service mTLS

For internal Kubernetes service traffic, use a service mesh or gateway pattern. In Istio, `PeerAuthentication` controls whether mTLS is required for incoming workload traffic; Istio documents `STRICT` mode as the setting used to require mTLS instead of accepting plaintext. ([Istio][3])

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payments
spec:
  mtls:
    mode: STRICT
```

Then add an authorization policy so only the expected workload identity can call the API:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-payments
  namespace: payments
spec:
  selector:
    matchLabels:
      app: payments-api
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/frontend/sa/frontend-api
      to:
        - operation:
            methods: ["POST"]
            paths: ["/v1/payments"]
```

This gives you transport authentication between workloads, but you should still enforce API authorization, scopes, object-level permissions, and request validation.

## 5. OAuth + mTLS-bound tokens

For high-assurance APIs, combine mTLS with OAuth so the access token is bound to the client certificate. RFC 8705 defines OAuth client authentication and certificate-bound access and refresh tokens using mutual TLS with X.509 certificates. The binding is carried in the token's `cnf` (confirmation) claim as `x5t#S256` — the base64url-encoded SHA-256 thumbprint of the client certificate. ([RFC Editor][4])

The API should validate all of:

```text
1. The client certificate is trusted and currently valid.
2. The access token is valid.
3. The token's cnf/x5t#S256 thumbprint matches the presented client certificate.
4. The token audience and scopes allow this API action.
```

That prevents a stolen bearer token from being replayed by a client that does not possess the matching private key. When TLS terminates at the gateway, the gateway must forward the client certificate (or its SHA-256 thumbprint) over a trusted channel so the backend can perform check 3.

## 6. Hardening checklist

```text
- Issue client certs from a dedicated client-auth CA.
- Require clientAuth EKU on client certificates.
- Validate SAN/service identity, not just CN.
- Use short-lived certs where possible.
- Rotate client certs automatically.
- Revoke compromised certs (CRL at the gateway; short TTLs where CRL/OCSP is impractical).
- Log certificate subject, issuer, serial, fingerprint, and mapped identity.
- Fail closed when cert validation fails.
- Set identity headers only from gateway TLS variables; never forward inbound ones.
- Do not pass raw client cert identity through untrusted networks.
- Restrict backend access so only the gateway/mesh can reach it.
- Keep OAuth/JWT authorization checks even when mTLS succeeds.
```

The clean pattern is:

```text
TLS proves the server.
mTLS proves the client system.
OAuth/JWT proves authorization.
Application logic proves the caller can perform this action on this object.
```

[1]: https://docs.nginx.com/nginx-instance-manager/system-configuration/secure-traffic/ "Secure client access and network traffic | NGINX Documentation"
[2]: https://nginx.org/en/docs/http/ngx_http_ssl_module.html "Module ngx_http_ssl_module"
[3]: https://istio.io/latest/docs/reference/config/security/peer_authentication/ "PeerAuthentication"
[4]: https://www.rfc-editor.org/info/rfc8705/ "RFC 8705: OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens"

---

## Related

- [TLS & mTLS for APIs](tls-and-mtls-for-apis.md)
- [Calling a Protected API with a Token](calling-a-protected-api.md)

See [References](references.md) for the full citation registry.
