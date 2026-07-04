# SAML SSO Security Best Practices

Yes — for **SAML**, the biggest risk is usually not the crypto algorithm itself. It is **bad Service Provider validation**.

SAML is XML-based SSO, so the secure baseline is: **validate the right signed assertion, from the right IdP, for the right SP, within the right time window, and only once.**

## Top SAML security practices

1. **Use a mature SAML library.**
   Do not hand-roll XML parsing, signature validation, or assertion processing. SAML is easy to get subtly wrong, especially around XML Signature Wrapping. OWASP specifically warns to validate signatures with a trusted library and avoid fragile custom processing. ([OWASP Cheat Sheet Series][1])

2. **Require signed SAML responses or signed assertions.**
   Decide which element must be signed and enforce it consistently. Then make sure your application uses the exact XML element that was validated, not a different unsigned assertion.

3. **Defend against XML Signature Wrapping.**
   Validate schema, reject duplicate IDs, enforce only expected XML structures, and bind application logic to the signed assertion object returned by the validation library. Signature wrapping attacks happen when one element is signed but the app reads attacker-controlled data elsewhere. ([arXiv][2])

4. **Validate issuer.**
   The `Issuer` must match the trusted IdP configured for that tenant/application. Do not accept assertions from arbitrary IdPs.

5. **Validate audience.**
   The `AudienceRestriction` must match your Service Provider entity ID. This prevents a token issued for another app from being replayed into yours.

6. **Validate destination and recipient.**
   Check `Destination`, `SubjectConfirmationData Recipient`, and ACS URL values so the assertion was intended for your endpoint.

7. **Validate `InResponseTo` for SP-initiated SSO.**
   Bind the SAML response to a login request you actually created. Reject unsolicited or mismatched responses unless you intentionally support IdP-initiated SSO and have separate controls.

8. **Enforce time windows.**
   Validate `NotBefore`, `NotOnOrAfter`, assertion expiration, and session lifetime with small clock skew. Do not accept stale assertions.

9. **Prevent replay.**
   Store assertion IDs or response IDs until expiration and reject duplicates. Replay protection is a core SAML control because a stolen valid assertion can otherwise be reused.

10. **Encrypt assertions when they contain sensitive attributes.**
    Signing gives integrity; encryption gives confidentiality. If assertions include sensitive identity attributes, groups, roles, or PII, require encrypted assertions.

## Strong SAML SP baseline

```text
SAML profile: Web Browser SSO, POST binding preferred
Library: mature, maintained, schema-validating
Signature: required on assertion or response
Algorithm: SHA-256 or stronger
Issuer: exact trusted IdP
Audience: exact SP entity ID
Recipient/Destination: exact ACS endpoint
InResponseTo: required for SP-initiated login
Time: NotBefore / NotOnOrAfter enforced
Replay: assertion ID cache
Attributes: minimal, encrypted if sensitive
Metadata: pinned IdP certificate, rotated deliberately
Transport: HTTPS only
```

## Common SAML mistakes

| Mistake                                                 | Why it matters                                   |
| ------------------------------------------------------- | ------------------------------------------------ |
| Accepting unsigned assertions                           | Anyone may forge login claims                    |
| Validating a signature but reading a different XML node | Enables XML Signature Wrapping                   |
| Not checking audience                                   | Assertion for App A works on App B               |
| Not checking issuer                                     | Rogue or wrong IdP accepted                      |
| Not checking `InResponseTo`                             | Login CSRF / unsolicited response abuse          |
| Not checking expiration                                 | Old assertions keep working                      |
| No replay cache                                         | Captured assertions can be reused                |
| Too many attributes                                     | Increases privacy and privilege risk             |
| Auto-provisioning admins from SAML attributes           | Attribute tampering becomes privilege escalation |
| Weak metadata/cert rotation process                     | Breaks trust or enables bad keys                 |

For your checklist, I’d phrase SAML like this:

> Harden SAML SSO by requiring signed assertions or responses, validating issuer, audience, recipient, destination, `InResponseTo`, time conditions, and replay IDs; defending against XML Signature Wrapping; pinning trusted IdP metadata and certificates; encrypting sensitive assertions; minimizing attributes; and using a mature maintained SAML library rather than custom XML handling.

[1]: https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html "SAML Security - OWASP Cheat Sheet Series"
[2]: https://arxiv.org/abs/1401.7483 "Secure SAML validation to prevent XML signature wrapping attacks"

---

## Related

- [OpenID Connect (OIDC) Security Best Practices](oidc-security-best-practices.md)
- [Okta Hardening Checklist](okta-hardening-checklist.md)

See [References](references.md) for the full citation registry.
