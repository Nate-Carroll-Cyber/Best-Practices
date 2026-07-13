# Langflow Hardening Guidance v1.1

## Executive Summary

Langflow must be secured as both a web application and a code-execution platform. Flows, custom components, integrations, and model-assisted workflows may execute Python, reach internal services, and use high-value credentials. A compromise must therefore be assumed capable of affecting the worker or container that executes the flow.

Production security depends on independently enforced controls around Langflow:

1. Keep administrative and build interfaces private.
2. Prevent direct access that bypasses the trusted proxy or gateway.
3. Separate administration, build, and execution workloads.
4. Enforce exact destination and resource permissions outside flow logic.
5. Isolate untrusted or mutually untrusted execution.
6. Keep secrets out of flow definitions and issue narrow, short-lived credentials.
7. Test object-level authorization for every resource type and action.
8. Pin, verify, patch, and continuously inventory the deployed software supply chain.
9. Monitor application, process, file, network, identity, and cloud activity together.
10. Rebuild compromised workloads and rotate credentials from a known-clean environment.

This document is implementation guidance, not a statement that any particular Langflow version is secure. Version-specific settings, routes, vulnerabilities, and configuration names must be validated against current official documentation and advisories before deployment.

---

## 1. Purpose and Scope

This guidance applies to Langflow deployments used for:

- Flow development and administration
- Internal or public flow execution
- Custom Python components
- Model and tool integrations
- Retrieval and data-processing workflows
- Shared workspaces
- Managed or multi-tenant services

It covers the Langflow application, reverse proxy, identity provider, database, workers, containers, orchestrator, network controls, secret stores, observability systems, and build pipeline.

### Security assumption

> A compromise of a Langflow execution environment may provide code execution with the operating-system, filesystem, network, database, and credential access available to that environment.

Prompt instructions, application settings, endpoint secrecy, random identifiers, and WAF signatures do not constitute isolation or authorization boundaries.

---

## 2. Deployment Classification

Classify every deployment before implementation:

| Attribute | Questions |
|---|---|
| Exposure | Is any UI or API reachable from the internet, partners, or unmanaged devices? |
| Users | Are all users trusted administrators, internal builders, external customers, or mutually untrusted tenants? |
| Execution | Can users create custom components, execute Python, import flows, or invoke arbitrary tools? |
| Data | Does the deployment process credentials, personal data, regulated data, source code, or production records? |
| Connectivity | Can workers reach internal networks, cloud APIs, databases, CI/CD, source control, or the public internet? |
| Impact | Can flows send messages, change production, transfer funds, delete data, or publish externally? |
| Tenancy | Are control plane, database, workers, secrets, or networks shared across trust boundaries? |

Higher-risk deployments require stronger isolation, shorter credential lifetimes, narrower connectivity, independent approval, and more intensive monitoring.

---

## 3. Version and Advisory Validation

Before relying on any route, environment variable, authentication option, or workaround, maintain a dated validation record:

| Item | Deployed value | Authoritative source | Affected or supported versions | Last validated | Owner |
|---|---|---|---|---|---|
| Langflow image and digest | TBD | Official release source | TBD | TBD | Platform |
| Authentication configuration | TBD | Official documentation | TBD | TBD | IAM |
| CORS configuration | TBD | Official documentation | TBD | TBD | Application security |
| Administrative routes | TBD | Running OpenAPI specification and source | TBD | TBD | Application owner |
| Applicable advisories and CVEs | TBD | Vendor advisory and recognized vulnerability source | TBD | TBD | Vulnerability management |
| Fixed release | TBD | Vendor advisory | TBD | TBD | Vulnerability management |

For every vulnerability, record:

- CVE or advisory identifier
- Affected and fixed versions
- Vulnerable feature or route
- Exploitation prerequisites
- Whether the issue is known to be exploited
- Temporary mitigations and their limitations
- Patch validation evidence
- Date the information was last checked

Endpoint blocking is a temporary containment measure. It does not replace patching or prove that alternate vulnerable routes do not exist.

---

## 4. Reference Architecture

### 4.1 Separate operational planes

Where practical, separate:

- **Access plane:** trusted gateway, identity enforcement, rate limits, and request validation
- **Administration plane:** users, projects, credentials, configuration, imports, exports, and flow editing
- **Build plane:** component validation, package resolution, and flow compilation
- **Execution plane:** constrained workers that run approved flows
- **Data plane:** databases, vector stores, files, and external services
- **Security plane:** secrets, signing keys, audit storage, policy, and monitoring

Execution workers must not possess administrative credentials or unrestricted access to the control plane. Administrative workloads should not execute untrusted flows.

### 4.2 Trust boundaries

Document trust boundaries among:

- Users and browsers
- Proxy and Langflow backend
- Langflow and identity provider
- Control plane and workers
- Workers and tool destinations
- Langflow and database
- Langflow and secret manager
- Tenants or workspaces
- Runtime and observability systems

Every boundary must define authentication, authorization, permitted data, permitted destinations, encryption, logging, timeout, and failure behavior.

---

## 5. Private-First Network Architecture

### 5.1 Inbound access

Do not expose Langflow directly to the public internet unless the use case explicitly requires it and the exposure has been reviewed, approved, and monitored.

Preferred access mechanisms include:

- Private load balancer
- VPN or private overlay network
- ZTNA with corporate identity
- Identity-aware proxy
- Service-mesh ingress
- Controlled administrative bastion

A public tunneling feature must not be described as private access. If public tunneling is used, treat the service as internet-exposed and apply the full public-service security baseline.

### 5.2 Trusted reverse proxy or gateway

Place Langflow behind a managed reverse proxy or API gateway that provides:

- TLS
- Authentication and session enforcement
- Request and response size limits
- Rate and concurrency limits
- Header normalization
- Trusted-proxy handling
- Route-level policy
- Access logging and correlation identifiers
- Optional WAF controls

The architecture must prevent proxy bypass:

- Langflow accepts inbound traffic only from the trusted proxy or service mesh.
- Backend ports are not publicly routable.
- Incoming identity and forwarding headers from clients are stripped.
- The proxy creates trusted identity headers only after authentication.
- Langflow or an authorization layer maps the trusted identity to application permissions.
- WebSocket, streaming, health, API, and static-resource routes are included in the design.
- Host and forwarded-host values are validated against approved domains.

Authentication at the proxy does not replace object-level authorization in Langflow.

### 5.3 Route exposure

Default-deny external routes and expose only those required by the approved use case. Derive the route inventory from the exact deployed version's OpenAPI specification, source, and observed traffic.

Classify routes such as:

- Authentication and session management
- User and role management
- Flow creation, build, import, export, and deletion
- Temporary or public flow building
- Component creation and package installation
- Credential, variable, and API-key management
- Project, folder, workspace, and file management
- Flow execution and response APIs
- Store, template, and sharing features
- Debug, telemetry, and health endpoints

Administrative, build, import/export, and credential-management routes must be restricted to authorized groups and trusted networks. Disable development conveniences, automatic login, anonymous access, and public build features outside isolated local development when the deployed version supports those features.

### 5.4 SSRF controls

Any component that accepts a URL or makes outbound requests must be treated as an SSRF surface.

Prefer exact approved destinations. Where general URL fetching is necessary:

1. Parse the URL with a standards-compliant parser.
2. Permit only required schemes, normally HTTPS.
3. Reject embedded credentials and ambiguous authority syntax.
4. Resolve all addresses through an approved resolver.
5. Reject loopback, link-local, multicast, unspecified, private, reserved, and organization-internal ranges unless specifically approved.
6. Handle IPv4, IPv6, IPv4-mapped IPv6, and alternate address representations consistently.
7. Validate the resolved address immediately before connection.
8. Revalidate every redirect and restrict redirect count.
9. Restrict ports, methods, request size, response size, and timeouts.
10. Prevent access to cloud metadata and workload-identity endpoints.
11. Enforce the same policy at an outbound proxy or firewall.

DNS blocking alone is insufficient because of rebinding and resolution-to-connection races.

### 5.5 Egress control

Do not allow broad cloud-domain wildcards such as entire cloud-provider or API namespaces. Attackers can often create resources beneath trusted provider domains.

Allowlist the smallest practical combination of:

- Exact hostname
- Account, tenant, project, or organization
- Region
- Port and protocol
- API path or service
- Destination certificate or private endpoint

Route outbound traffic through an authenticated egress proxy where possible. Log destination, resolved address, initiating flow, tenant, user, bytes transferred, and decision. Block direct internet access that bypasses the proxy.

### 5.6 Internal segmentation

Langflow workers must not have unrestricted access to:

- Kubernetes or container-management APIs
- Cloud metadata or workload-identity endpoints
- Domain controllers and identity infrastructure
- CI/CD and artifact-signing systems
- Source-control administration
- Production administration panels
- Secret-management administration APIs
- Unrelated databases or private services
- Host sockets or developer workstations

Grant connectivity per approved flow or worker class rather than to the entire Langflow deployment.

---

## 6. Sandboxed Execution and Resource Governance

### 6.1 Execution isolation

Do not execute untrusted flows or custom components in the same long-lived process that manages users, configuration, or sensitive credentials.

Select isolation according to risk:

- Dedicated worker containers for trusted internal flows
- Per-workspace or per-tenant workers for separated business units
- Ephemeral workers for untrusted or externally supplied flows
- Sandboxed runtimes such as gVisor or Kata Containers
- MicroVMs or dedicated compute for high-risk and mutually untrusted tenants
- Separate instances for strong tenant boundaries

Namespaces and service accounts improve organization but do not, by themselves, provide a secure multi-tenant boundary.

### 6.2 Container baseline

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

Also require:

- No privileged containers
- No host network, PID, IPC, or user namespace sharing
- No Docker or container-runtime socket
- No `hostPath` mounts unless formally approved
- Dedicated, size-limited ephemeral `/tmp`
- AppArmor, SELinux, or equivalent policy where supported
- Disabled automatic service-account token mounting unless required
- Explicit CPU, memory, process, file, storage, and network limits
- Immutable, digest-pinned images
- Runtime classes appropriate to the risk tier

### 6.3 Flow quotas and timeouts

Enforce per-user, per-workspace, per-tenant, and per-flow limits for:

- Concurrent executions
- Total execution time
- Model and tool calls
- Token and financial cost
- CPU and memory
- Temporary storage
- Input and output size
- Network requests and transferred bytes
- Retry count and recursive flow depth

Terminate and quarantine executions that exceed defined safety limits.

### 6.4 Process and file monitoring

Monitor process execution, but do not rely on executable names as a prevention boundary. Name-only blocking is bypassable and can disrupt legitimate behavior.

Correlate:

- Parent and ancestor processes
- Executable path and digest
- Command line
- Container image and workload identity
- Initiating user, tenant, flow, and request
- File creation and permission changes
- Network connections
- Credential and metadata access
- Package installation or dynamic loading

Start new runtime rules in alert mode, measure expected behavior, then promote narrowly tuned rules to prevention where operationally safe.

---

## 7. Authentication, Sessions, and API Keys

### 7.1 Corporate identity

Use an organization-managed identity provider for production access where supported. Require phishing-resistant MFA for administrators and high-impact approvals, centralized lifecycle management, group-based access, and conditional access appropriate to risk.

Retain at least one controlled recovery method, protect it with strong procedures, and monitor its use. Disable or tightly restrict local accounts after validating recovery and service-account requirements.

### 7.2 Token and JWT validation

Do not prescribe a JWT algorithm without confirming support and the complete key configuration for the deployed version.

Required controls include:

- Explicit accepted-algorithm allowlist
- Strong, separately managed signing keys
- Correct issuer and audience validation
- Expiration and not-before validation
- Key identifiers and safe rotation overlap
- Rejection of unsigned or unexpected algorithms
- Separate keys across environments and trust domains
- Private keys stored outside images and source control
- Documented emergency rotation

Asymmetric signing can separate signers from verifiers, but RS512 is not inherently preferable to RS256. A well-managed symmetric design may be suitable within a single tightly controlled trust boundary. Choose the design from the architecture and threat model, not the algorithm name alone.

### 7.3 Browser sessions

For cookie-authenticated sessions:

- Use `Secure`, `HttpOnly`, and an appropriate `SameSite` setting.
- Regenerate session identifiers after authentication and privilege changes.
- Set idle and absolute lifetimes.
- Revoke sessions after offboarding, credential change, or incident.
- Evaluate and test CSRF protection for every state-changing route.
- Do not place bearer credentials in URLs.

### 7.4 API keys

API keys must have an owner, purpose, scope, environment, creation time, last-use time, and expiration. Avoid shared administrator keys. Support immediate revocation, rotate keys through a tested process, and record key identifiers rather than key material in logs.

---

## 8. Object-Level Authorization and Tenant Isolation

### 8.1 Server-side authorization

Random or unguessable identifiers are not authorization controls. Every request must verify that the authenticated principal may perform the requested action on the referenced object.

Apply authorization to:

- Flows and flow versions
- Components and templates
- Variables and credentials
- API keys
- Files and datasets
- Projects, folders, and workspaces
- Users, roles, and invitations
- Execution records and outputs
- Imports, exports, clones, shares, and deletes

Object listing must return only objects the caller is authorized to know exist.

### 8.2 Authorization tests

Automate a matrix covering create, list, read, execute, update, clone, share, export, delete, and restore.

Minimum cross-user test:

```text
1. User A creates an object.
2. Capture its identifier and any endpoint alias.
3. Authenticate as User B in the same tenant.
4. Attempt every supported action on User A's object.
5. Confirm denial unless explicit sharing authorizes the action.
6. Repeat as a user in another tenant.
7. Test listing, search, error messages, logs, exports, and indirect references.
```

Test both `403` and non-enumerating `404` behavior according to the chosen disclosure policy. Also test batch APIs, GraphQL or alternate routes, WebSockets, background jobs, and race conditions where applicable.

### 8.3 Multi-tenant deployments

For mutually untrusted tenants, prefer separate:

- Langflow instances or control planes
- Databases or enforced tenant partitions
- Workers and service accounts
- Secrets and encryption keys
- Object storage and queues
- Network and egress policies
- Audit partitions and retention rules

Document any shared component, the isolation mechanism it provides, its failure mode, and the evidence that cross-tenant access is prevented.

---

## 9. Secret and Database Security

### 9.1 Secret handling

Do not place raw production secrets in flow definitions, exports, source control, container images, logs, prompts, or model context.

Prefer:

- Secret references resolved only at execution time
- Workload identity and cloud roles
- Token brokers and just-in-time credentials
- Per-flow, per-workspace, or per-tenant scope
- Short lifetimes and narrow audiences
- Destination-bound credentials

Environment variables are readable by a compromised process and often visible through debugging or runtime tooling. If they are used, mount only required values, keep privileges minimal, and rotate them through a controlled process.

Flow export and sharing must redact secret values and preserve only safe references. Treat an incorrectly exported or logged secret as compromised.

### 9.2 Secret-manager separation

Workers should receive permission to retrieve only the exact secrets needed for the current flow. They must not have administrative access to enumerate, create, modify, or delete unrelated secrets. Record secret identifier, version, requesting workload, flow, tenant, and decision without logging the value.

### 9.3 Canary credentials

Canaries must be nonprivileged, isolated, provider-compliant, and unable to trigger real transactions or material cost.

Different detection mechanisms are required for:

- **Read:** application instrumentation, file auditing, or runtime telemetry
- **Export or logging:** DLP and log scanning
- **Network transmission:** egress monitoring
- **Use:** provider-side or controlled service alerts

A canary does not inherently alert when merely read.

### 9.4 Database controls

- Use TLS in transit and encryption at rest.
- Restrict database connectivity to approved Langflow services.
- Use separate least-privilege identities for application, migration, backup, and administration.
- Prevent workers from accessing database administration functions.
- Monitor bulk reads, credential-table access, exports, and schema changes.
- Encrypt and access-control backups.
- Test restore and migration compatibility before upgrades.
- Define recovery-point and recovery-time objectives.

Encryption at rest protects media and snapshots; it does not protect data readable by a compromised live process.

---

## 10. Browser and Web Security

### 10.1 CORS

CORS is a browser policy, not authentication or authorization. Configure the exact deployed version to allow only approved frontend origins, required methods, and required headers. Do not combine wildcard origins with credentialed requests.

Validate the actual configuration syntax against current official documentation. Test preflight requests, credential behavior, error responses, and every approved frontend.

### 10.2 Content Security Policy

Begin with `Content-Security-Policy-Report-Only`, collect violations, and narrow the policy before enforcement.

Example target shape, requiring adaptation to the deployed UI:

```text
default-src 'self';
object-src 'none';
base-uri 'self';
frame-ancestors 'none';
script-src 'self' 'nonce-{per-response-nonce}';
style-src 'self' 'nonce-{per-response-nonce}';
img-src 'self' data:;
connect-src 'self' https://langflow.example.com wss://langflow.example.com;
```

Do not claim that the scheme source `wss:` is same-origin; it permits WebSocket connections to any host using that scheme. Avoid `'unsafe-inline'` where the application supports nonces or hashes. If it is temporarily required, document the residual risk and removal plan.

### 10.3 Security headers

Use, as applicable:

- `Strict-Transport-Security`
- `X-Content-Type-Options: nosniff`
- CSP `frame-ancestors` and, where required for legacy clients, `X-Frame-Options`
- `Referrer-Policy`
- `Permissions-Policy`
- Appropriate cache-control for sensitive responses

### 10.4 HSTS rollout

Deploy HSTS in stages:

1. Confirm HTTPS and certificate automation.
2. Start with a short `max-age`.
3. Increase after monitoring.
4. Add `includeSubDomains` only after every subdomain is HTTPS-ready.
5. Request preload only after deliberate organizational approval and verification of preload requirements.

Premature `includeSubDomains` or preload can make unrelated services unreachable.

---

## 11. Software Supply-Chain Security

Maintain an SBOM for:

- Langflow and its Python dependencies
- Base image and operating-system packages
- Custom components
- Frontend dependencies
- Database drivers
- Proxies, sidecars, and agents
- Infrastructure modules and deployment charts

Required controls:

- Pull images from approved registries.
- Pin production images by immutable digest.
- Verify signatures and provenance where available.
- Scan images, packages, and configuration before deployment.
- Restrict package installation and dynamic dependency retrieval at runtime.
- Protect CI/CD credentials and signing keys from Langflow workloads.
- Use admission policy to reject privileged, unsigned, mutable-tagged, or noncompliant deployments.
- Generate and retain deployment attestations.

Signatures establish provenance and integrity, not that an artifact is free of vulnerabilities.

---

## 12. Patch and Change Management

Deploy a supported release containing all applicable security fixes after expedited testing. Do not interpret “latest” as automatically approved or secure.

The patch process must:

1. Monitor official security advisories and recognized vulnerability sources.
2. Determine applicability to the exact deployed version and features.
3. Assign remediation deadlines based on exploitation and exposure.
4. Test database migrations, flows, integrations, authentication, and rollback.
5. Verify image signature and digest.
6. Deploy through controlled automation.
7. Verify the running version and digest.
8. Run security-control regression tests.
9. Remove temporary mitigations only after patch validation.

Flow, component, dependency, credential, route, and authorization changes require recorded approval and audit history. Monitor production for unauthorized flow or component changes.

---

## 13. WAF and Request Filtering

A WAF may reduce opportunistic exploitation or provide temporary containment, but it is not a substitute for patching, authentication, authorization, or isolation.

Avoid generic copy-paste keyword blocking as the primary control. Code-oriented workflows may legitimately contain shell or Python strings, while attackers can encode or restructure payloads.

WAF rules should combine:

- Exact vulnerable route and method
- Authentication state and user role
- Content type and schema
- Request size and rate
- Known exploit structure
- Source and threat intelligence
- Observed false-positive rate

Deploy new signatures in detection mode where possible, validate their effectiveness, and set an expiration date for temporary rules.

---

## 14. Logging and Detection

### 14.1 Application and gateway logging

Record:

- Timestamp and correlation identifier
- Authenticated user, service, tenant, and workspace
- Source and trusted proxy information
- Method, normalized route, status, and latency
- Request and response size
- Flow, component, project, or object identifier
- API-key identifier, not value
- Authorization decision and policy version
- Administrative and credential-management events

Do not log bearer tokens, secret values, complete credential objects, or sensitive prompts by default. Apply data classification, redaction, access control, integrity protection, and retention limits.

### 14.2 Detection use cases

Alert on correlated behaviors such as:

- Access to disabled or restricted build and administrative routes
- Cross-user or cross-tenant object access attempts
- Unusual object listing followed by execution or export
- New users, roles, API keys, variables, credentials, or components
- Unexpected flow imports, exports, clones, or modifications
- Flow execution followed by an unapproved destination
- Secret access followed by model submission or network transfer
- Shell or downloader execution from a worker
- Executable creation in temporary storage
- Reads of environment, local database, service-account, or cloud metadata data
- Disabled or missing logs from an active workload
- Large response volumes, token costs, retries, or execution fan-out

Prompt strings such as requests to reveal secrets are useful context but are weak indicators by themselves. Detect the resulting access and execution behavior.

### 14.3 Runtime and cloud telemetry

Correlate application logs with:

- Container runtime events
- Process, file, and network telemetry
- DNS and egress-proxy logs
- Identity-provider and API-key events
- Secret-manager audit logs
- Database audit records
- Cloud control-plane logs
- Kubernetes audit and admission events

Tune detections using representative benign workflows and adversarial tests.

---

## 15. Incident Response

### 15.1 Containment

1. Activate the incident-response lead and record the timeline.
2. Restrict public and administrative access at independently controlled network layers.
3. Isolate affected workers or instances without destroying volatile evidence unnecessarily.
4. Revoke active sessions and high-risk credentials through clean control systems.
5. Block malicious destinations and vulnerable routes.
6. Preserve application, proxy, identity, database, secret-manager, runtime, and cloud logs.
7. Capture forensic images or snapshots under approved chain-of-custody procedures.

Do not rotate credentials through a suspected compromised host.

### 15.2 Investigation

Determine:

- Initial access and vulnerable feature
- Affected version, instance, user, tenant, and flow
- Commands, components, files, and child processes executed
- Network destinations and transferred data
- Credentials and secrets accessible to the workload
- Database and object access
- New or modified users, roles, keys, variables, flows, and components
- Lateral movement into cloud, CI/CD, source control, or internal services
- Whether logging or security controls were altered

### 15.3 Eradication and recovery

- Rebuild from a known-good, verified, patched image.
- Remove unauthorized identities, keys, components, and persistence.
- Rotate exposed or potentially exposed credentials from a clean environment.
- Validate database integrity and restore where required.
- Reapply least privilege and exact egress controls.
- Test authentication, authorization, isolation, monitoring, and emergency controls.
- Restore service in phases behind authenticated access.
- Monitor closely for recurrence.

Do not restart a compromised container and treat the incident as resolved.

### 15.4 Post-incident work

- Complete root-cause and control-gap analysis.
- Assess notification and regulatory obligations.
- Update threat models, detections, tests, and architecture.
- Record lessons learned and remediation owners.
- Verify remediation independently before closing the incident.

---

## 16. Production Security Gate

A production deployment requires evidence that:

### Access and identity

- [ ] Langflow is private unless public exposure is explicitly approved.
- [ ] Direct backend access cannot bypass the proxy.
- [ ] Corporate identity and MFA protect administrative access.
- [ ] Trusted identity headers are stripped and regenerated by the proxy.
- [ ] Sessions, recovery accounts, and API keys meet policy.

### Routes and authorization

- [ ] The route inventory matches the deployed version.
- [ ] Development conveniences and unused public routes are disabled or blocked.
- [ ] Administrative and build routes are group-restricted.
- [ ] Object-level authorization covers every object and action.
- [ ] Cross-user and cross-tenant tests pass.
- [ ] CSRF behavior is tested for state-changing browser routes.

### Runtime

- [ ] Administration and untrusted execution are separated.
- [ ] Containers run non-root without privilege escalation.
- [ ] Root filesystems are read-only and capabilities are dropped.
- [ ] Runtime, seccomp, and mandatory-access-control policies are active.
- [ ] Resource, cost, concurrency, retry, and depth limits are enforced.
- [ ] Mutually untrusted tenants have an approved isolation architecture.

### Network

- [ ] Inbound access is restricted to approved gateways.
- [ ] SSRF defenses validate resolution, connection, and redirects.
- [ ] Egress uses exact destinations rather than broad cloud wildcards.
- [ ] Cloud metadata and management APIs are blocked unless required.
- [ ] Internal connectivity is granted per worker or flow class.

### Secrets and data

- [ ] Raw secrets are absent from flows, exports, images, logs, and prompts.
- [ ] Workers retrieve only task-required secrets.
- [ ] Credentials are narrow, short-lived, and independently revocable.
- [ ] Database access, backup, migration, and restore controls are tested.
- [ ] Canary credentials are safe, isolated, and monitored by the appropriate mechanism.

### Browser security

- [ ] CORS syntax and behavior are validated against the deployed version.
- [ ] CSP has been tested in report-only mode and reviewed before enforcement.
- [ ] WebSocket destinations are exact, not scheme-wide.
- [ ] HSTS scope has been staged and approved.
- [ ] Host-header and trusted-proxy behavior are tested.

### Supply chain and patching

- [ ] The image is supported, patched, signed where available, and digest-pinned.
- [ ] An SBOM and deployment attestation are retained.
- [ ] Applicable advisories and fixed versions are documented.
- [ ] Database migrations and rollback are tested.
- [ ] Admission policy rejects noncompliant workloads.

### Monitoring and response

- [ ] Application, gateway, runtime, identity, database, secret, and cloud logs are correlated.
- [ ] Cross-tenant, process-execution, egress, and secret-access alerts are tested.
- [ ] Logs are access-controlled, integrity-protected, and retained.
- [ ] Incident response, clean credential rotation, rebuild, and recovery have been exercised.
- [ ] Recovery objectives are documented.

Production approval must identify the accountable owner, evidence reviewed, residual risks, exceptions, and review date. Critical findings block deployment. High-severity findings require remediation or a time-bounded exception approved by the authorized risk owner.

---

## 17. Minimum Security Priorities

### Required before production

1. Private access or explicitly approved public architecture
2. Proxy anti-bypass and corporate authentication
3. Object-level and tenant-aware authorization
4. Separate, sandboxed execution workers
5. Exact egress and SSRF controls
6. Secret references and least-privilege credentials
7. Supported, patched, digest-pinned images
8. Application and runtime monitoring
9. Tested incident containment and rebuild
10. Evidence-based production approval

### Required before multi-tenant or external scale

1. Strong tenant isolation with documented shared components
2. Automated cross-tenant authorization testing
3. Per-tenant workers, secrets, keys, and egress policy
4. Flow quotas and financial guardrails
5. Supply-chain admission and attestations
6. Independent security assessment

---

## Conclusion

Langflow is safest when the application is treated as an orchestration and code-execution control plane rather than an ordinary internal web interface. The production boundary must be formed by private access, anti-bypass identity enforcement, server-side authorization, isolated workers, exact network policy, narrow credentials, verified artifacts, and correlated monitoring. Version-specific controls must be continuously revalidated because routes, defaults, authentication behavior, and vulnerabilities change over time.
