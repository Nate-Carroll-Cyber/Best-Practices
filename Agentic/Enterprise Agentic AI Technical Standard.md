# Enterprise Agentic AI Standard

The standard applies to every agentic AI system the enterprise builds, buys, or runs, including systems that use the Model Context Protocol or the Agent-to-Agent protocol. System owners design new agentic systems to this standard, security assesses running systems against it, and procurement records the gap for a product that cannot meet a recommendation.

## Purpose and scope

An agentic AI system is any AI system that calls tools, calls external services, keeps state between steps, delegates to other agents, or changes anything outside the model runtime. 

| System | How this standard applies |
| --- | --- |
| Production agents, including copilots, workflow agents, coding agents, and unattended services | The recommendations in this standard apply. |
| MCP servers, tool servers, connectors, plugins, and skills an agent can reach | The recommendations in this standard apply, and the MCP profile adds recommendations for servers, gateways, and hosts. |
| Agents that delegate to or receive work from other agents, including orchestrators | The recommendations in this standard apply, and the A2A profile adds recommendations for delegation, messaging, and orchestrators. |
| Developer workstations that run MCP hosts or servers | Production recommendations apply because these machines hold production-adjacent credentials. |
| AI features embedded in procured software | The feature is classified on the data it reaches and the actions it takes, and approval of the platform does not reduce what this standard recommends. Each recommendation the vendor cannot meet should be logged as a gap with a named owner. |
| Research and evaluation workloads that execute model-generated code | These workloads should follow the recommendations for agents that run model-generated code, including microVM isolation and no internet access. |

A model with no tool access, no persistent state, and no delegation path is out of scope and falls under the general AI Policy. Classification, scoring, and forecasting models with no action path are also out of scope.

Each item in this standard is a recommendation and uses should. A recommendation that is not met should be recorded with a reason, an owner, and a review date. The Policy prevails over this standard where the two conflict.

## Terms

| Term | Meaning |
| --- | --- |
| Consequential action | A tool call, memory write, memory or knowledge retrieval, context compaction, sub-agent start or stop, tool or skill registration, permission-mode change, or write to a location another run can read. |
| Enforcement broker | The policy engine outside the agent runtime that approves or denies consequential actions before they run. |
| MCP gateway | The per-tenant proxy that tool calls to MCP servers pass through. |
| Agent gateway | The proxy that agent-to-agent messages pass through. |
| Workload identity | The short-lived cryptographic identity of a running workload, issued by the enterprise identity authority. |
| Registry identifier | The persistent identifier that ties one agent, server, or tool to one registry entry across runs. |
| Decision-rights register | The record that assigns each automated action class to HiTL or GiTL and names its owner. |
| Human in the Loop (HiTL) | An action class where a person approves each action. High-impact and irreversible actions stay HiTL, including permanent deletion, backup deletion, retention or immutability changes, fund transfers, permission changes, PII disclosure, production modification, OT or safety-system containment, binding external commitments, pricing or billing changes, external transfers of internal data or code, writes to places other runs can read such as public repositories, package registries, paste sites, webhooks, or shared buckets, bulk production database changes, and any action with no rollback path. |
| Governance in the Loop (GiTL) | An action class that runs without per-action approval because it passed the register's admission tests and stays under its reversal ceiling. |
| Permission mode | The session setting that controls how much approval the runtime asks for. |
| Delegation chain | The sequence of agents from the person who started a task to the agent taking an action, carried in each token as the person as subject and the agents as actors. |
| On-behalf-of (OBO) exchange | The exchange of a person's token together with the agent's own actor credential for a downstream token that names the person as subject and the agent as actor, under RFC 8693 token exchange or a provider's equivalent flow. |
| Standing delegation grant | A refresh token or equivalent that lets an agent act through OBO for a person who is not present. |
| Orchestrator | An agent that splits, routes, retries, and recovers tasks for other agents. |
| Capability trifecta | One agent holding private-data access, untrusted-content intake, and the ability to send externally or act, all at once. |
| Anchored objective | The user's goal, fixed in system instructions at session start. |
| Tool manifest | The signed, versioned definition of a tool, Agent Card, OpenAPI document, plugin, or skill. |
| AIBOM | The AI bill of materials for a system, covering its models, software, data, prompts, tools, connectors, and agents. |
| Shadow agent or server | An agent or server found in the enterprise with no Approved or Active registry entry. |

## Identity and credentials

This section defines how agents, servers, and gateways prove who they are and how their credentials are issued, scoped, stored, and revoked.

| Plane | Capability | Recommendation | Verification |
|---|---|---|---|
| Identity and Trust | Cryptographic Agent Identity | Each agent, sub-agent, MCP server, and gateway should authenticate with its own workload identity from the enterprise identity authority, carried in a URI SAN or signed claim rather than a certificate Common Name. | A test connection that presents identity only in the Common Name is refused. |
| Identity and Trust | Secrets & Trust | Tools should resolve secrets from opaque handles at execution time so that the model, prompts, memory, logs, and source control hold no credentials, a signed-in person's token included, with signing keys kept in a KMS or HSM and static, long-lived, or leaked tokens retired and rotated. | A credential planted in a dependency error is redacted before it reaches the model or storage. |
| Identity and Trust | Secrets & Trust | A refresh token or other standing grant that lets an agent act for a person who is not present should be held in the vault and used by the credential service rather than the agent runtime, bound to one agent and one person, given its own expiry and revocation path, and logged on each use as an event distinct from the interactive path. | A standing grant is absent from the agent runtime and its environment in testing, and each test use produces its own audit event naming the agent and the person. |
| Identity and Trust | Access & Entitlements | Agent credentials should be issued per task, expire within minutes, name their target resource under RFC 8707, bind to the workload by mTLS or DPoP, and stay separate across development, staging, and production. | A token replayed from a second workload and a development credential are both refused at the production endpoints tested. |
| Identity and Trust | Access & Entitlements | Where an agent acts for a signed-in person, the credential service should run the OBO exchange, trading the person's token together with the agent's own actor credential for a downstream token that names the person as subject and the agent as actor, issue it only to an agent pre-authorized for that person through a may_act claim or an equivalent policy record, and limit its scope to what the person consented to, what the agent's registry entry declares, and what the task needs. | A test OBO exchange with no actor credential is refused, an exchange by an agent not pre-authorized for the test person is refused, and the issued token names the person as subject and the agent as actor with a scope inside all three limits. |
| Identity and Trust | Access & Entitlements | A person's or administrator's consent for an agent to act for that person should be recorded with the grantor, scope, grant time, and expiry, reviewed on a schedule, and withdrawable by the person. | The consent behind a sampled delegated action can be produced with grantor, scope, and expiry. |
| Identity and Trust | Identity Provider | The token service should revoke the delegation chains, outstanding OBO tokens, and pending approvals of a disabled person, or of a person who withdraws consent to an agent, within the published latency and suspend any agent whose only owner is disabled. | Disabling a test person, and withdrawing a test person's consent to an agent, each revoke the chains and tokens involved within the latency target. |
| Control Plane | Registries | Each agent, server, and tool should carry a persistent registry identifier that the broker stamps on each decision record. | A registry export joins to a sampled day of broker records. |
| Enforcement | Secrets Redaction | Before logs, tool output, or error payloads are stored or displayed, a secret scanner should check them against known secret patterns and keep only allowlisted fields. | A test credential under an unexpected field name does not reach the log. |
| Platform and Containment | Policy-as-Code | A permission change, or a repository setting that adds an MCP server or pre-approves a tool, should load only after policy-as-code review and the agent owner's approval. | An unapproved MCP definition committed to a test repository is unreachable at session start. |

## Authorization and enforcement

This section defines how an enforcement broker checks each consequential action an agent proposes and approves or denies it before it runs.

| Plane | Capability | Recommendation | Verification |
|---|---|---|---|
| Identity and Trust | Access & Entitlements | Each agent should have only the tools, data, and permissions its task needs, and those permissions should be reviewed regularly. | Effective permissions in a sample match documented permissions. |
| Control Plane | Policy Engine (PDP) | Before allowing an action, the broker should confirm who the user and the agent are, check the request against the tool’s approved version and input format, and then decide whether the action is permitted, including any request to loosen the session’s approval settings. | A schema-valid test request to an unauthorized recipient is denied with a 403, and an agent-requested mode change is denied. |
| Control Plane | Policy Engine (PDP) and Registries | Agent identities should have no write or delegated administrative access to the policy repository, registries, gateway configuration, decision-rights register, or AIBOM. | A test workload-identity write to each store is denied. |
| Control Plane | Deterministic Guardrail | Tools should take structured parameters from the model rather than command strings, and a deterministic guardrail should check each tool call against coded rules for that tool, including its input schema, allowed destinations, value and volume limits, approved programs, and resolved file paths, with untrusted values kept out of shell commands and SQL. | In testing, command injection, path traversal, and symlink cases fail, and calls to destinations off the allowlist or over their volume limit are blocked |
| Control Plane | Kill Switch & Circuit Breakers | Circuit breakers should stop an agent when its errors, loops, resource use, cost, or denied requests pass set limits, and a breaker on a high-impact action should stay tripped until an operator resets it. | A tripped breaker on a privileged class needs a recorded operator reset. |
| Orchestration | Controller / Execution Control | Each next action should be a new proposal with its own broker decision, so tool output does not trigger another tool and privileged calls do not retry automatically. | A tool result containing an embedded command causes no second execution. |
| Orchestration | Agent Runtime | Model-generated code should run in production only after a broker decision and only inside an isolated, network-segmented sandbox (Firecracker, Kata microVM, etc.). | Generated code without a permit is blocked in testing. |
| Enforcement | Enforcement Broker (PEP) | Consequential actions, including memory writes, should pass through an enforcement broker outside the agent runtime that limits task resources, rejects replayed requests, and denies the action when the authorization check times out, fails, or is unavailable. | With the policy service stopped a test action is denied, and a runaway test task stops at a named limit. |
| Enforcement | Data Access Controls | A service receiving an action under OBO or other delegated authority should check that the action is permitted for the person and permitted for the agent, deny when either check fails, and decide on the current actor's own identity and scope rather than on the actor chain, which is a record and not a permission. | A test action permitted for the person but not the agent is denied, and one permitted for the agent but not the person is denied. |
| Enforcement | Data Access Controls | Agents should reach production databases only through predefined parameterized queries with read-only identities for reads and a result limit on each query. | A code search of the agent finds no model-generated SQL path to production. |
| Inference | Model Gateway / Firewall | Model, region, sampling settings, output length, and token and cost budgets should be checked against an allowlist when configuration loads, and each invocation should record the resolved model and version. | A model not on the allowlist is refused at load. |

## Execution environment

This section defines the sandbox, network, and hardware isolation that agent code, tool calls, and model weights run inside.

| Plane | Capability | Recommendation | Verification |
|---|---|---|---|
| Orchestration | Agent Runtime | Untrusted code and high-risk tools should run in disposable non-root sandboxes on a Firecracker or Kata microVM runtime class with Linux capabilities dropped, a read-only root filesystem, workload-tested seccomp, resource limits, no network by default, and no credentials or host mounts. | Admission control rejects a test pod that lacks an approved runtime class or one of these settings. |
| Orchestration | Sandbox Egress | Sandbox egress should be blocked at both the sandbox and the cluster or cloud layer except through one allowlisted and logged proxy, with no metadata endpoint access and no internet access for workloads that run generated code or hold all three trifecta legs. | Tested disallowed destinations are blocked at both layers, and a metadata request fails. |
| Inference | On-Prem GPU Inference | Inference should run on GPU nodes separate from tool execution, and enterprise-owned weights should be decrypted only inside an attested trusted execution environment such as NVIDIA Hopper confidential computing on AMD SEV-SNP or Intel TDX hosts. | An induced attestation failure in staging blocks weight decryption, and a test execution pod cannot schedule a GPU. |
| Inference | Model Gateway / Firewall | The LLM gateway should run in its own zone with a separately revocable provider credential for each tenant and agent. | The gateway credential store shows one revocable credential per tenant and agent. |
| Platform and Containment | Zero Trust Control | Each agent workload should run in a default-deny network zone whose identities cannot create load balancers, private links, peering, tunnels, VPNs, DNS records, or unreviewed indirect routes to the internet. | The network policy export shows default deny in the zones reviewed, and a test attempt to create a tunnel is denied and raises an alert. |
| Platform and Containment | AIBOM and Provenance Gate | Workloads should verify images, dependencies, and configuration at start and refuse mutable tags, unsigned images, and cache entries with the wrong digest. | A mutable tag, an unsigned image, and a wrong-digest cache entry are each rejected at start. |

## Context, memory, and data

This section defines how content entering an agent's context, memory, and retrieval stores is labeled, screened, isolated, and kept current.

| Plane | Capability | Recommendation | Verification |
|---|---|---|---|
| Enforcement | Prompt, Response & Inline DLP | Content should carry source and trust labels that survive compaction and storage, be scanned for injection and concealment on the way into context, and be checked for destination, classification, secrets, and personal data on exit. | A test document carrying invisible-character text is flagged before it enters context, and a restricted value in a tool parameter bound for an outside destination is blocked. |
| Enforcement | Enforcement Broker (PEP) | Context compaction should be a broker decision that keeps security findings and refusals and preserves the pre-compaction version. | A compaction keeps an out-of-scope finding and a prior refusal. |
| Resources | Memory Store | Each tenant, user, and agent should have its own memory space enforced by the storage layer, and each memory entry should record who wrote it, where it came from, its classification, and when it expires, so a bad entry can be found and removed. | In testing, one tenant cannot read another tenant’s memory, and a memory write with no source or expiry is rejected. |
| Resources | Data Stores | The data owner should approve each data source and record its classification and residency in the AIBOM, keeping data collected or generated in appropriate regions. | Each source has an approval and residency record. |
| Resources | Knowledge Retrieval Security | Vector stores should accept writes only from the ingestion pipeline and record each chunk's source version and embedding model, so that changed sources are re-embedded, stale chunks are left out of retrieval, and a model change triggers re-indexing. | A test edit to a source document leads to re-embedding, and an agent-identity write to the vector store is denied. |
| Observability | Continuous Validation and Evals | Guardrail results should stay out of any reward, score, or selection signal for the agent. | Training and evaluation configuration excludes guardrail outputs. |
| Governance | Data Privacy | The broker should check user consent before data reaches a model or third-party tool wherever a jurisdiction requires consent. | A broker rule checks consent before model and tool calls. |

## Human oversight and decision rights

This section defines which action classes need a person's approval, how each approval request is built and bound to its action, and how approver quality is checked.

| Plane | Capability | Recommendation | Verification |
|---|---|---|---|
| Control Plane | Human Oversight (HiTL) | The broker should check each action against the decision-rights register and send high-impact or irreversible actions to a person for approval, showing the exact action that will run and, under delegated authority, the person it runs for and the actor chain, confirming the approver still holds the right role, voiding the approval if the action or the chain changes, and limiting how many approvals one person handles per hour. | In testing, an action with no register entry is denied, a parameter or actor changed after approval blocks execution, and a burst of approval requests queues past the cap. |
| Control Plane | Policy Engine (PDP) | The broker should escalate a GiTL action to HiTL when signals from outside the model, such as the amount, data class, target system, anomaly score, or a recent breaker trip, cross the owner's threshold, and it should not use the model's own stated confidence as a signal. | A GiTL action with a test anomaly score above the threshold goes to an approver. |
| Enforcement | Enforcement Broker (PEP) | Freezes and approval rules should be enforced by the broker rather than by prompt text. | Sampled controls in the evidence table map to a broker rule. |
| Observability | Grant and Approval Ledger | Each approval record should hold the approver, decision, time, signature, the request as displayed, the display and decision times, and whether review was mandatory, so approval rates can be reported separately for mandatory and discretionary review. | A sampled thirty days of approvals can be reconstructed from the records. |
| Observability | Behavioral Monitoring | Security operations should track each approver’s decision time, approval rate, and override rate from approval records, and flag an approver for review | Approval records include each approver’s median decision time, approval rate, and override rate. |

## Audit and observability

This section defines what each agent action records, where those records are kept, and what security operations detects from them.

| Plane | Capability | Recommendation | Verification |
|---|---|---|---|
| Identity and Trust | Identity Provider | Records should take the initiating person from a validated OIDC token or mTLS-bound assertion, label sessions without one as self-asserted, and record an action an agent took in a person's name under the agent with the person as subject, so the person's own activity record does not show it as theirs. | A test session with no assertion is labeled self-asserted, and a test action taken by an agent for a person appears under the agent identity with the person as subject. |
| Platform and Containment | Zero Trust Control | Each network-reachable component, including debug and health-check servers, should use one shared library that authenticates callers, caps request size, rate-limits by identity, returns fixed error codes with consistent status codes, and keeps stack traces and downstream error text out of model and caller responses. | In testing, oversized and over-limit requests are rejected before parsing, an unauthorized request returns a non-retryable 403, and a downstream error string does not reach the model. |
| Observability | Decision / Action Log | Each security-relevant event should be written to append-only or WORM storage the agent runtime cannot reach, recording the agent, initiating person, registry identifier, task and step, action and target, policy decision and version, approval, tool, model, outcome, correlation ID, the actor chain and whether the person was present where the action ran on delegated authority, and a structured decision summary in place of raw chain-of-thought. | A thirty-day sample shows the fields populated, and a test workload attempt to delete records or change retention is denied. |
| Observability | Telemetry Collection | Agent, gateway, tool, IAM, network-path, and control-plane telemetry should reach the SIEM with credentials and personal data masked, decision, approval, and identity records unsampled, and retention that rises with action impact and meets regional operational log retention requirements. | A SIEM search for one agent session returns its agent, gateway, and tool events alongside the IAM and network events for the same session, and the retention policy gives a retention period for each record type. |
| Observability | Forensic Capture | Investigators should reconstruct incidents and check agent claims of completed work against at least two record streams the agent cannot write, such as broker records, KMS and cloud logs, tool-side logs, and network telemetry, and against reasoning traces kept under audit access controls and excluded from training data. | A sampled incident report cites two independent record sources, such as broker records and cloud audit logs, for each finding, and the training data pipeline has no read access to the reasoning-trace store. |
| Observability | Monitoring & Detection | The SIEM should alert on new tools, destinations, or tool definitions, rate, cost, and denial anomalies, unusual delegation depth or read-then-send sequences, shared or unissued tokens, a person's token exchanged by two agents, an OBO exchange by an agent not pre-authorized for that person, an OBO token used after consent or session revocation, a first consent pairing a person with an agent, connections to unregistered endpoints, objective or memory drift, and use of decoy credentials, records, or honeypot endpoints that security operations places outside the agent runtime where agents can write to production or read restricted data. | Sampled signals map to a SIEM rule, and a decoy credential used in a drill opens an incident within the detection target. |

## Supply chain and AIBOM

This section defines how models, data, dependencies, and suppliers are recorded in the AIBOM, verified, and admitted through the internal mirror.

| Plane | Capability | Recommendation | Verification |
|---|---|---|---|
| Resources | Knowledge Retrieval Security | Training data and documents loaded into RAG knowledge bases should be scanned before use for hidden text, such as invisible characters, look-alike characters, encoded strings, and instructions hidden in document layers or metadata, and flagged files should be held for review. | A seeded invisible-character trigger in a test corpus is quarantined. |
| Inference | External Service / LLM | Teams should detect supplier model changes by comparing the model ID recorded on each call with the AIBOM and by tracking provider changelogs and sub-processor notices. | A test provider alias change is detected within one review cycle. |
| Platform and Containment | AIBOM and Provenance Gate | Each production agent should have an AIBOM that agents cannot write, recording its models, prompts, tools, servers, packages, datasets, embeddings, adapters, guardrails, sub-agents, and connectors with version, hash, owner, and license, separating declared, reachable, and observed dependencies, and blocking promotion when it is missing or stale. | A sampled AIBOM validates in a standard BOM format and lists the named component types, and a test promotion with a stale AIBOM is refused. |
| Platform and Containment | Artifact Mirror | Dependencies, images, models, plugins, and servers should be pinned to reviewed versions and pulled only through an internal mirror that verifies digest and signature, rejects model formats that run code on load, and accepts new entries only from its own service identity with a recorded approver. | A CI build that uses an unpinned version or pulls straight from a public registry fails, and the mirror rejects a pickle-format model file. |
| Platform and Containment | CI/CD Pipeline | The pipeline should sign internal checkpoints, training data, adapters, and infrastructure and policy code into a tamper-evident log, refuse a checkpoint whose hash differs from that log, record the generator of each synthetic training sample, and regenerate and sign the AIBOM on each build and on model, dependency, guardrail, or tool changes. | Pipeline attestations exist for sampled internal artifacts, a test swapped checkpoint is refused at load, and sampled builds emit a signed AIBOM. |
| Governance | Supply Chain and Third Party | Supplier management should put AIBOM attestation, change notice, and audit rights in AI contracts, track each component's maintenance status and advisories to an owner and patch state, and test third-party AI on enterprise data before onboarding as the MAS Guidelines expect. | Contract clauses, the vendor gap register, and advisory owners exist for sampled suppliers. |

## MCP profile

An agent that reaches a tool through the Model Context Protocol should follow these recommendations in addition to the general recommendations in this standard.

| Plane | Capability | Recommendation | Verification |
| --- | --- | --- | --- |
| Control Plane | Registries | Each MCP server and tool should be registered with a signed, versioned manifest bound to its definition hash, with the tool annotations `readOnlyHint`, `destructiveHint`, `idempotentHint`, and `openWorldHint` inside the signed definition, treated as destructive until the registry classifies it, and blocked after adding a tool, widening a scope, lowering a declared risk annotation, or reaching a new destination until the agent owner consents again. | A test unregistered server gets no credential, a server that adds a tool after approval is blocked, and a tool whose `destructiveHint` changes from true to false after approval is quarantined. |
| Enforcement | MCP Gateway | Each tool call should pass through the tenant's MCP gateway, which checks the server's workload identity against the registry, exchanges the inbound token for a new downstream token under RFC 8693 or the provider's OBO flow, keeping the person as subject and adding itself to the actor chain rather than replacing the agent, serves tool definitions from the signed manifest, scans manifest fields as untrusted text, and returns each tool result to the agent as untrusted data minimized to the fields the task needs with personal data and secrets redacted before the model sees it. | A direct connection fails, a forwarded token is rejected, a downstream test token names the person as subject with the agent and the gateway as actors, a schema change on the live server changes nothing the agent sees, and a seeded instruction in a test tool result causes no change in the agent's next action. |
| Enforcement | MCP Gateway | The MCP gateway should rate-limit and quota requests, tool calls, and token volume per verified identity, cap payload size and reject malformed and recursive payloads before parsing, apply a timeout and circuit breaker to each call, and submit each call to the enforcement broker as its own proposal so the broker's decision from the registry classification sets the gate rather than the server's declared annotation. | A deliberately runaway test task stops at a named limit, an oversized payload is rejected before parsing, and a test tool declaring `readOnlyHint` true that the registry classifies as destructive still reaches the broker gate. |
| Resources | MCP Servers and APIs | Each MCP server should use enterprise SSO or OAuth 2.1 with RFC 8707 resource indicators, mutual TLS when remote, STDIO as a non-root process when local, and a kill switch that cuts it off within minutes. | A token issued for one server is rejected by another, and a kill-switch drill cuts a test server off within minutes. |
| Resources | MCP Servers and APIs | Each MCP server should sign sensitive messages inside the JSON payload with an expiry and replay nonce, require an idempotency key on each consequential operation, and refuse expired or duplicate requests. | A replayed test request is refused, and a duplicated consequential request executes once. |
| Resources | MCP Servers and APIs | Each in-house or open-source MCP server should build from a pinned dependency set with an SBOM recorded in the AIBOM, run under an OS-level sandbox such as seccomp, AppArmor, or SELinux with file system and network access denied where the server does not need them, and send outbound traffic only through the allowlisted egress proxy pinned to approved URLs and methods. | A test server attempting an outbound connection to an unapproved destination is blocked, and a server with no SBOM entry fails admission. |
| Resources | MCP Servers and APIs | Each server should be recorded as all-local, single-tenant remote, or multi-tenant remote with that pattern's controls present, and a server shared by more than one tenant should enforce tenant isolation in the storage and network layer with per-tenant encryption and role-based access tested in CI. | The registry entry for each server names its deployment pattern, and in testing one tenant's query against a shared server returns no other tenant's records. |
| Observability | Decision / Action Log | The MCP gateway should write each tool call to the decision log with the verified user and agent identity, registry identifier, tool name and version, safe parameter metadata, a hash of the result, and time, and send those records to the SIEM unsampled. | The last thirty days of MCP tool-call records can be produced with no gap in identity, tool, parameter metadata, or time. |
| Observability | Shadow AI & MCP Discovery | Security operations should reconcile MCP-aware differential scans of networks, repositories, workstations, container registries, and CI with the registry, search repositories for MCP configuration files, and alert on connections to unregistered endpoints. | The latest scan has no unexplained entries, and a test server moved to a new port appears in the next differential report. |
| Orchestration | Agent Runtime | The MCP host should connect only to registered servers whose manifest signature, version, and workload identity check out, route calls through the MCP gateway so each call receives its own broker decision, treat a tool with no annotations as destructive, non-idempotent, and open-world, and keep client credentials in the OS keychain or a secrets manager. | A host configured with an unregistered server fails to connect, and an unannotated test tool is gated as destructive. |
| Governance | Supply Chain and Third Party | A procured product that acts as an MCP host or server should be classified on what it reaches, checked for maintenance status before adoption, send its audit log to the SIEM, have its MCP-related CVEs and advisories tracked to an owner and patch state, and have each unmet recommendation recorded in the vendor gap register. | Each embedded host or server has a gap register entry and a named advisory owner. |

## A2A profile

An agent that sends or receives agent-to-agent traffic should follow these recommendations in addition to the general recommendations in this standard.

| Plane | Capability | Recommendation | Verification |
| --- | --- | --- | --- |
| Control Plane | Registries | The registry should record each agent's owner, purpose, model, called agents, tools, data sources, trifecta scorecard, and state, and serve a signed, pinned Agent Card that cannot change without re-approval. | A change to a live Agent Card has no effect until it is re-approved. |
| Enforcement | Agent Gateway | Agent-to-agent messages should pass through the agent gateway, which authenticates the caller, allows calls only to registered agents, treats each delegation as a broker decision no wider than the parent's scope, and hashes each handoff at send and receipt. | A direct connection fails, and a delegation wider than the parent's scope is rejected. |
| Identity and Trust | Access & Entitlements | The token service should issue each agent its own token at each hop, naming the originating person as subject and adding the agent to the actor chain so that no hop impersonates the person or its parent, and revoke child credentials when the task or the parent ends. | Ending a test parent revokes its child credentials within the latency target, and a sampled hop token names the person as subject and the sending agent as actor. |
| Orchestration | Controller / Execution Control | An orchestrator should split, route, retry, and recover tasks without holding standing credentials to read source data or write to a destination. | The orchestrators sampled hold no standing data credential. |
| Control Plane | Policy Engine (PDP) | The broker should recheck policy at each delegation and tool boundary and prevent an agent from both proposing and approving the same action. | A delegation that merges propose and approve is rejected. |
| Observability | Monitoring & Detection | The security review should list each location two concurrent runs can both write and read, such as caches, repositories, wikis, and ticket systems, and split it per run or monitor it for encoded content, left-behind tokens, and write-then-read patterns. | A test that passes a token between two agents through a shared cache raises an alert. |
| Resources | Peer Agents | The receiving agent should treat a peer's message as data and take any resulting action only under its own broker decision, objective, and scope. | A peer instruction to act out of scope is denied. |
| Enforcement | Prompt, Response & Inline DLP | Messages should pass DLP keyed to the receiving agent's data zone before they leave the sender. | A restricted value sent to a lower-zone agent is blocked. |
| Observability | Decision / Action Log | Each inter-agent record should carry both agents' registry identifiers, the chain identifier and position, the originating person, and the permission mode. | Each action in a sampled chain traces to one agent and one person. |
| Control Plane | Kill Switch & Circuit Breakers | A breaker tripped by one agent should stop its whole chain. | A breaker trip in a child stops the parent chain. |
| Governance | Red Team / Threat Modeling | The red team should test whole chains against a hostile peer that sends objective rewrites, embedded instructions, altered handoffs, replays, and forged provenance, and record containment time and cross-agent detection rate. | Each hostile-peer case has a recorded result. |

## Framework mapping

Each row names the external controls a recommendation area supports. Support is not a claim that the area satisfies those controls. Privacy and Legal should confirm applicability for each deployment. 

| Recommendation area | ISO/IEC 42001:2023 Annex A | NIST AI RMF | OWASP Top 10 for Agentic Applications 2026 |
|---|---|---|---|
| Identity and credentials | A.3.2, A.4.2, A.6.2.6 | Govern, Manage | ASI03 Identity and Privilege Abuse |
| Authorization and enforcement | A.6.2.6, A.9.2, A.9.4 | Manage | ASI02 Tool Misuse and Exploitation, ASI03, ASI05 Unexpected Code Execution |
| Execution environment | A.4.5, A.6.2.3, A.6.2.5 | Map, Manage | ASI05, ASI02 |
| Context, memory, and data | A.7.2 to A.7.6, A.9.4 | Map, Manage | ASI01 Agent Goal Hijack, ASI06 Memory and Context Poisoning |
| Human oversight and decision rights | A.9.3, A.6.1.3, A.4.6 | Manage | ASI09 Human-Agent Trust Exploitation |
| Audit and observability | A.6.2.8, A.6.2.6 | Govern, Measure | ASI10 Rogue Agents, ASI08 Cascading Failures |
| Validation and evaluation | A.6.2.4, A.6.2.5, A.6.2.7, A.5.2 | Measure | All ten as test coverage |
| Supply chain and AIBOM | A.4.3 to A.4.5, A.7.5, A.10.2, A.10.3 | Govern, Map | ASI04 Agentic Supply Chain Vulnerabilities |
| MCP profile | A.4.4, A.6.2.3, A.10.2, A.10.3 | Manage | ASI02, ASI03, ASI04 |
| A2A profile | A.3.2, A.4.4, A.7.5, A.9.4, A.10.2 | Manage | ASI07 Insecure Inter-Agent Communication, ASI08, ASI03 |
