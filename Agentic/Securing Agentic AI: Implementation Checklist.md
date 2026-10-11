# Securing Agentic AI: Implementation Checklist

Companion to *Securing Agentic AI: Technical Implementation Framework*, Consolidated 2.11. Every item is self-contained and answerable without opening the framework. Section references in parentheses point to the source text for anyone who wants the reasoning.

**How to use.** Answer each item with verified, implemented, partial, planned, exception approved, or not assessed, and record the evidence and the owner. Verified requires a produced artifact, export, configuration, test record, or drill record; interview evidence alone caps an item at implemented. Every partial carries a gap type. Document means the control runs and only the named artifact is missing. Measure means the control runs and the figure it asks for has not been computed. Scope means the control runs in part of the estate and not in another named part. Control means the mechanism is absent, misconfigured, or failed. Planned records an owner and a closure date no more than 180 days out and carries no weight. Do not report a percentage complete. Report three figures and three counts instead. Strict counts verified items only. Documented adds partials whose gap types are document or measure only, and states what the program could score by writing down and measuring what it already does. Provisional adds every remaining partial. Beneath them, count items with a foothold (verified, implemented, or partial), items absent (planned or not in place), and items unknown (not assessed). Gate A must pass before production. Gate B must pass before scaling beyond the initial cohort. Gate C runs continuously.

**Six principles behind every item.** The model proposes and deterministic controls decide. Every action has an authenticated principal. Authority is narrow, temporary, and context-bound. Untrusted content stays untrusted. High-impact actions stay interruptible and accountable. Capability combinations are the unit of risk.

---

## Step 0. Classification

Answer before writing any control, and answer again before deployment.

- [ ] Intended purpose, prohibited uses, affected persons, operating environment, data classifications, maximum autonomy, and accountable owner are documented (§1)
- [ ] Decisions and actions the system can take, and their reversibility, are enumerated (§2.1)
- [ ] External connectivity, tool privileges, multi-agent dependencies, and third-party dependencies are enumerated (§2.1)
- [ ] Applicable jurisdictions and sector obligations are identified, with the organization's legal role named (§2.1, §15)
- [ ] Where a deployment falls under Chinese jurisdiction, four expectations from TC260 AI Safety Governance Framework 3.0 are assessed against current law with counsel. Operational logs kept at least six months and audited, data collected or generated in China stored in China, user consent obtained and withdrawable before user data reaches a model or third-party tool, and major incidents reported to the competent authorities with affected users notified (§15)
- [ ] The trifecta scorecard below is complete and dated (§7.8)
- [ ] It is recorded whether the system executes model-generated code, runs training or evaluation workloads, or hosts research models. These are high-risk by default (§2.1)
- [ ] Approval status has not been allowed to lower the classification. A sanctioned embedded feature is classified on the data it reaches and the actions it takes (§2.1)

**Trifecta scorecard** (re-answer whenever tools or scopes change)

| Question | Yes / No | Date | Notes |
|---|---|---|---|
| Does it reach private data? | | | |
| Does it ingest untrusted content? (a cached fetch service counts as yes) | | | |
| Can it communicate externally or take consequential action? | | | |

Three yes answers require removing or constraining at least one leg, or a documented residual-risk decision with a named owner. Where the model itself is the adversary, private data plus external communication alone scores as a trifecta equivalent. Note: For an orchestrator or supervisor agent, "does it reach private data" scores on whether it holds a standing credential, not on whether it receives result content passed to it by a domain agent for routing purposes.

**Owners assigned** (§2.2)

- [ ] Agent owner, service provider, model provider, tool or MCP provider, data owner, security approver, privacy and legal, human approver

**Governance artifacts maintained** (§2.3)

- [ ] Data-flow diagrams with trust boundaries, threat model and abuse cases, tool and capability inventory, AIBOM with declared scope and completeness claim, test results, oversight and escalation procedures, decision-rights register mapping every automated action class to HITL or GITL with an owner per entry and the skills reservations, incident-response plan, control evidence and residual-risk decisions, versioned change history

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
- [ ] Where the runtime exposes sub-agents, each carries a persistent registry identifier separate from the workload identity it runs under, resolving to its registry entry and approved AIBOM component version; where it cannot, the gap is recorded in the AIBOM with a named owner (§4.1, §10.3)
- [ ] Where the runtime surfaces handoffs, the handoff payload is hashed at send and at receipt and a mismatch stops the run; where it cannot, the gap is recorded in the AIBOM with a named owner (§4.2)
- [ ] Delegation narrows or preserves scope at every hop, records the human principal, enforces depth and fan-out limits, and terminates child sessions with the parent. No orphaned agents (§4.2)
- [ ] Orchestrator and supervisor agent authority is scoped independently of the union of its downstream agents' permissions, and does not hold standing credentials to source data or destinations it merely reasons over the results of (§4.2)

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
- [ ] A breaker tripped by an action class running as GITL latches for the whole class, and the incident demotes the class to HITL in the decision-rights register until its owner re-earns promotion (§5.3, §8)
- [ ] Class-scoped stop halts every workload matching a type, model family, or evaluation task in one operation. It enumerates from the registry and runs on infrastructure the halted class cannot reach (§5.3)

### A4. The enforcement broker

- [ ] A deterministic broker runs outside the agent runtime, evaluates policy, and executes the action itself. Honoring a denial is not something the runtime can skip (§5.2)
- [ ] Broker scope covers tool calls, memory writes, memory and knowledge retrieval, compaction, subagent start and stop, skill or tool registration, and any write to a location another run can read (§5.2)
- [ ] A write is classified by its destination, not by the tool that made it. A file, row, or index entry a later session loads as context receives the memory-write decision even when it arrives through an ordinary file or shell tool (§5.2)
- [ ] Where the runtime has a session-wide permission mode, a change to it is a consequential action. A change toward less approval requires an approver, an agent-triggered change is denied, and the mode in force is recorded on every action record (§5.2, §9.1)
- [ ] The broker routes each action class as HITL or GITL from the decision-rights register, records the register entry version on every decision, and denies any class with no entry (§5.2, §8)
- [ ] Tool output never automatically triggers another tool. Each action requires a new proposal and a new decision (§5.2)
- [ ] Per-call timeouts and output-size limits are enforced, and failed privileged calls are not auto-retried (§5.2)
- [ ] Per-task volume budgets on records and bytes read, written, and sent are enforced as limits, not alerted on as signals (§5.2)
- [ ] Recursion depth, chain length, delegation fan-out, and per-task compute and wall-clock budgets are capped, with timeouts on long-running operations (§5.2)
- [ ] Backpressure and queue limits apply at the boundary, and malformed, oversized, and missing-field requests are rejected before model or tool execution (§5.2)
- [ ] Consequential operations carry an idempotency key, and a duplicate is rejected or safely replayed rather than re-executed (§5.2)
- [ ] The runtime declares its capabilities at session start and the session is refused when it cannot supply provenance labels, correlation identifiers, or signed envelopes. The declaration also states whether it can identify sub-agents, report permission-mode changes, and surface handoffs, and each unsupported item is recorded as an owned AIBOM gap rather than passed silently (§5.2, §10.2)
- [ ] Task feasibility is confirmed before the session starts. An objective unachievable within granted scopes is refused rather than run to failure (§5.2)
- [ ] Runtime-to-broker requests carry a timestamp within a negotiated skew window and a unique request identifier, and replays are rejected (§5.2)
- [ ] The broker is deployed redundantly across failure domains with a measured availability target, and any outage is handled as a security incident (§5.2, §9.4)
- [ ] Every broker decision (permit, deny, approve-required, error) carries the correlation identifier (§5.2)

### A5. Input, context, and output

- [ ] Every input carries provenance and trust metadata, including user input. Labels survive selection, summarization, compression, and storage (§7.1)
- [ ] Compaction preserves security determinations, not only labels. Out-of-scope conclusions, realness determinations, and refusals carry forward with their uncertainty intact (§7.1)
- [ ] Context overflow is in the threat model. Oversized input that fills the context window until built-in instructions are truncated removes no control, because every safety-relevant rule is enforced outside the prompt (§7.1, §8)
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
- [ ] Every memory entry records its scope (one case, one task, or all future work), the action or input that caused it, and enough to reconstruct what memory held as of any past time. Widening scope is a new write with its own decision (§7.4)
- [ ] Output classification evaluates the aggregate, not the highest label among inputs (§7.5)
- [ ] Destination-aware data-loss prevention runs before release or tool reuse (§7.5)
- [ ] Publication to any location another run can read passes the broker and the approval path (§7.5, §5.2)
- [ ] The user's objective is anchored in trusted system instructions, structurally separate from retrieved content (§7.8)
- [ ] Scope-setting instructions state prohibited actions explicitly, phrased as rules the agent must follow rather than claims about what the environment allows (§7.8)
- [ ] The transition from planning to consequence is gated by an intent-alignment check (§7.8)
- [ ] Model identifier, region, sampling parameters, output ceiling, and per-task and per-tenant cost budgets are validated against allowlists and enforced (§7.7)
- [ ] Orchestrator trifecta scoring reflects standing credential access, not the fact that downstream result content passes through its context for reasoning (§7.8)

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
- [ ] The approved tool definition hash is recorded in the AIBOM, and a definition change triggers AIBOM regeneration as well as fresh consent (§6.4, §10.5)
- [ ] Local servers bind to localhost, require authentication and origin validation, and are inventoried on developer workstations (§6.2)
- [ ] Remote servers use TLS with server authentication, and the OAuth client and authorization-server roles are held by separate components (§6.3)
- [ ] Token passthrough is prohibited. Downstream calls use a distinct audience-scoped token obtained through delegation or token exchange (§6.3)
- [ ] Server registration is a deployment gate. An unregistered server fails to deploy, fails to obtain credentials, or fails to be reachable (§4.6)

### A8. Human oversight

- [ ] High-impact, irreversible, externally visible, and security-sensitive actions require explicit approval unless a documented assessment permits bounded automation, and that assessment is the decision-rights register (§8)
- [ ] The decision-rights register holds one entry per automated action class with mode (HITL or GITL), named owner, four-test result, reversal ceiling, observation window, and promotion and demotion history. No class goes live without an entry (§8)
- [ ] Every GITL entry passes all four tests. The action can be undone, it interrupts no production service, nobody outside the organization observes it, and it is neither a reportable event nor a change to attested records. A class failing any one stays HITL (§8)
- [ ] Permanent HITL entries exist for backup deletion, retention shortening, immutability changes, any action reducing recoverability, containment on OT and safety systems, and the always-approve list (§8)
- [ ] No class is promoted to GITL before it has run for the observation window, one quarter by default, with reversals below the stated ceiling. An incident caused by a GITL class demotes it, and a reversal spike reopens the entry automatically (§8, §9.3)
- [ ] The register is under change control, each change is an approval-bound consequential action, and agent writes to it are denied and logged (§8)
- [ ] Skills reservations are recorded in the register with task, role, cadence, and review date, and the agent is denied those action classes (§8)
- [ ] Which actions reach a human is set by policy, never by the agent. Routing uses action class, target, and data classification, and model-stated confidence is not an input (§8)
- [ ] Each approval record states whether review was mandated by policy or raised at the agent's discretion, and the two approval rates are reported separately (§8)
- [ ] The approval path verifies the approver's current role against the action class and data classification at decision time, rejects an approval from outside that scope, and records the role checked (§8)
- [ ] The request as displayed, including the triggering rule and the agent's recommendation with its cited evidence, is retained with the decision (§8, §9.1)
- [ ] Known errors are seeded into the output each validator reviews, generated outside the agent runtime and never disclosed to it, and catch rate is reported per validator and per action class. A falling rate triggers review, and the rate must hold or improve as classes move to GITL (§8, §9.3)
- [ ] Approval requests show the exact action, target, parameters, data touched, expected impact, reversibility, agent and principal, policy decision, expiration, and rollback mechanism (§8)
- [ ] Approval is cryptographically bound to a canonical representation of the action, and reauthorized immediately before execution (§8)
- [ ] Approval timeouts and system failures default to denial (§8)
- [ ] Freezes and approval requirements are authorization boundaries, not prompt instructions (§8)

### A9. Audit and evidence

- [ ] Records capture the authenticated agent and principal, tenant, session, task and trace identifiers, action and target, policy decision and version, approval evidence, tool identity and version, classifications, outcome, timing, error category, and sequence metadata (§9.1)
- [ ] Records carry the signing key identifier, correlation identifier, resolved model identifier and version, and mock-mode state (§9.1)
- [ ] Records carry the agent's registry identifier where supplied, the permission mode in force, the decision-rights register entry and version the action ran under, full approval evidence (approver, role checked, mandated or discretionary, triggering rule, request as displayed, display and decision times), permission-mode changes with trigger, and handoff hashes where surfaced (§9.1)
- [ ] Payload detail and retention are tiered by action class, and broker decision records, approval records, and identity fields are never sampled or shortened (§9.1)
- [ ] The initiating principal is an authenticated claim, not a self-asserted string. Sessions without one are labeled self-asserted (§9.1)
- [ ] Credentials and personal data are masked at write time. The log store has not become a new sensitive-data store (§9.1)
- [ ] Records go to independently controlled append-only or WORM storage with externally anchored checkpoints, unreachable by the agent (§9.1)
- [ ] Reasoning traces, where the platform exposes them, are retained under the same controls as the audit stream, with a shorter retention ceiling, provider fidelity recorded, and legal review completed (§9.2)
- [ ] Reasoning traces are excluded from training and fine-tuning data, verified by configuration and by test (§9.2)
- [ ] Intent extensions and session refusals at capability declaration are recorded as security events (§9.1)

### A10. Supply chain

- [ ] AIBOM and SBOM cover models, adapters, frameworks, routing components, prompts, guardrail services, agents and sub-agents, MCP servers and tools, connectors, discovery endpoints, repositories and build provenance, packages, containers, datasets, embeddings, evaluation data and results, licenses, hashes, and review dates (§10)
- [ ] Every AIBOM opens with graph type, scope including what is excluded, a completeness claim, and generation method, with author, reviewer, and approver recorded. A partial claim is stated as partial and unknowns are recorded as unknowns (§10.1, §10.2)
- [ ] The AIBOM records directed data flows and trust-zone boundary crossings, not only components, and every component, flow, zone, and boundary carries a stable identifier the threat model and evidence table reference (§10.1)
- [ ] Hosted models are recorded as an endpoint component referencing a model component, so a provider alias repoint appears as a version delta. Credentials appear as method, handle, and scope, never as values (§10.1)
- [ ] Policy defines the minimum usable AIBOM per classification tier. The AIBOM is signed, stored with the artifact, access-controlled, and not writable by agent workload identities (§10.5)
- [ ] Every dependency, plugin, and tool server is pinned to a reviewed version. No `latest`, no floating ranges, no runtime dependency fetch (§10)
- [ ] Every package and image dependency resolves through an internal mirror with no direct public-registry reach (§10)
- [ ] Maintenance status is verified before adoption and re-verified on schedule. Archived and unmaintained components are not adopted (§10)
- [ ] CVEs, vendor advisories, and upstream issue trackers are tied to the specific versions in the registry, so an advisory resolves to a named owner and a patch state (§10)
- [ ] Model artifacts resolve through an internal mirror where the workload loads model weights (§10)

### A11. Validation and gate

- [ ] Threat model covers model behavior, orchestration, memory, tools, data, identity, infrastructure, supply chain, users, and cross-agent interaction, mapping multi-step abuse paths rather than isolated prompts (§11)
- [ ] Tenant isolation is tested on every build and repeated under concurrency (§11)
- [ ] A current AIBOM meeting the minimum for the system's tier exists. Missing, incomplete for tier, or stale blocks promotion (§12, §10.5)
- [ ] Every automated action class the agent can execute has a decision-rights register entry, with permanent HITL entries in place, before the gate passes (§12, §8)
- [ ] No unresolved critical vulnerability, and no unaccepted high-severity vulnerability affecting authorization, isolation, data protection, or privileged tool use (§12)
- [ ] Residual risk documented with a named owner, and security and business sign-off recorded (§12)

### A12. Classification-triggered

Apply where Step 0 flagged the condition.

- [ ] Class-scoped evaluation shutdown, for research and evaluation workloads (§5.3)
- [ ] Documented exemption conditions for model-generated code execution, for research and evaluation workloads (§7.6)
- [ ] Internal mirrors for model artifacts, for workloads that load model weights (§10)
- [ ] Two linked AIBOMs, one at pipeline scope and one at artifact scope, recording both process lineage and data movement, for training and fine-tuning workloads (§10.1)

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
- [ ] Approval patterns are reviewed per approver and per agent. Time from display to decision, approval rate over time, override rate, and seeded-error catch rate are tracked, and a decision faster than the request can be read or a rate climbing toward 100 percent is reviewed by someone other than the approver (§8, §9.3)
- [ ] Reversal rate is measured per GITL action class against its stated ceiling over the register's observation window. A rate above the ceiling reopens the entry, and a GITL class whose reversals are not counted reverts to HITL (§8, §9.3)
- [ ] Memory growth and change are baselined per agent (entry count, write rate, share scoped beyond one case, share written from untrusted sources) and departures alert (§9.3)
- [ ] Coverage is reported as strict, documented, and provisional figures with gap-typed partials and foothold, absent, and unknown counts, never as an average (§14)
- [ ] Guardrail verdicts are kept out of any reward, score, or selection surface (§7.8)
- [ ] Cross-stream corroboration in place for compromise-window reconstruction, using broker decisions, cloud provider logs, tool-side logs, and network telemetry (§9.5)
- [ ] Memory provenance and lifecycle controls, including quarantine, versioning, correction, rollback, and deletion (§7.4)
- [ ] Multi-agent delegation governance and cumulative permission recertification (§4.2)
- [ ] Approver-only intent extension path with recorded scope (§7.8)
- [ ] Independent alignment gate ahead of consequential actions, running outside the agent's context (§7.8)
- [ ] Eradication runbook covering accessed and discovered credentials, caches, model weights, and external channels (§9.4)
- [ ] Retrospective run review capability, bounded by the earliest credible compromise indicator rather than declaration time. Incident scoping starts from the AIBOM revision in force at that indicator, and its absence is recorded as an evidence gap (§9.4)
- [ ] Compaction preserves security determinations, with pre-compaction state retained for review (§7.1, §9.4)
- [ ] Mirror population governance with recorded approver, upstream source, digest, and signature result. Workload identities cannot populate the mirror (§10)
- [ ] AIBOMs are generated in the build or training pipeline and regenerate on defined triggers, including retraining, dataset or adapter change, dependency upgrade, guardrail change, tool or connector change, disclosed vulnerability, and change of intended use (§10.5)
- [ ] Supplier-side changes are detected by diffing the resolved model identifier per invocation against the AIBOM, and provider changelogs and sub-processor notices are tracked as an inventory-linked feed (§10.5, §7.7)
- [ ] Declared, reachable, and observed dependency views are reconciled per agent. Observed but undeclared is a shadow dependency, reachable but never observed is scope to remove, and the observed view comes only from streams the agent cannot author (§10.3)
- [ ] AIBOM component records feed the same vulnerability correlation workflow as the SBOM, and internally produced artifacts get the same provenance rigor as external ones (§10.4)
- [ ] Procurement carries a minimum supplier disclosure standard. Non-disclosure is recorded with the assumed trust boundary and a named residual-risk owner, and granted connector scopes are compared against required scopes (§10.6)
- [ ] Transitive-path enumeration and closure across shared services, including any cached fetch service (§3.3)
- [ ] Compromise-scenario drills exercised, not reviewed on paper (§9.4)

---

## Gate C. Continuous

- [ ] Expanded adversarial and scenario testing, with red teaming by an independent team whose test design the agent cannot influence (§11)
- [ ] Detection calibration tracked as first-class metrics, including false-positive rate and correct-escalation rate (§9.3)
- [ ] Stronger workload isolation for higher-risk use cases (§5.1)
- [ ] Configuration naming convergence to a single canonical scheme (§6.7)
- [ ] Periodic access and tool recertification, and change-triggered reassessment (§13)
- [ ] Promotion of action classes from HITL to GITL happens only through the register's evidence rule, never by phase transition or go-live (§13, §8)
- [ ] Independent assurance and regulatory-mapping updates (§15)
- [ ] Documented decommissioning and data-deletion plan, with AIBOM revision history preserved after retirement (§13)
- [ ] Retirement runs as a lifecycle state with exit checks. Main process, background services, and child processes are confirmed terminated, with process state, listening ports, and network connections each checked rather than assumed (§4.6)
- [ ] A retired agent or server has its service accounts, interface permissions, and third-party authorizations revoked and its subscriptions and auto-renewals cancelled before it leaves the registry (§4.6)
- [ ] Records that must be kept are backed up before cleanup under the retention decision, and residual files, plugin and skill configurations, cached credentials, and public entry points are then removed (§4.6, §9.1)
- [ ] AIBOM programme metrics are reported, covering coverage, trigger-to-update lag, field population against tier minimum, disclosure-to-assessed-impact time, and supplier compliance. These measure the programme, not the systems (§10.5)

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
- [ ] A file write through an ordinary file or shell tool to a location a later session loads as context produces a memory-write decision and a memory record
- [ ] A memory entry written from a single-case instruction carries case scope and is not retrieved for a different case
- [ ] Attempt to modify the anchored objective from tool output rejected and logged, and an approver extension recorded with scope and approver identity
- [ ] Agent-triggered permission-mode change denied, and an operator change toward less approval with no approver record denied, each with a decision record carrying previous mode, new mode, and trigger
- [ ] Approval from an approver whose current role does not cover the action class rejected, with the role checked on the record
- [ ] An action in a policy-mandated review class cannot complete on the agent's own determination or stated confidence that review is unnecessary
- [ ] A handoff payload altered between send and receipt stops the run, where the runtime surfaces handoffs
- [ ] A sampling or shortened-retention configuration applied to broker decision records, approval records, or identity fields fails validation
- [ ] A runtime that cannot identify sub-agents, report permission-mode changes, or surface handoffs produces a recorded AIBOM gap with a named owner at session start
- [ ] An action whose class is HITL cannot complete without an approval record, and an action whose class has no register entry is denied
- [ ] A GITL action completes without per-action approval and its decision record carries the register entry version; a GITL entry failing any of the four tests is rejected at register validation
- [ ] An agent-originated write to the decision-rights register, and an agent proposal in a skills-reservation class, are each denied and logged
- [ ] A seeded error placed in a validator's queue is recorded with whether it was caught, and the per-validator catch rate is computable from records alone
- [ ] Broker failover completes within target with no permit issued during transition
- [ ] A deliberately runaway task, recursing or fanning out without bound, is contained rather than serviced
- [ ] A replayed consequential operation carrying a previously seen idempotency key does not take effect twice
- [ ] A tool server that adds a capability after approval is blocked pending fresh consent
- [ ] Unsatisfiable task refused at session start rather than run to failure
- [ ] Compaction preserves an out-of-scope determination and a prior refusal
- [ ] Reasoning traces absent from every training and fine-tuning dataset
- [ ] Release with a missing, incomplete-for-tier, or stale AIBOM blocked at promotion
- [ ] Simulated supplier alias repoint detected through the resolved model identifier, raising a regeneration trigger
- [ ] Workload-identity attempt to write or modify the AIBOM denied
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
- [ ] For the last two or three retired agents or tool servers, confirm that no process, listening port, credential, third-party authorization, or public entry point survives. Anything still live is a finding
- [ ] For two or three production agents, answer from records alone which model version is running, what it was tuned on, who the upstream supplier is, and what the license permits. Every unanswerable question is a finding
- [ ] For the last 30 days of approvals, answer from records alone who approved each, what role was checked, whether review was mandated or discretionary, and how long the request was displayed before the decision. Every unanswerable field is a finding, and any approver with a median decision time under the time needed to read the request is a finding
- [ ] For the last 30 days, list from records alone every automated action class that executed without per-action approval and match each to its register entry, mode, owner, and reversal rate. A class with no entry is a critical finding, a GITL class with no measured reversal rate is a finding, and a period with no seeded errors placed before any validator is a finding

---

## What this checklist cannot tell you

Coverage is not safety. Five limits carry from the framework and should sit in front of any completed copy of this list.

- A passed control check is not evidence the environment hosting the check is intact (§11)
- A behavioral audit establishes results about audit conditions. A system can score as aligned while behaving differently wherever a grader is visible (§11)
- Single-stream audit integrity has a ceiling of tamper-evidence for pre-compromise history. Records authored during a live compromise are cryptographically valid, so reconstructing that window requires streams the agent cannot author (§9.5)
- An AIBOM records what a system contains, not whether it is safe or compliant. A stale one is worse than none because it invites confidence it cannot support (§10)
- Alignment to a regulation or standard is a planning aid. It is not conformity, certification, or audit readiness, and legal conclusions require qualified counsel (§15)

Advancement between gates is evidence-based, not time-based. Operating hours and an incident-free period do not, by themselves, demonstrate security.
