# Securing Agentic AI: Implementation Checklist

Companion to *Securing Agentic AI: Technical Implementation Framework*, Consolidated 2.8. Every item is self-contained and answerable without opening the framework. Section references in parentheses point to the source text for anyone who wants the reasoning.

**How to use.** Answer each item with implemented, partial, planned, exception approved, or not assessed, and record the evidence and the owner. Do not report a percentage complete. Gate A must pass before production. Gate B must pass before scaling beyond the initial cohort. Gate C runs continuously.

**Six principles behind every item.** The model proposes and deterministic controls decide. Every action has an authenticated principal. Authority is narrow, temporary, and context-bound. Untrusted content stays untrusted. High-impact actions stay interruptible and accountable. Capability combinations are the unit of risk.

---

## Step 0. Classification

Answer before writing any control, and answer again before deployment.

- [ ] Intended purpose, prohibited uses, affected persons, operating environment, data classifications, maximum autonomy, and accountable owner are documented (§1)
- [ ] Decisions and actions the system can take, and their reversibility, are enumerated (§2.1)
- [ ] External connectivity, tool privileges, multi-agent dependencies, and third-party dependencies are enumerated (§2.1)
- [ ] Applicable jurisdictions and sector obligations are identified, with the organization's legal role named (§2.1, §15)
- [ ] The trifecta scorecard below is complete and dated (§7.8)
- [ ] It is recorded whether the system executes model-generated code, runs training or evaluation workloads, or hosts research models. These are high-risk by default (§2.1)
- [ ] Approval status has not been allowed to lower the classification. A sanctioned embedded feature is classified on the data it reaches and the actions it takes (§2.1)

**Trifecta scorecard** (re-answer whenever tools or scopes change)

| Question | Yes / No | Date | Notes |
|---|---|---|---|
| Does it reach private data? | | | |
| Does it ingest untrusted content? (a cached fetch service counts as yes) | | | |
| Can it communicate externally or take consequential action? | | | |

Three yes answers require removing or constraining at least one leg, or a documented residual-risk decision with a named owner. Where the model itself is the adversary, private data plus external communication alone scores as a trifecta equivalent.

**Owners assigned** (§2.2)

- [ ] Agent owner, service provider, model provider, tool or MCP provider, data owner, security approver, privacy and legal, human approver

**Governance artifacts maintained** (§2.3)

- [ ] Data-flow diagrams with trust boundaries, threat model and abuse cases, tool and capability inventory, provenance records, test results, oversight and escalation procedures, incident-response plan, control evidence and residual-risk decisions, versioned change history

---

## Gate A. Required before production

### A1. Identity and authorization

- [ ] Every agent workload has a unique cryptographically verifiable identity from an independent authority (SPIFFE or SPIRE, cloud workload identity, or managed PKI) (§4.1)
- [ ] Identity sits in a URI SAN or equivalent authenticated claim, not parsed from a certificate common name (§4.1)
- [ ] Roles and capabilities live in the authorization system, not baked into identity certificates (§4.1)
- [ ] No shared service accounts, no generic identities, no long-lived static keys as primary agent credentials (§4.1)
- [ ] No symmetric (HMAC) signing of audit or attestation records. Verifiers hold public material only (§4.1)
- [ ] Credentials are short-lived, audience-restricted, task-scoped, and bound to the workload with mTLS confirmation claims or DPoP (§4.3)
- [ ] Issuer, audience, expiry, scope, proof of possession, and revocation state are validated at every enforcement point, on every request, not once at session start (§4.3)
- [ ] Authorization is decided server-side from verified facts, never from client-supplied identity, roles, or scopes (§4.3)
- [ ] Where OAuth is used, tokens are bound to their target with resource indicators, and client credentials are not shared between agents (§4.3)
- [ ] Development, staging, and production are separated logically and physically, with no credential reuse across them (§4.3)
- [ ] Delegation narrows or preserves scope at every hop, records the human principal, enforces depth and fan-out limits, and terminates child sessions with the parent. No orphaned agents (§4.2)

### A2. Secrets

- [ ] The model never receives a credential value. The agent references an opaque handle the broker resolves at execution time (§4.5)
- [ ] Secrets live in a vault, resolved at execution time. Not in environment variables, config files, images, source control, prompts, or logs (§4.5)
- [ ] No secret enters prompts, context windows, memory, decision records, vector stores, or embeddings (§4.5)
- [ ] Secret detection and redaction run over prompt logs, application logs, tool output, and error payloads before persistence or display (§4.5)
- [ ] Repository-supplied agent configuration (settings files, MCP server definitions, tool manifests, auto-approval directives) is not loaded before an explicit trust decision by the owner (§4.5)
- [ ] Retrospective sweep completed on onboarding, covering repository history, configuration, and the full log-retention window. Everything found is rotated (§4.5)
- [ ] Every token issuer the organization controls is enrolled in the secret-scanning partner programs of public code hosts, model hubs, and package registries (§4.5)

### A3. Isolation and containment

- [ ] Untrusted code and high-risk tools run non-root, with no privilege escalation, all capabilities dropped, read-only root filesystem, seccomp and mandatory access control, and CPU, memory, process, file-size, and time limits (§5.1)
- [ ] Default-deny network access in the sandbox. No host sockets, no device access, no cross-agent volume mounts (§5.1)
- [ ] Higher-risk or multi-tenant workloads use a stronger boundary (microVM, sandboxed runtime, or dedicated per-tenant worker) (§5.1)
- [ ] Egress is enforced at two independent layers. Neither layer reads the other's configuration and compromising one does not relax the other (§3.3)
- [ ] Workload identities are denied cloud IAM permission to create load balancers, private links, peering, tunnels, VPN enrollments, or DNS records (§3.3)
- [ ] Transitive paths are enumerated and closed. Every shared service, cache, internal API, and private link the workload can reach inherits the workload's egress restrictions (§3.3)
- [ ] The cloud metadata endpoint is blocked from agent workloads unless a reviewed need exists (§3.3)
- [ ] DNS policy accounts for both UDP and TCP on port 53, and FQDN-level control uses a capable proxy rather than IP-based network policy (§3.3)
- [ ] A model routing layer or LLM gateway, where one exists, is isolated as its own trust zone with per-tenant and per-agent credential separation (§3.3)
- [ ] Emergency stop can reject actions, revoke credentials, close network paths, terminate tools, quarantine sessions, preserve evidence, and alert responders (§5.3)
- [ ] A per-server kill switch disables a specific tool server, revokes its credentials, and cuts its reachability within minutes without a deployment cycle (§5.3, §4.6)
- [ ] Breakers on privileged, destructive, financial, safety-critical, or externally visible actions remain latched until an authorized operator resets them (§5.3)
- [ ] Class-scoped stop halts every workload matching a type, model family, or evaluation task in one operation. It enumerates from the registry and runs on infrastructure the halted class cannot reach (§5.3)

### A4. The enforcement broker

- [ ] A deterministic broker runs outside the agent runtime, evaluates policy, and executes the action itself. Honoring a denial is not something the runtime can skip (§5.2)
- [ ] Broker scope covers tool calls, memory writes, memory and knowledge retrieval, compaction, subagent start and stop, skill or tool registration, and any write to a location another run can read (§5.2)
- [ ] Tool output never automatically triggers another tool. Each action requires a new proposal and a new decision (§5.2)
- [ ] Per-call timeouts and output-size limits are enforced, and failed privileged calls are not auto-retried (§5.2)
- [ ] Per-task volume budgets on records and bytes read, written, and sent are enforced as limits, not alerted on as signals (§5.2)
- [ ] Recursion depth, chain length, delegation fan-out, and per-task compute and wall-clock budgets are capped, with timeouts on long-running operations (§5.2)
- [ ] Backpressure and queue limits apply at the boundary, and malformed, oversized, and missing-field requests are rejected before model or tool execution (§5.2)
- [ ] Consequential operations carry an idempotency key, and a duplicate is rejected or safely replayed rather than re-executed (§5.2)
- [ ] The runtime declares its capabilities at session start and the session is refused when it cannot supply provenance labels, correlation identifiers, or signed envelopes (§5.2)
- [ ] Task feasibility is confirmed before the session starts. An objective unachievable within granted scopes is refused rather than run to failure (§5.2)
- [ ] Runtime-to-broker requests carry a timestamp within a negotiated skew window and a unique request identifier, and replays are rejected (§5.2)
- [ ] The broker is deployed redundantly across failure domains with a measured availability target, and any outage is handled as a security incident (§5.2, §9.4)
- [ ] Every broker decision (permit, deny, approve-required, error) carries the correlation identifier (§5.2)

### A5. Input, context, and output

- [ ] Every input carries provenance and trust metadata, including user input. Labels survive selection, summarization, compression, and storage (§7.1)
- [ ] Compaction preserves security determinations, not only labels. Out-of-scope conclusions, realness determinations, and refusals carry forward with their uncertainty intact (§7.1)
- [ ] Error payloads, tool metadata, and protocol fields are scanned as untrusted content on the same terms as document text (§7.1)
- [ ] Models propose typed actions, never executable command strings. Schemas reject unknown fields and canonicalize values (§7.2)
- [ ] Authorization and business rules are verified separately from schema validation (§7.2)
- [ ] No unrestricted model-generated SQL against production. Predefined parameterized operations through a query broker, read-only identities, and result-size limits (§7.3)
- [ ] Untrusted values are never concatenated into a command string. Structured process APIs with argument arrays, never shell execution (§7.6)
- [ ] File paths are canonicalized and symlinks resolved before the containment check, not after (§7.6)
- [ ] Model-generated code is not executed automatically in production agents. Research and evaluation workloads are exempt only under the documented conditions (§7.6)
- [ ] Sessions and memory are isolated by tenant, user, agent, and task, with infrastructure-enforced namespaces that make cross-tenant retrieval technically impossible (§7.4)
- [ ] Ownership is revalidated on every request. No cached or implicit session-to-user mapping (§7.4)
- [ ] Data is classified at ingestion, not at retrieval, and every context item has a TTL with enforced purge (§7.4)
- [ ] Output classification evaluates the aggregate, not the highest label among inputs (§7.5)
- [ ] Destination-aware data-loss prevention runs before release or tool reuse (§7.5)
- [ ] Publication to any location another run can read passes the broker and the approval path (§7.5, §5.2)
- [ ] The user's objective is anchored in trusted system instructions, structurally separate from retrieved content (§7.8)
- [ ] Scope-setting instructions state prohibited actions explicitly, phrased as rules the agent must follow rather than claims about what the environment allows (§7.8)
- [ ] The transition from planning to consequence is gated by an intent-alignment check (§7.8)
- [ ] Model identifier, region, sampling parameters, output ceiling, and per-task and per-tenant cost budgets are validated against allowlists and enforced (§7.7)

### A6. Error design and service surface

- [ ] One canonical error envelope across every component, with a stable enumerated code, generic message, escalation flag, correlation identifier, and timestamp (§6.5.2)
- [ ] The remediation hint the agent acts on is selected from a fixed enumeration, never interpolated from a downstream error string (§6.5.2)
- [ ] Four projections per failure (model, caller, operational log, audit) joined only by the correlation identifier (§6.5.1)
- [ ] Mock or stub mode is surfaced in the envelope, propagated to the audit stream, and refuses to start outside development (§6.5.2)
- [ ] Authorization denials are never retryable, and a 500 returns a generic message with detail confined to the operational log (§6.5.3)
- [ ] Redaction prefers an allowlist of loggable fields over a denylist, and runs before the error enters model context (§6.5.4)
- [ ] Any timeout, exception, or unavailability in the authorization path produces a denial. An unreachable policy service is a stop condition (§6.5.5)
- [ ] A service with no authentication configured refuses to start in any non-development environment (§6.5.5, §6.7)
- [ ] Authentication, request limits, and the response header baseline apply to every network-reachable component, including fallback, debug, and health-check servers (§6.7)
- [ ] Rate limiting keys on authenticated identity where available, falling back to client address only when it is not (§6.7)
- [ ] Caller-supplied correlation identifiers are accepted only if they validate, and carry no trust meaning (§6.7)

### A7. MCP and tool security

- [ ] Every approved tool has a versioned manifest with owner, artifact digest, schemas, required permissions and destinations, data classifications, side effects, resource limits, dependencies, and review dates (§6.4)
- [ ] Manifest signature and pinned version are validated at load and re-validated on every connection, with drift blocking execution (§6.4)
- [ ] The entire schema is scanned, including parameter names, enum values, and default values, not only the description field (§6.4)
- [ ] Display-control characters and ANSI escapes are stripped from schema fields before human review (§6.4)
- [ ] Any change to the advertised tool list or to a tool definition alerts. A tool that rewrites its own definition is a security event (§6.4)
- [ ] A connected server that adds a tool, widens a scope, or reaches a new destination requires a fresh consent decision by the accountable owner (§6.4)
- [ ] Local servers bind to localhost, require authentication and origin validation, and are inventoried on developer workstations (§6.2)
- [ ] Remote servers use TLS with server authentication, and the OAuth client and authorization-server roles are held by separate components (§6.3)
- [ ] Token passthrough is prohibited. Downstream calls use a distinct audience-scoped token obtained through delegation or token exchange (§6.3)
- [ ] Server registration is a deployment gate. An unregistered server fails to deploy, fails to obtain credentials, or fails to be reachable (§4.6)

### A8. Human oversight

- [ ] High-impact, irreversible, externally visible, and security-sensitive actions require explicit approval unless a documented assessment permits bounded automation (§8)
- [ ] Approval requests show the exact action, target, parameters, data touched, expected impact, reversibility, agent and principal, policy decision, expiration, and rollback mechanism (§8)
- [ ] Approval is cryptographically bound to a canonical representation of the action, and reauthorized immediately before execution (§8)
- [ ] Approval timeouts and system failures default to denial (§8)
- [ ] Freezes and approval requirements are authorization boundaries, not prompt instructions (§8)

### A9. Audit and evidence

- [ ] Records capture the authenticated agent and principal, tenant, session, task and trace identifiers, action and target, policy decision and version, approval evidence, tool identity and version, classifications, outcome, timing, error category, and sequence metadata (§9.1)
- [ ] Records carry the signing key identifier, correlation identifier, resolved model identifier and version, and mock-mode state (§9.1)
- [ ] The initiating principal is an authenticated claim, not a self-asserted string. Sessions without one are labeled self-asserted (§9.1)
- [ ] Credentials and personal data are masked at write time. The log store has not become a new sensitive-data store (§9.1)
- [ ] Records go to independently controlled append-only or WORM storage with externally anchored checkpoints, unreachable by the agent (§9.1)
- [ ] Reasoning traces, where the platform exposes them, are retained under the same controls as the audit stream, with a shorter retention ceiling, provider fidelity recorded, and legal review completed (§9.2)
- [ ] Reasoning traces are excluded from training and fine-tuning data, verified by configuration and by test (§9.2)
- [ ] Intent extensions and session refusals at capability declaration are recorded as security events (§9.1)

### A10. Supply chain

- [ ] AIBOM and SBOM cover models, frameworks, routing components, prompts, MCP servers and tools, repositories and build provenance, packages, containers, datasets, embeddings, licenses, hashes, and review dates (§10)
- [ ] Every dependency, plugin, and tool server is pinned to a reviewed version. No `latest`, no floating ranges, no runtime dependency fetch (§10)
- [ ] Every package and image dependency resolves through an internal mirror with no direct public-registry reach (§10)
- [ ] Maintenance status is verified before adoption and re-verified on schedule. Archived and unmaintained components are not adopted (§10)
- [ ] CVEs, vendor advisories, and upstream issue trackers are tied to the specific versions in the registry, so an advisory resolves to a named owner and a patch state (§10)
- [ ] Model artifacts resolve through an internal mirror where the workload loads model weights (§10)

### A11. Validation and gate

- [ ] Threat model covers model behavior, orchestration, memory, tools, data, identity, infrastructure, supply chain, users, and cross-agent interaction, mapping multi-step abuse paths rather than isolated prompts (§11)
- [ ] Tenant isolation is tested on every build and repeated under concurrency (§11)
- [ ] No unresolved critical vulnerability, and no unaccepted high-severity vulnerability affecting authorization, isolation, data protection, or privileged tool use (§12)
- [ ] Residual risk documented with a named owner, and security and business sign-off recorded (§12)

### A12. Classification-triggered

Apply where Step 0 flagged the condition.

- [ ] Class-scoped evaluation shutdown, for research and evaluation workloads (§5.3)
- [ ] Documented exemption conditions for model-generated code execution, for research and evaluation workloads (§7.6)
- [ ] Internal mirrors for model artifacts, for workloads that load model weights (§10)

---

## Gate B. Required before scaling

- [ ] Central authenticated service and tool registry, with signed integrity-protected responses (§4.4)
- [ ] Continuous shadow-server discovery across networks, repositories, developer environments, container registries, and CI/CD, reconciled against the registry in both directions (§4.6)
- [ ] Successive scans are diffed against each other, not only against the registry, so services that relocate or change between scans are surfaced (§4.6)
- [ ] Alerting whenever an agent connects to an endpoint absent from the approved registry (§4.6)
- [ ] Secure-by-default deployment templates so the compliant path is the fast path (§4.6)
- [ ] A single shared security library supplies authentication, rate limiting, correlation handling, error envelopes, and headers to every component (§6.7)
- [ ] CI-gated security regression suite covering error paths, injection, fail-closed behavior, and tenant isolation (§11)
- [ ] Full-schema metadata scanning and tool-definition change alerting (§6.4)
- [ ] Adversarial pre-validation. The model under test is tasked with escaping its own sandbox and defeating its own guardrails, supervised, offline, before any tool-granting run (§11)
- [ ] Cross-step behavioral monitoring with baselines established before anomaly alerting is relied on (§9.3)
- [ ] Agent, tool, platform, and host telemetry reaches the SIEM or XDR the security team actually monitors (§9.3)
- [ ] Platform monitoring baseline covering IAM events, network-path changes, and control-plane activity, separate from agent action records (§9.3)
- [ ] Auxiliary-model anomalies (summariser refusals, classifier errors, unexpected secondary-model output) are routed to security rather than discarded as pipeline faults (§9.3)
- [ ] Approver overrides of automated detections are recorded with justification and reviewed in aggregate (§9.3)
- [ ] Guardrail verdicts are kept out of any reward, score, or selection surface (§7.8)
- [ ] Cross-stream corroboration in place for compromise-window reconstruction, using broker decisions, cloud provider logs, tool-side logs, and network telemetry (§9.5)
- [ ] Memory provenance and lifecycle controls, including quarantine, versioning, correction, rollback, and deletion (§7.4)
- [ ] Multi-agent delegation governance and cumulative permission recertification (§4.2)
- [ ] Approver-only intent extension path with recorded scope (§7.8)
- [ ] Independent alignment gate ahead of consequential actions, running outside the agent's context (§7.8)
- [ ] Eradication runbook covering accessed and discovered credentials, caches, model weights, and external channels (§9.4)
- [ ] Retrospective run review capability, bounded by the earliest credible compromise indicator rather than declaration time (§9.4)
- [ ] Compaction preserves security determinations, with pre-compaction state retained for review (§7.1, §9.4)
- [ ] Mirror population governance with recorded approver, upstream source, digest, and signature result. Workload identities cannot populate the mirror (§10)
- [ ] Transitive-path enumeration and closure across shared services, including any cached fetch service (§3.3)
- [ ] Compromise-scenario drills exercised, not reviewed on paper (§9.4)

---

## Gate C. Continuous

- [ ] Expanded adversarial and scenario testing, with red teaming by an independent team whose test design the agent cannot influence (§11)
- [ ] Detection calibration tracked as first-class metrics, including false-positive rate and correct-escalation rate (§9.3)
- [ ] Stronger workload isolation for higher-risk use cases (§5.1)
- [ ] Configuration naming convergence to a single canonical scheme (§6.7)
- [ ] Periodic access and tool recertification, and change-triggered reassessment (§13)
- [ ] Independent assurance and regulatory-mapping updates (§15)
- [ ] Documented decommissioning and data-deletion plan (§13)

---

## Regression tests that gate CI

Run against every network-reachable component, including fallback and debug servers.

- [ ] Unauthenticated request denied
- [ ] Authenticated but unauthorized request denied with a distinct code, not retryable
- [ ] Unexpected content type rejected
- [ ] Oversized body rejected before parsing
- [ ] Malformed JSON rejected with the canonical envelope
- [ ] Rate limit engages with the correct status
- [ ] Correlation identifier returned, and an invalid caller-supplied one replaced rather than trusted
- [ ] Response header baseline present on success and error paths
- [ ] Secret redaction verified against a seeded credential inside a dependency error string
- [ ] Fail-closed verified under policy-service unavailability and dependency timeout
- [ ] Startup fails when authentication is unconfigured outside development, and when mock mode is enabled outside development
- [ ] Tenant A writes, tenant B retrieves nothing, repeated under concurrency
- [ ] Telemetry cannot be disabled, truncated, or redirected without an alert or independent approval
- [ ] Intent-misaligned action after untrusted-content injection is blocked by the alignment gate
- [ ] Egress probe blocked at both isolation layers independently. A block at only one layer is a failure
- [ ] Workload-identity attempt to create an outbound path denied at the cloud control plane, with the monitoring signal raised
- [ ] Mutable artifact reference, unsigned artifact, and mismatched cache digest each rejected at bind time
- [ ] Runtime declaring no provenance capability refused at session start
- [ ] Replayed request and out-of-skew timestamp each rejected
- [ ] Memory write, subagent start, and compaction each produce a broker decision record
- [ ] Attempt to modify the anchored objective from tool output rejected and logged
- [ ] Broker failover completes within target with no permit issued during transition
- [ ] A deliberately runaway task, recursing or fanning out without bound, is contained rather than serviced
- [ ] A replayed consequential operation carrying a previously seen idempotency key does not take effect twice
- [ ] A tool server that adds a capability after approval is blocked pending fresh consent
- [ ] Unsatisfiable task refused at session start rather than run to failure
- [ ] Compaction preserves an out-of-scope determination and a prior refusal
- [ ] Reasoning traces absent from every training and fine-tuning dataset
- [ ] Injection and traversal tests against every tool touching a process, filesystem, or query, including values arriving through metadata and protocol fields

**Control-environment attack cases**, run against the enforcement point rather than the agent.

- [ ] Instructions embedded in submitted content addressing the guardrail directly to elicit a permit
- [ ] Policy-relevant value obfuscated through runtime construction, defeating static inspection
- [ ] Workload attempt to edit its own permissions, policy file, sandbox configuration, or isolation-disable flag
- [ ] Attempt to start a second session or worker outside the monitored path
- [ ] Policy-violating operation buried inside a long otherwise-legitimate script or diff
- [ ] Attempt to terminate, blind, or rewrite the monitor and its records mid-run

---

## Verification probes

One-off exercises that produce evidence. Informative either way.

- [ ] Produce the last 30 days of tool-call logs. Any gap in availability, identity, tool, parameter metadata, or timestamp is a finding
- [ ] Search all repositories for MCP and agent configuration files, then reconcile against network scans and the registry
- [ ] Search the codebase for shell-invoking calls. Every occurrence is a finding until traced and replaced with an argument-array call
- [ ] Enumerate each agent's effective production permissions and compare against its documented permissions
- [ ] Trace every downstream call and flag any that forwards the caller's token unchanged
- [ ] Determine for each production agent whether its tool servers are vendor-built or drawn from an unreviewed repository
- [ ] Enumerate every outbound path from agent workloads, including shared services with their own egress, and compare against the allowlist. Any path a workload identity could have created is a critical finding
- [ ] Search public code hosts, model hubs, package registries, and paste services for tokens attributable to the organization. Revoke every match

---

## What this checklist cannot tell you

Coverage is not safety. Four limits carry from the framework and should sit in front of any completed copy of this list.

- A passed control check is not evidence the environment hosting the check is intact (§11)
- A behavioral audit establishes results about audit conditions. A system can score as aligned while behaving differently wherever a grader is visible (§11)
- Single-stream audit integrity has a ceiling of tamper-evidence for pre-compromise history. Records authored during a live compromise are cryptographically valid, so reconstructing that window requires streams the agent cannot author (§9.5)
- Alignment to a regulation or standard is a planning aid. It is not conformity, certification, or audit readiness, and legal conclusions require qualified counsel (§15)

Advancement between gates is evidence-based, not time-based. Operating hours and an incident-free period do not, by themselves, demonstrate security.
