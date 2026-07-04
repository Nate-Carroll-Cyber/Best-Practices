# Idempotency-Key & X-Request-ID Headers

These are **operational safety and observability headers**. They are not authentication secrets.

```http
Idempotency-Key: 8f7b4d37-2f4e-4c32-8b2d-3a0d2d6d0d91
X-Request-ID: 3d3d3f8e-91ff-4f92-9b18-9a7c3df42a12
```

## `Idempotency-Key`

An **Idempotency-Key** tells the server:

> “This POST represents one logical operation. If you see this same key again, do not perform the operation twice.”

This is important for state-changing APIs like:

```text
Create payment
Create order
Transfer funds
Submit application
Create user
Send message
Provision resource
```

Example problem:

```text
Client sends POST /payments
Server creates payment
Network times out before client receives response
Client retries
Without idempotency: duplicate payment may be created
With idempotency: server returns the original result
```

Typical behavior:

```http
POST /v1/payments
Idempotency-Key: abc-123
```

First request:

```text
Server processes payment.
Server stores result for key abc-123.
Returns 201 Created.
```

Retry with same key and same body:

```text
Server does not create another payment.
Server returns the original response.
```

Retry with same key but different body:

```text
Server should reject it, usually 409 Conflict or 400 Bad Request.
```

Security/implementation notes:

```text
- Generate a new key for each logical operation.
- Reuse the same key only for retries of the same operation.
- Use a random UUID or high-entropy value.
- Store the key server-side with the request hash and response.
- Scope it by user/client/tenant and endpoint.
- Expire it after a reasonable TTL.
- Do not put secrets or business data in the key.
```

## `X-Request-ID`

An **X-Request-ID** is for tracing and troubleshooting.

It tells every service:

> “This request is part of this one traceable transaction.”

Example:

```http
X-Request-ID: 3d3d3f8e-91ff-4f92-9b18-9a7c3df42a12
```

That ID can be logged by:

```text
API gateway
Auth service
Application API
Database layer
Queue worker
Payment service
SIEM/Splunk
Error monitoring
```

So when something fails, you can search one ID and follow the whole request path.

Example logs:

```text
gateway: request_id=3d3d3f8e status=200
auth:    request_id=3d3d3f8e token_valid=true
api:     request_id=3d3d3f8e created payment pay_789
worker:  request_id=3d3d3f8e queued receipt email
```

Security/implementation notes:

```text
- Accept a client-provided ID only if it matches a safe format and length.
- Generate one at the gateway if missing.
- Return it in the response.
- Do not treat it as authentication.
- Do not put secrets, usernames, emails, or tokens in it.
- Log it everywhere.
```

## Difference between them

| Header            |       Purpose | Prevents duplicate action? | Used for logging/tracing? |
| ----------------- | ------------: | -------------------------: | ------------------------: |
| `Idempotency-Key` |  Retry safety |                        Yes |                 Sometimes |
| `X-Request-ID`    | Observability |                         No |                       Yes |

A strong API request can include both:

```http
POST /v1/payments HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
Content-Type: application/json
Idempotency-Key: 8f7b4d37-2f4e-4c32-8b2d-3a0d2d6d0d91
X-Request-ID: 3d3d3f8e-91ff-4f92-9b18-9a7c3df42a12
```

The simple way to remember it:

```text
Idempotency-Key protects the business operation from accidental duplication.

X-Request-ID helps humans and systems trace what happened.
```

---

## Related

- [Calling a Protected API with a Token](calling-a-protected-api.md)
- [API Security Headers & Controls](api-security-headers-and-controls.md)

See [References](references.md) for the full citation registry.
