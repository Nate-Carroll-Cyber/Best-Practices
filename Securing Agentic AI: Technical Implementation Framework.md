# Securing Agentic AI: Technical Implementation Framework

**Version:** Consolidated 2.1
**Status:** Implementation guidance. Regulatory mappings are planning aids, not legal advice, certification, or evidence of conformity.
**Changes from 2.0:** self-verifying-signature anti-pattern and key-custody closure properties (§4.1); assertion-backed principal attribution (§9.1); compromise-window ceiling on single-stream integrity (§9.1); new compromise-resistance model, threat-closure table, and retrofit remediation order (§9.5); Priority 1 addition (§16); glossary addition (§17).

## Executive Summary

Agentic AI systems can plan, invoke tools, modify external systems, maintain state, and delegate work. Their security therefore depends on more than model-level safeguards. Production deployments require independently enforced identity, authorization, isolation, data protection, auditability, human oversight, and incident response.

This framework is built on five principles that govern every control below:

1. **The model proposes; deterministic controls decide.** A model must never grant itself authority or act as the final policy-enforcement point. Message roles, delimiters, and prompt separation improve model interpretation but are not access controls.
2. **Every action has an authenticated principal.** Human, service, agent, and delegated identities remain traceable throughout execution.
3. **Authority is narrow, temporary, and context-bound.** Permissions are task-scoped, audience-restricted, and resistant to replay.
4. **Untrusted content remains untrusted.** User input, retrieved content, tool output, memory, and inter-agent messages cannot become policy merely by entering the context window.
5. **High-impact actions remain interruptible and accountable.** Human approval, circuit breakers, and independently controlled audit records constrain autonomy.

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

### 4.4 Agent and Service Discovery

Discovery metadata must come from an authenticated registry and include service identity, approved endpoint, protocol, owner, current version, and health status. Registry responses must be integrity-protected (signed), access-controlled, logged, and subject to expiration. Require DNSSEC or equivalent for registry resolution where applicable. Discovery does **not** replace authentication or authorization at the destination service.

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

### 5.3 Circuit Breakers and Emergency Stops

Independent containment mechanisms must be able to reject actions, revoke credentials, close network paths, terminate tools, quarantine sessions, preserve forensic evidence, and alert responders. This also satisfies the "stop button" expectation for human oversight of high-risk systems.

Example trigger conditions: error-rate spikes, probable infinite loops (excessive loop depth), resource exhaustion, anomalous tool-call volume, repeated unauthorized tool access, and detected prompt-injection attempts.

Automatic recovery (a half-open retry) may be used for low-risk, idempotent operations. Breakers involving privileged, destructive, financial, safety-critical, or externally visible actions **remain latched** until an authorized operator reviews and resets them. Recovery tests must use non-destructive synthetic actions.

---

## 6. MCP and Tool Security

### 6.1 Threat Model

Treat MCP servers and other dynamic tool providers as untrusted supply-chain and execution dependencies. Relevant threats: malicious or changed tool descriptions; tool substitution or rug pulls; excessive or undisclosed permissions; token passthrough and confused-deputy behavior; cross-session or cross-tenant leakage; unsafe error disclosure; tool-output prompt injection; unauthorized network callbacks; dependency compromise; and sandbox escape.

### 6.2 Local Transports (STDIO / Unix Sockets)

Local servers must run as isolated child workloads with pinned artifacts (digest), minimal filesystem access, explicit environment variables, no inherited secrets, constrained network access, and deterministic shutdown. Local transport does **not** imply the tool implementation is trusted.

### 6.3 Remote Transports (Streamable HTTP)

Remote servers require TLS with server authentication, protocol validation, scoped authorization, replay resistance, rate limiting, and centralized policy enforcement.

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

> **Signatures and hashes establish artifact integrity and origin — not benign behavior.** Enforce declared permissions at runtime and audit actual tool behavior against the advertised capability. Require re-review whenever code, description, schema, permissions, owner, dependencies, or endpoint changes.

### 6.5 Error Handling

Return stable, minimal error codes to the model (e.g., `{"error": "DB_TIMEOUT", "request_id": "..."}`). Keep detailed stack traces, paths, query text, credentials, and infrastructure identifiers in access-controlled operational logs only. Apply secret detection and redaction before any error enters model context.

### 6.6 Product and Tool References

Named products and open-source projects (scanners, sandboxes, guardrail libraries, etc.) are candidates for evaluation, **not** endorsed controls. Before adoption, verify that each project exists, is actively maintained, has an identifiable owner, supports the claimed capability, and meets supply-chain requirements. Record its validated version, repository, license, security posture, and review date.

---

## 7. Input, Context, Memory, and Output Security

### 7.1 Untrusted-Content Handling

Label user input, retrieved documents, web content, tool output, files, memory, and inter-agent messages with provenance and trust metadata. Preserve those labels through selection, summarization, compression, and storage — aggressive compression must not strip security-critical markers.

Content scanning and prompt-injection classifiers are **signals, not authorization mechanisms**. They may block, warn, reduce privileges, or trigger review, but sensitive actions still require deterministic policy enforcement.

Known context-engineering threats to model against: context poisoning (malicious data injected via memory/tool output), context distraction (irrelevant content to derail focus), context confusion (contradictory content), and compression-induced loss of security markers.

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

### 7.5 Output and Data-Loss Controls

Before release or tool reuse, validate outputs for schema, destination, data classification, secrets, personal data, policy violations, and size. Apply destination-aware data-loss prevention. High-impact disclosures require bound approval and a final authorization check.

---

## 8. Human Oversight

High-impact, irreversible, externally visible, or security-sensitive actions require explicit human approval unless a documented assessment permits bounded automation. Typical always-approve actions: permanent deletion, fund transfers, permission changes, PII access/disclosure, and production modification.

Approval requests must show: the exact action, target, and material parameters; data accessed or disclosed; expected impact and reversibility; agent and initiating principal; the policy decision and relevant warnings; an expiration time; and the recovery/rollback mechanism.

**Bind approval cryptographically to a canonical representation of the proposed action.** Any change to the action, target, parameters, data classification, agent identity, or policy version invalidates the approval. **Reauthorize immediately before execution** to prevent time-of-check/time-of-use (TOCTOU) failures.

Critical actions may require multiple independent approvers and phishing-resistant authentication. Approval timeouts and system failures default to **denial**. All approval decisions are recorded in the audit trail with approver identity, decision, timestamp, and signature.

---

## 9. Audit, Monitoring, and Incident Response

### 9.1 Audit Records

Security-relevant events must record: authenticated agent and initiating principal; tenant, session, task, and trace identifiers; requested action and target; policy decision and policy version; human-approval evidence; tool identity, version, and execution boundary; input/output references, classifications, or approved hashes; outcome, timing, and error category; and sequence/integrity metadata. Include the signing key identifier (`key_id`) in every record so that records remain attributable across key rotations.

> **The initiating principal must be an authenticated claim, not a self-asserted string.** Derive the principal recorded in audit events from a validated identity assertion — an OIDC ID token verified against the identity provider's published keys (issuer, audience, expiry) or an mTLS-bound session assertion — and record the issuer, subject, assertion hash, and binding type. A session whose principal is not assertion-backed must be labeled as self-asserted; its records identify a session, not an accountable human. Retain the assertion hash so records can be correlated against identity-provider logs that the agent runtime cannot author. Note that a binding-type field written by the agent runtime is itself forgeable under compromise; its assurance value depends on that external correlation (§9.5).

Use canonical serialization, immutable verification copies, and independently controlled append-only or WORM storage, with externally anchored checkpoints.

> **Hash chaining alone does not prevent deletion, rollback, truncation, or wholesale replacement of a local chain.** It makes tampering *evident* only when combined with independent, access-controlled storage (e.g., S3 Object Lock, GCS retention/Bucket Lock) and external anchoring. The agent must have no access to log storage or modification infrastructure. Even then, records authored during a live compromise are cryptographically valid: single-stream integrity has a ceiling of tamper-evidence for pre-compromise history. Reconstructing events within a compromise window requires cross-stream corroboration (§9.5).

### 9.2 Traceable Decisions

Record concise, purpose-generated decision evidence: objective and relevant source provenance; proposed action; tools selected; material assumptions and uncertainty; policy checks and results; human approvals; and final action/outcome.

Do **not** require or retain raw private chain-of-thought tokens as a compliance artifact; generate structured decision summaries instead. Apply minimization, redaction, encryption, access control, and retention limits to decision records.

### 9.3 Behavioral Monitoring

Monitor for: new tools, destinations, or data types; rate, volume, cost, or latency anomalies; repeated denied requests; delegation depth or fan-out anomalies; read-then-exfiltrate action sequences (compositional risk); credential or policy failures; missing expected telemetry (evasion signature); cross-tenant access indicators; divergence between independently authored record streams (§9.5); and model, prompt, tool, or dependency changes.

Statistical and model-based detectors (baselines, z-scores, distribution-shift tests, goal-alignment scoring) produce **signals**; they do not replace policy enforcement. Calibrate thresholds against representative benign and malicious sessions, and track false-positive rate and correct-escalation rate as first-class metrics.

### 9.4 Incident Response

Procedures must support containment, credential revocation, evidence preservation, tenant notification, rollback, dependency quarantine, regulatory assessment, root-cause analysis, and controlled restoration. Rebuild compromised workloads from known-good images and rotate potentially exposed credentials from a clean environment. Exercise the plan before production and after material architectural changes.

During forensic reconstruction, treat the agent-authored record stream as one input among several; reconcile it against the independently authored streams in §9.5 before drawing conclusions about events inside the compromise window.

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

> Signatures establish origin and integrity, **not** safety; runtime isolation and authorization remain mandatory. When recording AIBOM entries, verify each component's real package ecosystem, version scheme, and identifiers rather than assuming them.

A supply-chain risk score (known CVEs, maintainer activity, version age, open security issues) can gate adoption into `approve`, `review-required`, or `reject`, with heavier runtime controls for higher scores.

---

## 11. Threat Modeling and Security Validation

Threat modeling must cover model behavior, orchestration, memory, tools, data, identity, infrastructure, supply chain, users, and cross-agent interactions. Map trust boundaries and multi-step abuse paths rather than evaluating individual prompts or tool calls in isolation.

A structured, layered approach such as CSA's **MAESTRO** framework can organize this analysis. MAESTRO defines a **seven-layer** reference architecture — foundation models; data operations; agent frameworks; deployment and infrastructure; evaluation and observability; security and compliance (vertical); and the agent ecosystem — and maps threats and mitigations across those layers. Use the framework's actual layer structure; do not substitute an invented layer count.

Validation must include: direct and indirect prompt injection; goal hijacking and policy conflict; tool substitution and manifest changes; confused-deputy and token misuse; cross-session and cross-tenant leakage; memory poisoning and provenance failures; privilege amplification through safe-looking action sequences; approval tampering and stale approvals; sandbox escape and resource exhaustion; data exfiltration and covert destinations; logging interruption and evidence manipulation; forgery and false-record authorship during simulated runtime compromise (validating the §9.5 corroboration path); and recovery/emergency-stop operation.

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
| Human oversight (§8) | MANAGE | Human oversight / stop mechanism | Bound approval and emergency stop for high-impact actions |
| Audit & monitoring (§9) | GOVERN / MEASURE | Record-keeping & traceability | Tamper-evident, independently stored decision records |

---

## 16. Implementation Priorities

**Priority 1 — Required Before Production**

- Workload identity and external (deterministic) policy enforcement
- Least privilege and task-scoped, proof-of-possession-bound authorization
- Execution sandboxing and network containment
- Structured tool schemas and safe database access
- Bound human approval for high-impact actions (with TOCTOU-safe re-check)
- Append-only/WORM audit storage and tested incident response
- Assertion-backed principal attribution in audit records (§9.1)
- Emergency stops and credential revocation
- Threat modeling and a passed production security gate

**Priority 2 — Required Before Scaling**

- Central authenticated service and tool registry
- Automated artifact and dependency verification
- Cross-step behavioral monitoring
- Cross-stream corroboration coverage for compromise-window reconstruction (§9.5)
- Memory provenance and lifecycle controls
- Multi-agent delegation governance
- Complete control evidence and recertification

**Priority 3 — Continuous Improvement**

- Expanded adversarial and scenario testing
- Detection calibration and response automation
- Stronger workload isolation for higher-risk use cases
- Independent assurance and regulatory-mapping updates
- Environmental and operational efficiency monitoring

---

## 17. Glossary

- **Agent:** An AI-enabled system that can plan or select actions and invoke tools.
- **AIBOM:** Inventory of models, data, prompts, tools, and supporting software used by an AI system.
- **Cross-stream corroboration:** Reconciling the agent-authored audit stream against independently authored record streams (broker decisions, KMS/CSP logs, tool-side logs, network telemetry) that the agent runtime cannot write; the required control for reconstructing events during a live compromise.
- **DID:** Decentralized Identifier (W3C); requires a method defining issuance, resolution, rotation, and revocation.
- **DPoP:** Demonstrating Proof of Possession; binds token use to a key.
- **HITL:** Human-in-the-loop review or approval for defined actions.
- **JIT:** Just-in-time issuance of short-lived, scoped credentials.
- **MAESTRO:** CSA's seven-layer agentic AI threat-modeling framework.
- **MCP:** Model Context Protocol; exposes context and tools to compatible clients.
- **mTLS:** Mutual TLS; bidirectional certificate authentication.
- **Policy decision point (PDP):** Evaluates an action against authorization policy.
- **Policy enforcement point (PEP):** Permits, denies, or constrains execution based on a policy decision.
- **Proof of possession:** Cryptographic evidence that a caller holds the key to which a credential is bound.
- **Self-verifying identity (anti-pattern):** An identity or audit scheme in which issuance, signing, and verification reside in the same process or trust domain, or a symmetric scheme in which verifiers can forge; provides cryptographic structure without a trust boundary.
- **SPIFFE/SPIRE:** Standard and runtime for securely identifying software workloads.
- **TOCTOU:** Time-of-check/time-of-use; the gap between approval and execution that must be closed by re-authorization.
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
- OAuth 2.0, token exchange (RFC 8693), mTLS token binding, and DPoP specifications
- OpenID Connect Core (ID token validation, for principal assertion binding)
- CSA MAESTRO and MITRE ATLAS threat-modeling sources
- Current MCP protocol and authorization specifications
- CIS Kubernetes Benchmark (for container/orchestration hardening)

---

## Conclusion

Secure agentic AI requires independently enforced controls **around** the model. Cryptographic workload identity, short-lived proof-of-possession authority, isolated execution, provenance-aware context, brokered and schema-constrained tools, cryptographically bound approvals, tamper-resistant and independently stored evidence, and tested containment together form the production security boundary. Model instructions, delimiters, and classifiers add valuable defense in depth, but they do not replace deterministic authorization or operational accountability. Scope every integrity control to the failure mode it actually closes, and assume that reconstructing a compromise requires corroboration across streams the agent cannot author. Treat every code snippet here as a sketch to validate, every framework mapping as a planning aid to verify, and autonomy as something earned through evidence rather than asserted.
