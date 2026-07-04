# API Keys: Format, Header, and Choosing an Identifier

This guide covers three related questions about API keys: what format the key should take, whether UUIDs are a good choice, and which HTTP header to send them in.

## Which HTTP header for API keys?

The standard, recommended approach is:

```http
Authorization: Bearer <api_key>
```

Example:

```http
GET /v1/users HTTP/1.1
Authorization: Bearer sk_live_xxxxxxxxx
```

This has several advantages:

- Standard across REST APIs.
- Supported by API gateways, SDKs, proxies, and tooling.
- Easy to migrate later to OAuth or JWT without changing clients.

If you want to explicitly distinguish API keys from bearer tokens, the conventional custom header is:

```http
X-API-Key: abc123...
```

or

```http
Api-Key: abc123...
```

`X-API-Key` is by far the most common custom header. Although the historical `X-` prefix is no longer recommended for new HTTP header names, it remains widely recognized and supported. Avoid non-standard variants like `XApi`, `Xapi`, or `XApiKey` — they may confuse developers.

## Should API keys be UUIDs?

UUIDs can work as API keys, but they usually aren't the best choice by themselves.

A random UUID (especially UUIDv4) provides about 122 bits of randomness, which is sufficient for many applications.

Example:

```text
550e8400-e29b-41d4-a716-446655440000
```

However, there are better options:

| Type | Good for API keys? | Notes |
| --- | --- | --- |
| UUIDv4 | ✅ Good | Widely available, enough entropy. |
| UUIDv7 | ⚠️ Not ideal | Time-ordered, so it leaks creation time. Great for database IDs, not secret keys. |
| 256-bit random token | ⭐ Best | More entropy and designed to be secret. |
| URL-safe Base64 token | ⭐ Best | Shorter and easier to transmit. |
| Hex-encoded random bytes | ⭐ Best | Simple and secure. |

For example, generate 32 random bytes:

```text
5hZL5C2QK2wM1m0nA0m5iGQw7V4P8tR1KxE9YzL8BnM
```

or

```text
9d8a4d4d0b0ef2a8cdb8f3f9b52cbe7cfe0c65f42d3b6f4dcbacb69c71d4e7e5
```

These are harder to guess than UUIDs and are commonly used by major APIs.

## Recommendation

For a new API:

- **Key format:** 256-bit cryptographically secure random token (Base64 URL-safe or hex encoded).
- **Header:** `Authorization: Bearer <key>`.

If you want the key to be visually identifiable, use a prefix:

```text
sk_live_4Jx...
sk_test_9Lp...
pk_live_...
```

Prefixes make it easy to distinguish production vs. test keys, identify the credential type, and detect accidental exposure in logs or code. This is the pattern used by APIs such as Stripe and many other modern services.

See also [Token Security](token-security-guide.md) for how API keys differ from access, ID, refresh, and session tokens.

---

## Related

- [Token Security: Types, Usage & Failure Modes](token-security-guide.md)
- [API Security Headers & Controls](api-security-headers-and-controls.md)

See [References](references.md) for the full citation registry.
