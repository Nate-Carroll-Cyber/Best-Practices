# Securing Agentic AI: Technical Implementation Framework v1.2

## Executive Summary

Agentic AI systems can plan, invoke tools, modify external systems, and delegate work. Their security therefore depends on more than model-level safeguards. Production deployments require independently enforced identity, authorization, isolation, data protection, auditability, human oversight, and incident-response controls.

This framework defines a defense-in-depth architecture built on five principles:

1. **The model proposes; deterministic controls decide.** A model must never grant itself authority or act as the final policy-enforcement point.
2. **Every action has an authenticated principal.** Human, service, agent, and delegated identities remain traceable throughout execution.
3. **Authority is narrow, temporary, and context-bound.** Permissions are task-scoped, audience-restricted, and resistant to replay.
4. **Untrusted content remains untrusted.** User input, retrieved content, tool output, memory, and inter-agent messages cannot become policy merely by entering the context window.
5. **High-impact actions remain interruptible and accountable.** Human approval, circuit breakers, and independently controlled audit records constrain autonomy.

This document provides implementation guidance. Regulatory mappings are planning aids, not legal advice, certification, or evidence of conformity.

---

## 1. Purpose, Scope, and Audience

This framework applies to AI systems that can independently select or invoke tools, interact with external services, maintain state, delegate tasks, or cause material effects outside the model runtime.

It is intended for:

- System and security architects
- Engineering and platform teams
- Product and model owners
- Security operations and incident-response teams
- Privacy, legal, compliance, and risk functions
- Internal and external assessors

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

| Role | Core responsibility |
|---|---|
| Agent owner | Defines purpose, acceptable use, risk tolerance, and operating constraints; accepts documented residual risk within delegated authority. |
| Service provider | Designs, operates, patches, tests, and documents the agent service. |
| Model provider | Supplies model capabilities and relevant documentation under the applicable service agreement. |
| Tool or MCP provider | Secures tool behavior, interfaces, dependencies, credentials, and change lifecycle. |
| Data owner | Authorizes data use, classification, access, retention, and deletion. |
| CISO or security delegate | Approves security architecture, testing criteria, exceptions, and incident-response integration. |
| Privacy and legal functions | Determine applicable privacy, AI, consumer, employment, and sector-specific obligations. |
| Human approver | Reviews high-impact actions within a defined scope and remains accountable for the approval decision. |

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

Security controls mediate every execution cycle:

```text
PERCEIVE -> LABEL AND VALIDATE -> PLAN -> PROPOSE ACTION
        -> AUTHENTICATE -> AUTHORIZE -> APPROVE IF REQUIRED
        -> EXECUTE IN CONSTRAINTS -> OBSERVE -> RECORD -> CONTINUE OR STOP
```

The agent may plan and propose. External policy-enforcement points authorize and constrain execution.

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

Message roles, delimiters, XML or JSON encoding, and prompt separation may improve model interpretation, but they are not access controls. Security decisions must fail closed outside the model.

### 3.3 Trust Zones

At minimum, separate:

- Untrusted ingestion and retrieval
- Model inference and orchestration
- Policy decision and enforcement
- Tool execution
- Credential issuance and key management
- Memory and data stores
- Audit and monitoring infrastructure
- Privileged administration

Default-deny network policies must restrict both ingress and egress. DNS must use approved resolvers; Kubernetes policies that permit DNS should account for both UDP and TCP port 53. Domain-level egress control requires a capable proxy or gateway because standard network policies do not enforce fully qualified domain names.

---

## 4. Identity and Authorization

### 4.1 Machine and Workload Identity

**Requirement:** Every deployed agent workload must have a unique, cryptographically verifiable identity issued by an organization-approved identity authority. Shared agent service accounts are prohibited.

- Prefer established workload identity such as SPIFFE/SPIRE, cloud workload identity, or an organization-managed PKI.
- Place workload identity in a defined URI SAN or equivalent authenticated claim; do not rely on certificate Common Name parsing.
- If decentralized identifiers are used, select or define a DID method with documented issuance, resolution, rotation, recovery, and revocation.
- Keep mutable roles and capabilities in the authorization system rather than identity certificates.
- Automate certificate and key rotation, revocation, and compromise response.

Identity establishes the calling workload. It does not grant permission by itself.

### 4.2 Least Agency and Delegation

Each agent receives only the tools, resources, data, destinations, and action types required for its declared task.

Delegation must satisfy all of the following:

- A child cannot receive authority exceeding its parent's delegable authority.
- Each hop narrows or preserves scope; it never silently expands it.
- Delegation records identify the human principal, parent, child, purpose, scope, and expiration.
- Maximum delegation depth and fan-out are enforced.
- Child credentials and sessions terminate with the task or parent unless explicitly and safely re-parented.
- Policy is re-evaluated at every delegation and tool boundary.

### 4.3 Just-in-Time Authorization

Credentials must be short-lived, audience-restricted, task-scoped, and resistant to replay.

1. Authenticate the agent using workload identity.
2. Evaluate the requested action against user authority, task, resource, environment, and policy.
3. Issue only the required scopes for the intended audience.
4. Bind the token to the workload using an approved proof-of-possession mechanism, such as mTLS confirmation claims or DPoP.
5. Validate issuer, audience, expiration, scope, proof-of-possession, and revocation state at every enforcement point.
6. Revoke the credential or allow it to expire when the task ends.

The `jti` claim supports token uniqueness and replay detection; it does not make a bearer token non-transferable.

### 4.4 Agent and Service Discovery

Discovery metadata must come from an authenticated registry and include service identity, approved endpoint, protocol, owner, current version, and health status. Registry responses must be integrity-protected, access-controlled, logged, and subject to expiration. Discovery does not replace authentication or authorization at the destination service.

---

## 5. Execution Isolation and Containment

### 5.1 Sandboxing

Run untrusted code and high-risk tools in isolated, disposable execution environments.

Minimum controls include:

- Non-root execution
- No privilege escalation
- All Linux capabilities dropped unless explicitly justified
- Read-only root filesystem
- Dedicated ephemeral writable paths
- Seccomp and mandatory access-control policy
- CPU, memory, process, file-size, and execution-time limits
- Default-deny network access
- No host sockets, device access, or cross-agent volume mounts
- Image and dependency integrity verification

For higher-risk or multi-tenant workloads, use stronger boundaries such as microVMs, sandboxed runtimes, or dedicated workers.

Example Kubernetes security context:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

volumeMounts:
  - name: tmp
    mountPath: /tmp

volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 100Mi
```

Do not mount a ConfigMap over `/` as a substitute for a read-only container filesystem.

### 5.2 Tool Execution Broker

All consequential tool calls pass through an enforcement broker that:

- Authenticates the initiating principal and agent
- Validates the tool and pinned version
- Validates a strict request schema
- Evaluates policy using canonical action data
- Applies resource, destination, and rate limits
- Obtains bound human approval where required
- Issues task-scoped credentials
- Executes in the required isolation boundary
- Validates and classifies results
- Records the decision and outcome

Tool output must never automatically trigger another tool. Each subsequent action requires a new proposal and policy decision.

### 5.3 Circuit Breakers and Emergency Stops

Independent containment mechanisms must be able to reject actions, revoke credentials, close network paths, terminate tools, quarantine sessions, preserve forensic evidence, and alert responders.

Automatic recovery may be used for low-risk, idempotent operations. Breakers involving privileged, destructive, financial, safety-critical, or externally visible actions remain latched until an authorized operator reviews and resets them. Recovery tests must use non-destructive synthetic actions.

---

## 6. MCP and Tool Security

### 6.1 Threat Model

Treat MCP servers and other dynamic tool providers as untrusted supply-chain and execution dependencies. Relevant threats include:

- Malicious or changed tool descriptions
- Tool substitution or rug pulls
- Excessive or undisclosed permissions
- Token passthrough and confused-deputy behavior
- Cross-session or cross-tenant leakage
- Unsafe error disclosure
- Tool-output prompt injection
- Unauthorized network callbacks
- Dependency compromise and sandbox escape

### 6.2 Local Transports

STDIO and local-socket servers must run as isolated child workloads with pinned artifacts, minimal filesystem access, explicit environment variables, no inherited secrets, constrained network access, and deterministic shutdown. Local transport does not imply that the tool implementation is trusted.

### 6.3 Remote Transports

Remote servers require TLS, server authentication, protocol validation, scoped authorization, replay resistance, rate limiting, and centralized policy enforcement. Do not forward a client's bearer token to downstream services. Obtain a distinct downstream token with the correct audience and minimum scope through an approved delegation or token-exchange flow.

### 6.4 Tool Manifests and Change Control

Approved tools must have a versioned manifest recording:

- Stable identifier and owner
- Artifact digest and provenance
- Input and output schemas
- Required permissions and destinations
- Data classifications
- Side effects and reversibility
- Resource limits
- Dependency inventory
- Review and expiration dates

Signatures and hashes establish artifact integrity, not benign behavior. Enforce permissions at runtime, and require re-review when code, description, schema, permissions, owner, dependencies, or endpoint changes.

### 6.5 Error Handling

Return stable, minimal error codes to the model. Keep detailed stack traces, paths, query text, credentials, and infrastructure identifiers in access-controlled operational logs. Apply secret detection and redaction before any error enters model context.

### 6.6 Product and Tool References

Named products and open-source projects are candidates for evaluation, not endorsed controls. Before publication or adoption, verify that each project exists, is actively maintained, has an identifiable owner, supports the claimed capability, and meets supply-chain requirements. Record its validated version, repository, license, security posture, and review date.

---

## 7. Input, Context, Memory, and Output Security

### 7.1 Untrusted-Content Handling

Label user input, retrieved documents, web content, tool output, files, memory, and inter-agent messages with provenance and trust metadata. Preserve those labels through selection, summarization, compression, and storage.

Content scanning and prompt-injection classifiers are signals, not authorization mechanisms. They may block, warn, reduce privileges, or trigger review, but sensitive actions still require deterministic policy enforcement.

### 7.2 Structured Actions and Schema Enforcement

Models must propose typed actions rather than executable command strings. Schemas must reject unknown fields, enforce lengths and formats, canonicalize values, and validate cross-field constraints. The enforcement layer must separately verify authorization and business rules.

### 7.3 Safe Database Access

Agents must not execute unrestricted model-generated SQL against production databases.

- Prefer predefined, parameterized operations exposed through a query broker.
- Map each operation to approved tables, columns, filters, and result limits.
- Use read-only identities for read operations.
- Apply row- and column-level controls where available.
- Enforce time, concurrency, cost, and result-size limits.
- Disable multi-statement execution and dangerous functions.

If free-form SQL is unavoidable, use a database-aware parser and an isolated read replica. Regular expressions and blocked-keyword lists are not sufficient controls.

### 7.4 Session and Memory Security

Isolate state by tenant, user, agent, and task. Temporary data must use encrypted ephemeral storage with explicit retention and cleanup rules. When cryptographic erasure is required, use per-session encryption keys and destroy the relevant key. Ordinary object deletion, garbage collection, and file removal are not cryptographic zeroing.

Every memory entry must record its authenticated writer, source, tenant, task, creation time, confidence, classification, and expiration. Separate observed facts, user assertions, model inferences, and executable instructions. Support quarantine, versioning, correction, rollback, and deletion.

A signature proves that content has not changed since signing; it does not prove that the content was true or safe when accepted.

### 7.5 Output and Data-Loss Controls

Before release or tool reuse, validate outputs for schema, destination, data classification, secrets, personal data, policy violations, and size. Apply destination-aware data-loss prevention. High-impact disclosures require bound approval and a final authorization check.

---

## 8. Human Oversight

High-impact, irreversible, externally visible, or security-sensitive actions require explicit approval unless a documented assessment permits bounded automation.

Approval requests must show:

- Exact action, target, and material parameters
- Data accessed or disclosed
- Expected impact and reversibility
- Agent and initiating principal
- Policy decision and relevant warnings
- Expiration time
- Recovery or rollback mechanism

Bind approval cryptographically to a canonical representation of the proposed action. A change to the action, target, parameters, data classification, agent identity, or policy version invalidates the approval. Reauthorize immediately before execution to prevent time-of-check/time-of-use failures.

Critical actions may require multiple independent approvers and phishing-resistant authentication. Approval timeouts and system failures default to denial.

---

## 9. Audit, Monitoring, and Incident Response

### 9.1 Audit Records

Security-relevant events must record:

- Authenticated agent and initiating principal
- Tenant, session, task, and trace identifiers
- Requested action and target
- Policy decision and policy version
- Human approval evidence
- Tool identity, version, and execution boundary
- Input and output references, classifications, or approved hashes
- Outcome, timing, and error category
- Sequence and integrity metadata

Use canonical serialization, immutable verification copies, independently controlled append-only or WORM storage, and externally anchored checkpoints. Hash chaining alone does not prevent deletion, rollback, truncation, or replacement of an entire local chain.

### 9.2 Traceable Decisions

Record concise, purpose-generated decision evidence:

- Objective and relevant source provenance
- Proposed action
- Tools selected
- Material assumptions and uncertainty
- Policy checks and results
- Human approvals
- Final action and outcome

Do not require or retain private chain-of-thought tokens. Apply minimization, redaction, encryption, access control, and retention limits to decision records.

### 9.3 Behavioral Monitoring

Monitor for:

- New tools, destinations, or data types
- Rate, volume, cost, or latency anomalies
- Repeated denied requests
- Delegation depth or fan-out anomalies
- Read-then-exfiltrate action sequences
- Credential or policy failures
- Missing expected telemetry
- Cross-tenant access indicators
- Model, prompt, tool, or dependency changes

Thresholds must be calibrated against representative benign and malicious sessions. Statistical and model-based detectors produce signals; they do not replace policy enforcement.

### 9.4 Incident Response

Response procedures must support containment, credential revocation, evidence preservation, tenant notification, rollback, dependency quarantine, regulatory assessment, root-cause analysis, and controlled restoration. Exercise the plan before production and after material architectural changes.

---

## 10. Supply-Chain Security

Maintain an AI and software bill of materials covering:

- Models and model versions
- Runtime and agent frameworks
- System and developer prompts
- MCP servers and tools
- Source repositories and build provenance
- Packages, containers, and infrastructure modules
- Datasets, embeddings, and knowledge sources
- Licenses, owners, and support status
- Artifact hashes and signatures
- Vulnerabilities, exceptions, and review dates

Verify provenance and signatures where available, pin production artifacts, scan dependencies, protect the build pipeline, and require review for material changes. Signatures establish origin and integrity, not safety; runtime isolation and authorization remain mandatory.

---

## 11. Threat Modeling and Security Validation

Threat modeling must cover model behavior, orchestration, memory, tools, data, identity, infrastructure, supply chain, users, and cross-agent interactions. Map trust boundaries and multi-step abuse paths rather than evaluating individual prompts or tool calls in isolation.

Validation must include:

- Direct and indirect prompt injection
- Goal hijacking and policy conflict
- Tool substitution and manifest changes
- Confused-deputy and token misuse
- Cross-session and cross-tenant leakage
- Memory poisoning and provenance failures
- Privilege amplification through safe-looking action sequences
- Approval tampering and stale approvals
- Sandbox escape and resource exhaustion
- Data exfiltration and covert destinations
- Logging interruption and evidence manipulation
- Recovery and emergency-stop operation

Measure detection rate, false-positive rate, unsafe action completion rate, correct escalation rate, containment time, and recovery time. Test complete sessions and realistic action sequences, not only isolated prompts.

---

## 12. Production Security Gate

An agent must not enter or remain in production with an unresolved critical vulnerability or an unaccepted high-severity vulnerability affecting authorization, isolation, data protection, or privileged tool use.

Production approval requires:

- Documented remediation and regression testing
- Representative adversarial testing
- Verified policy enforcement and emergency controls
- Operational monitoring and response readiness
- Privacy and legal review where applicable
- Documented residual risk and named risk owner
- Time-bounded, approved exceptions where policy permits them
- Security and business-owner sign-off

Example blocking decision:

> **Production decision: NOT APPROVED.** A high-severity privilege-escalation path remains unresolved. Deployment is blocked until remediation is complete and independent retesting confirms that the execution chain is prevented.

---

## 13. Phased Deployment

### Phase 1: Isolated Evaluation

- Synthetic or approved test data
- No production credentials
- Read-only or simulated tools
- Security-team access
- Threat-model and abuse-case validation

### Phase 2: Internal Staging

- Production-like isolation and monitoring
- Narrow internal use cases
- Limited, reversible writes
- Human approval for consequential actions
- Incident-response exercise completed

### Phase 3: Limited Production

- Defined user cohort and use cases
- Bounded tools, destinations, and data
- Enhanced monitoring and daily review
- Tested rollback and kill switch
- Explicit go/no-go criteria

### Phase 4: General Production

- Scale only within validated boundaries
- Continuous control evidence and testing
- Periodic access and tool recertification
- Change-triggered reassessment
- Documented decommissioning and data-deletion plan

Advancement is evidence-based, not time-based. A fixed number of operating hours or a zero-incident period does not demonstrate security by itself.

---

## 14. Control Coverage and Evidence

Do not report unsupported percentage-complete claims. Track implementation and effectiveness through evidence.

| Risk domain | Control | Status | Evidence | Test result | Residual risk | Owner | Review date |
|---|---|---|---|---|---|---|---|
| Runtime execution | External policy enforcement | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Prompt injection | Untrusted-content isolation | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Privilege escalation | Tool authorization | Not assessed | Pending | Not tested | Unknown | TBD | TBD |
| Accountability | Append-only audit storage | Not assessed | Pending | Not tested | Unknown | TBD | TBD |

Allowed statuses are `Not assessed`, `Planned`, `Partial`, `Implemented`, `Verified`, and `Exception approved`. If coverage percentages are used, document the scoring method, evidence standard, treatment of partial controls, and independent validation process.

---

## 15. Regulatory and Standards Alignment

This framework supports preparation for applicable AI, cybersecurity, privacy, and sector-specific obligations. Applicability depends on intended purpose, deployment context, affected persons, jurisdiction, and the organization's legal role.

For every mapping:

- Cite the exact regulation or standard edition, article or control, and publication date.
- Identify the regulated actor and system category.
- Separate legal requirements from organizational policy and optional guidance.
- Record the implementation artifact and evidence owner.
- Obtain qualified legal review for legal interpretations.
- Obtain auditor review before representing ISO/IEC 27001 or SOC 2 alignment.

Alignment does not establish conformity, certification, or audit readiness. ISO/IEC 27001 control references must identify the applicable edition. SOC 2 mappings must use the applicable Trust Services Criteria and should not describe section labels as certifications.

---

## 16. Implementation Priorities

### Priority 1: Required Before Production

- Workload identity and external policy enforcement
- Least privilege and task-scoped authorization
- Execution sandboxing and network containment
- Structured tool schemas and safe database access
- Bound human approval for high-impact actions
- Append-only audit storage and incident response
- Emergency stops and credential revocation
- Threat modeling and production security gate

### Priority 2: Required Before Scaling

- Central service and tool registry
- Automated artifact and dependency verification
- Cross-step behavioral monitoring
- Memory provenance and lifecycle controls
- Multi-agent delegation governance
- Complete control evidence and recertification

### Priority 3: Continuous Improvement

- Expanded adversarial and scenario testing
- Detection calibration and response automation
- Stronger workload isolation for higher-risk use cases
- Independent assurance and regulatory mapping updates
- Environmental and operational efficiency monitoring

---

## 17. Glossary

- **Agent:** An AI-enabled system that can plan or select actions and invoke tools.
- **AIBOM:** Inventory of models, data, prompts, tools, and supporting software used by an AI system.
- **DPoP:** Demonstrating Proof of Possession, a mechanism for binding token use to a key.
- **HITL:** Human-in-the-loop review or approval for defined actions.
- **MCP:** Model Context Protocol, a protocol for exposing context and tools to compatible clients.
- **Policy decision point:** Component that evaluates an action against authorization policy.
- **Policy enforcement point:** Component that permits, denies, or constrains execution based on a policy decision.
- **Proof of possession:** Cryptographic evidence that a caller holds the key to which a credential is bound.
- **SPIFFE:** A standard for securely identifying software workloads.
- **WORM:** Write once, read many storage used to resist alteration or deletion.

---

## 18. Reference Categories

Before release, maintain dated references to the authoritative versions used by the organization, including:

- Applicable EU AI Act text and implementation guidance
- NIST AI Risk Management Framework and relevant profiles
- NIST cybersecurity and identity guidance
- ISO/IEC 27001 and related AI management standards
- AICPA Trust Services Criteria
- W3C DID specifications when DIDs are used
- SPIFFE/SPIRE documentation when workload identity is used
- OAuth, token exchange, mTLS token binding, and DPoP specifications
- MITRE ATLAS and other adopted AI threat-modeling sources
- Current MCP protocol and authorization specifications

Every reference entry should record its title, publisher, version or date, URL, owner, and last validation date.

---

## Conclusion

Secure agentic AI requires independently enforced controls around the model. Identity, short-lived authority, isolated execution, provenance-aware context, constrained tools, bound approvals, tamper-resistant evidence, and tested containment form the production security boundary. Model instructions and classifiers add useful defense in depth, but they do not replace deterministic authorization or operational accountability.
