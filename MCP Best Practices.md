# MCP Security Best Practices

This guide consolidates the actionable security practices in the supplied text. It covers the complete MCP Top 10, removes case-study narrative and repetition, and retains preventive controls, detection checks, immediate actions, and program-level guidance. Items marked **(NSA CSI)** are additions drawn from NSA, *Model Context Protocol (MCP): Security Design Considerations for AI-Driven Automation*, U/OO/6030316-26, May 2026 — including a new MCP11 availability category.

## Core Principles

1. **Keep credentials out of the model context.** The model does not need to see a credential to use a tool. Put credential-handling middleware between the model and the protected service.
2. **Grant only temporary, task-specific authority.** Use least privilege and just-in-time, short-lived, revocable tokens instead of standing permissions.
3. **Treat tool metadata and configuration as executable-risk input.** Names, descriptions, parameters, defaults, schemas, manifests, and project settings can influence agent behavior and must be verified before use.
4. **Verify every component in the trusted path.** Sign, pin, inventory, scan, and sandbox schemas, plugins, servers, SDKs, connectors, and dependencies.
5. **Never turn untrusted text into executable strings.** Use structured process APIs, parameterized queries, validated paths, and allowlisted operations.
6. **Constrain external communication.** Limit the channels through which credentials and private data can leave, and monitor unexpected outbound connections.
7. **Make changes and actions attributable.** Log permission changes and preserve per-agent action attribution so incidents can be investigated and contained.
8. **Treat retrieved content as data, never authority.** Keep system instructions separate from untrusted context and validate that planned actions remain aligned with the user’s original intent.
9. **Isolate every security boundary.** Enforce separation between tenants, sessions, environments, identities, and OAuth roles rather than relying on conventions or assumed ownership.
10. **Break the lethal trifecta.** For every agent, determine whether it can access private data, ingest untrusted content, and communicate externally. Remove at least one leg wherever possible.
11. **Protect availability.** Bound task complexity, recursion, and resource consumption; rate-limit and quota agent workloads so exhaustion techniques cannot stall the system. **(NSA CSI)**
12. **Bind messages to time and context.** Sign sensitive MCP messages, include expiry and replay protection, and require idempotent handling of re-delivered requests. **(NSA CSI)**

## MCP01 — Token Mismanagement and Secret Exposure

### Preventive controls

- Store secrets in a dedicated secrets vault or secrets manager—not in `.env` files, configuration files, prompts, or logs.
- Fetch credentials at runtime instead of placing them in environment variables or static configuration.
- Use short-lived tokens scoped to a single session or task; credentials should not outlive the work for which they were issued.
- Keep credentials out of prompts, context windows, model memory, vector stores, and other model-readable state.
- Place middleware between the LLM and credentials so the LLM never sees secret values.
- Redact token strings and other secret material from prompt, application, and observability logs.
- Avoid shared or static service accounts. Prefer narrowly scoped credentials that support attribution and independent revocation.
- Restrict unauthorized external communication so credentials and sensitive data do not have an exfiltration path.
- Route all outbound connections through a filtering egress proxy (e.g., Squid, tinyproxy) or enterprise DLP solution, pinned to specific resource URLs and access methods. **(NSA CSI)**
- Treat repository and project configuration—such as MCP or agent settings files—as potentially active, untrusted instructions. Do not apply untrusted project configuration before the user has established trust.

### Detection and review

- Scan repositories and configuration files for hard-coded tokens and token-like strings.
- Inspect prompts and model context for live credentials or retained secrets.
- Check logs for unredacted prompts and secret material.
- Identify tokens whose lifetime exceeds the session or task.
- Inventory shared and static service accounts.
- Review whether project configuration can redirect authenticated traffic or auto-approve MCP servers before a trust decision.

### Immediate action

- Move every static API key into a vault.
- Scan at least the last seven days of logs for token-like strings and remediate any exposure found.

## MCP02 — Privilege Escalation via Scope Creep

### Preventive controls

- Apply least privilege by design: start with the narrowest scope required instead of reducing broad access later.
- Use just-in-time tokens that are:
  - issued only when an action is needed;
  - scoped to that specific action;
  - valid for only minutes; and
  - revocable on demand.
- Eliminate standing credentials wherever possible.
- Manage permissions as policy-as-code in CI/CD so scope changes are reviewed and logged.
- Physically and logically separate development, staging, and production.
- Ensure development credentials cannot reach production systems or data.
- Do not reuse roles or credentials across environments.
- Give each agent or workflow its own identity instead of using shared agent or service accounts.
- Preserve per-agent action attribution.
- Add expiry to permissions and tokens so access is periodically re-evaluated.
- Enforce freezes, approval requirements, and other critical controls as hard authorization boundaries—not prompt instructions.
- Remove write and delete capabilities unless they are strictly required.
- Align tools and models with data classification zones: group public tools for public datasets; explicitly control and segregate tools that interact with sensitive or regulated data. **(NSA CSI)**
- Prefer locally deployed MCP server instances when processing private data. **(NSA CSI)**

### Detection and review

- Flag permission or scope changes that lack an audit record identifying who changed them and why.
- Identify shared agent and service accounts.
- Find scopes and tokens with no expiry.
- Verify that testing or development changes cannot be pushed directly to production.
- Verify that every agent action can be attributed to a distinct identity.
- Review cumulative permissions, not only individual grants, to detect quiet scope growth.

### Immediate action

- Inventory every write and delete capability available to each agent and remove every capability that is not necessary.

## MCP03 — Tool Poisoning

### Preventive controls

- Digitally sign tool schemas and manifests; reject definitions that do not have a valid signature.
- Pin tool definitions at the host using trust on first use (TOFU), and flag later changes.
- Bind each tool to known provenance and a known-good version.
- Scan the **entire** tool schema—not just the tool name or description—including:
  - tool and parameter names;
  - descriptions;
  - parameter definitions; and
  - default values.
- Treat every metadata field as instruction-grade, untrusted content because it enters model context.
- Strip ANSI escape sequences and other display-control characters from schema fields so reviewers see the actual content.
- Require RBAC for any writable schema registry.
- Require human review before schema edits are promoted to production.
- Reject or quarantine unexpected runtime schema changes.
- Re-prompt for informed user consent whenever a connected server’s capabilities or data access change; never let an already-trusted server expand silently. **(NSA CSI)**
- Do not assume a user approval step is sufficient if the model has already consumed untrusted tool metadata; validate metadata before it reaches the model.
- Isolate tools so one tool cannot silently manipulate another tool into accessing or exporting unrelated private data.

### Detection and review

- Identify schemas fetched without signature or integrity verification.
- Check whether a schema registry is writable without RBAC.
- Flag schema edits that are automatically promoted.
- Detect hosts that accept runtime schema changes without review.
- Find tools that lack provenance or version binding.
- Alert whenever the MCP tool list or any tool definition changes.
- Treat a tool rewriting its own definition as a security signal.

### Immediate action

- Pin and sign every tool schema, then scan all names, descriptions, parameters, and defaults.

## MCP04 — Supply Chain and Dependency Tampering

### Preventive controls

- Require signed components with verifiable provenance; do not run components whose authenticity or origin cannot be established.
- Maintain a complete software bill of materials (SBOM) for every MCP server and plugin.
- Pin every dependency, plugin, and MCP server to a reviewed, known-good version; do not use `latest` or other floating versions.
- Upgrade deliberately only after review.
- Avoid fetching dependencies dynamically at runtime. Build and deploy a fixed, vetted dependency set.
- Scan dependencies automatically in CI before release.
- Verify maintenance status before adoption: many popular MCP servers are archived or unmaintained; prefer reference implementations and actively supported projects, and apply the most stringent code-audit profile to newer integrations. **(NSA CSI)**
- Keep MCP development and test tooling patched — e.g., the MCP Inspector RCE (CVE-2025-49596) fixed in v0.14.1. **(NSA CSI)**
- Sandbox third-party plugins so a compromised component cannot compromise the entire environment.
- Monitor and restrict plugin network access, especially calls to unknown domains.
- Inventory MCP servers running on developer machines.
- Bind local development servers and tools to `127.0.0.1`/localhost rather than `0.0.0.0` or all network interfaces.
- Require authentication and origin checks for local web interfaces and reject unauthorized cross-origin requests.

### Detection and review

- Find unsigned plugin installations.
- Identify dependencies fetched at runtime.
- Verify that each server and plugin has a complete SBOM.
- Find dependencies using `latest`, ranges, or other floating versions.
- Monitor plugins for connections to unknown or unapproved domains.
- Check local MCP servers for unauthenticated interfaces, permissive origin handling, and network-wide binding.
- Operate a formal process for tracking MCP-related CVEs, vendor advisories, and open-source issue trackers, tied to the server inventory and patch history. **(NSA CSI)**

### Immediate action

- Inventory every MCP server on developer machines, pin every version, and bind local services to localhost.

## MCP05 — Command Injection and Execution

### Preventive controls

- Never construct shell command strings from user input, model output, remote responses, OAuth metadata, URLs, file content, or other untrusted values.
- Avoid passing untrusted strings to `exec`, `eval`, or APIs configured with `shell: true`.
- Use structured process APIs such as `execFile` or `spawn`, passing the executable and arguments separately as an array.
- Use `--` to terminate option parsing when the invoked command supports it, ensuring subsequent values are treated as data rather than flags.
- Remove, reject, or safely escape shell metacharacters as defense in depth, including semicolons, pipes, backticks, and command-substitution syntax such as `$()`.
- Do not automatically execute model-generated code. Add an explicit validation and authorization gate.
- Validate and constrain file paths to prevent traversal (for example, `../`) and access outside approved directories.
- Parameterize SQL queries; never interpolate values into SQL strings.
- Allowlist permitted commands, verbs, and operations so destructive or unexpected actions cannot run.
- Enforce strict schema validation on all deserialized context, configuration, and payload objects; maintain hard isolation between code and data during deserialization (classic insecure-deserialization risk applied to MCP’s serialized context). **(NSA CSI)**
- Block or restrict parameter forwarding when the origin of the data is ambiguous or potentially user-supplied, so inputs intended for one component cannot be reused by another. **(NSA CSI)**
- Sandbox local MCP servers and run agents as non-root, without administrator or unrestricted `sudo` privileges.
- Isolate each tool’s execution context with OS-level mechanisms — seccomp, AppArmor, SELinux, or Windows AppContainers — and explicitly deny runtime access to file systems, model/data files, and internal networks the tool does not require. **(NSA CSI)**
- Apply the same injection defenses to metadata and protocol fields—such as OAuth authorization URLs—as to conventional user input.

### Detection and review

- Search for shell commands assembled through concatenation or interpolation.
- Trace whether user input, model output, or remote responses flow into executable command strings.
- Find uses of `exec`, `eval`, and `shell: true` that can receive untrusted content.
- Identify generated code that is executed automatically.
- Review file operations for unsanitized or unconstrained paths.
- Find SQL statements built through string interpolation.
- Review protocol handlers, including OAuth flows, for values inserted into shell commands.
- Map execution findings to CWE-77, CWE-78, CWE-94, and CWE-95 for vulnerability management. **(NSA CSI)**

### Immediate action

- Search the codebase for `child_process.exec`, `os.system`, and `shell: true`.
- Treat each occurrence as a finding, review its data flow, and replace unsafe string-based execution with structured argument-array calls.

## MCP06 — Intent Flow Subversion

### Preventive controls

- Anchor the user’s original objective in the system prompt as a fixed, authoritative reference.
- Treat all retrieved pages, documents, issues, files, code comments, and tool output as untrusted data—not instructions.
- Tag or fence retrieved content explicitly as untrusted context before it reaches the planning model.
- Keep trusted system instructions structurally separate from retrieved content; do not merge them into an undifferentiated prompt.
- Validate that every planned or requested action remains aligned with the user’s original intent.
- Add a gate between planning and consequential actions so the agent cannot move directly from reading context to deleting, sending, modifying, or exporting.
- Use an independent guardrail model or control outside the primary agent’s context to detect intent drift and mismatched actions.
- Re-anchor the original intent during long-running sessions and multi-step workflows.
- Treat context preconditioning as an attack vector: probabilistic interpretation of prior context means an actor can seed context history to steer the agent toward a specific unsafe outcome; constrain how much prior context can influence consequential decisions. **(NSA CSI)**
- Inspect and filter each tool’s output before passing it to the next component in a chained pipeline — content length checks, disallowed-keyword scanning, and detection of injected prompts or code elements that could alter downstream behavior. **(NSA CSI)**
- Avoid giving one agent or credential simultaneous access to untrusted public content, sensitive private data, and unrestricted outbound actions.
- Break at least one leg of the private-data/untrusted-content/external-communications trifecta.

### Detection and review

- Find workflows with no intent-alignment validation.
- Identify agents that can treat retrieved text as instructions.
- Identify “blind planning” paths that execute immediately after retrieving context.
- Find prompts that merge system instructions and untrusted context without clear boundaries.
- Test long-running sessions for gradual intent drift.
- Test identical prompts under varied context histories for divergent, unsafe interpretations — non-deterministic behavior across deployments is itself an exploitable gap. **(NSA CSI)**
- Inventory production agents that possess all three trifecta capabilities.

### Immediate action

- Build a trifecta scorecard for every production agent:
  - Does it access private data?
  - Does it ingest untrusted content?
  - Can it communicate externally or take consequential action?
- Prioritize agents for which all three answers are yes, then remove or constrain at least one leg.

## MCP07 — Insufficient Authentication and Authorization

### Preventive controls

- Treat MCP as an API and apply established API-security controls to every request.
- Use OAuth 2.1 with short-lived, session-scoped tokens and per-user delegation.
- Require mutual authentication between clients, agents, tools, and servers; use mTLS where appropriate.
- Validate every token server-side, on every request, for authenticity, validity, expiry, intended resource, and authorization for the specific action.
- Make authorization decisions server-side from verified facts, never from client-supplied identity, roles, or scopes.
- Enforce scope checks for every tool and resource request.
- Use OAuth resource indicators to bind a token to its intended target server and prevent cross-resource replay.
- Never pass a client token through to a downstream service.
- Use token exchange or an on-behalf-of flow so an agent receives its own narrowly scoped downstream token.
- Separate the OAuth client and authorization-server roles and preserve independent trust boundaries.
- Avoid shared client IDs, shared credentials, and token reuse across agents.
- Rotate credentials and eliminate static or long-lived tokens.
- Cryptographically sign sensitive MCP messages within the JSON payload; include expiration timestamps and replay-protection metadata so each request is bound to its time and context (per OWASP ASVS V7 session-management guidance). **(NSA CSI)**
- Enforce idempotency for consequential operations — the protocol leaves it to JSON-RPC and it is often unimplemented; use idempotency keys so replayed or duplicated messages cannot re-execute actions. **(NSA CSI)**
- Correlate every logged action with a verified user and agent identity.

### Detection and review

- Check for missing mutual authentication between agents and tools.
- Inventory static or long-lived tokens.
- Find authorization decisions derived from client-controlled input.
- Identify logs that lack verified identity correlation.
- Alert when the same token appears across different agents or IP addresses.
- Review OAuth implementations for combined client/authorization-server roles, shared client IDs, redirect weaknesses, token passthrough, and missing resource binding.
- Test message handling for replay acceptance and duplicate re-execution of consequential operations. **(NSA CSI)**

### Immediate action

- Audit every downstream call made by agents and flag any flow that forwards the client’s token unchanged.

## MCP08 — Lack of Audit and Telemetry

### Preventive controls

- Record agent activity as structured, queryable, tamper-evident logs.
- Capture every tool call with, at minimum:
  - verified identity;
  - tool name;
  - parameters or safe parameter metadata; and
  - timestamp.
- Where feasible, record cryptographic hashes of tool results and outputs to support forensic reconstruction. **(NSA CSI)**
- Capture the prompt/context and network events needed to reconstruct intent changes and external communication, while applying data minimization and masking.
- Capture source information from HTTP headers where available, and alert natively on protocol anomalies: repeated or malformed requests, authorization failures, and RBAC violations. **(NSA CSI)**
- Centralize MCP telemetry and send it to the organization’s SIEM or XDR.
- Protect audit controls so the same people or processes being monitored cannot silently disable, delete, or rewrite them.
- Retain logs long enough to support incident response, forensic reconstruction, and compliance.
- Mask PII, credentials, and other sensitive payload content so the logging system does not become a new sensitive-data store.
- Establish behavioral baselines for normal agent and tool activity.
- Alert on anomalies such as unusual data volume, new tools or endpoints, identity changes, atypical call sequences, and goal drift.
- Run incident-response and telemetry drills before a real event occurs.

### Detection and review

- Find agent activity that is unlogged or exists only as free-text console output.
- Identify logs that are local, deletable, editable, or otherwise not tamper-evident.
- Verify that tool calls, relevant prompts/context changes, and network calls are captured.
- Check whether MCP telemetry reaches the SIEM or XDR used by the security team.
- Verify that anomaly detection and a normal-behavior baseline exist.
- Test whether telemetry can be disabled without an alert or independent approval.

### Immediate action

- Attempt to produce the last 30 days of MCP tool-call logs. Any gap in availability, identity, tool, parameter metadata, or time is a finding.

## MCP09 — Shadow MCP Servers

### Preventive controls

- Maintain a central registry and current asset inventory for every MCP server, with a named owner and lifecycle state.
- Make registration mandatory: unregistered servers must fail deployment or access.
- Enroll approved local servers in a registry service (e.g., the MCP Registry) so vetted implementations are available internally through a controlled path. **(NSA CSI)**
- Continuously scan networks, repositories, developer environments, and CI/CD for unregistered MCP services.
- Monitor agents for connections to endpoints that are absent from the approved registry.
- Provide secure-by-default configuration templates with hardening built in and a low-friction deployment path.
- Require SSO or OIDC on every server and tie every service to central identity.
- Eliminate default credentials and permissive development configurations before production use.
- Use network segmentation to prevent unknown or forgotten servers from reaching production systems and sensitive data.
- Ensure every production path includes security review, registration, centralized authentication, and telemetry.
- Revalidate registered servers continuously rather than treating approval as permanent.

### Detection and review

- Find servers deployed through paths that bypass registration.
- Compare active services with the asset inventory and investigate gaps.
- Scan common development ports such as `8000` and `8080`.
- Use MCP-aware scanners (e.g., MCP Scanner, Ramparts, CyberMCP, Proximity) to detect unauthenticated, vulnerable, unauthorized, or internet-exposed MCP services. **(NSA CSI)**
- Run periodic differential scans — MCP servers can dynamically change ports, so diff reports are needed to surface new or modified services. **(NSA CSI)**
- Check whether continuous discovery scanning exists and covers repositories, dotfiles, cloned projects, hosts, and CI.
- Alert when agents call unknown or unregistered endpoints.
- Search repositories for `mcp.json` and related MCP configuration files.

### Immediate action

- Search organization repositories for `mcp.json` to create a first-pass inventory, then reconcile the results with network scans and the central registry.

## MCP10 — Context Injection and Oversharing

### Preventive controls

- Use ephemeral, per-session context that is destroyed when the session ends.
- Isolate context by tenant, user, agent, and workflow using infrastructure-enforced namespaces.
- Ensure one tenant’s namespace is technically incapable of answering another tenant’s queries.
- Revalidate session and tenant ownership on every request; never trust cached or implicit session-to-user mappings.
- Avoid shared singleton state, shared context buffers, or shared vector stores unless isolation is explicitly enforced and tested.
- Classify and tag sensitive data as it enters context.
- Apply policy at ingestion to block, mask, minimize, or restrict sensitive content before it is stored.
- Give every context item a time-to-live and enforce automatic purging.
- Treat persistent context as a governed data store with access controls, classification, retention, and deletion requirements.
- Prevent injected content in shared memory from becoming instructions in later sessions.
- Add automated tenant-isolation testing to CI.

### Detection and review

- Inventory shared context buffers and vector stores used by multiple users, tenants, agents, or workflows.
- Check whether context persists across users or sessions.
- Find reused context or server-side state whose ownership is not revalidated on every request.
- Identify sensitive data that enters context without classification or tags.
- Find context with no TTL, expiry, or automatic purge.
- Test timing- and concurrency-dependent cross-tenant leakage conditions.

### Immediate action

- Write and run a tenant-isolation test that accesses tenant A and then verifies tenant B can see none of tenant A’s data. Add the test to CI for every build.

## MCP11 — Resource Exhaustion and Availability **(NSA CSI)**

### Preventive controls

- Rate-limit and quota agent workloads per verified identity: requests, token volume, tool calls, and concurrent tasks.
- Bound task complexity: cap recursion depth, chain length, fan-out, and per-task compute/time budgets; enforce timeouts and circuit breakers on long-running operations.
- Reject malformed, missing-field, and oversized inputs at the MCP boundary before they reach model or tool execution.
- Apply backpressure and queue limits so prompt storms degrade service gracefully instead of stalling the broader workflow.
- Budget for “lethargy” techniques — legitimate-looking but overly complex tasks designed to exhaust resources — with per-session resource ceilings.

### Detection and review

- Baseline normal resource consumption per agent and tool; alert on sustained deviation.
- Distinguish exhaustion attacks from valid high load by correlating resource spikes with identity, task type, and input characteristics.
- Monitor for recursive or self-amplifying task request patterns.
- Verify that a single client or session cannot consume unbounded server resources.

### Immediate action

- Set hard rate limits, timeouts, and recursion caps on every MCP server, then verify that a deliberately runaway task is contained rather than serviced.

## Consolidated Security Review Checklist

### Secrets and identity

- [ ] Secrets are stored in a vault and fetched only at runtime.
- [ ] No secret enters model context, prompts, configuration, environment variables, or logs.
- [ ] Logs redact token-like values.
- [ ] Tokens are short-lived, task-scoped, and revocable.
- [ ] Agents have distinct identities; no shared static service accounts are used.

### Authorization and environments

- [ ] Every agent follows least privilege.
- [ ] Write and delete access is explicitly justified.
- [ ] Permission changes are reviewed, logged, and managed as code.
- [ ] Development, staging, and production credentials and systems are isolated.
- [ ] Every action is attributable to a specific agent identity.
- [ ] Critical boundaries are enforced by authorization controls rather than natural-language instructions.
- [ ] Tools are aligned to data classification zones; sensitive-data tools are segregated and, where possible, local. **(NSA CSI)**

### Tools and schemas

- [ ] Schemas and manifests are signed, pinned, version-bound, and traceable to known provenance.
- [ ] Full schema content—including parameters and defaults—is scanned before model exposure.
- [ ] ANSI/display-control characters are stripped during review.
- [ ] Schema registries enforce RBAC and reviewed promotion.
- [ ] Tool-list and runtime schema changes trigger alerts.
- [ ] Capability or data-access changes on a connected server trigger renewed user consent. **(NSA CSI)**

### Supply chain and networking

- [ ] Every server and plugin has a complete SBOM.
- [ ] Dependencies and servers are pinned; no floating versions are used.
- [ ] Dependency scanning runs in CI.
- [ ] Third-party plugins are sandboxed.
- [ ] Unexpected outbound domains are blocked or alerted on.
- [ ] Outbound traffic flows through a filtering egress proxy or DLP pinned to approved resource URLs and methods. **(NSA CSI)**
- [ ] Maintenance status is verified; archived/unmaintained servers are not adopted, and MCP CVEs are formally tracked. **(NSA CSI)**
- [ ] Local services bind only to localhost and enforce authentication and origin checks.

### Execution safety

- [ ] Untrusted values never enter shell command strings.
- [ ] Processes are launched with structured argument arrays.
- [ ] Option parsing is terminated with `--` where supported.
- [ ] Generated code is not executed without validation and authorization.
- [ ] File paths are normalized and constrained to approved roots.
- [ ] SQL queries are parameterized.
- [ ] Commands and operations are allowlisted.
- [ ] Deserialization is schema-validated with hard code/data isolation. **(NSA CSI)**
- [ ] Tool execution is confined by OS-level sandboxing (seccomp, AppArmor, SELinux, AppContainers). **(NSA CSI)**
- [ ] Agents do not run with unnecessary root or administrator privileges.

### Intent integrity

- [ ] The user’s objective is anchored in trusted system instructions.
- [ ] Retrieved content is explicitly marked and handled as untrusted data.
- [ ] System instructions and retrieved content remain structurally separate.
- [ ] Consequential actions require an intent-alignment check.
- [ ] An independent guardrail detects intent drift.
- [ ] Chained tool outputs are inspected before reaching the next component. **(NSA CSI)**
- [ ] Every production agent has a current trifecta scorecard.

### Authentication and authorization

- [ ] Clients, agents, tools, and servers mutually authenticate.
- [ ] Every token is validated server-side on every request.
- [ ] Authorization is enforced server-side using verified identity and scope.
- [ ] Tokens are bound to their intended resource.
- [ ] Client tokens are never passed through downstream.
- [ ] Downstream access uses token exchange or on-behalf-of delegation.
- [ ] Sensitive messages carry payload signatures, expiry, and replay protection; consequential operations are idempotent. **(NSA CSI)**
- [ ] Logs correlate actions to verified user and agent identities.

### Audit and response

- [ ] Tool-call logs are structured, queryable, centralized, and tamper-evident.
- [ ] Every tool call records identity, tool, safe parameter metadata, and time.
- [ ] Relevant prompt/context changes and outbound events are observable.
- [ ] Sensitive log content is masked.
- [ ] MCP telemetry is integrated with the SIEM or XDR.
- [ ] Behavioral baselines and anomaly alerts are active.
- [ ] Protocol anomalies (malformed/repeated requests, authorization failures, RBAC violations) generate native alerts. **(NSA CSI)**
- [ ] Telemetry cannot be silently disabled or rewritten.
- [ ] Incident-response and telemetry drills are performed regularly.

### Inventory and governance

- [ ] Every MCP server is registered, inventoried, reviewed, and assigned an owner.
- [ ] Unregistered servers fail deployment or access.
- [ ] Continuous discovery scans networks, repositories, hosts, and CI/CD.
- [ ] MCP-aware scanners run differential scans to catch dynamic port changes and modified services. **(NSA CSI)**
- [ ] Agents calling unknown endpoints trigger alerts.
- [ ] Secure-by-default deployment templates are available.
- [ ] Every server uses central identity through SSO or OIDC.
- [ ] Network segmentation limits the reach of unknown services.

### Context and tenant isolation

- [ ] Context is ephemeral and scoped to one session.
- [ ] Tenant, user, agent, and workflow namespaces are infrastructure-enforced.
- [ ] Ownership is revalidated on every request.
- [ ] Sensitive context is classified and tagged at ingestion.
- [ ] Context has enforced TTLs and automatic purging.
- [ ] Tenant-isolation tests run in CI.

### Availability **(NSA CSI)**

- [ ] Rate limits and quotas are enforced per identity for requests, tool calls, and token volume.
- [ ] Recursion depth, chain length, and per-task time/compute budgets are capped with timeouts and circuit breakers.
- [ ] Malformed and oversized inputs are rejected at the MCP boundary.
- [ ] Resource-consumption baselines and deviation alerts are active.

## MCP Deployment Minimum Bar

Every MCP deployment should meet these six pillars:

1. **Identity, authentication, and policy**
   - OAuth 2.1.
   - Short-lived, scoped tokens.
   - No token passthrough.
   - Per-user delegation.
   - Every request tied to a verified identity with minimum privilege for minimum time.
2. **Isolation and lifecycle**
   - Tenant boundaries are tested, not assumed.
   - Workloads run in containers or equivalent isolation.
   - Services run as non-root and bind locally when remote exposure is unnecessary.
   - Context and credentials expire automatically.
   - Resource ceilings, rate limits, and timeouts bound every workload. **(NSA CSI)**
3. **Trusted, controlled tooling**
   - Every server is inventoried.
   - Tools are vendor-built or independently security-reviewed.
   - Components are signed, pinned, provenance-verified, and continuously revalidated.
4. **Schema-driven validation**
   - Inputs and outputs are validated at the MCP boundary.
   - Retrieved text is always treated as data, never as a command.
   - Sensitive messages are signed, time-bound, and replay-protected; deserialization is schema-enforced. **(NSA CSI)**
5. **Hardened deployment and oversight**
   - Structured tool-call logs flow to a SIEM or XDR.
   - Behavioral baselines and anomaly alerts are active.
6. **Trifecta scorecard**
   - Score every agent and workflow for private-data access, untrusted-content ingestion, and external communication.
   - Remove or constrain at least one leg wherever possible.

## Continuous MCP Security Program

A mature MCP security program continuously performs five functions:

1. **Discovery**
   - Scan networks, repositories, developer environments, and CI/CD continuously.
   - Run differential scans to catch dynamically relocated services. **(NSA CSI)**
   - Reconcile discovered systems with the official inventory.
2. **Governance**
   - Operate a trusted registry with a submit, review, deploy, and revalidate lifecycle.
   - Assign a named owner to every server.
   - Maintain a kill switch that can disable a compromised server within minutes.
3. **Identity and access**
   - Use OAuth 2.1, just-in-time tokens, and on-behalf-of delegation.
   - Tie every action to a real identity with minimum privilege for minimum time.
4. **Defense in depth**
   - Combine gateways, runtime policies, tool guardrails, network controls, and container isolation.
   - Design each layer to contain failures that pass through an earlier layer.
5. **Detection and response**
   - Maintain structured audit pipelines and anomaly detection.
   - Track and patch MCP-related CVEs as a formal, inventory-linked process. **(NSA CSI)**
   - Exercise compromise scenarios: determine who is paged, where the logs are, and how the kill switch is activated.

## Immediate-Priority Checklist

1. Audit every MCP source: determine whether each server is vendor-built or from an unreviewed repository.
2. Move static API keys to a vault and scan logs for token-like values.
3. Inventory all agent write and delete capabilities and remove those that are unnecessary.
4. Validate inputs at the MCP boundary and ensure retrieved text is never treated as a command.
5. Search for string-interpolated shell calls and replace them with structured process APIs.
6. Audit each agent’s effective production permissions—not only its documented permissions.
7. Confirm that client tokens are never passed through to downstream services.
8. Enable structured tool-call logging with identity, tool, safe parameter metadata, and time.
9. Run the tenant-isolation test and add it to CI.
10. Build a trifecta scorecard for every production agent and workflow.
11. Set hard rate limits, timeouts, and recursion caps on every MCP server and verify runaway tasks are contained. **(NSA CSI)**
12. Enable message expiry, replay protection, and idempotent handling for consequential operations. **(NSA CSI)**

## References

1. Base document: consolidated MCP Top 10 security practices (MCP01–MCP10).
2. NSA. *Model Context Protocol (MCP): Security Design Considerations for AI-Driven Automation*. U/OO/6030316-26, May 2026.
