# Okta Hardening Checklist

Yes — Okta should be hardened like a **Tier 0 identity platform**, not just another SaaS app.

Here’s the checklist I’d use.

## Okta hardening checklist

1. **Require phishing-resistant MFA for admins first.**
   Use Okta FastPass or Passkeys/FIDO2 WebAuthn for administrators and high-risk users. Okta identifies Okta FastPass and Passkeys/FIDO2 WebAuthn as phishing-resistant authenticators. ([Okta Docs][1])

2. **Lock down the Okta Admin Console app.**
   Create a dedicated Admin Console app sign-in policy requiring phishing-resistant MFA, device posture, network conditions, and re-authentication. Okta documents Admin Console MFA enforcement through the Okta Admin Console app policy. ([Okta Docs][2])

3. **Minimize Super Admins.**
   Treat Super Admin as a break-glass or very small trusted role. Okta describes Super Admin as the highest permission level with full management access. ([Okta Docs][3]) Use custom admin roles and least privilege for help desk, app admins, group admins, and SecOps staff.

4. **Harden help desk reset workflows.**
   MFA reset and device enrollment are prime targets. CISA reports Scattered Spider actors use social engineering to convince help desk staff to reset passwords or MFA tokens. ([CISA][4]) Require strong identity proofing, manager approval, callback to trusted numbers, ticket evidence, and alerting for factor resets.

5. **Require phishing-resistant auth before enrolling new authenticators.**
   This helps prevent an attacker who has a password or weak factor from adding their own MFA method. Okta supports requiring a phishing-resistant authenticator before enrolling additional authenticators. ([Okta Docs][5])

6. **Use network zones in policies.**
   Add network zones to Global Session Policies and app sign-in policies so access differs by trusted, untrusted, anonymizer, or high-risk networks. Okta documents adding network zones to Global Session Policies. ([Okta Docs][6])

7. **Enable ThreatInsight in block mode where appropriate.**
   Okta ThreatInsight evaluates sign-in attempts for suspicious activity and can log or deny access from potentially suspicious traffic; Okta notes it is not a guarantee, so treat it as one signal. ([Okta Docs][7])

8. **Use risk and behavior-based policies.**
   Configure step-up authentication for risky sign-ins, new devices, new locations, impossible travel-style behavior, or unusual session context. Okta risk scoring evaluates sign-in attempts using factors such as IP, device, and behavior. ([Okta Docs][8])

9. **Add device assurance.**
   Require managed, healthy, patched devices for privileged apps and sensitive data. Okta device assurance can check security-related device attributes, such as OS version or patch level, in app sign-in policies. ([Okta Docs][9])

10. **Restrict and rotate API tokens.**
    Okta API tokens should be named, owned by service accounts, restricted to network zones, monitored, and rotated. Okta allows SSWS API tokens to be restricted by IP/network zone and rate-limited per token. ([Okta Docs][10])

11. **Prefer scoped OAuth service apps over broad SSWS tokens.**
    Use the narrowest possible scopes, short-lived credentials, private key auth where supported, and separate clients per integration. Do not share one token across automation, SIEM, HR, and provisioning jobs.

12. **Harden SAML/OIDC apps.**
    For OIDC, require Authorization Code + PKCE, exact redirect URIs, strict issuer/audience validation, and minimal scopes. For SAML, require signed assertions or responses, validate audience and recipient, encrypt sensitive assertions, and minimize attributes.

13. **Control app assignments through groups.**
    Avoid direct user assignments where possible. Use group-based access, approval workflows for privileged apps, and periodic access reviews. This makes access easier to revoke and audit.

14. **Deprovision aggressively.**
    HR should be the source of truth. Disable Okta accounts quickly on termination, remove app assignments, revoke sessions, revoke refresh tokens where applicable, and suspend downstream SaaS accounts through SCIM/provisioning.

15. **Stream logs to your SIEM.**
    Send Okta System Log events to Splunk or another SIEM. Okta log streaming supports exporting System Log events to platforms including Splunk Cloud and Amazon EventBridge for monitoring, automation, alerting, root-cause analysis, and compliance retention. ([Okta Docs][11])

## Events I would alert on immediately

```text
New Super Admin assigned
Admin role granted or changed
MFA factor reset
New authenticator enrolled
Password reset for privileged user
API token created, revoked, or used from new IP
OAuth app created or modified
SAML certificate or metadata changed
Sign-in from suspicious IP
ThreatInsight detection
Risk level changed to high
Device assurance failure
Global Session Policy changed
App sign-in policy changed
Network zone changed
User added to privileged group
```

## Strong baseline

```text
Admins: phishing-resistant MFA only
Users: phishing-resistant MFA for sensitive apps
Help desk: no MFA reset without verified identity proofing
Sessions: risk-based, device-aware, location-aware
Apps: least privilege, group assigned, reviewed
API tokens: restricted, rotated, monitored
Logs: streamed to SIEM with detections
Break glass: few, monitored, offline-protected
```

For Okta, the highest-value controls are: **phishing-resistant MFA, admin role minimization, help desk reset controls, API token restrictions, device/risk-based policies, and System Log monitoring.**

[1]: https://help.okta.com/oie/en-us/content/topics/identity-engine/authenticators/phishing-resistant-auth.htm "Phishing-resistant authentication"
[2]: https://help.okta.com/oie/en-us/content/topics/security/healthinsight/mfa-admin-console.htm "MFA for the Admin Console"
[3]: https://help.okta.com/oie/en-us/content/topics/security/administrators-super-admin.htm "Super administrators"
[4]: https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-320a "Scattered Spider"
[5]: https://help.okta.com/oie/en-us/content/topics/identity-engine/authenticators/require-phishing-resistant-authenticator.htm "Phishing-resistant authenticator enrollment"
[6]: https://help.okta.com/oie/en-us/content/topics/security/network/add-network-zone-signon-policy.htm "Add a network zone to policies"
[7]: https://help.okta.com/oie/en-us/content/topics/security/threat-insight/about-threatinsight.htm "About Okta ThreatInsight"
[8]: https://help.okta.com/oie/en-us/content/topics/identity-engine/oie-risk-behavior-eval.htm "Risk and behavior evaluation"
[9]: https://help.okta.com/oie/en-us/content/topics/identity-engine/devices/device-assurance.htm "Device assurance"
[10]: https://help.okta.com/en-us/content/topics/security/api.htm "Manage Okta API tokens"
[11]: https://help.okta.com/oie/en-us/content/topics/reports/log-streaming/about-log-streams.htm "Log streaming"

---

## Related

- [OpenID Connect (OIDC) Security Best Practices](oidc-security-best-practices.md)
- [SAML SSO Security Best Practices](saml-security-best-practices.md)
- [Token Security: Types, Usage & Failure Modes](token-security-guide.md)

See [References](references.md) for the full citation registry.
