# Securing Agentic AI: Technical Implementation Framework

**Version:** Consolidated 2.3
**Status:** Implementation guidance. Regulatory mappings are planning aids, not legal advice, certification, or evidence of conformity.
**Changes from 2.2:** sixth governing principle on capability combinations; trifecta added to system classification (§2.1); server-side authorization decisions, OAuth 2.1, and resource-indicator token binding (§4.3); server registration and shadow discovery (§4.6); per-server kill switch (§5.3); OAuth client/authorization-server role separation (§6.3); infrastructure-enforced tenant isolation and ingestion-time classification (§7.4); **intent integrity and the capability trifecta (§7.8)**; log-store minimization and retention (§9.1); SIEM/XDR integration and three new monitoring signals (§9.3); compromise-scenario drills (§9.4); tenant-isolation and telemetry tests plus verification probes (§11); new coverage rows (§14); Priority 1 and 2 additions (§16); glossary and reference additions (§17, §18).
**Changes from 2.1 (delivered in 2.2):** secret custody and model-context exclusion, and untrusted project configuration (§4.5); local-transport binding and origin validation (§6.2); full-schema metadata scanning, display-control stripping, and TOFU pinning (§6.4); error design (§6.5); service surface hardening baseline (§6.7); safe command and process execution (§7.6); model invocation guardrails (§7.7).

**Numbering note:** All 2.2 and 2.3 additions append at the tail of an existing section. Section numbers 1–18 and every cross-reference established in 2.1 are unchanged.

## Executive Summary

Agentic AI systems can plan, invoke tools, modify external systems, maintain state, and delegate work. Their security therefore depends on more than model-level safeguards. Production deployments require independently enforced identity, authorization, isolation, data protection, auditability, human oversight, and incident response.

This framework is built on six principles that govern every control below:

1. **The model proposes; deterministic controls decide.** A model must never grant itself authority or act as the final policy-enforcement point. Message roles, delimiters, and prompt separation improve model interpretation but are not access controls.
2. **Every action has an authenticated principal.** Human, service, agent, and delegated identities remain traceable throughout execution.
3. **Authority is narrow, temporary, and context-bound.** Permissions are task-scoped, audience-restricted, and resistant to replay.
4. **Untrusted content remains untrusted.** User input, retrieved content, tool output, memory, and inter-agent messages cannot become policy merely by entering the context window — and they cannot redefine the objective the agent was given.
5. **High-impact actions remain interruptible and accountable.** Human approval, circuit breakers, and independently controlled audit records constrain autonomy.
6. **Capability combinations are the unit of risk.** An agent that can reach private data, ingest untrusted content, and communicate externally is exploitable regardless of how well each capability is individually controlled. Assess and constrain the combination, not only the parts.

Throughout this document, code and configuration snippets are **illustrative sketches that must be validated against current library, protocol, and platform documentation** before use. They are not drop-in reference implementations.

---

## Table of Contents

- [1. Purpose, Scope, and Audience](#1-purpose-scope-and-audience)
- [2. Governance, Classification, and Accountability](#2-governance-classification-and-accountability)
- [3. Reference Architecture and Trust Boundaries](#3-reference-architecture-and-trust-boundaries)
- [4. Identity and Authorization](#4-identity-and-authorization)
- [5. Execution Isolation and Containment](#5-execution-isolation-and-containment)
- [6. MCP and Tool Security](#6-mcp-and-tool-security)
- [7. Input, Context, Memory, and Output Security](#7-input-context-memory-and-output-security)
- [8. Human Oversight](#8-human-oversight)
- [9. Audit, Monitoring, and Incident Response](#9-audit-monitoring-and-incident-response)
- [10. Supply-Chain Security](#10-supply-chain-security)
- [11. Threat Modeling and Security Validation](#11-threat-modeling-and-security-validation)
- [12. Production Security Gate](#12-production-security-gate)
- [13. Phased Deployment](#13-phased-deployment)
- [14. Control Coverage and Evidence](#14-control-coverage-and-evidence)
- [15. Regulatory and Standards Alignment](#15-regulatory-and-standards-alignment)
- [16. Implementation Priorities](#16-implementation-priorities)
- [17. Glossary](#17-glossary)
- [18. Reference Categories](#18-reference-categories)

---

## 1. Purpose, Scope, and Audience

This framework applies to AI systems that can independently select or invoke tools, interact with external services, maintain state, delegate tasks, or cause material effects outside the model runtime.

It is intended for system and security architects; engineering and platform teams; product and model owners; security operations and incident-response teams; privacy, legal, compliance, and risk functions; and internal and external assessors.

Each deployment must define its intended purpose, prohibited uses, affected persons, operating environment, data classifications, maximum autonomy, and accountable owner before production use.

---

## 2. Governance, Classification, and Accountability

### 2.1 System Classification

Before development and again before deployment, classify the system using:

- Intended purpose and reasonably foreseeable misuse
- Decisions or actions the system can perform
- Data sensitivity and affected populations
- Financial, legal, safety, privacy, and operational impact
- Degree of autonomy and reversibility
- External connectivity and tool privileges
- Multi-agent and third-party dependencies
- Whether the system simultaneously holds all three legs of the capability trifecta in §7.8 — private-data access, untrusted-content ingestion, and external communication
- Applicable jurisdictions and sector-specific obligations

Do not treat a simplified risk-tier label as a complete legal classification. Regulatory applicability depends on the system, use case, jurisdiction, and the organization's legal role.

### 2.2 Responsibility Model

Security across the agentic supply chain is a shared responsibility. Assign each role an accountable owner.

| Role | Core responsibility |
|---|---|
| **Agent owner** | Defines purpose, acceptable use, policies, guardrails, risk tolerance, and operating constraints; accepts documented residual risk within delegated authority. |
| **Service provider** | Designs, operates, patches, tests, and documents the agent service, including design integrity and skills. |
| **Model provider** | Supplies model capabilities and relevant documentation under the applicable service agreement. |
| **Tool or MCP provider** | Secures tool behavior, interfaces, dependencies, credentials, and change lifecycle; provides supply-chain transparency. |
| **Data owner** | Authorizes data use, classification, access, retention, and deletion. |
| **CISO or security delegate** | Approves security architecture, testing criteria, exceptions, and incident-response integration. |
| **Privacy and legal functions** | Determine applicable privacy, AI, consumer, employment, and sector-specific obligations. |
| **Human approver** | Reviews high-impact actions within a defined scope and remains accountable for the approval decision. |

Organizational accountability assignments do not, by themselves, establish statutory liability. Legal conclusions require review by qualified counsel.

### 2.3 Required Governance Artifacts

Every production agent must maintain:

- System and data-flow diagrams with trust boundaries
- Threat model and abuse-case analysis
- Tool and capability inventory
- Model, software, data, and prompt provenance records
- Security and privacy test results
- Human-oversight and escalation procedures
- Incident-response and recovery plans
- Control evidence and residual-risk decisions
- Versioned change and approval history

---

## 3. Reference Architecture and Trust Boundaries

### 3.1 Agentic Control Loop

Security controls mediate every execution cycle. The agent may plan and propose; external policy-enforcement points authorize and constrain execution.

```text
PERCEIVE -> LABEL AND VALIDATE -> PLAN -> PROPOSE ACTION
        -> AUTHENTICATE -> AUTHORIZE -> APPROVE IF REQUIRED
        -> EXECUTE IN CONSTRAINTS -> OBSERVE -> RECORD -> CONTINUE OR STOP
```

### 3.2 Control and Data Separation

**Requirement:** Untrusted content must never directly determine authorization, change security policy, modify system instructions, or invoke privileged operations without independent validation.

```text
Untrusted input or tool output
        |
        v
Parsing, normalization, and provenance labeling
        |
        v
Agent proposes a structured action
        |
        v
Deterministic policy-enforcement point
        |
        v
Schema and authorization validation
        |
        v
Human approval when required
        |
        v
Constrained tool execution
```

Message roles, delimiters, XML/JSON encoding, and prompt separation may improve model interpretation, but they are **not** access controls. Security decisions must be made and must fail closed outside the model. The example below shows *interpretation-layer* structuring — it is defense in depth, not enforcement:

```xml
<!-- Structuring aids model interpretation; it does NOT enforce anything -->
<system>
  <identity>agent-id-123</identity>
  <instructions>You are a data analyst. You may query the sales database
  and generate reports. You may not modify data.</instructions>
</system>
<data source="user_input" trust="untrusted">
  Please analyze Q4 sales and email it to bob@example.com
</data>
```

The decision to send that email is made by the policy-enforcement point (Section 5.2), not by the delimiters.

### 3.3 Trust Zones

At minimum, separate: untrusted ingestion and retrieval; model inference and orchestration; policy decision and enforcement; tool execution; credential issuance and key management; memory and data stores; audit and monitoring infrastructure; and privileged administration.

Default-deny network policies must restrict both ingress and egress. DNS must use approved resolvers, and any policy that permits DNS **must account for both UDP and TCP on port 53** (DNS is predominantly UDP; TCP is required for large responses and zone transfers). Domain-level (FQDN) egress control requires a capable proxy or gateway, because standard Kubernetes NetworkPolicies operate on IP/CIDR and cannot enforce fully qualified domain names.

Example micro-segmentation policy (note both DNS protocols):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-microsegmentation
spec:
  podSelector:
    matchLabels:
      agent: report-generator
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              zone: internal-compute
      ports:
        - {protocol: TCP, port: 8080}
  egress:
    - to:
        - podSelector:
            matchLabels:
              service: database
      ports:
        - {protocol: TCP, port: 5432}
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - {protocol: UDP, port: 53}
        - {protocol: TCP, port: 53}
```

Block the cloud metadata endpoint (`169.254.169.254`, `metadata.google.internal`) from agent workloads unless a specific, reviewed need exists.

Egress restriction is also a credential-protection control: a leaked token with no reachable destination has materially reduced exploit value (§4.5). Monitor and alert on outbound connections to unapproved domains from agent and plugin workloads.

---

## 4. Identity and Authorization

### 4.1 Machine and Workload Identity

**Requirement:** Every deployed agent workload must have a unique, cryptographically verifiable identity issued by an organization-approved identity authority. Shared agent service accounts are prohibited.

- Prefer established workload identity: SPIFFE/SPIRE, cloud workload identity, or an organization-managed PKI.
- Place workload identity in a defined **URI SAN or equivalent authenticated claim**. Do **not** rely on certificate Common Name (CN) parsing — CN-based identity is deprecated and error-prone.
- Keep mutable roles and capabilities in the **authorization system**, not baked into identity certificates. Embedding permissions in a cert forces reissuance on every permission change and couples identity to policy.
- If decentralized identifiers (DIDs) are used, select or define a DID method with documented issuance, resolution, rotation, recovery, and revocation.
- Automate certificate and key rotation, revocation, and compromise response.

**Prohibited anti-patterns:** shared service accounts across agents; generic identities (`agent-1`, `automation-bot`); long-lived static API keys or bearer tokens as primary agent credentials; capability lists embedded in certificate SANs; **symmetric-key (HMAC) signing of audit or attestation records, where any authorized verifier can also forge; and self-verifying identity, where issuance, signing, and verification reside in the same process or trust domain.** With symmetric schemes, verification capability equals forgery capability, so no non-repudiation exists. Use asymmetric signing with verifiers that hold only public material and validate a trust chain to an independent authority.

> **Key custody closes exfiltration, not live misuse.** Holding agent signing keys in a KMS/HSM — the workload calls a signing API and never handles raw key bytes — prevents key theft and establishes a hard revocation boundary: records signed after credential revocation are impossible. It does **not** prevent a live, compromised workload from producing validly signed but false records through its legitimate signing access, because the signer is the compromised party. Treat KMS/HSM custody as a revocability control and pair it with the corroboration requirements in §9.5.

Identity establishes the calling workload. It does not grant permission by itself.

### 4.2 Least Agency and Delegation

Each agent receives only the tools, resources, data, destinations, and action types required for its declared task. Prefer capability-scoped allowlists over broad grants, with explicit denies for sensitive operations.

Delegation must satisfy all of the following:

- A child cannot receive authority exceeding its parent's delegable authority.
- Each hop narrows or preserves scope; it never silently expands it.
- Delegation records identify the human principal, parent, child, purpose, scope, and expiration (provenance tracking back to the accountable owner).
- Maximum delegation depth and fan-out are enforced.
- Child credentials and sessions terminate with the task or parent unless explicitly and safely re-parented. Orphaned agents are prohibited.
- Policy is re-evaluated at every delegation and tool boundary.

Review **cumulative** effective permissions per agent identity, not only individual grants. Scope creep is usually additive and individually defensible; it is visible only in aggregate. Remove write and delete capability unless a specific declared task requires it, and re-justify what remains at each recertification (§16, Priority 2).

### 4.3 Just-in-Time Authorization

Credentials must be short-lived, audience-restricted, task-scoped, and resistant to replay.

1. Authenticate the agent using workload identity.
2. Evaluate the requested action against user authority, task, resource, environment, and policy.
3. Issue only the required scopes for the intended audience, with a short TTL (typically minutes).
4. **Bind the token to the workload using an approved proof-of-possession mechanism** — mTLS confirmation (`cnf`) claims or DPoP. This is what prevents token transfer.
5. Validate issuer, audience, expiration, scope, proof-of-possession, and revocation state at **every** enforcement point.
6. Revoke the credential or allow it to expire when the task ends.

> **Correction of a common myth:** Including an agent fingerprint in the `jti` claim does **not** make a token non-transferable. The `jti` supports uniqueness and replay detection only. Non-transferability requires proof-of-possession binding (mTLS `cnf` or DPoP).

Implementation options: OAuth 2.0 client credentials + mTLS for agent-to-service; SPIFFE/SPIRE for Kubernetes-native workload identity; cloud STS role assumption for cloud deployments. Maintain a revocation mechanism (CRL/OCSP or short TTLs) with a defined maximum revocation latency.

Permissions and scopes must themselves carry expiry so that access is re-evaluated rather than inherited indefinitely. Manage permission definitions as policy-as-code in the change pipeline so every scope change has a reviewer, a reason, and an audit record; a scope change with no attributable author is a finding.

Development, staging, and production must be separated logically and physically. Development credentials must not reach production systems or data, and roles and credentials must not be reused across environments.

**Authorization is decided server-side from verified facts.** Never derive an authorization decision from client-supplied identity, roles, or scopes. Validate every token at the receiving service on every request — signature, issuer, audience, expiry, intended resource, and authorization for the specific action — rather than once at session establishment.

Where OAuth is used, prefer OAuth 2.1 with per-user delegation, and bind each token to its intended target using **resource indicators** (RFC 8707) so a token issued for one server cannot be replayed against another. Do not share client identifiers or client credentials across agents, and do not reuse tokens between agents. The same token presented under two agent identities, or from two network locations, is an incident signal (§9.3).

### 4.4 Agent and Service Discovery

Discovery metadata must come from an authenticated registry and include service identity, approved endpoint, protocol, owner, current version, and health status. Registry responses must be integrity-protected (signed), access-controlled, logged, and subject to expiration. Require DNSSEC or equivalent for registry resolution where applicable. Discovery does **not** replace authentication or authorization at the destination service.

### 4.5 Secret Custody and Model-Context Exclusion

**Requirement:** The model never receives a credential value. A tool does not need the model to see a secret in order to use it — credential-resolving middleware sits between the model runtime and the protected service, and the agent references an opaque handle that the broker (§5.2) resolves at execution time.

- Store secrets in a managed vault or secrets manager. Not in `.env` files, configuration files, container images, source control, prompts, or logs.
- Fetch credentials at execution time. Long-lived secrets injected as environment variables are prohibited for production agents; where a bootstrap secret is unavoidable, scope it to vault authentication only and nothing else.
- No secret may enter prompts, context windows, model memory, decision records, vector stores, embeddings, or any other model-readable state.
- Prefer short-lived, task-scoped credentials (§4.3) over any static key. A credential must not outlive the work it was issued for.
- Run secret detection and redaction over prompt logs, application logs, tool output, and error payloads before persistence or display (§6.5.4).
- Shared and static service accounts are prohibited (§4.1). Every credential must be independently attributable and independently revocable.

**Untrusted project configuration.** Repository and workspace configuration — agent settings files, MCP server definitions, tool manifests committed to a project, auto-approval directives, and IDE or runtime config — is *active input*, not inert data. It can redirect authenticated traffic, register additional servers, or pre-approve tools. Do not load project-supplied agent configuration before an explicit trust decision by the accountable owner, and treat configuration changes with the same review requirement as code.

**Retrospective sweep for existing systems.** On onboarding a system that predates these controls, scan repository history, configuration, and at minimum the full log-retention window for token-like strings. Rotate everything found. Absence of evidence of exposure is not evidence of non-exposure; where log retention is shorter than the credential's lifetime, rotate by default.

### 4.6 Server Registration and Shadow Discovery

The registry in §4.4 answers "where is this service." This section answers the harder question: **what is running that nobody registered.** Unregistered tool servers — stood up on a workstation for an afternoon, cloned from a repository, left running after a prototype shipped — hold real credentials, sit outside monitoring, and are absent from every inventory the incident-response team will consult.

- **Registration is a deployment gate, not a courtesy.** An unregistered server must fail to deploy, fail to obtain credentials, or fail to be reachable. A registry that depends on voluntary compliance measures compliance, not exposure.
- Every registered server carries a named owner and a lifecycle state, and is revalidated on a schedule. Approval is not permanent.
- **Run continuous discovery** across networks, source repositories, developer environments, container registries, and CI/CD — not a one-time inventory. Reconcile discovered services against the registry and investigate gaps in both directions.
- Alert whenever an agent connects to an endpoint absent from the approved registry. This is the highest-yield detection available for shadow infrastructure, because it catches servers no scan reached.
- Search repositories for MCP and agent configuration files (`mcp.json` and equivalents) as a first-pass inventory, then reconcile against network scans and the registry.
- Include common development ports in discovery scans, recognizing that this finds only the careless cases.
- **Make the compliant path the fast path.** Publish secure-by-default deployment templates with hardening, central identity, and telemetry already wired in. Shadow servers proliferate wherever the sanctioned route is slower than the unsanctioned one; this is a friction problem before it is a policy problem.
- Require SSO/OIDC on every server and tie every service to central identity. Eliminate default credentials and permissive development configurations before any production use.
- Use network segmentation so an unknown or forgotten server cannot reach production systems or sensitive data.
- Maintain a **kill switch** capable of disabling a specific server within minutes (§5.3).

---

## 5. Execution Isolation and Containment

### 5.1 Sandboxing

Run untrusted code and high-risk tools in isolated, disposable execution environments.

Minimum controls: non-root execution; no privilege escalation; all Linux capabilities dropped unless explicitly justified; read-only root filesystem; dedicated ephemeral writable paths; seccomp and mandatory access-control (AppArmor/SELinux) policy; CPU, memory, process, file-size, and execution-time limits; default-deny network access; no host sockets, device access, or cross-agent volume mounts; and image/dependency integrity verification.

For higher-risk or multi-tenant workloads, use stronger boundaries: microVMs (Firecracker), sandboxed runtimes (gVisor), Kata Containers, or dedicated per-tenant workers.

Example Kubernetes security context — the read-only filesystem is enforced by `readOnlyRootFilesystem`, **not** by mounting a ConfigMap over `/`:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop: ["ALL"]
resources:
  limits: {memory: "256Mi", cpu: "500m"}
  requests: {memory: "128Mi", cpu: "250m"}
volumeMounts:
  - {name: tmp, mountPath: /tmp}
volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 100Mi
```

> **Seccomp caution:** A syscall allowlist for a tool sandbox must permit the syscalls the runtime actually needs. Blanket-denying `execve`, `fork`, and `clone` will prevent most interpreters and subprocess-based tools from running at all. Build and test the profile against the real workload rather than copying a restrictive template.

Agents must not run with root, administrator, or unrestricted `sudo` privileges for operational convenience.

### 5.2 Tool Execution Broker

All consequential tool calls pass through a deterministic enforcement broker (policy-decision + policy-enforcement point) that:

- Authenticates the initiating principal and agent
- Validates the tool and its pinned version
- Validates a strict request schema
- Evaluates policy using canonical action data
- Applies resource, destination, and rate limits
- Obtains bound human approval where required
- Issues task-scoped credentials
- Executes in the required isolation boundary
- Validates and classifies results
- Records the decision and outcome

**Tool output must never automatically trigger another tool.** Each subsequent action requires a new proposal and a new policy decision. Enforce per-call timeouts and output-size limits, and do not auto-retry failed privileged calls.

The broker's decision records constitute an independently authored event stream when the broker runs outside the agent runtime; §9.5 relies on this stream for cross-stream corroboration.

Every broker decision — permit, deny, approve-required, and error — carries the correlation identifier defined in §6.5.2, so that the model-facing error, the client response, the operational log, and the audit record for a single event can be joined without inference.

### 5.3 Circuit Breakers and Emergency Stops

Independent containment mechanisms must be able to reject actions, revoke credentials, close network paths, terminate tools, quarantine sessions, preserve forensic evidence, and alert responders. This also satisfies the "stop button" expectation for human oversight of high-risk systems.

Example trigger conditions: error-rate spikes, probable infinite loops (excessive loop depth), resource exhaustion, anomalous tool-call volume, repeated unauthorized tool access, and detected prompt-injection attempts.

Automatic recovery (a half-open retry) may be used for low-risk, idempotent operations. Breakers involving privileged, destructive, financial, safety-critical, or externally visible actions **remain latched** until an authorized operator reviews and resets them. Recovery tests must use non-destructive synthetic actions.

Containment must extend to infrastructure, not only to sessions. An authorized operator must be able to disable a specific tool server, revoke its credentials, and cut its network reachability within minutes, without waiting on a deployment cycle (§4.6).

---

## 6. MCP and Tool Security

### 6.1 Threat Model

Treat MCP servers and other dynamic tool providers as untrusted supply-chain and execution dependencies. Relevant threats: malicious or changed tool descriptions; tool substitution or rug pulls; excessive or undisclosed permissions; token passthrough and confused-deputy behavior; cross-session or cross-tenant leakage; unsafe error disclosure; tool-output prompt injection; unauthorized network callbacks; dependency compromise; and sandbox escape.

### 6.2 Local Transports (STDIO / Unix Sockets)

Local servers must run as isolated child workloads with pinned artifacts (digest), minimal filesystem access, explicit environment variables, no inherited secrets, constrained network access, and deterministic shutdown. Local transport does **not** imply the tool implementation is trusted.

Local does not mean unreachable:

- Bind local servers, debug interfaces, and development UIs to `127.0.0.1`/localhost. Binding to `0.0.0.0` or all interfaces exposes a "local" service to the entire network segment.
- Require authentication and `Origin`/`Host` validation on any local HTTP interface, and reject unauthorized cross-origin requests. Unauthenticated localhost interfaces are reachable from a browser tab via DNS rebinding.
- Inventory MCP servers running on developer workstations. Developer machines hold production-adjacent credentials and are in scope, not out of it.

### 6.3 Remote Transports (Streamable HTTP)

Remote servers require TLS with server authentication, protocol validation, scoped authorization, replay resistance, rate limiting, and centralized policy enforcement. Apply the surface baseline in §6.7 to every remote tool endpoint. Treat MCP as an API and apply the organization's established API-security controls to every request; the protocol's newness does not create an exemption.

**Keep the OAuth client and authorization-server roles separate.** A component that both issues tokens and consumes them has no trust boundary between issuance and use — the §4.1 self-verifying anti-pattern reappearing at the protocol layer. Preserve independent trust domains, validate redirect URIs strictly, and bind tokens to their target with resource indicators (§4.3).

**Prohibit token passthrough (confused deputy).** Do not forward a client's bearer token to downstream services. Obtain a **distinct downstream token** with the correct audience and minimum scope through an approved delegation or token-exchange flow (e.g., RFC 8693).

```python
# CORRECT: exchange for a new, audience-scoped downstream token
delegated = token_exchange(
    subject_token=incoming_client_token,
    resource="https://database-api.example.com",
    scope="read:sales",              # minimum needed
)
# WRONG: requests.get(url, headers={"Authorization": f"Bearer {incoming_client_token}"})
```

### 6.4 Tool Manifests and Change Control

Approved tools must have a versioned manifest recording: stable identifier and owner; artifact digest and provenance; input/output schemas; required permissions and destinations; data classifications; side effects and reversibility; resource limits; dependency inventory; and review/expiration dates.

Validate the manifest signature and pinned version at load time, and re-validate on every connection. Detect drift by comparing the current artifact hash to the approved hash; block execution and require human re-approval on mismatch.

**Scan the entire schema, not the description field.** Every metadata field reaches model context and is therefore instruction-grade untrusted content. Review and scan tool names, parameter names, descriptions, parameter definitions, enum values, and **default values**. Default values are the most commonly skipped field and the most useful to an attacker, because they take effect without the model ever mentioning them.

- **Strip ANSI escape sequences and other display-control characters** from schema fields before human review. Content that renders as benign in a terminal or web reviewer can carry hidden instructions; reviewers must see the actual bytes.
- Where a signing authority is unavailable, pin at the host on first use (TOFU) and flag every subsequent change. Treat TOFU as a weaker fallback than signature verification, not an equivalent.
- Require RBAC on any writable schema registry, and require human review before schema edits are promoted. Reject or quarantine unexpected runtime schema changes.
- Alert on any change to the advertised tool list or to any tool definition. **A tool that rewrites its own definition is a security event, not a version update.**
- Human approval of an *action* is not mitigation for poisoned *metadata*, because the model consumed the metadata before the approval prompt was generated. Validate metadata before it reaches context.
- Isolate tools from one another so that one tool cannot induce another into accessing or exporting unrelated data.

> **Signatures and hashes establish artifact integrity and origin — not benign behavior.** Enforce declared permissions at runtime and audit actual tool behavior against the advertised capability. Require re-review whenever code, description, schema, permissions, owner, dependencies, or endpoint changes.

### 6.5 Error Design

Errors are not an afterthought of the happy path; they are a security surface in four directions at once. An error can leak internals to a caller, inject attacker-controlled text into model context, dump credentials into a log, or silently degrade a denial into a permit. Design the error path with the same rigor as the authorization path.

#### 6.5.1 Four audiences, four projections

A single failure produces different content for different audiences. Do not emit one blob to all of them.

| Audience | Receives | Must never receive |
|---|---|---|
| **Model / agent runtime** | Stable error code, a fixed-vocabulary remediation hint, an escalate flag, correlation ID | Stack traces, query text, paths, hostnames, downstream error strings, any credential |
| **Calling client / user** | Status code, stable error code, generic message, correlation ID | Internal identifiers, dependency names, schema internals, stack traces |
| **Operational log** | Full detail: stack trace, request context, dependency response, timing | Unredacted secrets or personal data (§6.5.4) |
| **Audit stream (§9.1)** | Authenticated principal, action, decision, error category, correlation ID, outcome | Free-form downstream text that could be attacker-controlled |

The four projections are joined by one correlation identifier and nothing else. An operator reconstructs the incident by correlation ID; the model never needs the detail, and the caller never receives it.

#### 6.5.2 Canonical error envelope

Every service, tool, and broker emits the same error shape. Divergent error shapes between components are a real vulnerability class: they make failures unparseable, force per-caller special-casing, and cause callers to fall back on string matching against downstream text.

```json
{
  "error_code": "UNAUTHORIZED",
  "error": "Request could not be authorized.",
  "retry_with": null,
  "escalate": false,
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-07-25T12:00:00+00:00"
}
```

| Field | Purpose | Constraint |
|---|---|---|
| `error_code` | Stable, enumerated machine identifier | Drawn from a fixed vocabulary; never generated from downstream text |
| `error` | Short human-readable message | Generic; contains no internal detail |
| `retry_with` | Structured remediation hint the agent may act on | **Selected from a fixed enumeration, never interpolated from a downstream error string** |
| `escalate` | Whether the agent must stop and hand to a human rather than retry | Set by policy, not by the model |
| `correlation_id` | Joins the four projections | Validated UUID; generated if the caller did not supply a valid one |
| `timestamp` | Event time | UTC, ISO 8601 |

> **`retry_with` is the injection surface.** It is the only error field that shapes subsequent agent behavior, which makes it the field an attacker wants to control. If a downstream service's error text is passed through into `retry_with`, a compromised or hostile dependency can steer the agent through the error path — the same threat as tool-output prompt injection (§7.1), arriving through a channel that is usually excluded from content scanning. Map downstream failures to your own enumerated hints; never forward their prose.

The success path takes a matching envelope so that telemetry is uniform across outcomes:

```json
{
  "tool": "list_aws_resources",
  "input": {"service_type": "ecs", "region": "us-east-1"},
  "status": "success",
  "latency_ms": 42,
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-07-25T12:00:00+00:00",
  "mock_mode": false,
  "data": {"count": 3}
}
```

> **`mock_mode` is a security field, not a development convenience.** A mocked or stubbed tool result that is indistinguishable from a real one means the audit record cannot establish whether an action actually occurred. Surface the flag in the envelope, propagate it to the audit stream, and fail closed — refuse to start — if mock mode is enabled in a non-development environment.

#### 6.5.3 Status and escalation semantics

Standardize the mapping so that identical conditions produce identical codes across every component.

| Condition | Status | Escalate | Agent retry |
|---|---|---|---|
| Malformed body / JSON parse failure | 400 | No | Retry with corrected structure |
| Schema validation failure | 422 | No | Retry with corrected parameters |
| Missing or invalid credential | 401 | Yes | No retry |
| Authenticated but not authorized | 403 | Yes | No retry — this is a policy decision, not a transient fault |
| Unexpected content type | 415 | No | Retry with correct type |
| Body exceeds size limit | 413 | No | Retry with reduced payload |
| Rate or quota exceeded | 429 | No | Retry after the indicated interval |
| Dependency timeout | 504 | Contextual | Bounded retry for idempotent reads only |
| Unhandled internal failure | 500 | Yes | No retry |

Two rules apply throughout: **an authorization denial is never retryable** — an agent that retries a 403 is probing, and repeated denials are a monitoring signal (§9.3) — and **a 500 returns a generic message to the caller with detail confined to the operational log**.

#### 6.5.4 Redaction and truncation

- Redact known-sensitive keys before any value is logged, returned, or entered into model context: `authorization`, `token`, `api_key`, `secret`, `password`, `cookie`, `set-cookie`, `private_key`, `session`, plus organization-specific names.
- **Prefer allowlisting the fields that may be logged over denylisting the fields that may not.** A denylist fails silently against a field name nobody anticipated; an allowlist fails safe.
- Run pattern-based secret detection in addition to key-name matching, because credentials appear in free-text error strings from dependencies.
- Truncate oversized string fields in logs with an explicit truncation marker, so that a large error payload cannot be used to flood or evade log storage.
- Apply redaction *before* the error enters model context, not only before it reaches durable storage.

#### 6.5.5 Fail-closed defaults

- A timeout, exception, or unavailability anywhere in the authorization path produces a **denial**, never a permit. An unreachable policy service is a stop condition.
- An unhandled exception in a tool handler produces a denial and a latched outcome for privileged operations; it does not fall through to a default execution path.
- Missing configuration must not mean "no control." A service with no authentication configured must refuse to start in any non-development environment rather than start unauthenticated (§6.7).
- Approval timeouts default to denial (§8).

#### 6.5.6 Error paths are tested surface

Error paths are the least-exercised code in most deployments and the most likely to leak. They belong in the regression suite (§11), not in manual review.

### 6.6 Product and Tool References

Named products and open-source projects (scanners, sandboxes, guardrail libraries, etc.) are candidates for evaluation, **not** endorsed controls. Before adoption, verify that each project exists, is actively maintained, has an identifiable owner, supports the claimed capability, and meets supply-chain requirements. Record its validated version, repository, license, security posture, and review date.

### 6.7 Service Surface Hardening Baseline

This baseline applies to **every** network-reachable component in the deployment: the agent application, each tool or MCP server, the enforcement broker, the registry, and any fallback, minimal, debug, or health-check server. Parity gaps between components are the dominant finding in practice — a mature primary service alongside a fallback server that was never brought to the same standard.

**One implementation, not one per service.** Authentication, rate limiting, correlation-ID handling, error envelope construction, and response headers must come from a single shared library used by every component. Independently reimplemented security logic drifts, and the drift is invisible until an incident. Consolidating these into a shared module reduces maintenance cost and, more importantly, removes an entire class of inconsistency bugs.

**Authentication policy contract.** Declare one contract and implement it identically everywhere:

| Configuration | Behavior |
|---|---|
| No API key, no JWT | Permitted in local development only; **must fail to start** in any non-development environment |
| API key configured | Key authentication required |
| JWT configured | Bearer required; validate signature, issuer, audience, and expiry via JWKS |
| Both configured | Either accepted |
| RBAC scopes configured | Scope or role check applied after authentication |

The property that matters: absence of configuration must never silently mean "authentication not required." Fail closed at startup.

**Request limits.** Enforce a maximum body size, validate `Content-Type` and `Content-Length`, and apply sliding-window rate limiting. Rate-limit on authenticated identity where available and fall back to client address only when it is not; per-IP limiting alone is trivially defeated by a distributed caller and unfairly aggregates callers behind a shared egress.

**Correlation identifiers.** Accept a caller-supplied identifier only if it validates as a UUID; otherwise generate one. Echo it in the response header, include it in every log line and audit record, and propagate it across service hops. A caller-supplied correlation ID is a tracing convenience and carries **no** trust or authorization meaning.

**Response header baseline** — applied to every response, on every method, on success and error paths alike:

```text
X-Content-Type-Options: nosniff
Cache-Control: no-store
Pragma: no-cache
Referrer-Policy: no-referrer
X-Frame-Options: DENY
Content-Security-Policy: default-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'none'; object-src 'none'
```

For pure JSON APIs these are inexpensive defense in depth; they become load-bearing wherever any response can be rendered by a browser, which includes debug endpoints and error pages that were never intended to be rendered.

**Configuration naming.** Standardize the *semantics* of security configuration across components even where prefixes differ per service, and converge on a single canonical prefix over time with backward-compatible aliases during migration. Divergent naming is how production gets misconfigured: an operator who correctly sets a limit on one service reasonably assumes the same variable name applies to the next.

---

## 7. Input, Context, Memory, and Output Security

### 7.1 Untrusted-Content Handling

Label user input, retrieved documents, web content, tool output, files, memory, and inter-agent messages with provenance and trust metadata. Preserve those labels through selection, summarization, compression, and storage — aggressive compression must not strip security-critical markers.

Content scanning and prompt-injection classifiers are **signals, not authorization mechanisms**. They may block, warn, reduce privileges, or trigger review, but sensitive actions still require deterministic policy enforcement.

Known context-engineering threats to model against: context poisoning (malicious data injected via memory/tool output), context distraction (irrelevant content to derail focus), context confusion (contradictory content), and compression-induced loss of security markers.

Error payloads, tool metadata, and protocol fields are untrusted content on the same terms as document text, and are more frequently exempted from scanning by accident (§6.4, §6.5.2).

### 7.2 Structured Actions and Schema Enforcement

Models must propose **typed actions**, not executable command strings. Schemas must reject unknown fields, enforce lengths and formats, canonicalize values, and validate cross-field constraints. The enforcement layer must **separately** verify authorization and business rules — a valid schema is not an authorization decision.

```python
# Schema validation is necessary but NOT sufficient — authorization is separate.
class SendEmailAction(BaseModel):
    to: EmailStr
    subject: constr(max_length=200)
    body: constr(max_length=50000)
# After schema validation, the broker still checks:
#   - is this agent authorized to email this recipient?
#   - does policy require human approval for external recipients?
```

### 7.3 Safe Database Access

Agents must not execute unrestricted model-generated SQL against production databases.

- Prefer predefined, parameterized operations exposed through a query broker.
- Map each operation to approved tables, columns, filters, and result limits.
- Use read-only identities for read operations; apply row- and column-level controls where available.
- Enforce time, concurrency, cost, and result-size limits; disable multi-statement execution and dangerous functions.

> **Regular expressions and blocked-keyword lists are not a SQL security boundary.** A `^SELECT` prefix check is bypassable via CTEs, subqueries, comments, stacked statements, and expensive read-only queries. If free-form SQL is unavoidable, use a database-aware parser and an isolated read replica.

### 7.4 Session and Memory Security

Isolate state by tenant, user, agent, and task ("one task, one session"). Temporary data must use encrypted ephemeral storage with explicit retention and cleanup rules.

> **On erasure:** ordinary object deletion, garbage collection, `del`, and file removal are **not** cryptographic zeroing. When cryptographic erasure is required, encrypt session data under a per-session key and destroy that key.

Every memory entry must record its authenticated writer, source, tenant, task, creation time, confidence, classification, and expiration (TTL). Separate observed facts, user assertions, model inferences, and executable instructions. Support quarantine, versioning, correction, rollback, and deletion.

> **On signed memory:** a signature proves content has not changed since signing; it does **not** prove the content was true or safe when accepted. Validate provenance and trust at write time, not just integrity at read time.

**Tenant isolation must be infrastructure-enforced, not conventional.** Namespaces separating tenant, user, agent, and workflow must make cross-tenant retrieval technically impossible rather than merely unrequested — one tenant's namespace should be *incapable* of answering another tenant's query, not simply disinclined to. Avoid shared singleton state, shared context buffers, and shared vector stores unless isolation is explicitly enforced and tested.

**Revalidate ownership on every request.** Never trust a cached or implicit session-to-user mapping. Cross-tenant leakage in agent systems is frequently timing- and concurrency-dependent, which means it surfaces under load and not in functional testing (§11).

**Classify at ingestion, not at retrieval.** Tag sensitive data as it enters context and apply policy at that boundary — block, mask, or minimize — so restricted content is never stored in a form a later retrieval can surface. Give every context item a TTL and enforce automatic purge. Persistent context is a governed data store: it inherits the access-control, classification, retention, and deletion obligations of any other store holding the same data, and injected content in shared memory must not be able to become instructions in a later session.

### 7.5 Output and Data-Loss Controls

Before release or tool reuse, validate outputs for schema, destination, data classification, secrets, personal data, policy violations, and size. Apply destination-aware data-loss prevention. High-impact disclosures require bound approval and a final authorization check.

### 7.6 Safe Command and Process Execution

**Requirement:** Untrusted values must never be concatenated or interpolated into a command string. Untrusted here includes user input, model output, retrieved content, tool responses, file contents, filenames, URLs, and protocol or metadata fields.

- Use structured process APIs — `execFile`, `spawn`, `subprocess.run` with an argument list — passing the executable and each argument separately. Never `shell=True` / `shell: true`, and never `exec` or `eval` on content that can be influenced from outside.
- Terminate option parsing with `--` where the invoked command supports it, so that a value beginning with `-` is treated as data rather than a flag.
- Rejecting or escaping shell metacharacters (`;`, `|`, `` ` ``, `$()`, `&`, newline, redirection) is **defense in depth only**. The argument-array boundary is the control; metacharacter filtering is the backstop that catches the case where someone reintroduces a shell.
- Allowlist permitted executables, verbs, and operations. Deny by default, so that a destructive or unanticipated operation cannot run even if it reaches the handler.
- **Model-generated code is never executed automatically.** It passes an explicit validation and authorization gate (§5.2) and executes only inside the sandbox of §5.1.
- File paths: canonicalize and resolve symlinks first, then verify the result lies within an approved root. Reject traversal after normalization, not before — a pre-normalization check on `../` is bypassable by encoding, symlinks, and mixed separators.
- Apply the same rules to protocol and metadata fields — OAuth authorization URLs, redirect URIs, header values, filenames from remote responses. These are routinely overlooked because they do not look like "user input," and they are attacker-reachable.

§7.3 is the database-specific case of this same principle: parameterize the operation, never build the statement from untrusted text.

### 7.7 Model Invocation Guardrails

Validate model invocation parameters at the configuration boundary, not only at call time, and fail closed on any value outside policy.

- **Model identifier** against an explicit allowlist. Deny unknown or unapproved models, including newly released ones that appear in a provider's catalog without review.
- **Region or endpoint** against an expected format and an approved list.
- **Temperature and sampling parameters** within defined ranges.
- **Maximum output tokens** against a hard ceiling.
- **Per-task and per-tenant token and cost budgets**, enforced rather than alerted on.

Model selection is a data-governance control, not only a cost control: an unapproved model may sit in a different jurisdiction, carry different retention terms, or fall outside an executed agreement. Record the resolved model identifier and version in the audit record for every invocation (§9.1).

### 7.8 Intent Integrity and the Capability Trifecta

Sections 3.2 and 7.1 stop untrusted content from becoming *policy*. This section addresses a distinct failure mode: untrusted content that leaves policy intact while redirecting the *objective*. Every action passes authorization, every schema validates, every credential is correctly scoped — and the agent is pursuing a goal no one gave it.

**Intent anchoring**

- Anchor the user's original objective in trusted system instructions as a fixed, authoritative reference, structurally separate from retrieved content. Do not merge instructions and context into an undifferentiated prompt.
- Tag or fence retrieved pages, documents, issues, files, code comments, and tool output explicitly as untrusted context before it reaches the planning step.
- **Gate the transition from planning to consequence.** An agent must not move directly from reading context to deleting, sending, modifying, or exporting. Insert an intent-alignment check between the two that evaluates whether the proposed action still serves the objective the user actually stated.
- Re-anchor the original intent during long-running sessions and multi-step workflows. Drift accumulates across steps that are each individually reasonable.
- Run intent-drift detection in an **independent guardrail outside the primary agent's context**. A check that lives inside the context an attacker has already influenced is not a check. This is the §9.5 principle applied to reasoning rather than to records: the evaluator must be something the compromised party cannot author.

**The capability trifecta**

An agent is exploitable when it simultaneously holds:

1. access to private data,
2. ingestion of untrusted content, and
3. the ability to communicate externally or take consequential action.

Any two are containable. All three compose into exfiltration regardless of how well each leg is individually secured, because the attacker supplies the content, the agent supplies the data, and the outbound channel supplies the delivery. Individually correct controls do not sum to a safe combination — this is the concrete case of governing principle 6.

Maintain a **trifecta scorecard** for every production agent and workflow: answer the three questions explicitly, date the answer, and re-answer it when tools or scopes change. Prioritize any agent scoring yes three times, and remove or constrain at least one leg — narrow the data scope, eliminate the untrusted ingestion path, or restrict outbound destinations to an allowlist. Where all three legs are genuinely required, that is a documented residual-risk decision with a named owner (§2.3), not a default.

---

## 8. Human Oversight

High-impact, irreversible, externally visible, or security-sensitive actions require explicit human approval unless a documented assessment permits bounded automation. Typical always-approve actions: permanent deletion, fund transfers, permission changes, PII access/disclosure, and production modification.

Approval requests must show: the exact action, target, and material parameters; data accessed or disclosed; expected impact and reversibility; agent and initiating principal; the policy decision and relevant warnings; an expiration time; and the recovery/rollback mechanism.

**Bind approval cryptographically to a canonical representation of the proposed action.** Any change to the action, target, parameters, data classification, agent identity, or policy version invalidates the approval. **Reauthorize immediately before execution** to prevent time-of-check/time-of-use (TOCTOU) failures.

Critical actions may require multiple independent approvers and phishing-resistant authentication. Approval timeouts and system failures default to **denial**. All approval decisions are recorded in the audit trail with approver identity, decision, timestamp, and signature.

Freezes, approval requirements, and other critical constraints are **authorization boundaries, not prompt instructions**. A rule that exists only in a system prompt is a suggestion to the model, not a control.

---

## 9. Audit, Monitoring, and Incident Response

### 9.1 Audit Records

Security-relevant events must record: authenticated agent and initiating principal; tenant, session, task, and trace identifiers; requested action and target; policy decision and policy version; human-approval evidence; tool identity, version, and execution boundary; input/output references, classifications, or approved hashes; outcome, timing, and error category; and sequence/integrity metadata. Include the signing key identifier (`key_id`) in every record so that records remain attributable across key rotations.

Records must also carry the correlation identifier (§6.5.2), the resolved model identifier and version (§7.7), and the `mock_mode` state of any executed tool, so that an audit record establishes whether an action actually took effect.

Capture enough to reconstruct intent changes and external communication — prompt and context deltas, and network events, not only tool calls — while applying minimization and masking. **The log store must not become a new sensitive-data store.** Mask credentials and personal data in payloads at write time; an audit pipeline that faithfully records everything an agent saw has recreated the exposure it was built to detect, inside a system with broader read access. Set retention long enough to support forensic reconstruction and applicable compliance obligations, and record the retention decision and its owner.

> **The initiating principal must be an authenticated claim, not a self-asserted string.** Derive the principal recorded in audit events from a validated identity assertion — an OIDC ID token verified against the identity provider's published keys (issuer, audience, expiry) or an mTLS-bound session assertion — and record the issuer, subject, assertion hash, and binding type. A session whose principal is not assertion-backed must be labeled as self-asserted; its records identify a session, not an accountable human. Retain the assertion hash so records can be correlated against identity-provider logs that the agent runtime cannot author. Note that a binding-type field written by the agent runtime is itself forgeable under compromise; its assurance value depends on that external correlation (§9.5).

Use canonical serialization, immutable verification copies, and independently controlled append-only or WORM storage, with externally anchored checkpoints.

> **Hash chaining alone does not prevent deletion, rollback, truncation, or wholesale replacement of a local chain.** It makes tampering *evident* only when combined with independent, access-controlled storage (e.g., S3 Object Lock, GCS retention/Bucket Lock) and external anchoring. The agent must have no access to log storage or modification infrastructure. Even then, records authored during a live compromise are cryptographically valid: single-stream integrity has a ceiling of tamper-evidence for pre-compromise history. Reconstructing events within a compromise window requires cross-stream corroboration (§9.5).

### 9.2 Traceable Decisions

Record concise, purpose-generated decision evidence: objective and relevant source provenance; proposed action; tools selected; material assumptions and uncertainty; policy checks and results; human approvals; and final action/outcome.

Do **not** require or retain raw private chain-of-thought tokens as a compliance artifact; generate structured decision summaries instead. Apply minimization, redaction, encryption, access control, and retention limits to decision records.

### 9.3 Behavioral Monitoring

Monitor for: new tools, destinations, or data types; rate, volume, cost, or latency anomalies; repeated denied requests; delegation depth or fan-out anomalies; read-then-exfiltrate action sequences (compositional risk); credential or policy failures; missing expected telemetry (evasion signature); cross-tenant access indicators; divergence between independently authored record streams (§9.5); and model, prompt, tool, or dependency changes.

Error-shape and tool-definition signals:

- Any change to the advertised tool list or to a tool definition, including a tool that modifies its own definition (§6.4).
- Shifts in error-category distribution per agent identity. A rise in 401/403 outcomes indicates probing, credential expiry, or misconfiguration, and an agent retrying a denial is behaving anomalously by definition (§6.5.3).
- Responses or log lines missing an expected correlation identifier, which indicates either a component outside the shared implementation or a path that bypasses the broker.
- Outbound connections from tool or plugin workloads to unapproved domains (§3.3).
- The same token presented under two agent identities, or from two network locations (§4.3).
- Agent connections to endpoints absent from the approved registry (§4.6).
- Divergence between an agent's stated objective and its action sequence, evaluated outside the agent's own context (§7.8).

Agent and tool telemetry must reach the SIEM or XDR the security team actually monitors. Telemetry that lives only in an application-specific dashboard is not integrated with detection and will not be consulted during an incident. Establish behavioral baselines for normal agent and tool activity before relying on anomaly alerting.

Statistical and model-based detectors (baselines, z-scores, distribution-shift tests, goal-alignment scoring) produce **signals**; they do not replace policy enforcement. Calibrate thresholds against representative benign and malicious sessions, and track false-positive rate and correct-escalation rate as first-class metrics.

### 9.4 Incident Response

Procedures must support containment, credential revocation, evidence preservation, tenant notification, rollback, dependency quarantine, regulatory assessment, root-cause analysis, and controlled restoration. Rebuild compromised workloads from known-good images and rotate potentially exposed credentials from a clean environment. Exercise the plan before production and after material architectural changes.

During forensic reconstruction, treat the agent-authored record stream as one input among several; reconcile it against the independently authored streams in §9.5 before drawing conclusions about events inside the compromise window.

Exercise concrete compromise scenarios rather than reviewing the plan on paper: who is paged, which store holds the relevant logs, who is authorized to activate the kill switch, and how long disabling a specific tool server actually takes. Each of these is a question that cannot be answered quickly under pressure, so it must be answered beforehand.

### 9.5 Compromise-Resistance Model and Remediation Order

Each identity and audit-integrity control closes a narrower failure mode than "safe under compromise." Scope every control's claim precisely, and order retrofit remediation by what each control actually buys:

| Control | Closes | Does not close |
|---|---|---|
| Independent WORM/append-only storage with external anchoring | Retroactive deletion, truncation, or rewriting of pre-compromise history | False records authored during a live compromise |
| KMS/HSM key custody (workload never holds raw key bytes) | Key exfiltration; forgery after credential revocation | Live signing of false records via the workload's legitimate signing access |
| Assertion-backed principal binding (§9.1) | Fabricated human attribution at session start | Compromised-runtime reuse of an already-valid session |
| External identity authority (SPIFFE/SPIRE, PKI) | Identity minting; cross-agent impersonation; post-revocation identity reuse | Abuse of an identity that is already valid |
| Out-of-process PDP/PEP with its own record stream (§5.2) | Single-stream falsification; policy bypass or patching within the agent runtime | — (this control supplies the corroboration baseline) |

**The live-compromise residual.** Because the signer is the compromised party, no key ceremony prevents a live compromised runtime from emitting internally consistent, validly signed false records. The ceiling of any single record stream is tamper-evidence for history. Answering "what happened during the compromise window" — as opposed to "what one stream claims happened" — requires at least two independently authored streams that the agent runtime cannot write: the enforcement broker's decision records, KMS/cloud-provider signing and access logs (e.g., CloudTrail), tool- and destination-side logs, and network telemetry. Reconcile these against the agent's stream; divergence or absence between streams is itself a detection signal (§9.3).

**Recommended remediation order for retrofits**, ranked by forensic value:

1. Independent append-only storage with external anchoring — the only control in the set that preserves history.
2. KMS/HSM key custody — revocability and a hard post-revocation forgery boundary.
3. Assertion-backed principal binding — accountable human attribution.
4. External identity authority — anti-minting and impersonation resistance.
5. Out-of-process enforcement with an independently authored stream — cross-stream corroboration; last in sequence, not least in value.

---

## 10. Supply-Chain Security

Maintain an AI and software bill of materials (AIBOM/SBOM) covering: models and versions; runtime and agent frameworks; system and developer prompts; MCP servers and tools; source repositories and build provenance; packages, containers, and infrastructure modules; datasets, embeddings, and knowledge sources; licenses, owners, and support status; artifact hashes and signatures; and vulnerabilities, exceptions, and review dates.

Verify provenance and signatures where available, pin production artifacts by digest, scan dependencies continuously, protect the build pipeline, and require review for material changes.

Operational requirements that follow from this:

- Pin every dependency, plugin, and tool server to a reviewed, known-good version. No `latest`, no floating ranges. Upgrade deliberately after review.
- Do not fetch dependencies dynamically at runtime; build and deploy a fixed, vetted set.
- Run dependency scanning in CI before release, not only on a periodic schedule.
- Sandbox third-party plugins (§5.1) so that one compromised component does not compromise the environment.
- Monitor and restrict plugin network access, particularly calls to unknown domains.
- Include developer workstations in the inventory (§6.2).

> Signatures establish origin and integrity, **not** safety; runtime isolation and authorization remain mandatory. When recording AIBOM entries, verify each component's real package ecosystem, version scheme, and identifiers rather than assuming them.

A supply-chain risk score (known CVEs, maintainer activity, version age, open security issues) can gate adoption into `approve`, `review-required`, or `reject`, with heavier runtime controls for higher scores.

---

## 11. Threat Modeling and Security Validation

Threat modeling must cover model behavior, orchestration, memory, tools, data, identity, infrastructure, supply chain, users, and cross-agent interactions. Map trust boundaries and multi-step abuse paths rather than evaluating individual prompts or tool calls in isolation.

A structured, layered approach such as CSA's **MAESTRO** framework can organize this analysis. MAESTRO defines a **seven-layer** reference architecture — foundation models; data operations; agent frameworks; deployment and infrastructure; evaluation and observability; security and compliance (vertical); and the agent ecosystem — and maps threats and mitigations across those layers. Use the framework's actual layer structure; do not substitute an invented layer count.

Validation must include: direct and indirect prompt injection; goal hijacking and policy conflict; tool substitution and manifest changes; confused-deputy and token misuse; cross-session and cross-tenant leakage; memory poisoning and provenance failures; privilege amplification through safe-looking action sequences; approval tampering and stale approvals; sandbox escape and resource exhaustion; data exfiltration and covert destinations; logging interruption and evidence manipulation; forgery and false-record authorship during simulated runtime compromise (validating the §9.5 corroboration path); and recovery/emergency-stop operation.

**Error-path and surface regression tests.** These are the least-exercised paths in most deployments and belong in an automated suite gated in CI on every change, run against every network-reachable component including fallback and debug servers:

- Unauthenticated request is denied.
- Authenticated-but-unauthorized request is denied with a distinct code, and is not retryable.
- Unexpected content type is rejected.
- Oversized body is rejected before parsing.
- Malformed JSON is rejected with the canonical envelope.
- Rate limit engages and returns the correct status.
- Correlation identifier is returned, and an invalid caller-supplied identifier is replaced rather than trusted.
- Response header baseline is present on success **and** error paths.
- Secret redaction is verified against a seeded credential in a dependency error string, not only against a key name.
- Fail-closed behavior is verified under policy-service unavailability and dependency timeout.
- Startup fails when authentication is unconfigured in a non-development environment, and when mock mode is enabled outside development.
- Tenant isolation holds: a test that writes as tenant A and asserts tenant B can retrieve none of it, run on every build and repeated under concurrency.
- Telemetry cannot be disabled, truncated, or redirected without an alert or independent approval.
- An intent-misaligned action proposed after untrusted-content injection is blocked by the alignment gate (§7.8).

**Injection and traversal tests** against every tool that touches a process, a filesystem, or a query: shell metacharacters, argument-injection via leading `-`, encoded and symlinked path traversal, and untrusted values arriving through metadata and protocol fields rather than through the primary parameter.

**Verification probes.** These are one-off exercises that produce evidence rather than pass/fail assertions. Each is diagnostic because the result is informative either way:

- Produce the last 30 days of tool-call logs. Any gap in availability, identity, tool, parameter metadata, or timestamp is a finding.
- Search all organization repositories for MCP and agent configuration files, then reconcile against network scans and the registry (§4.6).
- Search the codebase for `child_process.exec`, `os.system`, `subprocess` with `shell=True`, and `shell: true`. Treat every occurrence as a finding, trace its data flow, and replace it with a structured argument-array call (§7.6).
- Enumerate each agent's *effective* production permissions rather than its documented permissions, and compare the two (§4.2).
- Trace every downstream call an agent makes and flag any that forwards the caller's token unchanged (§6.3).
- Determine, for each production agent, whether its tool servers are vendor-built or drawn from an unreviewed repository (§10).

Measure detection rate, false-positive rate, unsafe-action completion rate, correct-escalation rate, containment time, and recovery time. **Test complete sessions and realistic action sequences, not only isolated prompts.** Red teaming should be conducted by an independent team whose test design the agent cannot influence.

---

## 12. Production Security Gate

An agent must not enter or remain in production with an unresolved **critical** vulnerability or an unaccepted **high-severity** vulnerability affecting authorization, isolation, data protection, or privileged tool use.

Production approval requires: documented remediation and regression testing; representative adversarial testing; verified policy enforcement and emergency controls; operational monitoring and response readiness; privacy and legal review where applicable; documented residual risk and a named risk owner; time-bounded, approved exceptions where policy permits; and security and business-owner sign-off.

Example blocking decision:

> **Production decision: NOT APPROVED.** A high-severity privilege-escalation path (tool chaining: query database → reuse output in a delete operation) remains unresolved. Deployment is blocked until remediation is complete and independent retesting confirms the execution chain is prevented.

---

## 13. Phased Deployment

Advancement is **evidence-based, not time-based**. A fixed number of operating hours or a zero-incident period does not, by itself, demonstrate security.

**Phase 1 — Isolated Evaluation:** synthetic or approved test data; no production credentials; read-only or simulated tools; security-team access; threat-model and abuse-case validation.

**Phase 2 — Internal Staging:** production-like isolation and monitoring; narrow internal use cases; limited, reversible writes; human approval for consequential actions; incident-response exercise completed.

**Phase 3 — Limited Production:** defined user cohort and use cases; bounded tools, destinations, and data; enhanced monitoring and daily review; tested rollback and kill switch; explicit go/no-go criteria.

**Phase 4 — General Production:** scale only within validated boundaries; continuous control evidence and testing; periodic access and tool recertification; change-triggered reassessment; documented decommissioning and data-deletion plan.

Each transition requires security, operations, product, and compliance sign-off plus a tested rollback plan.

---

## 14. Control Coverage and Evidence

Do **not** report unsupported percentage-complete claims (e.g., "100% coverage"). Track implementation and effectiveness through evidence.

| Risk domain | Control | Status | Evidence | Test result | Residual risk | Owner | Review date |
|---|---|---|---|---|---|---|---|
| Runtime execution | External policy enforcement | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Prompt injection | Untrusted-content isolation | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Privilege escalation | Tool authorization | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Accountability | Append-only audit storage | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Accountability | Assertion-backed principal attribution | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Compromise resistance | Cross-stream corroboration | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Credential protection | Secrets excluded from model context | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Error handling | Redacted, fail-closed error paths | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Service surface | Uniform authentication and request limits | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Execution safety | No untrusted values in command strings | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Model governance | Model, region, and budget allowlists | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Intent integrity | Alignment gate and independent drift detection | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Capability exposure | Current trifecta scorecard per agent | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Tenant isolation | Infrastructure-enforced namespaces, CI-tested | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Inventory | Mandatory registration and shadow discovery | Not assessed | Pending | Not tested | Unknown | TBD | TBD |

Allowed statuses: `Not assessed`, `Planned`, `Partial`, `Implemented`, `Verified`, `Exception approved`. If coverage percentages are used, document the scoring method, evidence standard, treatment of partial controls, and independent-validation process.

---

## 15. Regulatory and Standards Alignment

This framework supports preparation for applicable AI, cybersecurity, privacy, and sector-specific obligations. Applicability depends on intended purpose, deployment context, affected persons, jurisdiction, and the organization's legal role.

For every mapping: cite the exact regulation/standard edition, article or control, and publication date; identify the regulated actor and system category; separate legal requirements from organizational policy and optional guidance; record the implementation artifact and evidence owner; obtain qualified legal review for legal interpretations; and obtain auditor review before representing ISO/IEC 27001 or SOC 2 alignment.

> **Alignment does not establish conformity, certification, or audit readiness.** Map controls to frameworks (e.g., NIST AI RMF's GOVERN / MAP / MEASURE / MANAGE functions; relevant EU AI Act articles on risk management, record-keeping, transparency, human oversight, and cybersecurity) as a planning aid, and record which article/control each control *supports* — not which it "satisfies." Do not describe SOC 2 criteria labels as certifications, and always identify the ISO/IEC 27001 edition in use.

The following illustrates the intended mapping *shape* — each row supports, and provides evidence toward, the cited requirement:

| Framework control | NIST AI RMF function | EU AI Act area (verify article/edition) | Objective |
|---|---|---|---|
| Identity & authorization (§4) | GOVERN / MANAGE | Cybersecurity & access control | Cryptographically verified identity; ephemeral, task-scoped access |
| Isolation & containment (§5) | MAP / MANAGE | Robustness & resilience | Limit blast radius; interruptible execution |
| Error design & data protection (§6.5) | MANAGE / MEASURE | Cybersecurity; accuracy & robustness | Fail-closed failure handling without disclosure or injection |
| Human oversight (§8) | MANAGE | Human oversight / stop mechanism | Bound approval and emergency stop for high-impact actions |
| Audit & monitoring (§9) | GOVERN / MEASURE | Record-keeping & traceability | Tamper-evident, independently stored decision records |

---

## 16. Implementation Priorities

**Priority 1 — Required Before Production**

- Workload identity and external (deterministic) policy enforcement
- Least privilege and task-scoped, proof-of-possession-bound authorization
- Secrets in a vault, resolved at execution time, and never present in model context (§4.5)
- Execution sandboxing and network containment
- Structured tool schemas, safe database access, and no untrusted values in command strings (§7.6)
- Canonical, redacted, fail-closed error design across every component (§6.5)
- Authentication, request limits, and the header baseline on every network-reachable surface, including fallback and debug servers (§6.7)
- Bound human approval for high-impact actions (with TOCTOU-safe re-check)
- Append-only/WORM audit storage and tested incident response
- Assertion-backed principal attribution in audit records (§9.1)
- Emergency stops, credential revocation, and a per-server kill switch (§5.3, §4.6)
- Mandatory server registration enforced as a deployment gate (§4.6)
- Infrastructure-enforced tenant and session isolation, verified by test (§7.4, §11)
- A current trifecta scorecard for every production agent, with at least one leg constrained wherever feasible (§7.8)
- Threat modeling and a passed production security gate

**Priority 2 — Required Before Scaling**

- Central authenticated service and tool registry
- Automated artifact and dependency verification
- Shared security utility library replacing per-component reimplementation (§6.7)
- CI-gated security regression suite covering error paths, injection, and fail-closed behavior (§11)
- Full-schema metadata scanning and tool-definition change alerting (§6.4)
- Model, region, parameter, and budget guardrails (§7.7)
- Cross-step behavioral monitoring
- Cross-stream corroboration coverage for compromise-window reconstruction (§9.5)
- Memory provenance and lifecycle controls
- Multi-agent delegation governance
- Intent anchoring and an independent alignment gate ahead of consequential actions (§7.8)
- Continuous shadow-server discovery and unregistered-endpoint alerting (§4.6)
- Telemetry integrated with the SIEM/XDR, with behavioral baselines established (§9.3)
- Secure-by-default deployment templates that make the compliant path the fast path (§4.6)
- Cumulative permission review and recertification (§4.2)
- Complete control evidence and recertification

**Priority 3 — Continuous Improvement**

- Expanded adversarial and scenario testing
- Detection calibration and response automation
- Stronger workload isolation for higher-risk use cases
- Configuration naming convergence to a single canonical scheme (§6.7)
- Independent assurance and regulatory-mapping updates
- Environmental and operational efficiency monitoring

---

## 17. Glossary

- **Agent:** An AI-enabled system that can plan or select actions and invoke tools.
- **AIBOM:** Inventory of models, data, prompts, tools, and supporting software used by an AI system.
- **Capability trifecta:** The combination of private-data access, untrusted-content ingestion, and external communication in one agent; exploitable as a combination even where each leg is individually well controlled (§7.8).
- **Confused deputy:** A component with legitimate authority induced to exercise it on behalf of a caller that lacks that authority; in agentic systems, most often via token passthrough (§6.3).
- **Correlation identifier:** A validated, generated-if-absent identifier propagated across services that joins the model-facing error, client response, operational log, and audit record for a single event.
- **Cross-stream corroboration:** Reconciling the agent-authored audit stream against independently authored record streams (broker decisions, KMS/CSP logs, tool-side logs, network telemetry) that the agent runtime cannot write; the required control for reconstructing events during a live compromise.
- **DID:** Decentralized Identifier (W3C); requires a method defining issuance, resolution, rotation, and revocation.
- **DPoP:** Demonstrating Proof of Possession; binds token use to a key.
- **Error envelope:** The single canonical error shape emitted by every component (§6.5.2), comprising a stable code, generic message, enumerated remediation hint, escalation flag, correlation identifier, and timestamp.
- **Fail closed:** The property that a timeout, exception, or missing configuration produces a denial or a refusal to start, never a permit or an unauthenticated service.
- **HITL:** Human-in-the-loop review or approval for defined actions.
- **Intent anchoring:** Holding the user's original objective in trusted, structurally separate system instructions and validating that proposed actions still serve it (§7.8).
- **JIT:** Just-in-time issuance of short-lived, scoped credentials.
- **MAESTRO:** CSA's seven-layer agentic AI threat-modeling framework.
- **MCP:** Model Context Protocol; exposes context and tools to compatible clients.
- **mTLS:** Mutual TLS; bidirectional certificate authentication.
- **Policy decision point (PDP):** Evaluates an action against authorization policy.
- **Policy enforcement point (PEP):** Permits, denies, or constrains execution based on a policy decision.
- **Proof of possession:** Cryptographic evidence that a caller holds the key to which a credential is bound.
- **Resource indicator:** An OAuth parameter (RFC 8707) binding a token to its intended target server, preventing replay against a different resource.
- **Self-verifying identity (anti-pattern):** An identity or audit scheme in which issuance, signing, and verification reside in the same process or trust domain, or a symmetric scheme in which verifiers can forge; provides cryptographic structure without a trust boundary.
- **Shadow server:** A tool or MCP server running outside the registry — unregistered, unowned, unmonitored, and absent from the inventory consulted during an incident (§4.6).
- **SPIFFE/SPIRE:** Standard and runtime for securely identifying software workloads.
- **TOCTOU:** Time-of-check/time-of-use; the gap between approval and execution that must be closed by re-authorization.
- **TOFU:** Trust on first use; pinning an artifact or schema at first observation and alerting on later change. A fallback where signature verification is unavailable, not an equivalent to it.
- **WORM:** Write once, read many storage used to resist alteration or deletion.

---

## 18. Reference Categories

Maintain dated references to the authoritative versions the organization actually uses, recording title, publisher, version/date, URL, owner, and last validation date:

- Applicable EU AI Act text and implementation guidance
- NIST AI Risk Management Framework and relevant profiles
- NIST cybersecurity and identity guidance
- ISO/IEC 27001 and related AI management standards (identify edition)
- AICPA Trust Services Criteria
- W3C DID specifications (when DIDs are used)
- SPIFFE/SPIRE documentation (when workload identity is used)
- OAuth 2.0 / 2.1, token exchange (RFC 8693), resource indicators (RFC 8707), mTLS token binding, and DPoP specifications
- OpenID Connect Core (ID token validation, for principal assertion binding)
- CSA MAESTRO and MITRE ATLAS threat-modeling sources
- OWASP Top 10 for LLM Applications and OWASP agentic security guidance
- Current MCP protocol and authorization specifications
- CIS Kubernetes Benchmark (for container/orchestration hardening)
- Provider model-service documentation for model identifiers, regions, and retention terms

---

## Conclusion

Secure agentic AI requires independently enforced controls **around** the model. Cryptographic workload identity, short-lived proof-of-possession authority, isolated execution, provenance-aware context, brokered and schema-constrained tools, cryptographically bound approvals, tamper-resistant and independently stored evidence, and tested containment together form the production security boundary. Model instructions, delimiters, and classifiers add valuable defense in depth, but they do not replace deterministic authorization or operational accountability. Failure paths carry the same weight as success paths: an error that leaks a credential, injects attacker text into context, or degrades a denial into a permit defeats every control upstream of it. Capability combinations carry weight independently of the capabilities themselves: an agent whose every individual permission is correct can still be exploitable because of what those permissions compose into. Scope every integrity control to the failure mode it actually closes, and assume that reconstructing a compromise requires corroboration across streams the agent cannot author. Treat every code snippet here as a sketch to validate, every framework mapping as a planning aid to verify, and autonomy as something earned through evidence rather than asserted.
