# API Keys: Format, Header, and Choosing an Identifier

This guide covers what format an API key should take, whether UUIDs are a good choice, which HTTP header to send them in, and how to handle keys server-side.

## Which HTTP header for API keys?

The most common approach is to reuse the bearer scheme:

```http
Authorization: Bearer <api_key>
```

Example:

```http
GET /v1/users HTTP/1.1
Authorization: Bearer sk_live_xxxxxxxxx
```

This has several advantages:

- Standard-shaped across REST APIs and used by major providers.
- Supported by API gateways, SDKs, proxies, and tooling.
- Easy to migrate later to OAuth or JWT without changing clients.

One caveat: `Bearer` is technically the OAuth 2.0 scheme (RFC 6750), so if your API accepts *both* OAuth access tokens and API keys, putting them in the same scheme forces the server to distinguish them by format. In that case use a separate header for keys:

```http
X-API-Key: abc123...
```

or

```http
Api-Key: abc123...
```

`X-API-Key` is by far the most common custom header. The `X-` prefix is formally deprecated for new header names (RFC 6648 ([IETF Datatracker][1])), but it remains widely recognized and supported. Avoid non-standard variants like `XApi`, `Xapi`, or `XApiKey` — they may confuse developers. Whatever you choose, keep keys in headers — never in URLs or query strings, which leak through logs, history, and referrers.

## Should API keys be UUIDs?

UUIDs can work as API keys, but they usually aren't the best choice by themselves.

A random UUID (specifically UUIDv4) provides about 122 bits of randomness, which is sufficient for many applications — provided it comes from a cryptographically secure generator, which not every UUID library guarantees.

Example:

```text
550e8400-e29b-41d4-a716-446655440000
```

However, there are better options:

| Type | Good for API keys? | Notes |
| --- | --- | --- |
| UUIDv4 | ✅ Acceptable | Widely available, enough entropy if the generator is CSPRNG-backed. |
| UUIDv7 | ❌ No | Time-ordered, so it leaks creation time and reduces effective randomness. Great for database IDs, not secret keys. |
| 256-bit random token | ⭐ Best | More entropy and designed to be secret. |
| URL-safe Base64 token | ⭐ Best | Same entropy, shorter and easier to transmit. |
| Hex-encoded random bytes | ⭐ Best | Simple and secure. |

For example, 32 cryptographically random bytes, Base64url- or hex-encoded:

```text
5hZL5C2QK2wM1m0nA0m5iGQw7V4P8tR1KxE9YzL8BnM
```

```text
9d8a4d4d0b0ef2a8cdb8f3f9b52cbe7cfe0c65f42d3b6f4dcbacb69c71d4e7e5
```

These are harder to guess than UUIDs and are commonly used by major APIs.

## Handling keys server-side

An API key is a credential — treat it like a password, not like an ID:

```text
- Store only a hash of the key (SHA-256 is fine for high-entropy
  random keys; no salt/stretching needed at 256 bits).
- Keep a non-secret key ID or prefix in plaintext for lookup and logs.
- Compare hashes in constant time.
- Show the full key exactly once, at creation.
- Scope each key: allowed APIs, environments, tenants, rate limits.
- Support rotation: allow two active keys per client so rotation
  is zero-downtime; expire the old one.
- Revoke immediately on exposure; monitor for keys committed to
  repos or pasted in tickets.
- Never log the raw key — log the key ID/prefix and decision.
- Never ship keys in frontend code or mobile binaries; a key that
  reaches the client is public.
```

## Recommendation

For a new API:

- **Key format:** 256-bit cryptographically secure random token (Base64 URL-safe or hex encoded).
- **Header:** `Authorization: Bearer <key>`, or `X-API-Key` if the API also accepts OAuth tokens.
- **Storage:** hashed server-side, plaintext shown once at creation.

If you want the key to be visually identifiable, use a prefix:

```text
sk_live_4Jx...
sk_test_9Lp...
pk_live_...
```

Prefixes make it easy to distinguish production vs. test keys, identify the credential type, and detect accidental exposure in logs or code — secret-scanning tools key on exactly these prefixes. This is the pattern used by APIs such as Stripe and many other modern services.

See also [Token Security](token-security-guide.md) for how API keys differ from access, ID, refresh, and session tokens.

[1]: https://datatracker.ietf.org/doc/html/rfc6648 "RFC 6648 - Deprecating the \"X-\" Prefix and Similar Constructs in Application Protocols"

---

## Related

- [Token Security: Types, Usage & Failure Modes](token-security-guide.md)
- [API Security Headers & Controls](api-security-headers-and-controls.md)
- [Calling a Protected API with a Token](calling-a-protected-api.md)

See [References](references.md) for the full citation registry.
