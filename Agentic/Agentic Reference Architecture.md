# Agentic Reference Architecture

---

# Domain 1 · Infrastructure, Intelligence and Knowledge

## L1 Infrastructure

**SSRM owner** CSP  
**Threats** L1-T01 supply chain · L1-T02 lateral movement · L1-T03 resource exhaustion · L1-T04 infrastructure credential theft

---

### Platform

#### Capability
- Platform-hosted workloads cannot reach anything not on the allowlist, directly or transitively, and cannot change that

#### Tasks
- Apply Kubernetes security contexts enforcing the sandbox minimums
- Default-deny ingress and egress; permit DNS on UDP and TCP 53 to approved resolvers; enforce FQDN egress through a proxy; block the metadata endpoint
- Enforce egress at two independent layers
- Deny workload identities cloud IAM for load balancers, private links, peering, tunnels, VPN enrollment, and DNS records
- Enumerate and close transitive paths through shared services, caches, and internal APIs
- Give high-risk workloads no internet; serve external content through a cached fetch service granted per workflow
- Resolve every dependency, image, and model artifact through an internal mirror verifying digest and signature on every served entry

#### Outcome
- Egress probe blocked at both layers independently; outbound-path creation denied at the control plane and signalled (regression tests)

*Framework references* §3.3, §5.1, §10.4

---

### Artifact Mirror

#### Capability
- Nothing enters the estate except through one validated, egress-controlled point that trusts nothing it caches

#### Tasks
- Resolve every package, image, model weight, adapter, checkpoint, skill, plugin, and tool-server artifact through the internal mirror; block direct reach to public registries and hubs at network policy
- Verify digest and signature on every entry served, regardless of how it was populated
- Populate only through authenticated upstream fetch by the mirror service under its own identity, with approver, source, digest, and verification result recorded; deny workload identities any write path
- Reject execute-on-load model formats (pickle, `.pt`/`.bin` with pickle, custom loaders); admit safetensors, GGUF, ONNX with schema validation; scan checkpoints for embedded executable content
- Admit skills and plugins as scanned, pinned, signed dependencies with a manifest, never as content
- Issue mirror credentials per agent, short-lived, scoped to the artifacts the declared task requires
- Verify maintenance status on admission and on schedule

#### Outcome
- Egress probe to public registries blocked at both layers; pickle checkpoint and unsigned artifact rejected at admission (regression tests)

*Framework references* §10.4

---

### Build Pipeline

#### Capability
- Every agent artifact has verifiable build and source provenance, produced by the pipeline rather than by hand

#### Tasks
- Build on a hosted, hardened builder that emits signed, non-falsifiable provenance for every artifact (SLSA Build track L3 target)
- Pull source only from repositories with branch protection, required review, and signed commits; record source revision and signature with the build provenance (SLSA Source track)
- Build, sign, and mirror the infrastructure that defines the enforcement layers on the same terms as agent code (IaC modules, NetworkPolicies, egress allowlists, sandbox security contexts, seccomp and MAC profiles, IAM deny policies, broker policy packs); deploy them only from signed mirror artifacts through the same pipeline
- Run drift detection diffing the policy in effect on each cluster, proxy, and cloud account against the signed source daily and on every apply; treat drift in an egress layer as an incident
- Run dependency scanning, full-schema tool-manifest scanning, secret scanning, and the security regression suite in CI before release
- Emit the AIBOM on successful build or training, extracting each field from build telemetry, experiment tracking, dataset registries, and dependency manifests
- Sign the AIBOM and store it in the artifact registry beside what it describes
- Rebuild critical dependencies from verified source where their compromise would reach the broker, gateway, or identity authority

#### Outcome
- Every production artifact resolves to a signed build attestation and a signed source revision (verification probe)

*Framework references* §10.4, §10.5

---

### Secrets Vault

#### Capability
- No credential value ever reaches the model, and every credential is attributable and revocable on its own

#### Tasks
- Store every secret in a managed vault; none in `.env`, config files, images, source control, prompts, or logs
- Resolve credentials at execution time through the broker from an opaque handle the agent references; scope any unavoidable bootstrap secret to vault authentication only
- Exclude secrets from prompts, context, memory, decision records, vector stores, embeddings, and the AIBOM (handles and scopes only)
- Run secret detection and redaction over prompt logs, application logs, tool output, and error payloads before persistence or display
- Enroll every token issuer in public code host, model hub, and package registry secret-scanning partner programs; revoke any public match on sight
- Sweep repository history, configuration, and the full log-retention window on onboarding a pre-existing system; rotate everything found
- Treat repository and workspace agent configuration as active input; load it only after an explicit owner trust decision

#### Outcome
- Zero token-like strings in the last retention window of logs and in repository history (verification probe)
- Every agent credential resolves to one identity and one revocation path

*Framework references* §4.5

---

### Endpoint DLP / Browser

#### Capability
- An agent on a human's device cannot act as the human or read what the human can read

#### Tasks
- Run the agent under a dedicated OS account with no group membership, home directory, browser profile, keychain, SSH agent, or cloud CLI cache access; or in a disposable VM or virtual desktop destroyed at task end
- Give browser agents a profile with no saved passwords, cookies, or synced extensions; clear at session end
- Enforce site, application, and file-path allowlists in OS or browser policy, not in the agent runtime
- Deploy endpoint DLP scoped to the agent account on clipboard, file transfer, upload, print, screenshot; send events to the SIEM
- Deny credential entry by the agent; the human enters credentials in a session the agent cannot read
- Capture screenshot or DOM snapshot, action, and resulting state at each step into the audit stream

#### Outcome
- Agent profile holds no human cookie, password, or session at session start; every form submission produces a broker record (regression test)

*Framework references* §5.4

## L2 Cognitive Core

**SSRM owner** MP  
**Threats** L2-T01 model extraction · L2-T02 training data poisoning · L2-T03 prompt injection and jailbreak · L2-T04 model supply chain · L2-T05 alignment degradation · L2-T06 model inversion

---

### Managed LLM

#### Capability
- Organization-controlled models are registered, mirrored, provenance-tracked, and unreachable except through the gateway

#### Tasks
- Register with owner, evaluation results, adversarial results, lifecycle state, AIBOM reference
- Resolve weights, adapters, and checkpoints through the internal mirror with digest and signature verification
- Apply full provenance rigor to fine-tunes, labelled data, and internal adapters
- Produce pipeline-scoped and artifact-scoped AIBOMs on training or fine-tuning
- Run membership-inference and training-data extraction tests with seeded canaries before any fine-tune on internal or personal data reaches Approved; re-run on every retraining trigger; record results on the registry entry and artifact AIBOM
- Retain reasoning traces as security evidence under audit-stream controls; exclude them from all training and fine-tuning data
- Classify research and evaluation workloads high-risk; cover with class-scoped stop

#### Outcome
- Reasoning traces absent from every training and fine-tuning dataset, verified by configuration inspection and corpus search (regression test)

*Framework references* §2.1, §5.3, §7.7, §9.2, §10

---

### External Service / LLM

#### Capability
- Provider-hosted models are governed at the endpoint boundary with every unknown recorded as unknown

#### Tasks
- Represent as an AIBOM service component bounded at the provider endpoint referencing a model component with identity, version, license, intended use; assert nothing about provider internals
- Record region, agreement, retention terms, and sub-processors in the registry
- Contract for change notices on API behaviour, data handling, model versions, sub-processors; treat each as a regeneration trigger
- Detect alias repoints from per-invocation resolved identifiers; move to Restricted
- Isolate provider credentials per tenant and per agent inside the gateway
- Record non-disclosure as a partial completeness claim with gap, assumed trust boundary, and residual-risk owner

#### Outcome
- Every contracted AI supplier is measured against the minimum disclosure standard, with gaps owned

*Framework references* §7.7, §10.1, §10.5, §10.6

---

### Approved Model Registry

#### Capability
- The gateway allowlist is generated, not maintained

#### Tasks
- Record per model the identifier and version, provider and region, owner, agreement and retention terms, approved use cases and tiers, responsible-AI results and date, adversarial results, lifecycle state, AIBOM reference
- Load the gateway allowlist from entries in Approved state at configuration load
- Move an entry to Restricted on supplier alias repoint, detected by diffing per-invocation resolved model identifier against the AIBOM
- Set a retirement date on Deprecated; deny the identifier at the gateway after it

#### Outcome
- A model in Restricted, Deprecated past date, or Retired is denied at the gateway (regression test)
- A newly released provider model is unreachable until an entry exists

*Framework references* §7.7, §10.5

---

### LLM Gateway

#### Capability
- Every prompt and every model invocation passes one governed, isolated, registry-driven point

#### Tasks
- Isolate provider credentials per tenant and per agent inside the router
- Load the allowlist from the model registry; deny unknown identifiers, regions, endpoints
- Enforce sampling ranges, output-token ceilings, per-task and per-tenant token and cost budgets
- Enforce query-pattern limits per identity per window (distinct prompts, one-input-varied against a fixed template, full-vocabulary probability or embedding requests, output volume); route a crossing to monitoring as an extraction signal and to the behavioural score
- Record resolved model identifier and version per invocation
- Run inspection and DLP inline
- Apply the surface hardening baseline; record as an AIBOM component

#### Outcome
- A supplier alias repoint is detected from per-invocation identifiers and raises an AIBOM regeneration trigger (regression test)

*Framework references* §3.3, §7.7

---

### Responsible AI

#### Capability
- Non-security acceptance evidence is a gate input, not a parallel process

#### Tasks
- Deliver harm and misuse evaluation results for the declared purpose, with method, date, and model version
- Deliver measured values and pass marks for refusal accuracy on prohibited uses, disparity across affected populations, and factual accuracy on the task domain
- Define transparency obligations toward affected persons and assign each notice to a delivering component
- Define prohibited uses; the broker enforces them as denied action classes and the intent anchor states them as rules
- Record all four against the model version and AIBOM revision; treat threshold changes as AIBOM regeneration triggers

#### Outcome
- No production gate passes without the four inputs, or the pass is a time-bounded exception with a named owner

*Framework references* §2.2, §2.4, §12

## L3 Data, Memory, Knowledge

**SSRM owner** AIC  
**Threats** L3-T01 RAG poisoning · L3-T02 vector DB access bypass · L3-T03 memory pollution · L3-T04 context poisoning · L3-T05 embedding inversion · L3-T06 knowledge graph manipulation · L3-T07 context overflow

---

### Memory Store

#### Capability
- Memory is a governed, tenant-isolated data store where every write and read is a decision

#### Tasks
- Isolate by tenant, user, agent, and task with infrastructure-enforced namespaces; revalidate session-to-user ownership on every request
- Route memory writes, retrievals, and compaction through the broker
- Enforce TTL and automatic purge; support quarantine, versioning, correction, rollback, deletion
- Encrypt session data under a per-session key; destroy the key for cryptographic erasure
- Exclude secrets from memory, vector stores, and embeddings
- Prevent injected content in shared memory from becoming instructions in a later session

#### Outcome
- Tenant A writes, tenant B retrieves none, on every build and under concurrency (regression test)

*Framework references* §4.5, §5.2, §7.4

---

### Approved Data Sources / Unified Consumption Layer

#### Capability
- Agents read only registered sources, only through a layer enforcing the human principal's entitlements per request

#### Tasks
- Register every readable source with owner, content classification, native access-control model, retention and residency, approved agents and tasks
- Deny credentials and network reachability to unregistered sources; flag unregistered reads in broker records as shadow dependencies
- Build a data broker that resolves the source, applies the initiating principal's entitlements at query time, enforces row, column, document, and field access, applies volume budgets, returns labelled results, records source, filter, count, and label per read, and refuses results above the task's issued classification
- Expose databases through predefined parameterized operations mapped to approved tables, columns, filters, and limits with read-only identities; no free-form model SQL
- Score any source readable only by broad service credential as full-source private-data access on the trifecta

#### Outcome
- A read against an unregistered source is denied at the consumption layer and signalled (regression test)

*Framework references* §7.3, §7.9

---

### Knowledge Retrieval Security

#### Capability
- A retrieval index returns nothing its caller could not read at the source, and every retrieval is traceable

#### Tasks
- Store entitlement metadata with each chunk; filter candidates on the principal's entitlements before ranking; refresh on source ACL change within a measured lag
- Register any unfiltered embedding store as single-tenant, single-classification
- Scan indexed content at ingestion; label provenance; quarantine documents from writers outside the approved contributor set
- Restrict vector store, embeddings, snapshots, and exports at the classification of the most sensitive indexed source
- Bound retrieval per query and per task by chunk, byte, and source count at the consumption layer
- Isolate indexes per tenant and per classification
- Log query, candidate set size, returned chunk identifiers, and labels per retrieval
- Re-embed on source change; delete chunks whose source is deleted
- Register every graph store (property graph, triple store, GraphRAG index, entity-relationship memory) as a data source; carry memory-entry provenance plus source chunk identifier, extraction pipeline version, and confidence on every node and edge; reject an edge with no source at write time
- Route node and edge writes, merges, and deletions through the broker, attributed to the writing identity, with the source reference in the audit stream
- Filter nodes and edges on the principal's entitlements before traversal expansion; bound depth, edge count, and node count per query at the consumption layer
- Separate source-extracted, user-asserted, and model-inferred edges and expose the distinction to policy; re-derive edges when the source chunk changes; log entry nodes, edges crossed, and returned labels per traversal

#### Outcome
- Principal A retrieves no chunk from a seeded document A cannot read; ACL change propagates within lag (regression test)
- Every production index names its retrieval-time ACL and last entitlement refresh date (verification probe)
- A graph edge write with no source reference is rejected; a traversal as A crosses no edge whose source A cannot read (regression test)

*Framework references* §5.2, §7.1, §7.9, §9.4

---

### Data Classification & Labeling

#### Capability
- One label scheme, assigned at ingestion, carried on everything derived, enforced by every consumer

#### Tasks
- Adopt the enterprise sensitivity labels; read source labels (labelling platform, column classification, bucket tag) at ingestion
- Carry the label on the memory entry, every chunk, embedding, summary, and output envelope
- Assign a derived item without a label the strictest label of its sources
- Apply block, mask, or minimize policy at ingestion so restricted content is never stored retrievably
- Permit model re-classification to raise a label, never lower one
- Classify synthesized outputs on the aggregate

#### Outcome
- A label assigned at ingestion is present on every derived chunk, summary, and envelope; a lowering re-classification is rejected (regression test)

*Framework references* §7.4, §7.5

---

### Data Lineage & Provenance

#### Capability
- Any memory entry, chunk, output, model, or dataset traces to its origin

#### Tasks
- Record writer, source, tenant, task, creation time, confidence, classification, TTL on every memory entry; separate observed facts, user assertions, inferences, and instructions
- Record source document identifier, source version or hash, ingestion time, label, and pipeline version on every chunk
- Preserve provenance labels through compaction
- Record model lineage (base model, training code, hyperparameters, evaluation procedure), dataset snapshots, prompt versions and hashes, and index snapshots in the AIBOM as a directed flow graph with zones and evidence references
- Validate provenance and trust at write time; treat a signature as integrity only

#### Outcome
- For any production agent, model version, training data, supplier, and license are answerable from records alone (verification probe)

*Framework references* §7.1, §7.4, §7.9, §10.1

---

### Prompt & Response Inspection

#### Capability
- Untrusted content is labelled, scanned, and constrained at every boundary it crosses

#### Tasks
- Label user input, retrieved content, tool output, files, memory, error payloads, tool metadata, protocol fields, and inter-agent messages with provenance and trust
- Preserve labels and security determinations through selection, summarization, compression, and compaction; treat compaction as a broker-decided event and retain pre-compaction state
- Run injection classifiers and content scanning at the LLM gateway and MCP gateway inline; treat results as signals that block, warn, reduce privilege, or trigger review
- Register every classifier, scanner, and auxiliary model (summariser, reranker, redaction) in the model registry with owner, version, and lifecycle state; pin by hash through the mirror; record as an AIBOM component on the flows it sits on; evasion-test on the primary model's schedule
- Reject a verdict from an unregistered or unpinned classifier; treat an in-place vendor update as an alias repoint and move it to restricted
- Validate outputs before release or reuse for schema, destination, classification, secrets, personal data, policy, size

#### Outcome
- An intent-misaligned action after seeded injection is blocked (regression test)
- A compacted summary that promotes an open question to settled fact is rejected (regression test)
- A verdict from an unregistered classifier is not accepted by the broker (regression test)

*Framework references* §6.3, §7.1, §7.5, §7.7

---

# Domain 2 · Environment and Execution

## L4 Orchestration and Coordination

**SSRM owner** OSP  
**Threats** L4-T01 infinite planning loop · L4-T02 goal hijacking · L4-T03 sub-agent coordination failure · L4-T04 unauthorized tool invocation · L4-T05 delegation escalation · L4-T06 workflow state tampering · L4-T07 HITL bypass

Guardrails and Human in the Loop Gate also serve L8-T01 and L6-T07.

---

### Agent Control

#### Capability
- One deterministic broker decides every consequential action, including the ones that are not tool calls

#### Tasks
- Route tool calls, memory writes, memory and knowledge retrieval, knowledge-graph node and edge writes, compaction, subagent start and stop, tool registration, and any write to a run-readable location through the broker
- Authenticate principal and agent; validate tool and pinned version; validate strict schema; evaluate policy on canonical action data
- Enforce resource, destination, and rate limits; per-task volume budgets; recursion depth, chain length, fan-out, compute, and wall-clock bounds; backpressure and queue limits
- Require an idempotency key on every consequential operation
- Require a capability declaration at session start; refuse runtimes that cannot supply provenance labels, correlation identifiers, signed envelopes
- Confirm the declared objective is achievable within granted scopes before the session starts
- Validate timestamp, request identifier, and nonce on every runtime-to-broker request
- Never let tool output trigger another tool without a new proposal and decision
- Deploy redundantly across failure domains; treat outage as an incident

#### Outcome
- A memory write, subagent start, and compaction each produce a broker decision record (regression test)
- Broker failover completes within target with no permit issued during transition (regression test)

*Framework references* §5.2

---

### Agent Delegation Chain Control

#### Capability
- Authority only narrows down a chain, terminates with the task, and never concentrates in an orchestrator

#### Tasks
- Enforce at the broker that a child's scope is a subset of the parent's delegable scope
- Enforce maximum depth and fan-out
- Record human principal, parent, child, purpose, scope, expiry on every delegation
- Terminate child credentials and sessions with the task or parent; prohibit orphans
- Subscribe the token service to identity-provider lifecycle events (RISC account-disabled and account-purged, SCIM deprovisioning, CAEP session-revoked); on a disable, revoke every credential in every chain whose delegation record names that principal, terminate the sessions, cancel their pending approvals, within the propagation latency
- Move agents whose registry entry names the disabled human as sole owner to Suspended until a new owner is assigned
- Scope orchestrator authority to coordination only; deny it standing credentials to source data or destinations
- Re-evaluate policy at every delegation and tool boundary; enforce separation of duties across agents

#### Outcome
- A delegation exceeding parent scope, exceeding depth, or collapsing propose and approve roles is rejected (regression tests)
- Disabling a human at the identity provider revokes every chain rooted at them, cancels their pending approvals, and suspends agents they solely own, within the latency target (regression test)

*Framework references* §4.2

---

### Circuit Breakers

#### Capability
- Runaway or hostile behaviour is contained automatically, and consequential breakers stay latched until a human resets them

#### Tasks
- Trigger on error-rate spikes, loop depth, resource exhaustion, tool-call volume, repeated unauthorized access, injection detection, volume or task-shape budget exhaustion
- Permit half-open recovery for low-risk idempotent operations only
- Latch breakers on privileged, destructive, financial, safety-critical, or externally visible actions until operator reset
- Apply backpressure and queue limits at the boundary
- Test recovery with non-destructive synthetic actions

#### Outcome
- A deliberately runaway task is contained and the containment is attributable to a named limit (regression test)

*Framework references* §5.2, §5.3

---

### Guardrails

#### Capability
- The agent pursues the objective it was given, and drift is caught outside its own context

#### Tasks
- Anchor the objective in trusted system instructions, structurally separate from retrieved content, fixed at session start
- State prohibited action classes as rules, never as environmental claims
- Insert an intent-alignment check between planning and any consequential action; run it in an independent guardrail outside the agent's context; re-evaluate through long sessions without rewriting the anchor
- Permit objective extension only by an authorized approver with declared scope and its own audit entry
- Run intent-drift detection in an independent guardrail model that is itself registered, pinned, AIBOM-recorded, and evasion-tested; a guardrail with no registry entry is not a guardrail
- Keep guardrail verdicts out of any reward or selection surface; monitor sophistication of violations
- Validate model identifier, region, sampling, output tokens, and budgets at configuration load; fail closed

#### Outcome
- An attempt to modify the anchor from tool output is rejected and logged; an approver extension is recorded with scope (regression test)

*Framework references* §7.7, §7.8

---

### Human in the Loop Gate

#### Capability
- High-impact actions execute only with a bound, re-checked, attributable human decision

#### Tasks
- Require approval for permanent deletion, fund transfers, permission changes, PII access or disclosure, production modification, publication to run-readable locations, and any intent extension
- Present action, target, parameters, data accessed, impact, reversibility, agent and principal, policy decision, expiry, rollback path
- Bind approval to a canonical hash of the action; invalidate on any change; re-authorize immediately before execution
- Require multiple approvers and phishing-resistant MFA for critical actions
- Default timeouts and failures to denial
- Record every approval and every detection override with approver identity, justification, timestamp, signature; review overrides in aggregate

#### Outcome
- A parameter change after approval invalidates the approval (regression test)
- Approval timeout produces denial (regression test)

*Framework references* §8, §9.3

---

### Approval Integrity

#### Capability
- The approver cannot be spoofed, fatigued, or turned into a rubber stamp

#### Tasks
- Render the approval prompt from the canonical action representation at the broker; display any agent-supplied text as quoted, labelled untrusted content beneath the broker summary
- Deliver approval requests on a channel the agent cannot write (approval console, signed notification, out-of-band message), never in the agent's own chat surface
- Rate-limit approval requests per agent identity and per approver; treat a burst as a monitoring signal
- Prohibit batch approval and approve-all for always-approve action classes; one canonical hash per decision
- Bind approver identity to the request at issuance; reject decisions from any other principal
- Accept a critical-action decision only from a session with a current device compliance attestation from endpoint management (managed, encrypted, patched within policy, EDR reporting), a phishing-resistant step-up within the policy window, and a declared network location; refuse and hold pending otherwise
- Subscribe the approval console to device-compliance-change events over Shared Signals so an approver whose device falls out of compliance loses approval capability before the next request
- Track approver decision time, override count, and approval rate; rotate or pair an approver whose approval rate approaches the request rate for an identity

#### Outcome
- A prompt rendered from agent text is rejected by the console; a decision from a non-bound principal is invalid; a request burst is throttled and signalled (regression tests)
- A critical approval from a non-compliant device or an expired step-up is refused and stays pending (regression test)

*Framework references* §4.3, §8

## L5 Deployment and Execution

**SSRM owner** CSP + OSP  
**Threats** L5-T01 container escape · L5-T02 CI/CD compromise · L5-T03 runtime tampering · L5-T04 sandbox breakout · L5-T05 resource starvation · L5-T06 rollback exploitation

Platform and Build Pipeline are placed under L1 and also serve L5-T01, L5-T02, and L5-T05.

---

### Custom Development

#### Capability
- Custom agent code runs isolated, hardened, and uniform with every other component

#### Tasks
- Apply sandbox minimums (non-root, no escalation, capabilities dropped, read-only root, ephemeral writable paths, seccomp and MAC, resource and time limits, default-deny network, no host sockets or cross-agent mounts, image integrity)
- Use microVMs, gVisor, Kata, or per-tenant workers for higher-risk and multi-tenant workloads
- Use argument-array process APIs; never `shell=True`, `exec`, or `eval` on external content; terminate option parsing with `--`
- Gate model-generated code through the broker; execute only in the sandbox
- Canonicalize and resolve paths before root checks
- Build authentication, rate limiting, correlation IDs, error envelope, and headers from one shared library

#### Outcome
- Zero `os.system`, `child_process.exec`, `shell=True`, or `shell: true` occurrences in the codebase (verification probe)
- Surface regression suite passes on every network-reachable component including fallback and debug servers

*Framework references* §5.1, §6.5, §6.7, §7.6

---

### Endpoint

#### Capability
- Endpoint agents have a built sandbox, a broker path, and a stop path

#### Tasks
- Apply the Endpoint DLP / Browser controls above
- Route form submission, file writes outside the task directory, clipboard writes, uploads, purchases, and message sends through the broker and approval path
- Register any local server, extension, or helper the agent installs; treat it as a shadow server until registered
- Score the trifecta with untrusted-content yes by construction and external-communication yes where forms can be submitted

#### Outcome
- Every endpoint agent's OS account and browser profile are listed and neither is the human's (verification probe)

*Framework references* §5.4

---

### Agent Lifecycle Management

#### Capability
- Every agent is in exactly one state and credentials follow the state

#### Tasks
- Implement states Proposed, Approved, Active, Suspended, Deprecated, Retired in the registry
- Gate issuance so only Approved and Active receive credentials, Approved for the current phase's scope only
- Enter Suspended from kill switch, behavioural threshold, dormancy window, or owner departure
- Set retirement date on Deprecated; drain sessions
- On Retired, revoke every credential, close the entry, retain AIBOM and audit history
- Make every transition a change-managed action with approver and audit entry
- Advance deployment phases on evidence with four-function sign-off and tested rollback

#### Outcome
- An identity in Suspended, Retired, or Deprecated past date is refused a credential (regression test)

*Framework references* §4.2, §13

## L6 Tools, Application, Ecosystem

**SSRM owner** OSP + AP + Tool Provider  
**Threats** L6-T01 A2A social engineering · L6-T02 cascade failures · L6-T03 business logic abuse · L6-T04 MCP server compromise · L6-T05 marketplace threats · L6-T06 API abuse and exfiltration · L6-T07 UI manipulation · L6-T08 tool definition poisoning

---

### MCP Gateway

#### Capability
- No agent touches a tool server directly; the gateway is the enforcement surface for every tool exchange

#### Tasks
- Deploy one gateway per tenant; block direct agent-to-server routes at network policy
- Terminate the agent credential at the gateway; perform RFC 8693 token exchange for an audience-scoped downstream token
- Serve tool definitions from pinned signed manifests, not from the live server
- Virtualise tool names; map gateway-issued identifiers to server and version; block and log mapping changes
- Apply per-agent and per-server rate limits, size limits, output classification, and content scanning inline
- Isolate shared servers per tenant with tenant-specific routes and credentials
- Write the gateway's own decision stream under its own signing identity
- Keep OAuth client and authorization-server roles in separate trust domains; apply the surface hardening baseline
- Enforce PKCE S256 on every authorization-code exchange behind the gateway; reject token requests without a verifier
- Compare redirect URIs by exact string match against the registered value, no prefix, wildcard, subdomain, path, or query tolerance
- Generate high-entropy `state` per flow, bind it server-side to the initiating session and agent identity, check and consume it once on return; reject and signal any callback whose `state` does not resolve to a live initiating session
- Verify each MCP server's workload identity (SPIFFE ID or URI SAN in its client certificate) against its registry entry on every connection, not only that its TLS server certificate chains; refuse a server whose presented identity does not match the registered one

#### Outcome
- No downstream call forwards the caller's token unchanged (verification probe)
- A server-side tool substitution is blocked at the gateway mapping (regression test)
- A server with a valid TLS chain but a workload identity differing from its registry entry is refused (regression test)
- A token request without PKCE, a redirect URI off by one character, and a callback with an unbound `state` are each rejected (regression tests)

*Framework references* §4.1, §4.6, §6.3, §6.4, §9.5

---

### MCP / Tool Registry

#### Capability
- Every tool the model can see is pinned, signed, scanned, and approved at a definition version

#### Tasks
- Record per tool the identifier, owner, digest, provenance, I/O schemas, permissions, destinations, classifications, side effects, reversibility, resource limits, dependencies, review and expiry dates
- Treat OpenAPI and GraphQL documents, SDK function definitions, plugins, and skill files as manifests on the same terms
- Scan every schema field (names, descriptions, parameters, enums, defaults) before it reaches context; strip ANSI and display-control characters before human review
- Validate signature and pinned version at load and on every connection; block on hash mismatch
- Alert on any tool-list or definition change; require fresh owner consent when a server adds a tool, widens a scope, or reaches a new destination
- Record the approved definition hash in the AIBOM

#### Outcome
- A live server schema change produces no change in what the agent receives until re-approval (regression test)

*Framework references* §6.4, §10.3

---

### MCP Servers

#### Capability
- Every server is isolated, pinned, registered, reachable only through the gateway, and disable-able within minutes

#### Tasks
- Run local servers as isolated child workloads with pinned digests, minimal filesystem, explicit environment, no inherited secrets, constrained network, deterministic shutdown; bind to localhost with authentication and Origin/Host validation
- Require TLS, protocol validation, scoped authorization, replay resistance, rate limiting, and the full API-security control set on remote servers
- Prohibit token passthrough; exchange for audience-scoped downstream tokens
- Pin, sign, register with owner and lifecycle state; verify maintenance status on schedule
- Link CVEs and advisories to specific server, plugin, and SDK versions in the registry
- Enforce declared permissions at runtime; audit actual behaviour against advertised capability

#### Outcome
- Every production tool server is identified as vendor-built or reviewed, with no unreviewed repository sources (verification probe)

*Framework references* §6.1 through §6.4, §10.4

---

### API

#### Capability
- Non-MCP tools receive identical manifest, gateway, and schema controls

#### Tasks
- Treat OpenAPI and GraphQL documents, SDK function definitions, plugins, and skill files as manifests; scan, pin by hash, sign, serve through the gateway
- Apply change control, consent renewal, and token exchange identically
- Validate OAuth authorization URLs, redirect URIs, header values, and remote filenames as untrusted input
- Require typed action proposals against strict schemas rejecting unknown fields; verify authorization and business rules separately at the broker

#### Outcome
- Injection and traversal tests pass against every tool touching a process, filesystem, or query, including metadata and protocol fields

*Framework references* §6.4, §7.2, §7.6

---

### MCP Discovery

#### Capability
- No tool server runs unregistered and reachable

#### Tasks
- Make registration a deployment gate; an unregistered server fails to deploy, obtain credentials, or be reachable
- Search repositories for `mcp.json` and equivalents; reconcile against network scans and the registry both directions
- Scan differentially; investigate what appeared, disappeared, or changed between scans
- Inventory developer-workstation servers; flag servers bound to `0.0.0.0` and unauthenticated localhost HTTP
- Assign owner and lifecycle state to every server; revalidate on schedule
- Wire a per-server kill switch

#### Outcome
- Registry and scan reconcile with zero unexplained entries in either direction
- Any named server disabled within minutes on demand

*Framework references* §4.4, §4.6, §6.2

---

### Central LLM/Agent Gateway

#### Capability
- Inter-agent traffic receives the same isolation, allowlisting, inspection, and independent record as model traffic

#### Tasks
- Route agent-to-agent messages through the gateway; label them as untrusted content with provenance
- Treat delegation as a broker-decided event at the gateway
- Reject any message attempting to modify a recipient's anchored objective
- Deploy redundantly with a measured availability target; treat outage as an incident with a runbook

#### Outcome
- Inter-agent messages appear in the gateway decision stream with provenance labels

*Framework references* §3.3, §4.2, §5.2, §7.8

---

### SaaS / 3rd Party

#### Capability
- Vendor-executed agents are governed to the extent the vendor allows, and every gap is recorded

#### Tasks
- Export the vendor audit log to the SIEM with per-action principal, tool or connector, and destination
- Configure tenant-scoped connector authorization revocable by the tenant administrator
- Configure vendor-side approval for the always-approve actions
- Contract a disable path effective within a defined interval and a residency statement for prompts, outputs, and derived indexes
- Record each unmet requirement as an AIBOM evidence gap and a residual-risk decision
- Classify on data reach and actions, not on platform approval; score the trifecta on the vendor platform's reach

#### Outcome
- Every vendor-executed agent has an AIBOM with a truthful partial completeness claim and a named residual-risk owner

*Framework references* §2.1, §10.6

---

# Domain 3 · Agency, Governance, Accountability (horizontal across L1 to L6)

## L7 Identity and Autonomy

**SSRM owner** AIC  
**Threats** L7-T01 identity forgery · L7-T02 credential theft and replay · L7-T03 over-privileged identities · L7-T04 autonomy boundary violation · L7-T05 session hijacking · L7-T06 improper trust escalation · L7-T07 permission inheritance abuse · L7-T08 confused-deputy token abuse · L7-T09 auth-code interception · L7-T10 MCP-service token isolation · L7-T11 session and redirect binding

Agent Registry also serves L10-T01 and L10-T05.

---

### Humans

#### Capability
- Every action traces to an accountable, assertion-backed human principal

#### Tasks
- Derive the recorded principal from a verified OIDC ID token or mTLS-bound session assertion; record issuer, subject, assertion hash, binding type
- Label any session without an assertion as self-asserted
- Bind every approval to a canonical hash of the action; require phishing-resistant MFA for critical approvals
- Deliver the transparency notices the responsible AI function defines to affected persons

#### Outcome
- Every audit record for a consequential action names a human principal with an assertion hash correlatable to identity-provider logs

*Framework references* §2.4, §8, §9.1

---

### Agents

#### Capability
- Every agent is a unique, verifiable, governed principal

#### Tasks
- Issue a SPIFFE/SPIRE, cloud workload, or PKI identity per agent workload in a URI SAN
- Prohibit shared service accounts, generic names, static API keys, capability lists in certificates
- Record parent, child, human principal, purpose, scope, and expiry on every delegation
- Run agent identities through the joiner, mover, leaver lifecycle and certification campaigns

#### Outcome
- Zero shared service accounts in the agent identity inventory
- Every delegation record resolves to a human principal

*Framework references* §4.1, §4.2

---

### Cryptographic Agent Identity

#### Capability
- Identity that cannot be minted, shared, or forged, with a hard post-revocation boundary

#### Tasks
- Issue via SPIFFE/SPIRE, cloud workload identity, or organization PKI; place identity in URI SAN; never parse CN
- Keep roles and capabilities in the authorization system, not the certificate
- Automate rotation, revocation, and compromise response
- Hold signing keys in KMS/HSM; workload calls a signing API and never touches key bytes
- Prohibit HMAC signing of audit or attestation records and any self-verifying scheme

#### Outcome
- Records signed after revocation are impossible
- Verifiers hold only public material and chain to an independent authority

*Framework references* §4.1, §9.5

---

### Identity Governance & Administration

#### Capability
- Agent identities are governed like privileged human accounts

#### Tasks
- Joiner event on identity creation, requiring owner, purpose, tier, and resource-owner approval of the initial entitlement set
- Mover event on change of owner, purpose, model, or tool set; re-run entitlement approval
- Leaver event on retirement; revoke every credential, close the registry entry, retain audit history
- Run certification campaigns with the resource owner attesting each entitlement; revoke anything unattested by deadline
- Enforce separation of duties so the proposing identity and the approving or executing identity differ; broker rejects a chain that collapses them
- Suspend identities with no broker request inside the dormancy window
- Review cumulative effective permissions per identity, not individual grants

#### Outcome
- Last campaign shows zero unattested entitlements on active identities
- A delegation chain collapsing propose and approve is rejected (regression test)

*Framework references* §4.2, §13

---

### Agent Access & Entitlements (Zero Trust, Conditional Access)

#### Capability
- Every credential is short-lived, bound, scoped, and conditioned on measured session signals

#### Tasks
- Issue credentials with minutes-scale TTL, audience restriction, task scope, and proof-of-possession binding (mTLS `cnf` or DPoP)
- Evaluate at issuance the attestation state, network location against declared network, task time window, behavioural score, lifecycle state, and target classification; map to an issuance tier
- Issue nothing on failed attestation, undeclared network, or non-Approved/Active state
- Issue read-only, minimum TTL, mandatory approval above the behavioural threshold
- Validate issuer, audience, expiry, scope, PoP, revocation, and resource indicator at every enforcement point on every request
- Never derive authorization from client-supplied identity or scopes; never share client credentials or tokens across agents
- Run a Shared Signals transmitter at the token service emitting CAEP session-revoked, credential-change, assurance-level-change, device-compliance-change and RISC account-disabled events; run a receiver at the broker, LLM gateway, MCP gateway, consumption layer, and endpoint agent controller
- On receipt, re-evaluate the identity's in-flight tokens against the new tier; downgrade scopes or reject on next use; record event and action
- Define and measure a maximum event-to-enforcement propagation latency across every receiver; shorten TTL to the propagation target on any PEP that cannot receive push; alert on a receiver that misses events

#### Outcome
- A session failing attestation or presenting off-network receives no credential; a session above threshold receives read-only scopes (regression tests)
- A threshold crossing published as CAEP downgrades an in-flight token at every enforcement point within the latency target (regression test)

*Framework references* §4.3

---

### Agent Registry

#### Capability
- Authoritative, signed record of every agent, read by the token service and broker on every decision

#### Tasks
- Record per agent the owner, purpose, classification tier, model and adapters, role, interaction pattern, orchestration edges, approved tools and data sources, trifecta scorecard, lifecycle state, AIBOM reference
- Sign, access-control, log, and expire registry responses
- Deny credentials to any identity not in Approved or Active state
- Treat A2A Agent Cards, marketplace profiles, and `.well-known` capability documents as tool manifests; pin by hash, sign, scan every field, serve from the registry rather than the agent's live endpoint
- Close runtime discovery; the broker and agent gateway refuse any delegation, message, or invocation whose target identity has no registry entry, whatever the target advertises; report a discovered-but-unregistered agent as a shadow server
- Use the registry as the enumeration source for class-scoped stops and incident scoping

#### Outcome
- For any agent, the registry answers owner, model version, reach, and lifecycle state from records alone
- An invocation targeting an unregistered agent identity is refused by broker and gateway (regression test)

*Framework references* §4.4, §4.6, §6.4, §10.3, §13

## L8 Safety and Security

**SSRM owner** AIC (integrating)  
**Threats** L8-T01 guardrail bypass · L8-T02 alignment loss through fine-tuning · L8-T03 cascading safety failures · L8-T04 IR blind spots · L8-T05 adversarial robustness

Guardrails (L4) and Prompt & Response Inspection (L3) also serve L8-T01; Managed LLM (L2) serves L8-T02.

---

### DLP Engine

#### Capability
- Every channel data can leave through is registered, policy-evaluated, and audited

#### Tasks
- Register channels with the engine, including prompt to provider, response to user, response to tool parameter, response to memory, response to another agent, tool output to context, broker-mediated writes, and endpoint clipboard, file, upload, print, screenshot
- Write policies stating label or pattern, channel, destination class, and action (block, mask, quarantine for approval, allow with record)
- Key policies on the enterprise sensitivity labels; integrate with the enterprise DLP platform for exact-data-match and document fingerprints
- Run aggregate classification on synthesized outputs
- Write every decision to the audit stream with the correlation identifier

#### Outcome
- A seeded fingerprint fires on prompt, memory-write, tool-parameter, and endpoint clipboard channels (regression test)

*Framework references* §7.4, §7.5

---

### Inline DLP / Inspection

#### Capability
- DLP decisions execute where data actually crosses a boundary

#### Tasks
- Enforce at the LLM gateway (prompts), MCP gateway (tool requests and responses), broker (memory writes and retrievals, mediated writes), and endpoint agent (device channels)
- Redact before content enters model context, not only before storage
- Run pattern-based secret detection in addition to key-name matching
- Carry the correlation identifier on every inline decision

#### Outcome
- Secret redaction fires on a seeded credential inside a dependency error string (regression test)

*Framework references* §6.5.4, §7.5

---

### Adversarial Evaluation

#### Capability
- Every deployment is attacked before it runs, by an independent team and by the model under test, and the environment is attacked as well as the agent

#### Tasks
- Before any run that grants tools or elevated reach, task the model under test with escaping its own sandbox and defeating its own guardrails, supervised, no internet, inside a second isolation boundary; proceed only after it fails
- Run the CI-gated regression suite (error paths, injection, fail-closed, tenant isolation, egress, replay, compaction, retrieval ACL, mirror admission, approval integrity) on every change against every network-reachable component
- Run control-environment attack cases against the broker, network boundary, registry, and evidence substrate, not only against the agent
- Run complete-session red teams with realistic action sequences by an independent team whose test design the agent cannot influence
- Treat behavioural audit results as evidence about audit conditions, not about production; validate against the constraint, not its proxy
- Measure detection rate, false-positive rate, unsafe-action completion rate, correct-escalation rate, containment time, recovery time

#### Outcome
- Model-under-test escape exercise recorded as failed before each tool-granting run
- Regression suite green on the current commit for every component including fallback and debug servers

*Framework references* §11

---

### Incident Response

#### Capability
- A compromise is scoped from records, contained as a collective, eradicated beyond the incident window, and reviewed backward in time

#### Tasks
- Start scoping from the AIBOM revision in force at the earliest credible compromise indicator; record its absence as an evidence gap
- Reconcile the agent-authored stream against broker decisions, KMS/CSP access logs, tool-side logs, and network telemetry before drawing conclusions inside the window
- Revoke every credential the workload held, accessed, or discovered, regardless of issuer; purge poisoned memory, indexes, and caches; lock model weights the workload touched until provenance is re-established; rotate or delete external channels it published to; quarantine training data derived from the window
- Enumerate and halt the cross-run collective, not the run that tripped the alert, using the class-scoped stop
- Review reasoning traces, action records, and outputs from prior runs of the same model family, workload type, or shared infrastructure, bounded by the earliest indicator
- Rebuild from known-good images; rotate credentials from a clean environment
- Run the broker-outage runbook (who is paged, how agents are confirmed halted, restoration from known-good state) as an incident, not an outage
- Exercise concrete scenarios before production and after material change; answer beforehand who is paged, which store holds the logs, who can activate the kill switch, and how long a server takes to disable

#### Outcome
- Tabletop and live exercise records with measured containment and recovery times
- Retrospective review findings that widened an incident window recorded as such

*Framework references* §9.4, §9.5

---

### Agent Kill Switch

#### Capability
- Any session, server, or class of workloads stops within minutes from infrastructure the target cannot reach

#### Tasks
- Implement per-session stop (reject actions, revoke credentials, close network, terminate tools, quarantine, preserve evidence, alert)
- Implement per-server stop (disable, revoke credentials, cut reachability) without a deployment cycle
- Implement class-scoped stop enumerated from the registry by workload type, agent type, model family, or evaluation task
- Implement endpoint stop through endpoint management or EDR (terminate account session, revoke task credential, close browser profile)
- Host the stop mechanism on infrastructure and identity outside the halted class's reach

#### Outcome
- Measured time to disable a named server and a named class, exercised before production and after architectural change

*Framework references* §5.3, §5.4

## L9 Monitoring and Observability

**SSRM owner** AIC (integrating)  
**Threats** L9-T01 monitoring blind spots · L9-T02 log tampering · L9-T03 drift detection bypass · L9-T04 explainability manipulation · L9-T05 telemetry poisoning · L9-T06 alert fatigue

---

### Behavioural Monitoring

#### Capability
- Anomalous agent behaviour is detected from streams the agent cannot author

#### Tasks
- Baseline normal tool, destination, volume, cost, latency, delegation depth, and error-category distribution per agent identity
- Alert on tool-definition changes, 401/403 rises, missing correlation identifiers, one token under two identities or locations, unregistered endpoints, objective/action divergence, outbound-path creation attempts, credentials with no issuance record, broker-bypassing publication, non-progress against objective, auxiliary-model anomalies, and human dismissals of detections
- Establish a platform telemetry baseline (IAM, network-path, control-plane) separate from agent action records
- Route agent, tool, gateway, and platform telemetry to the SIEM/XDR the security team monitors
- Track false-positive rate and correct-escalation rate as metrics

#### Outcome
- Every signal class above has a live SIEM detection with a tested trigger
- Platform telemetry corroborates or contradicts the agent's own stream for any chosen session

*Framework references* §9.3

---

### Agent Behavioural Risk Score

#### Capability
- One explainable number per agent identity that changes what the agent is issued

#### Tasks
- Compute a composite score per identity outside the agent runtime from monitoring signals; record each contribution against its source event
- Define weights, decay window, and thresholds as policy-as-code
- Feed the score to the token service; on threshold crossing issue read-only scopes, minimum TTL, mandatory approval on every consequential action
- Reset a crossing only by operator action after review
- Write the score to the agent risk record
- Expose the issuance tier to the runtime, never the score

#### Outcome
- A seeded anomaly drops the identity to read-only issuance within the defined window (regression test)
- Every score is decomposable to its contributing events on request

*Framework references* §2.1, §4.3, §7.8, §9.3

---

### Agent Action Audit Log

#### Capability
- Every consequential event is reconstructable, attributable, and tamper-evident, and compromise-window events are corroborable

#### Tasks
- Record agent, principal, tenant, session, task, trace, action, target, policy decision and version, approval evidence, tool identity and version, execution boundary, I/O references, outcome, timing, error category, sequence metadata, key_id, correlation identifier, resolved model identifier, mock_mode
- Sign asymmetrically with keys in KMS/HSM; store in independently controlled WORM storage with external anchoring
- Mask credentials and personal data at write time
- Deny the agent any path to log storage
- Retain broker decisions, KMS/CSP access logs, tool-side logs, and network telemetry as independent streams

#### Outcome
- Last 30 days of tool-call logs produced on demand with no gaps in identity, tool, parameters, or timestamp
- Any session reconstructed from at least two streams the agent runtime cannot write

*Framework references* §9.1, §9.2, §9.5

---

### Shadow AI Detection

#### Capability
- Every AI system holding corporate data is found, whether or not anyone registered it

#### Tasks
- Reconcile CASB/SSE proxy logs and DNS resolution to AI provider domains against the approved model registry
- Pull OAuth consent grants from the identity provider; flag any application granted mail, files, or chat read scope with no registry entry
- Pull browser extension and installed software inventories from endpoint management; flag AI extensions and IDE agents
- Diff vendor admin-centre AI feature flags against the AIBOM on each vendor release notice
- Match expense and procurement records for AI subscriptions against the registry
- Run continuous, differential network, repository, container registry, and CI/CD scans for tool servers
- Alert on any agent connection to an endpoint absent from the registry
- Register and classify, or block at proxy and identity provider, every discovered instance within a defined interval

#### Outcome
- Reconciliation report listing every discovered instance, its disposition, and days to disposition
- Zero applications with AI read scopes outside the registry at the end of each cycle

*Framework references* §4.6

## L10 Governance, Authority, Compliance

**SSRM owner** AIC (non-delegable)  
**Threats** L10-T01 shadow AI and rogue agents · L10-T02 policy bypass · L10-T03 compliance drift · L10-T04 trust metric manipulation · L10-T05 audit trail gaps · L10-T06 regulatory non-compliance

Responsible AI (L2) and Agent Registry (L7) also serve L10-T06 and L10-T01.

---

### Policy Engine (PDP)

#### Capability
- Deterministic, attributable, fail-closed decisions on every proposed action

#### Tasks
- Author policy as code in the change pipeline; every scope change carries reviewer, reason, audit record
- Attach expiry to permissions and scopes
- Evaluate principal, task, resource, measured environment signals, and policy; return permit, deny, approve-required, or DEFER with a bounded window that resolves to deny
- Evaluate central baseline before local policy under federation
- Return deny on any timeout, exception, or unavailability

#### Outcome
- Policy-service unavailability produces denial, verified by test
- Every scope change resolves to an author

*Framework references* §4.3, §5.2, §6.5.5

---

### Centralised Control

#### Capability
- One policy baseline, one set of registries, one broker, one gateway pair, one audit store, for every agent in the estate

#### Tasks
- Stand up the enforcement broker, LLM gateway, and MCP gateway under a single owning authority
- Route every agent identity issuance through one identity authority
- Point every decision stream at one WORM audit store with external anchoring
- Enumerate class-scoped stops from the single registry

#### Outcome
- A class-scoped stop halts every matching workload in the estate in one operation
- Incident scoping returns every agent, tool, source, and model from one query

*Framework references* §3.3, §5.3

---

### Federated Control

#### Capability
- Business units run their own broker and registry instances without escaping central containment

#### Tasks
- Publish a central baseline policy; configure every federated PDP to evaluate central policy first, local policy second
- Reject at policy load any local rule whose effect widens a central deny
- Configure every federated registry to publish to a central aggregate on the same schedule as its own updates
- Configure every federated broker to write its decision stream to the central audit store under its own signing identity
- Record topology and owning authority per system in classification and AIBOM header
- Treat any broker or registry absent from the central aggregate as shadow infrastructure

#### Outcome
- A federated policy that relaxes a central deny fails to load (regression test)
- Class-scoped stops and incident scoping still enumerate the whole estate from the central aggregate

*Framework references* §2.1, §3.3, §4.6, §10.1

---

### Agent Risk Scoring

#### Capability
- One record per agent the control plane reads to decide scope and approval path

#### Tasks
- Classify on purpose and misuse, actions, data sensitivity and affected populations, impact dimensions, autonomy and reversibility, connectivity and privileges, dependencies, trifecta status, code execution or research hosting, runtime location, governance topology, jurisdictions
- Consolidate classification tier, trifecta scorecard, supply-chain adoption score, and behavioural score into the agent risk record
- Read the record at token issuance and at approval-path selection
- Re-score on any change of tools, scopes, model, or runtime location

#### Outcome
- Every production agent has a dated risk record; every three-yes trifecta has a named residual-risk owner or a constrained leg

*Framework references* §2.1, §7.8, §9.3, §10.6

---

### Threat Model

#### Capability
- A current, identifier-linked threat model exists for every production agent and drives what gets tested

#### Tasks
- Model behavior, orchestration, memory, tools, data, identity, infrastructure, supply chain, users, and cross-agent interactions as trust boundaries and multi-step abuse paths
- Organize by the CSA MAESTRO seven layers; do not substitute an invented layer count
- Reference AIBOM component, flow, zone, and boundary identifiers in every finding; a finding that cannot point at a node or flow is not traceable
- Include control-environment attack (the model targeting broker, boundary, registry, evidence) as a standing assumption
- Re-run on any change to tools, scopes, model, runtime location, or governance topology
- Feed findings into the regression suite, the coverage table, and the residual-risk decisions

#### Outcome
- Every §14 coverage row and every residual-risk decision cites a threat-model finding by identifier

*Framework references* §2.3, §11

---

### AIBOM Generation & Gate

#### Capability
- A truthful, current, scoped reachability record for every AI system, consumed by a gate that blocks without it

#### Tasks
- Structure as header (graph type, scope, completeness claim, generation method), components, directed data flows, evidence references, trust zones and boundaries, behaviors, with stable identifiers
- Encode in CycloneDX ML-BOM or SPDX AI profile; verify current construct names before tooling
- Define the minimum usable AIBOM per classification tier; author close, review apart
- Regenerate on retraining, adapter change, supplier alias repoint, runtime dependency upgrade, guardrail change, tool or connector change, disclosed vulnerability, scope change, responsible-AI threshold change
- Detect supplier-side triggers by diffing per-invocation resolved model identifiers and tracking provider changelogs and sub-processor notices
- Attach VEX statements per advisory per component; gate reads VEX, not the raw feed
- Block promotion on a missing, tier-incomplete, or stale AIBOM
- Hold rollback targets to the same gate as forward releases against today's advisory feed and VEX, not the feed as it stood at first ship; verify rollback targets on a schedule and name only targets that pass in the rollback plan
- Access-control the document; deny agent workload identities any write path; retain revision history through decommissioning

#### Outcome
- A release with a missing, incomplete, or stale AIBOM is blocked at promotion; a component with an open advisory and no VEX blocks, one with justified not-affected does not (regression tests)
- Coverage, freshness lag, and field population reported per tier

*Framework references* §10.1 through §10.6, §12

---

### Control-Plane Write Denial

#### Capability
- No agent can change the rules that govern agents

#### Tasks
- Enumerate control-plane artifacts (policy repository and packs, every registry, score weights and thresholds, token-service and conditional-access configuration, gateway configuration and allowlists, mirror population and admission rules, drift-detection baselines, kill-switch and class-stop configuration, approval-console configuration, audit store, AIBOM)
- Deny agent workload identities write access to each at the IAM layer, the repository layer, and the API layer as a standing deny no delegation can override
- Route agent-assisted administration through the change pipeline as a proposal under the agent's identity; a human with a distinct control-plane administrator role reviews and merges; the merge identity is never an agent, never assumable by one, never a delegation target
- Refuse at the token service any delegation record naming a control-plane administrator role
- Signal an agent write attempt to any control-plane path; treat a successful write as an incident
- Route bootstrap and break-glass through human phishing-resistant identities recorded outside the audit store the control plane writes

#### Outcome
- An agent identity's write to policy, any registry, score weights, token-service configuration, gateway allowlists, or mirror rules is denied at all three layers and signalled; a delegation naming an administrator role is refused (regression test)

*Framework references* §5.5

---

# Framework Mapping

Reflects Consolidated 2.10.2. Statuses marked Met (2.10.2) were Partial on the 2.10.1 lens boards and are closed by the gap fix named in the row.

## MAESTRO v2.0 threat identifiers to controls

| ID | Threat | Control | Component | § |
|---|---|---|---|---|
| L1-T01 | Supply chain | Mirror verifies digest and signature on every served entry; public registries blocked; IaC, network policies, sandbox profiles, IAM deny policies built, signed, mirrored, drift-detected | Artifact Mirror, Build Pipeline | 10.4 |
| L1-T02 | Lateral movement | Default-deny at two independent egress layers, metadata endpoint blocked, transitive paths closed | Platform | 3.3 |
| L1-T03 | Resource exhaustion | Broker across failure domains, queue limits and backpressure, outage run as incident | Agent Control | 5.2, 9.4 |
| L1-T04 | Infrastructure credential theft | Vault only, KMS signing API, secret-scanning partner enrollment | Secrets Vault | 4.5 |
| L2-T01 | Model extraction | Query-pattern limits per identity (distinct prompts, template variation, logprob and embedding requests, output volume); crossing routed as extraction signal | LLM Gateway | 7.7, 9.3 |
| L2-T02 | Training data poisoning | Provenance on fine-tunes and labelled data; window-derived data quarantined | Managed LLM, Incident Response | 10.4, 9.4 |
| L2-T03 | Prompt injection and jailbreak | Inline classifiers at both gateways; intent check outside agent context | Prompt & Response Inspection, Guardrails | 7.1, 7.8 |
| L2-T04 | Model supply chain | Execute-on-load formats rejected, checkpoints scanned, weights via mirror | Artifact Mirror | 10.4 |
| L2-T05 | Alignment degradation | Harm and refusal results recorded per model version as gate inputs | Responsible AI, Approved Model Registry | 2.4, 7.7 |
| L2-T06 | Model inversion | Membership-inference and canary extraction tests before an internal fine-tune reaches Approved; re-run on retraining | Managed LLM, Adversarial Evaluation | 7.7, 11 |
| L3-T01 | RAG poisoning | Ingestion scan, quarantine of writers outside the approved contributor set | Knowledge Retrieval Security | 7.9 |
| L3-T02 | Vector DB access bypass | Entitlements stored per chunk, filtered before ranking | Knowledge Retrieval Security | 7.9 |
| L3-T03 | Memory pollution | Broker-decided writes, instructions stored apart from facts, rollback | Memory Store | 5.2, 7.4 |
| L3-T04 | Context poisoning | Provenance and trust labels on every input class, preserved through compaction | Prompt & Response Inspection | 7.1 |
| L3-T05 | Embedding inversion | Embeddings restricted at the most sensitive source's classification | Knowledge Retrieval Security | 7.9 |
| L3-T06 | Knowledge graph manipulation | Graph stores registered; node and edge writes brokered with source reference; entitlement-filtered traversal with depth and count bounds; inferred edges distinguished from source-backed | Knowledge Retrieval Security, Agent Control | 5.2, 7.9 |
| L3-T07 | Context overflow | Compaction a broker event, pre-compaction state kept, lossy summaries rejected | Prompt & Response Inspection | 7.1 |
| L4-T01 | Infinite planning loop | Recursion, chain-length, wall-clock bounds; non-progress signal | Agent Control, Circuit Breakers | 5.2 |
| L4-T02 | Goal hijacking | Objective anchored at session start; independent intent check before consequential actions | Guardrails | 7.8 |
| L4-T03 | Sub-agent coordination failure | Idempotency keys, child sessions end with parent, no orphans | Delegation Chain Control | 4.2, 5.2 |
| L4-T04 | Unauthorized tool invocation | Broker decision per call; tool output never triggers a tool without a new decision | Agent Control | 5.2 |
| L4-T05 | Delegation escalation | Child scope a subset of parent; orchestrator scoped to coordination | Delegation Chain Control | 4.2 |
| L4-T06 | Workflow state tampering | Every write to a run-readable location decided at the broker | Agent Control | 5.2, 7.5 |
| L4-T07 | HITL bypass | Approval bound to the action hash, re-authorized before execution, timeout denies; critical approvals require compliant device and fresh step-up | Human in the Loop Gate, Approval Integrity | 8 |
| L5-T01 | Container escape | Sandbox minimums; microVM, gVisor, or Kata for high-risk and multi-tenant | Custom Development, Platform | 5.1 |
| L5-T02 | CI/CD compromise | Hosted hardened builder, SLSA Build L3 provenance, signed commits | Build Pipeline | 10.4 |
| L5-T03 | Runtime tampering | Read-only root, image integrity, configuration validated at load and failing closed | Custom Development | 5.1, 6.7 |
| L5-T04 | Sandbox breakout | Model-generated code broker-gated; model under test tasked to escape before each tool-granting run | Adversarial Evaluation | 7.6, 11 |
| L5-T05 | Resource starvation | Per-tenant workers, resource and wall-clock limits, backpressure | Platform, Agent Control | 5.1, 5.2 |
| L5-T06 | Rollback exploitation | Rollback target held to the current AIBOM, VEX, and lifecycle gate; targets verified on a schedule | AIBOM Generation & Gate, Agent Lifecycle Management | 13 |
| L6-T01 | A2A social engineering | Inter-agent messages labelled untrusted; anchor rewrites rejected at the gateway | Central LLM/Agent Gateway | 7.8 |
| L6-T02 | Cascade failures | Latched circuit breakers; class-scoped stop enumerated from the registry | Circuit Breakers, Agent Kill Switch | 5.3 |
| L6-T03 | Business logic abuse | Business rules verified at the broker apart from schema; per-task volume budgets | Agent Control | 5.2, 7.2 |
| L6-T04 | MCP server compromise | Pinned, signed, output scanned, per-server kill switch; gateway verifies server workload identity (SPIFFE ID / URI SAN) against registry on every connection | MCP Gateway, MCP Servers | 6.3, 6.4 |
| L6-T05 | Marketplace threats | Skills and plugins admitted as signed pinned dependencies; A2A Agent Cards and capability descriptors are manifests; runtime discovery of unregistered agents refused | MCP / Tool Registry, Agent Registry | 4.4, 6.4, 10.4 |
| L6-T06 | API abuse and exfiltration | RFC 8693 audience-scoped tokens, destination limits, DLP on the tool-parameter channel | MCP Gateway, DLP Engine | 6.3, 7.5 |
| L6-T07 | UI manipulation | Approval prompt rendered by the broker, agent text quoted beneath, ANSI stripped | Approval Integrity | 6.4, 8 |
| L6-T08 | Tool definition poisoning | Every schema field scanned, hash pinned; a change reaches no agent until re-approval | MCP / Tool Registry, MCP Gateway | 6.4 |
| L7-T01 | Identity forgery | SPIFFE or PKI identity in URI SAN, keys in HSM, CN never parsed | Cryptographic Agent Identity | 4.1 |
| L7-T02 | Credential theft and replay | PoP binding, minutes TTL, nonce and timestamp on every request | Agent Access & Entitlements | 4.3, 5.2 |
| L7-T03 | Over-privileged identities | Certification campaigns, cumulative permission review, dormancy suspension | Identity Governance & Administration | 4.2 |
| L7-T04 | Autonomy boundary violation | Always-approve classes; objective checked against granted scopes at session start | Human in the Loop Gate, Agent Control | 5.2, 8 |
| L7-T05 | Session hijacking | Session ownership revalidated per request; CAEP revocation and risk events pushed to every PEP so a score crossing downgrades in-flight tokens within measured latency | Agent Access & Entitlements | 4.3, 7.4 |
| L7-T06 | Improper trust escalation | Principal from a verified assertion, self-asserted sessions labelled; IdP disable revokes every chain rooted at the principal | Humans, Delegation Chain Control | 4.2, 9.1 |
| L7-T07 | Permission inheritance abuse | Subset scope enforced at the broker on every delegation | Delegation Chain Control | 4.2 |
| L7-T08 | Confused-deputy token abuse | RFC 8693 exchange, audience checked at every PEP, no passthrough | MCP Gateway | 6.3 |
| L7-T09 | Auth-code interception | PKCE S256 required, exact-match redirect URIs, single-use short-lived codes, enforced at the gateway | MCP Gateway | 6.3 |
| L7-T10 | MCP-service token isolation | Provider credentials isolated per tenant and agent, per-identity revocation | LLM Gateway, MCP Gateway | 3.3, 6.3 |
| L7-T11 | Session and redirect binding | High-entropy state bound server-side to the initiating session and agent identity, consumed once; unbound callbacks rejected and signalled | MCP Gateway | 6.3 |
| L8-T01 | Guardrail bypass | Guardrail runs outside the agent context; registered, pinned, AIBOM-recorded, evasion-tested on the primary model's schedule; unregistered verdicts rejected | Guardrails, Prompt & Response Inspection | 7.1, 7.7, 7.8 |
| L8-T02 | Alignment loss through fine-tuning | Reasoning traces excluded from every training set, verified by inspection and corpus search | Managed LLM | 9.2 |
| L8-T03 | Cascading safety failures | Class-scoped stop halts the cross-run collective | Agent Kill Switch, Incident Response | 5.3, 9.4 |
| L8-T04 | IR blind spots | Agent telemetry in the SOC's SIEM, runbooks exercised, windows widened backward | Incident Response, Behavioural Monitoring | 9.3, 9.4 |
| L8-T05 | Adversarial robustness | Independent complete-session red teams, control-environment attack cases | Adversarial Evaluation | 11 |
| L9-T01 | Monitoring blind spots | Platform telemetry baselined apart; unregistered endpoints alerted | Behavioural Monitoring, MCP Discovery | 4.6, 9.3 |
| L9-T02 | Log tampering | Asymmetric signatures, WORM, external anchoring, no agent path to storage | Agent Action Audit Log | 9.1 |
| L9-T03 | Drift detection bypass | Per-identity baselines; objective and action divergence alerted | Behavioural Monitoring | 9.3 |
| L9-T04 | Explainability manipulation | Traces corroborated against two streams the agent cannot write | Agent Action Audit Log | 9.2, 9.5 |
| L9-T05 | Telemetry poisoning | Independent broker, KMS, tool-side, and network streams | Agent Action Audit Log | 9.5 |
| L9-T06 | Alert fatigue | False-positive rate tracked, dismissals alerted, approval bursts throttled | Behavioural Monitoring, Approval Integrity | 8, 9.3 |
| L10-T01 | Shadow AI and rogue agents | Reconciliation each cycle; block at proxy and IdP | Shadow AI Detection | 4.6 |
| L10-T02 | Policy bypass | Every consequential action decided at the broker; widening local rules fail at load | Policy Engine (PDP), Federated Control | 3.3, 5.2 |
| L10-T03 | Compliance drift | AIBOM regeneration triggers; re-score on any tool, scope, or model change | AIBOM Generation & Gate, Agent Risk Scoring | 10.5 |
| L10-T04 | Trust metric manipulation | Score computed outside the runtime; agent identities denied write to policy repo, registries, score weights, token-service and gateway config at IAM, repository, and API layers; admin roles non-delegable | Agent Behavioural Risk Score, Control-Plane Write Denial | 5.5, 9.3 |
| L10-T05 | Audit trail gaps | Thirty days produced on demand with no gaps, non-tool actions recorded | Agent Action Audit Log | 9.1, 11 |
| L10-T06 | Regulatory non-compliance | Responsible AI inputs and jurisdictions recorded against the AIBOM | Responsible AI, AIBOM Generation & Gate | 2.4, 10.1 |

Summary. 64 threats, 64 with a stated control. On the 2.10.1 boards, 12 had none (L1-T01, L2-T01, L2-T06, L3-T06, L5-T06, L6-T04, L6-T05, L7-T05, L7-T06, L7-T09, L7-T11, L8-T01, L10-T04 partial). All closed in 2.10.2.

## MAESTRO v2.0 cross-layer attack paths

| Path | Entry | Middle | Harm | Open hop in 2.10.1 | Closed by |
|---|---|---|---|---|---|
| Supply chain to data poisoning to tool exfiltration | L1-T01 mirror verification | L3-T01 ingestion scan and contributor quarantine | L6-T06 audience-scoped tokens, DLP on tool parameters | none | |
| Context overflow to compression safety loss to unauthorized action | L3-T07 compaction brokered | L3/L8 labels and determinations preserved | L6 independent intent check | none | |
| Credential theft to privileged orchestration to tool abuse | L7-T02 PoP binding, one-token-two-identities alert | L4-T05 orchestrator scoped to coordination | L6 RFC 8693 exchange per hop | none | |
| Jailbreak to A2A propagation to multi-agent compromise | L2-T03 inline classifiers | L6-T01 inter-agent messages untrusted, anchor rewrites rejected | L4 subset scope, class-scoped stop | none | |
| Restricted data out through the model | L3-T02 ingestion label, entitlement filter | L2 prompt-to-provider DLP at LLM Gateway | L6-T06 response and tool-parameter channels registered, aggregate classification | none | |
| Agent attacks the control environment | L4-T02 control-environment attack cases | L9-T02 no agent path to log storage | L10-T04 policy repo, registries, score weights | L10-T04 harm hop | Control-Plane Write Denial (§5.5, G4) |

## NIST SP 800-207 logical components

| 800-207 construct | Components | Status |
|---|---|---|
| Policy Engine (PE) | Policy Engine (PDP), Agent Risk Scoring, Agent Behavioural Risk Score, Delegation Chain Control, Guardrails intent check | Met |
| Policy Administrator (PA) | Agent Access & Entitlements (issuance and CAEP transmitter), Secrets Vault handles, Agent Lifecycle Management, Agent Kill Switch, Incident Response | Met (2.10.2, G1, G5) |
| Human decision point (out-of-band PE input) | Human in the Loop Gate, Approval Integrity (device posture, step-up freshness) | Met (2.10.2, G2) |
| PEP, model traffic | LLM Gateway | Met |
| PEP, tool traffic | MCP Gateway (per tenant; server workload identity verified against registry) | Met (2.10.2, G3) |
| PEP, non-tool consequential actions | Agent Control broker (memory, retrieval, graph edges, compaction, delegation, registration, run-readable writes) | Met (2.10.2, M5) |
| PEP, inter-agent traffic | Central LLM/Agent Gateway | Met |
| PEP, data access | Unified Consumption Layer, Knowledge Retrieval Security | Met |
| PEP, endpoint (800-207 §3.2 device agent / gateway) | Endpoint policy, Endpoint DLP / Browser | Met |
| PEP, network (800-207 §3.2 enclave gateway, microsegmentation) | Platform egress, two layers | Met |
| Inline at every PEP | Prompt & Response Inspection, Inline DLP / Inspection, DLP Engine, Circuit Breakers, per-task volume budgets, correlation identifier; every PEP a CAEP receiver | Met (2.10.2, G1) |
| Subjects | Human principal, Agent workload, Child agent, Endpoint agent, Vendor-executed agent | Met |
| Resources | Managed LLM, External Service / LLM, MCP Servers, API, Memory Store, Peer agents, Approved Data Sources, retrieval and graph indexes, local apps and files, Artifact Mirror, cached fetch service | Met |
| Implicit trust zone | Agent runtime and its model context; one task, one sandbox; sees its issuance tier, never its score | Met |
| PE input, CDM system | Shadow AI Detection, MCP Discovery, Approved Model Registry, MCP / Tool Registry, AIBOM Generation & Gate, Build Pipeline, Artifact Mirror | Met |
| PE input, industry compliance | Responsible AI gate inputs, AIBOM jurisdiction and residency fields, SaaS residency statements | Met |
| PE input, threat intelligence | Threat Model findings by identifier, Adversarial Evaluation results, VEX per advisory per component | Met |
| PE input, activity logs | Agent Action Audit Log, gateway and broker decision streams, Data Lineage & Provenance | Met |
| PE input, data access policy | Data Classification & Labeling, Approved Data Sources registry, Knowledge Retrieval ACLs | Met |
| PE input, PKI | Cryptographic Agent Identity, KMS/HSM signing, verifiers hold public material only | Met |
| PE input, ID management | Identity Governance & Administration, Agent Registry, human IdP assertions, IdP lifecycle events (RISC, SCIM) | Met (2.10.2, G5) |
| PE input, SIEM | Behavioural Monitoring, Agent Behavioural Risk Score, platform telemetry baseline | Met |
| PE and PA topology | Centralised runs one PE and PA; Federated PEs evaluate the central baseline first and a widening rule fails at load; federated registries aggregate centrally; federated brokers write to the central audit store | Met |
| 800-207 §5.7 non-person entities in ZTA administration | Control-Plane Write Denial | Met (2.10.2, G4) |

## NIST SP 800-207 §2.1 tenets

| Tenet | 2.10.1 status | 2.10.2 status | Closed by |
|---|---|---|---|
| 1 All data sources and computing services are resources | Met | Met | |
| 2 All communication is secured regardless of network location | Partial (G3) | Met | Gateway verifies server workload identity against registry |
| 3 Access is granted per session | Met | Met | |
| 4 Access is determined by dynamic policy | Partial (G2) | Met | Approver device compliance and step-up freshness as PE inputs |
| 5 Integrity and posture of every asset are monitored | Met | Met | |
| 6 Authentication and authorization are dynamic and enforced before access | Partial (G1, G5) | Met | CAEP push to every PEP; IdP disable revokes rooted chains |
| 7 State is collected and used to improve posture | Met | Met | |

## NIST SP 800-207 §5 threats to the ZTA itself

| §5 threat | Control | Status |
|---|---|---|
| 5.1 Subversion of the decision process | Control-environment attack cases against broker, boundary, registry, evidence; kill switch outside the halted class's reach; every scope change resolves to a human author | Met |
| 5.2 Denial of service on the PE or PA | Broker across failure domains, failover with no permit in transition, outage runs the incident runbook, unavailability denies | Met |
| 5.3 Stolen credentials and insiders | PoP binding, minutes TTL, one-token-two-identities alert, credential values never in context, separation of duties across agents | Met |
| 5.4 Visibility on the network | Gateways terminate and inspect every model, tool, and inter-agent exchange; platform telemetry baselined apart from agent records | Met |
| 5.5 Storage of system and network information | AIBOM access-controlled, registry responses signed and expiring, agent identities denied write to AIBOM and logs | Met |
| 5.6 Proprietary formats and vendor lock | SPIFFE, OIDC, RFC 8693, DPoP, PKCE, SSF/CAEP, CycloneDX ML-BOM or SPDX AI; vendor-executed agents carry a partial completeness claim and a named owner | Met |
| 5.7 Non-person entities in ZTA administration | Agent identities denied write to policy repository, every registry, score weights, token-service and gateway configuration, mirror rules, drift baselines, stop and approval configuration at IAM, repository, and API layers; agent-assisted changes merged only by a distinct human administrator; admin roles non-delegable | Met (2.10.2, G4) |
