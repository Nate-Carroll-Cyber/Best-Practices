# Enterprise Agentic AI Standard

This standard sets recommendations for agentic AI systems the enterprise builds, buys, or runs, including systems that use the Model Context Protocol or the Agent-to-Agent protocol.

## Purpose and scope

An agentic AI system is any AI system that calls tools, calls external services, keeps state between steps, delegates to other agents, or changes anything outside the model runtime. The Agentic AI Policy states why each recommendation exists, and the Agentic AI Technical Standards state how each one is tested.

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
| Decision-rights register | The record that assigns each automated action class to HITL or GITL and names its owner. |
| HITL | An action class where a person approves each action. |
| GITL | An action class that runs without per-action approval because it passed the register's admission tests and stays under its reversal ceiling. |
| Permission mode | The session setting that controls how much approval the runtime asks for. |
| Delegation chain | The sequence of agents from the person who started a task to the agent taking an action. |
| Orchestrator | An agent that splits, routes, retries, and recovers tasks for other agents. |
| Capability trifecta | One agent holding private-data access, untrusted-content intake, and the ability to send externally or act, all at once. |
| Anchored objective | The user's goal, fixed in system instructions at session start. |
| Tool manifest | The signed, versioned definition of a tool, Agent Card, OpenAPI document, plugin, or skill. |
| AIBOM | The AI bill of materials for a system, covering its models, software, data, prompts, tools, connectors, and agents. |
| Shadow agent or server | An agent or server found in the estate with no Approved or Active registry entry. |

## Identity and credentials

How agents, servers, and gateways prove who they are, and how their credentials are issued, scoped, stored, and revoked.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Identity and Trust | Cryptographic Agent Identity | Each agent, sub-agent, MCP server, and gateway should authenticate with its own workload identity from the enterprise identity authority, carried in a URI SAN or signed claim rather than a certificate Common Name. | A test connection that presents identity only in the Common Name is refused. |
| Identity and Trust | Secrets & Trust | Tools should fetch each secret from the vault at execution time using a reference ID, so the model only ever sees the reference or the secret value. Neither the model, prompts, memory, logs, and source control should hold no credentials, with signing keys kept in a KMS or HSM and static, long-lived, or leaked tokens retired and rotated. | A credential planted in a dependency error is redacted before it reaches the model or storage. |
| Identity and Trust | Access & Entitlements | Agent credentials should be issued per task, expire within minutes, name their target resource under RFC 8707, bind to the workload by mTLS or DPoP, and stay separate across development, staging, and production. | A token replayed from a second workload and a development credential are both refused at the production endpoints tested. |
| Identity and Trust | Identity Provider | The token service should revoke the delegation chains and pending approvals of a disabled person within the published latency and suspend any agent whose only owner is disabled. | Disabling a test user revokes the chains they started within the latency target. |
| Control Plane | Registries | Each agent, server, and tool should carry a persistent registry identifier that the broker stamps on each decision record. | A registry export joins to a sampled day of broker records. |
| Enforcement | Secrets Redaction | Before logs, tool output, or error payloads are stored or displayed, a secret scanner should check them against known secret patterns and keep only allowlisted fields. | A test credential under an unexpected field name does not reach the log. |
| Platform and Containment | Policy-as-Code | A permission change, or a repository setting that adds an MCP server or pre-approves a tool, should load only after policy-as-code review and the agent owner’s approval. | An unapproved MCP definition committed to a test repository is unreachable at session start. |

## Authorization and enforcement

How an enforcement broker checks each consequential action an agent proposes and approves or denies it before it runs.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Identity and Trust | Access & Entitlements | Each agent should hold only the tools, data, destinations, and action types its declared task needs, with effective permissions recertified each quarter. | Effective permissions in a sample match documented permissions. |
| Control Plane | Policy Engine (PDP) | The broker should authenticate the principal and agent, check the tool's pinned version, validate the request against a strict schema, and then authorize it as a separate step that also covers changes to the session permission mode. | A schema-valid test request to an unauthorized recipient is denied with a 403, and an agent-requested mode change is denied. |
| Control Plane | Policy Engine (PDP) and Registries | Agent identities should have no write or delegated administrative access to the policy repository, registries, gateway configuration, decision-rights register, or AIBOM. | A test workload-identity write to each store is denied. |
| Control Plane | Deterministic Guardrail | Models should propose typed actions that tools run from argument arrays, an executable allowlist, and symlink-resolved paths, with no untrusted values inserted into commands or SQL. | Shell metacharacter, leading-dash, encoded traversal, and symlink test cases fail against the process, file, and query tools tested. |
| Control Plane | Kill Switch & Circuit Breakers | Circuit breakers should trip on error spikes, consecutive tool failures, loop depth, resource exhaustion, call volume, repeated denials, injection detection, and budget exhaustion, and should stay tripped on privileged, destructive, financial, safety, or externally visible actions until an operator resets them. | A tripped breaker on a privileged class needs a recorded operator reset. |
| Orchestration | Controller / Execution Control | Each next action should be a new proposal with its own broker decision, so tool output does not trigger another tool and privileged calls do not retry automatically. | A tool result containing an embedded command causes no second execution. |
| Orchestration | Agent Runtime | Model-generated code should run in production only after a broker decision and inside a Firecracker or Kata microVM. | Generated code without a permit is blocked in testing. |
| Enforcement | Enforcement Broker (PEP) | Consequential actions, including memory writes, should pass through one redundant enforcement broker outside the agent runtime that caps task resources, rejects replays, refuses runtimes missing required security features, and denies on a timeout, error, or outage. | With the policy service stopped a test action is denied, and a runaway test task stops at a named limit. |
| Enforcement | Data Access Controls | Agents should reach production databases only through predefined parameterized queries with read-only identities for reads and a result limit on each query. | A code search of the agent finds no model-generated SQL path to production. |
| Inference | Model Gateway / Firewall | Model, region, sampling settings, output length, and token and cost budgets should be checked against an allowlist when configuration loads, and each invocation should record the resolved model and version. | A model not on the allowlist is refused at load. |

## Execution environment

The sandbox, network, and hardware isolation that agent code, tool calls, and model weights run inside.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Orchestration | Agent Runtime | Untrusted code and high-risk tools should run in disposable non-root sandboxes on a Firecracker or Kata microVM runtime class with Linux capabilities dropped, a read-only root filesystem, workload-tested seccomp, resource limits, no network by default, and no credentials or host mounts. | Admission control rejects a test pod that lacks an approved runtime class or one of these settings. |
| Orchestration | Sandbox Egress | Sandbox egress should be blocked at both the sandbox and the cluster or cloud layer except through one allowlisted and logged proxy, with no metadata endpoint access and no internet access for workloads that run generated code or hold all three trifecta legs. | Tested disallowed destinations are blocked at both layers, and a metadata request fails. |
| Inference | On-Prem GPU Inference | Inference should run on GPU nodes separate from tool execution, and enterprise-owned weights should be decrypted only inside an attested trusted execution environment such as NVIDIA Hopper confidential computing on AMD SEV-SNP or Intel TDX hosts. | An induced attestation failure in staging blocks weight decryption, and a test execution pod cannot schedule a GPU. |
| Inference | Model Gateway / Firewall | The LLM gateway should run in its own zone with a separately revocable provider credential for each tenant and agent. | The gateway credential store shows one revocable credential per tenant and agent. |
| Platform and Containment | Zero Trust Control | Each agent workload should run in a default-deny network zone whose identities cannot create load balancers, private links, peering, tunnels, VPNs, DNS records, or unreviewed indirect routes to the internet. | The network policy export shows default deny in the zones reviewed, and a test attempt to create a tunnel is denied and raises an alert. |
| Platform and Containment | AIBOM and Provenance Gate | Workloads should verify images, dependencies, and configuration at start and refuse mutable tags, unsigned images, and cache entries with the wrong digest. | A mutable tag, an unsigned image, and a wrong-digest cache entry are each rejected at start. |

## Context, memory, and data

How content entering an agent’s context, memory, and retrieval stores is labeled, screened, isolated, and kept current.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Control Plane | Deterministic Guardrail | A guardrail with none of the agent's permissions should check each delete, send, modify, or export against the user's objective held by the control plane outside the agent's context and escalate actions that fall outside the objective's scope or exceed the owner's embedding-distance threshold. | In testing, an injected action is blocked even after the agent's own copy of the objective has been overwritten. |
| Control Plane | Human Oversight (HITL) | A write to a place another run can read, such as a public repository, model hub, package registry, paste site, webhook, or shared bucket, should go to human approval by default. | A write to a shared bucket without a permit is blocked. |
| Orchestration | Controller / Execution Control | The prompt should state as rules that the agent does not act outside the declared targets, use social engineering, create third-party accounts, or publish where others can read. | Prompt review finds these rules and no claims that the environment prevents the actions. |
| Enforcement | Prompt, Response & Inline DLP | Content should carry source and trust labels that survive compaction and storage, be scanned for injection and concealment on the way into context, and be checked for destination, classification, secrets, and personal data on the way out. | A test document carrying invisible-character text is flagged before it enters context, and a restricted value in a tool parameter bound for an outside destination is blocked. |
| Enforcement | Enforcement Broker (PEP) | Context compaction should be a broker decision that keeps security findings and refusals and preserves the pre-compaction version. | A compaction keeps an out-of-scope finding and a prior refusal. |
| Resources | Memory Store | Memory should be isolated by tenant, user, agent, and task in infrastructure-enforced namespaces, with each entry recording its writer, source, classification, expiry, and scope and supporting quarantine and rollback. | Data written as tenant A is not returned to tenant B under concurrent load, and a test write missing a field is rejected. |
| Resources | Data Stores | The data owner should approve each data source and record its classification and residency in the AIBOM, keeping data collected or generated in China in China as TC260 Framework 3.0 requires. | Each source has an approval and residency record. |
| Resources | Knowledge Retrieval Security | Vector stores should accept writes only from the ingestion pipeline and record each chunk's source version and embedding model, so that changed sources are re-embedded, stale chunks are left out of retrieval, and a model change triggers re-indexing. | A test edit to a source document leads to re-embedding, and an agent-identity write to the vector store is denied. |
| Observability | Continuous Validation and Evals | Guardrail results should stay out of any reward, score, or selection signal for the agent. | Training and evaluation configuration excludes guardrail outputs. |
| Governance | Data Privacy | The broker should check user consent before data reaches a model or third-party tool wherever a jurisdiction requires consent, including China under TC260 Framework 3.0. | A broker rule checks consent before model and tool calls. |

## Human oversight and decision rights

Which action classes need a person’s approval, how each approval request is built and bound to its action, and how approver quality is checked.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Control Plane | Human Oversight (HITL) | The broker should route each automated action class by its register entry, hold permanent deletion, backup deletion, retention or immutability changes, fund transfers, permission changes, PII disclosure, production modification, OT or safety-system containment, binding external commitments, pricing or billing changes, external transfers of internal data or code, bulk production database changes, and any class with no rollback path at HITL, and send each approval request built from the canonical action to an approver whose current role it checks, whose decision it binds cryptographically to that exact action, and whose hourly volume it caps. | In testing, an action with no register entry is denied, a parameter changed after approval blocks execution, and a burst of approval requests queues past the cap. |
| Control Plane | Policy Engine (PDP) | The broker should escalate a GITL action to HITL when signals from outside the model, such as the amount, data class, target system, anomaly score, or a recent breaker trip, cross the owner's threshold, and it should not use the model's own stated confidence as a signal. | A GITL action with a test anomaly score above the threshold goes to an approver. |
| Enforcement | Enforcement Broker (PEP) | Freezes and approval rules should be enforced by the broker rather than by prompt text. | Sampled controls in the evidence table map to a broker rule. |
| Resources | MCP Servers and APIs | Each consequential tool should offer a draft stage with no side effects, a commit stage that runs only on timely confirmation of that draft, and an undo path, with the second confirmation TC260 Framework 3.0 requires before deleting files, sending data, or changing configuration under Chinese jurisdiction. | A test confirmation for a superseded or expired draft is refused. |
| Observability | Grant and Approval Ledger | Each approval record should hold the approver, decision, time, signature, the request as displayed, the display and decision times, and whether review was mandatory, so approval rates can be reported separately for mandatory and discretionary review. | A sampled thirty days of approvals can be reconstructed from the records. |
| Observability | Behavioral Monitoring | Security operations should seed known errors into reviewer queues and track decision time, approval rate, override rate, and catch rate per approver and agent, sending an approver for independent review when decisions come faster than the request can be read, approvals near 100 percent, or catch rate falls below that reviewer's baseline. | Dashboards, seeded-item results, and review records exist for the four measures. |

## Audit and observability

What each agent action records, where those records are kept, and what security operations detects from them.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Identity and Trust | Identity Provider | Records should take the initiating person from a validated OIDC token or mTLS-bound assertion and label sessions without one as self-asserted. | A test session with no assertion is labeled self-asserted. |
| Platform and Containment | Zero Trust Control | Each network-reachable component, including debug and health-check servers, should use one shared library that authenticates callers, caps request size, rate-limits by identity, returns fixed error codes with consistent status codes, and keeps stack traces and downstream error text out of model and caller responses. | In testing, oversized and over-limit requests are rejected before parsing, an unauthorized request returns a non-retryable 403, and a downstream error string does not reach the model. |
| Observability | Decision / Action Log | Each security-relevant event should be written to append-only or WORM storage the agent runtime cannot reach, recording the agent, initiating person, registry identifier, task and step, action and target, policy decision and version, approval, tool, model, outcome, correlation ID, and a structured decision summary in place of raw chain-of-thought. | A thirty-day sample shows the fields populated, and a test workload attempt to delete records or change retention is denied. |
| Observability | Telemetry Collection | Agent, gateway, tool, IAM, network-path, and control-plane telemetry should reach the SIEM with credentials and personal data masked, decision, approval, and identity records unsampled, and retention that rises with action impact and meets the six-month minimum TC260 Framework 3.0 sets for operational logs in China. | SIEM ingestion covers both streams in the sampled environments, and the retention policy names its periods by action impact. |
| Observability | Forensic Capture | Investigators should reconstruct incidents and check agent claims of completed work against at least two record streams the agent cannot write, such as broker records, KMS and cloud logs, tool-side logs, and network telemetry, and against reasoning traces kept under audit access controls and excluded from training data. | A simulated compromise with forged agent records is caught by stream mismatch, and a sampled training corpus contains no reasoning traces. |
| Observability | Monitoring & Detection | The SIEM should alert on new tools, destinations, or tool definitions, rate, cost, and denial anomalies, unusual delegation depth or read-then-send sequences, shared or unissued tokens, connections to unregistered endpoints, objective or memory drift, and use of decoy credentials, records, or honeypot endpoints that security operations places outside the agent runtime where agents can write to production or read Restricted data. | Sampled signals map to a SIEM rule, and a decoy credential used in a drill opens an incident within the detection target. |

## Validation and evaluation

The testing an agent passes before release, including threat modeling, attacks on the enforcement point, release evaluation, and red teaming.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Platform and Containment | CI/CD Pipeline | CI should run the error-path, service-surface, and tenant-isolation regression suites on each change, including against debug and fallback servers, and block the merge on a failure. | CI lists the suites as required checks. |
| Observability | Continuous Validation and Evals | Release evaluation should run in CI against the production sandbox and gateway, cover prompt injection, goal hijacking, tool substitution, token misuse, cross-tenant leakage, memory poisoning, privilege escalation, approval tampering, sandbox escape, covert exfiltration, logging interruption, and emergency stop, gate the release on a failure threshold, and compare evaluated with production success rates to catch sandbagging. | The evaluation log names the production environment, shows a result for each listed case, and meets the threshold. |
| Governance | Red Team / Threat Modeling | An independent red team should threat-model each system across model behavior, orchestration, memory, tools, data, identity, infrastructure, supply chain, and cross-agent paths, attack the enforcement point and guardrails in complete sessions, and have the model try offline to escape its own sandbox before it receives code-execution or production tools. | The red-team report covers the listed domains, reports detection, unsafe-action, and containment measures, and records the pre-validation result. |
| Governance | Safety | An agent should not reach production or roll back to a version with an open critical vulnerability or an unaccepted high vulnerability in authorization, isolation, data protection, or privileged tools. | A test rollback to an image that fails the gate is refused. |
| Governance | Audit and Assurance | Reviewers independent of the development team should validate each use case before deployment and retest each finding, adding formal scheduled validation for high-materiality use cases under the MAS Guidelines. | Sampled closed findings name their retester. |

## Supply chain and AIBOM

How models, data, dependencies, and suppliers are recorded in the AIBOM, verified, and admitted through the internal mirror.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Resources | Knowledge Retrieval Security | Training data and retrieval corpora should be screened before use for invisible characters, homoglyphs, encoded strings, hidden document layers and metadata instructions, and image or audio perturbations, with flagged items quarantined and unverified media kept out unless its provenance is recorded. | A seeded invisible-character trigger in a test corpus is quarantined. |
| Inference | External Service / LLM | Teams should detect supplier model changes by comparing the model ID recorded on each call with the AIBOM and by tracking provider changelogs and sub-processor notices. | A test provider alias change is detected within one review cycle. |
| Platform and Containment | AIBOM and Provenance Gate | Each production agent should have a graph AIBOM that agents cannot write, recording its models, prompts, tools, servers, packages, datasets, embeddings, adapters, guardrails, sub-agents, and connectors with version, hash, owner, and license, separating declared, reachable, and observed dependencies, and blocking promotion when it is missing or stale. | A sampled AIBOM validates in a standard BOM format and lists the named component types, and a test promotion with a stale AIBOM is refused. |
| Platform and Containment | Artifact Mirror | Dependencies, images, models, plugins, and servers should be pinned to reviewed versions and pulled only through an internal mirror that verifies digest and signature, rejects model formats that run code on load, and accepts new entries only from its own service identity with a recorded approver. | A test floating version and a test direct public fetch both fail CI, and a test unsafe model file is rejected at the mirror. |
| Platform and Containment | CI/CD Pipeline | The pipeline should sign internal checkpoints, training data, adapters, and infrastructure and policy code into a tamper-evident log, refuse a checkpoint whose hash differs from that log, record the generator of each synthetic training sample, and regenerate and sign the AIBOM on each build and on model, dependency, guardrail, or tool changes. | Pipeline attestations exist for sampled internal artifacts, a test swapped checkpoint is refused at load, and sampled builds emit a signed AIBOM. |
| Governance | Supply Chain and Third Party | Supplier management should put AIBOM attestation, change notice, and audit rights in AI contracts, track each component's maintenance status and advisories to an owner and patch state, and test third-party AI on enterprise data before onboarding as the MAS Guidelines expect. | Contract clauses, the vendor gap register, and advisory owners exist for sampled suppliers. |

## Incident response

How an agent, a server, or a class of workloads is stopped, investigated, and restored after a compromise or failure.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Control Plane | Kill Switch & Circuit Breakers | An operator should be able to disable one agent or server, or halt the workloads of a given type, model, server version, or tenant, within minutes from infrastructure the halted workloads cannot reach. | A drill measures the time to disable one server and the time to halt one class. |
| Control Plane | Human Oversight (HITL) | An incident in a GITL class should demote that class to HITL during containment, before root-cause work begins. | Register history shows the demotion before the root-cause record. |
| Platform and Containment | CI/CD Pipeline | The response team should rebuild compromised workloads from known-good images and rotate exposed credentials from a clean environment. | Rebuild and rotation records exist for the incidents sampled. |
| Observability | Forensic Capture | Responders should scope each incident from the AIBOM version in force at the earliest sign of compromise, using at least two record streams the agent could not write and a review of earlier runs of the same model, workload type, or infrastructure. | The incident record cites the AIBOM version, the record streams used, and the earlier runs checked. |
| Governance | Incident Response | Each production agent should have a drilled response plan that treats a broker or gateway outage as an incident, halts runs that exchanged messages with the compromised run, revokes the credentials it held, used, or found, purges affected memory and caches, locks touched model weights, and quarantines training data from the incident window. | The latest drill record shows the timings, the connected runs halted, and the revocations, purges, and quarantines performed. |
| Governance | Legal and Regulatory | Privacy and Legal should assess regulatory duties in each incident, including the report to the competent authorities and the user notice that TC260 Framework 3.0 requires under Chinese jurisdiction. | The incident record holds the notification decision and its owner. |

## MCP profile

An agent that reaches a tool through the Model Context Protocol should follow these recommendations in addition to the general recommendations in this standard.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Control Plane | Registries | Each MCP server and tool should be registered with a signed, versioned manifest bound to its definition hash, treated as destructive until the registry classifies it, and blocked after adding a tool, widening a scope, or reaching a new destination until the agent owner consents again. | A test unregistered server gets no credential, and a server that adds a tool after approval is blocked. |
| Enforcement | MCP Gateway | Each tool call should pass through the tenant's MCP gateway, which checks the server's workload identity against the registry, exchanges the agent's token for a new downstream token under RFC 8693, serves tool definitions from the signed manifest, and scans manifest fields as untrusted text. | A direct connection fails, a forwarded token is rejected, and a schema change on the live server changes nothing the agent sees. |
| Resources | MCP Servers and APIs | Each MCP server should use enterprise SSO or OAuth 2.1 with RFC 8707 resource indicators, mutual TLS when remote, STDIO as a non-root process when local, and a kill switch that cuts it off within minutes. | A token issued for one server is rejected by another, and a kill-switch drill meets its time target. |
| Observability | Shadow AI & MCP Discovery | Security operations should reconcile discovery scans of networks, repositories, workstations, container registries, and CI with the registry and alert on connections to unregistered endpoints. | The latest scan has no unexplained entries. |
| Orchestration | Agent Runtime | The MCP host should connect only to registered servers whose manifest signature, version, and workload identity check out, route calls through the broker, and keep client credentials in the OS keychain or a secrets manager. | A host configured with an unregistered server fails to connect. |
| Governance | Supply Chain and Third Party | A procured product that acts as an MCP host or server should be classified on what it reaches, send its audit log to the SIEM, and have each unmet recommendation recorded in the vendor gap register. | Each embedded host or server has a gap register entry. |

## A2A profile

An agent that sends or receives agent-to-agent traffic should follow these recommendations in addition to the general recommendations in this standard.

| Plane | Capability | Recommendation | Evidence |
| --- | --- | --- | --- |
| Control Plane | Registries | The registry should record each agent's owner, purpose, model, called agents, tools, data sources, trifecta scorecard, and state, and serve a signed, pinned Agent Card that cannot change without re-approval. | A change to a live Agent Card has no effect until it is re-approved. |
| Enforcement | Agent Gateway | Agent-to-agent messages should pass through the agent gateway, which authenticates the caller, allows calls only to registered agents, treats each delegation as a broker decision no wider than the parent's scope, and hashes each handoff at send and receipt. | A direct connection fails, and a delegation wider than the parent's scope is rejected. |
| Identity and Trust | Access & Entitlements | The token service should issue each agent its own token at each hop and revoke child credentials when the task or the parent ends. | Ending a test parent revokes its child credentials within the latency target. |
| Orchestration | Controller / Execution Control | An orchestrator should split, route, retry, and recover tasks without holding standing credentials to read source data or write to a destination. | The orchestrators sampled hold no standing data credential. |
| Control Plane | Policy Engine (PDP) | The broker should recheck policy at each delegation and tool boundary and prevent an agent from both proposing and approving the same action. | A delegation that merges propose and approve is rejected. |
| Observability | Monitoring & Detection | The security review should list each location two concurrent runs can both write and read, such as caches, repositories, wikis, and ticket systems, and split it per run or monitor it for encoded content, left-behind tokens, and write-then-read patterns. | A test that passes a token between two agents through a shared cache raises an alert. |
| Resources | Peer Agents | The receiving agent should treat a peer's message as data and take any resulting action only under its own broker decision, objective, and scope. | A peer instruction to act out of scope is denied. |
| Enforcement | Prompt, Response & Inline DLP | Messages should pass DLP keyed to the receiving agent's data zone before they leave the sender. | A restricted value sent to a lower-zone agent is blocked. |
| Observability | Decision / Action Log | Each inter-agent record should carry both agents' registry identifiers, the chain identifier and position, the originating person, and the permission mode. | Each action in a sampled chain traces to one agent and one person. |
| Control Plane | Kill Switch & Circuit Breakers | A breaker tripped by one agent should stop its whole chain. | A breaker trip in a child stops the parent chain. |
| Governance | Red Team / Threat Modeling | The red team should test whole chains against a hostile peer that sends objective rewrites, embedded instructions, altered handoffs, replays, and forged provenance, and record containment time and cross-agent detection rate. | Each hostile-peer case has a recorded result. |

## Framework mapping

Each row names the external controls a recommendation area supports. Support is not a claim that the area satisfies those controls. Privacy and Legal should confirm applicability for each deployment. ISO/IEC 27001 or SOC 2 alignment should be stated only after auditor review.

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
| Incident response | A.6.2.8, A.8.4 | Manage | ASI08, ASI10 |
| MCP profile | A.4.4, A.6.2.3, A.10.2, A.10.3 | Manage | ASI02, ASI03, ASI04 |
| A2A profile | A.3.2, A.4.4, A.7.5, A.9.4, A.10.2 | Manage | ASI07 Insecure Inter-Agent Communication, ASI08, ASI03 |
