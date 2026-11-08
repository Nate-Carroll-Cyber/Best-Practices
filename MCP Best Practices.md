# MCP Security Best Practices

This guide consolidates the actionable security practices in the supplied text. It covers the complete MCP Top 10, removes case-study narrative and repetition, and retains preventive controls, detection checks, immediate actions, and program-level guidance. Items marked **(NSA CSI)** are additions drawn from NSA, *Model Context Protocol (MCP): Security Design Considerations for AI-Driven Automation*, U/OO/6030316-26, May 2026 — including a new MCP11 availability category. Items marked **(CoSAI)** are additions drawn from the Coalition for Secure AI, *Model Context Protocol (MCP) Security* white paper, approved 8 January 2026, including a new section on deployment pattern requirements. Items marked **(MCP Spec)** are additions on tool annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`) drawn from the MCP specification and the MCP maintainers' post *Tool Annotations as Risk Vocabulary*, 16 March 2026, together with the stateless-core, caching, authorization, and deprecation changes of revision 2026-07-28. Items marked **(OBO)** are additions on on-behalf-of delegation drawn from RFC 8693, *OAuth 2.0 Token Exchange*, the Microsoft Entra Agent ID and Amazon Bedrock AgentCore Identity OBO documentation, the IETF identity chaining and Identity Assertion JWT Authorization Grant drafts, and the MCP Enterprise-Managed Authorization extension. Items marked **(Six Pillars)** are additions drawn from the CSA AI Safety Working Group draft *The Six Pillars of MCP Security* (2026 working draft), kept where a shipping implementation or a direct protocol fact backs them and marked as a target state where none does. Items marked **(mcp-gateway)** are additions drawn from the openziti/mcp-gateway code base at v0.1.11 (August 2026), where each control is implemented, and each names the test that proves it is in place.

## Core Principles

1. **Keep credentials out of the model context.** The model does not need to see a credential to use a tool. Put credential-handling middleware between the model and the protected service.
2. **Grant only temporary, task-specific authority.** Use least privilege and just-in-time, short-lived, revocable tokens instead of standing permissions. When the authority comes from a signed-in person, carry it as a delegation that names the person as subject and the agent as actor, never as an impersonation of the person. **(OBO)**
3. **Treat tool metadata and configuration as executable-risk input.** Names, descriptions, parameters, defaults, annotations, schemas, manifests, and project settings can influence agent behavior and must be verified before use.
4. **Verify every component in the trusted path.** Sign, pin, inventory, scan, and sandbox schemas, plugins, servers, SDKs, connectors, and dependencies.
5. **Never turn untrusted text into executable strings.** Use structured process APIs, parameterized queries, validated paths, and allowlisted operations.
6. **Constrain external communication.** Limit the channels through which credentials and private data can leave, and monitor unexpected outbound connections.
7. **Make changes and actions attributable.** Log permission changes and preserve per-agent action attribution so incidents can be investigated and contained.
8. **Treat retrieved content as data, never authority.** Keep system instructions separate from untrusted context and validate that planned actions remain aligned with the user’s original intent. Label each item that enters context with its origin, and give anything derived from several items the most restrictive label among them. **(Six Pillars)**
9. **Isolate every security boundary.** Enforce separation between tenants, sessions, environments, identities, and OAuth roles rather than relying on conventions or assumed ownership.
10. **Break the lethal trifecta.** For every agent, determine whether it can access private data, ingest untrusted content, and communicate externally. Remove at least one leg wherever possible.
11. **Protect availability.** Bound task complexity, recursion, and resource consumption; rate-limit and quota agent workloads so exhaustion techniques cannot stall the system. **(NSA CSI)**
12. **Bind messages to time and context.** Sign sensitive MCP messages, include expiry and replay protection, and require idempotent handling of re-delivered requests. **(NSA CSI)**
13. **Make consequential actions previewable and reversible.** Every other control can fail. A preview stage before commit and a rollback path after it bound the damage when one does. **(CoSAI)**

## MCP01 — Token Mismanagement and Secret Exposure

### Preventive controls

- Store secrets in a dedicated secrets vault or secrets manager—not in `.env` files, configuration files, prompts, or logs.
- Fetch credentials at runtime instead of placing them in environment variables or static configuration.
- Use short-lived tokens scoped to a single session or task; credentials should not outlive the work for which they were issued.
- Keep credentials out of prompts, context windows, model memory, vector stores, and other model-readable state.
- Place middleware between the LLM and credentials so the LLM never sees secret values.
- Treat a person's access token, and any refresh token held so an agent can act for that person later, as secrets under the same rule. The host validates the inbound user token at ingress and references it by handle afterward, and the on-behalf-of (OBO) exchange runs in the middleware rather than in the agent runtime. **(OBO)**
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
- Find user tokens or refresh tokens held in agent runtime memory, environment variables, or configuration rather than in the vault. **(OBO)**
- Find stdio MCP servers whose process environment contains gateway or host secrets they were never configured to receive. **(mcp-gateway)**
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
- Scope an OBO token to the intersection of what the person consented to, what the agent's registry entry declares, and what the task needs. A may_act pre-authorization names who may act for the person and says nothing about what they may do, so scope narrowing is a separate control applied at exchange time. **(OBO)**
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
- Prefer locally deployed MCP server instances when processing private data that the organization controls. Where the data already lives with a service provider in a multi-tenant deployment, prefer the server that provider hosts directly over a third-party intermediary, since the intermediary adds a party to the data path without reducing the provider's access. See Deployment Pattern Requirements. **(NSA CSI, CoSAI)**

### Detection and review

- Flag permission or scope changes that lack an audit record identifying who changed them and why.
- Identify shared agent and service accounts.
- Find scopes and tokens with no expiry.
- Find OBO tokens issued at the person's full scope rather than the intersection of consent, registry entry, and task. **(OBO)**
- Verify that testing or development changes cannot be pushed directly to production.
- Verify that every agent action can be attributed to a distinct identity.
- Review cumulative permissions, not only individual grants, to detect quiet scope growth.

### Immediate action

- Inventory every write and delete capability available to each agent and remove every capability that is not necessary.

## MCP03 — Tool Poisoning

### Preventive controls

- Digitally sign tool schemas and manifests; reject definitions that do not have a valid signature.
- Pin tool definitions at the host using trust on first use (TOFU), and flag later changes. Compare a changed definition against the pinned baseline instead of treating the change itself as an attack, because revision 2026-07-28 marks `tools/list` responses cacheable with `ttlMs` and `cacheScope` and a mid-session definition change is a normal event. Hash-baseline comparison of this kind ships in mcp-scan tool pinning and in runtime proxies such as Pipelock and Aegis. **(Six Pillars)**
- Include `annotations` in the signed and pinned definition. Quarantine a tool whose `destructiveHint` changes from `true` to `false`, or whose `readOnlyHint` changes from `false` to `true`, after pinning. A lowered risk declaration on an already-trusted tool is a rug-pull signal. **(MCP Spec)**
- Bind each tool to known provenance and a known-good version.
- Scan the **entire** tool schema—not just the tool name or description—including:
  - tool and parameter names;
  - descriptions;
  - parameter definitions;
  - default values; and
  - annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`). **(MCP Spec)**
- Treat every metadata field as instruction-grade, untrusted content because it enters model context.
- Strip ANSI escape sequences and other display-control characters from schema fields so reviewers see the actual content.
- Require RBAC for any writable schema registry.
- Require human review before schema edits are promoted to production.
- Reject or quarantine unexpected runtime schema changes.
- Reject a malformed or non-conforming tool definition at discovery one tool at a time, and keep the server's other tools available, so one bad definition neither takes the server down nor hides behind an outage. **(Six Pillars)**
- Page `tools/list` to the last `nextCursor` before applying any allow or deny filter, and fail the backend when a page request errors, because a filter applied to the first page alone leaves every tool on later pages unfiltered and reachable by name. openziti/mcp-gateway shipped that defect until v0.1.8. Test it with a backend that advertises more tools than one page holds and confirm the gateway's exposed tool count equals the backend's advertised count. **(mcp-gateway)**
- Re-prompt for informed user consent whenever a connected server’s capabilities or data access change; never let an already-trusted server expand silently. When a pinned definition changes, show the approved text and the current text together and require the person's acknowledgment before the changed tool runs again. Current clients prompt once per server at first approval and do not prompt again on a definition change, so this is a requirement to place on the client rather than a behavior to assume. **(NSA CSI, Six Pillars)**
- Do not assume a user approval step is sufficient if the model has already consumed untrusted tool metadata; validate metadata before it reaches the model.
- Isolate tools so one tool cannot silently manipulate another tool into accessing or exporting unrelated private data. Namespace every exposed tool as the backend identifier, a separator, and the original tool name, so two servers that both expose `read_file` cannot shadow each other and the call names the server that receives it. openziti/mcp-gateway does this with `_` as the default separator, giving `filesystem_read_file`. The prefix is unambiguous only when the separator cannot appear inside a backend identifier, and mcp-gateway does not enforce that, so backend `fs` with tool `x_y` and backend `fs_x` with tool `y` both become `fs_x_y` and the later registration silently replaces the earlier one in `aggregator/namespace.go`. Reject backend identifiers that contain the separator and fail startup on any post-prefix collision instead of overwriting. Test it with two backends whose prefixed names collide and confirm the gateway refuses to start. **(mcp-gateway)**
- Design each tool for a single purpose with explicit boundaries. A "do anything" tool cannot be scoped, allowlisted, or reasoned about, and its schema is the largest possible poisoning surface. **(CoSAI)**
- Set `destructiveHint` to the worst outcome any accepted argument can produce. A tool with a `mode`, `force`, or `overwrite` parameter that can delete or overwrite carries `destructiveHint: true`, or is split into separate additive and destructive tools so each hint is exact. **(MCP Spec)**

### Detection and review

- Identify schemas fetched without signature or integrity verification.
- Check whether a schema registry is writable without RBAC.
- Flag schema edits that are automatically promoted.
- Detect hosts that accept runtime schema changes without review.
- Find tools that lack provenance or version binding.
- Alert whenever the MCP tool list or any tool definition changes.
- Treat a tool rewriting its own definition as a security signal.
- Find clients that re-verify a changed definition silently and run the tool without showing the person what changed. **(Six Pillars)**
- Find gateways that apply tool filters before paging `tools/list` to completion. **(mcp-gateway)**
- Find gateways that overwrite rather than refuse when two namespaced tool names collide. **(mcp-gateway)**
- Find tools whose description or parameter set spans several unrelated capabilities. **(CoSAI)**
- Find tools whose name, description, or backing API call uses a delete, drop, overwrite, truncate, or purge verb while `destructiveHint` is `false` or absent. **(MCP Spec)**
- Run each tool in a sandbox and compare observed side effects against its declared hints. A tool declared `readOnlyHint: true` that writes, or `destructiveHint: false` that deletes or overwrites, is a finding. **(MCP Spec)**

### Immediate action

- Pin and sign every tool schema, then scan all names, descriptions, parameters, and defaults.

## MCP04 — Supply Chain and Dependency Tampering

### Preventive controls

- Require signed components with verifiable provenance; do not run components whose authenticity or origin cannot be established. Verify the signature in the MCP client before the server is loaded, not after it is already serving tools. **(CoSAI)**
- Maintain a complete software bill of materials (SBOM) for every MCP server and plugin.
- Pin every dependency, plugin, and MCP server to a reviewed, known-good version; do not use `latest` or other floating versions.
- Upgrade deliberately only after review.
- Avoid fetching dependencies dynamically at runtime. Build and deploy a fixed, vetted dependency set.
- Scan dependencies automatically in CI before release.
- Verify maintenance status before adoption: many popular MCP servers are archived or unmaintained; prefer reference implementations and actively supported projects, and apply the most stringent code-audit profile to newer integrations. **(NSA CSI)**
- Keep MCP development and test tooling patched — e.g., the MCP Inspector RCE (CVE-2025-49596) fixed in v0.14.1. **(NSA CSI)**
- Sandbox third-party plugins so a compromised component cannot compromise the entire environment.
- Monitor and restrict plugin network access, especially calls to unknown domains.
- Configure the gateway's own HTTP clients to backends so they do not follow redirects and do not read `HTTP_PROXY`, `HTTPS_PROXY`, or `ALL_PROXY` from the environment, and require an explicit per-backend opt-in for either. A backend that answers a tool call with a 302 to an attacker host otherwise moves the gateway's request, `Authorization` header included, to that host, and an inherited proxy variable routes every call through a host the operator never approved. openziti/mcp-gateway refuses both by default since v0.1.8 and exposes `allow_redirects` and `allow_environment_proxy` for the exceptions. Test it by pointing a backend at a server that returns a 302 and confirming the gateway returns an error rather than following it. **(mcp-gateway)**
- Inventory MCP servers running on developer machines.
- Bind local development servers and tools to `127.0.0.1`/localhost rather than `0.0.0.0` or all network interfaces.
- Require authentication and origin checks for local web interfaces and reject unauthorized cross-origin requests.
- Use `stdio` transport for local servers. A pipe has no network listener, which removes DNS rebinding and cross-origin attacks against the local server entirely. Reserve HTTP streaming transport for servers that must be reached remotely, and apply the remote transport controls under Deployment Pattern Requirements. **(CoSAI)**

### Detection and review

- Find unsigned plugin installations.
- Identify dependencies fetched at runtime.
- Verify that each server and plugin has a complete SBOM.
- Find dependencies using `latest`, ranges, or other floating versions.
- Monitor plugins for connections to unknown or unapproved domains.
- Check local MCP servers for unauthenticated interfaces, permissive origin handling, and network-wide binding.
- Find local servers exposed over HTTP where `stdio` would serve. **(CoSAI)**
- Find gateway backend clients that follow redirects or inherit proxy variables. **(mcp-gateway)**
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
- Validate and constrain file paths to prevent traversal (for example, `../`) and access outside approved directories. Enforce the same constraint in the gateway, where the server cannot bypass it. For each tool that takes a path, name the argument and the allowed absolute roots in the gateway configuration, have the gateway resolve symlinks, refuse `..` segments and unresolvable paths, and return a tool-level denial without calling the backend when the resolved path falls outside every root. openziti/mcp-gateway implements this as `policy.paths` entries with `tool`, `argument`, and `roots`, and requires each root to be an existing directory at startup. Test it with a symlink inside an allowed root that points outside it and confirm the gateway denies the call. **(mcp-gateway)**
- Parameterize SQL queries; never interpolate values into SQL strings.
- Apply context-aware output encoding for every sink a tool writes to, including HTML and other markup rendered to a user or another agent, not only SQL and shell. **(CoSAI)**
- Allowlist permitted commands, verbs, and operations so destructive or unexpected actions cannot run.
- Enforce strict schema validation on all deserialized context, configuration, and payload objects; maintain hard isolation between code and data during deserialization (classic insecure-deserialization risk applied to MCP’s serialized context). **(NSA CSI)**
- Parse each tool call's arguments exactly once at the gateway, reject the call when the JSON carries a duplicate key or more than one top-level value, run every policy check against that parsed object, and forward the same parsed object to the backend rather than the original bytes. Two parsers with different duplicate-key handling otherwise let `{"path":"/safe","path":"/etc/shadow"}` pass the gateway check as `/safe` and reach the server as `/etc/shadow`. openziti/mcp-gateway does this in `aggregator/policy.go` with a decoder that rejects duplicate keys and returns the settled map for dispatch. Test it by sending a duplicate-key argument object through the gateway and confirming a policy error with no backend call. **(mcp-gateway)**
- Block or restrict parameter forwarding when the origin of the data is ambiguous or potentially user-supplied, so inputs intended for one component cannot be reused by another. **(NSA CSI)**
- Sandbox local MCP servers and run agents as non-root, without administrator or unrestricted `sudo` privileges. Start each stdio server with an explicit environment and nothing inherited from the gateway or host process, because any process in the spawned tree can read the parent's variables from `/proc/<pid>/environ` on Linux. openziti/mcp-gateway exposes this as `transport.env_policy: closed`, and its default `additive` inherits everything. Verify by reading `/proc/<backend pid>/environ` from inside the backend's sandbox and confirming only the configured keys appear. **(mcp-gateway)**
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
- Find gateways that forward the client's original argument bytes rather than the object the policy evaluated. **(mcp-gateway)**
- Map execution findings to CWE-77, CWE-78, CWE-94, and CWE-95 for vulnerability management. **(NSA CSI)**

### Immediate action

- Search the codebase for `child_process.exec`, `os.system`, and `shell: true`.
- Treat each occurrence as a finding, review its data flow, and replace unsafe string-based execution with structured argument-array calls.

## MCP06 — Intent Flow Subversion

### Preventive controls

- Anchor the user’s original objective in the system prompt as a fixed, authoritative reference.
- Treat all retrieved pages, documents, issues, files, code comments, and tool output as untrusted data—not instructions.
- Tag or fence retrieved content explicitly as untrusted context before it reaches the planning model.
- Label every item that enters context with the server it came from, the principal that authorized the fetch, whether it came from the open world, and whether the person typed it, and give anything the model derives from several inputs the most restrictive label among them, so a summary of a web page, a plan built from it, a tool argument, and a state handle all carry the page's untrusted label. Microsoft Purview applies the most restrictive source sensitivity label to Copilot and Agent 365 output, and Microsoft Agent Framework ships FIDES as experimental middleware that propagates the most restrictive input label to outputs and blocks a tool call while untrusted content is in scope. The first FIDES release accepted a `security_label` key inside attacker-supplied data as an authoritative trust label, so take a label only from the middleware's own record of where content came from and never from a field inside the content. **(Six Pillars)**
- Treat lowering a label as a target-state control with no current implementation. Where a host supports it, record each downgrade as a scoped, expiring declassification that names the authorizing person, and refuse any session-wide or blanket downgrade. **(Six Pillars)**
- Keep trusted system instructions structurally separate from retrieved content; do not merge them into an undifferentiated prompt.
- Validate that every planned or requested action remains aligned with the user’s original intent.
- Add a gate between planning and consequential actions so the agent cannot move directly from reading context to deleting, sending, modifying, or exporting.
- Use a two-stage commit for high-impact actions. The first stage produces a draft or preview with a draft ID and no side effects. The second stage commits only on explicit confirmation of that specific draft. Where the target supports it, bind the commit to a time window so a stale confirmation cannot execute a changed action. **(CoSAI)**
- Let tool annotations only raise a gate. `destructiveHint: true` routes the call to the preview and confirm stage. `destructiveHint: false` or `readOnlyHint: true` skips a confirmation only when the server is in the trusted registry, because the spec requires clients to treat annotations from untrusted servers as untrusted and a server can declare `readOnlyHint: true` and still delete. **(MCP Spec)**
- Treat a tool with no annotations as destructive, non-idempotent, and open-world, matching the spec defaults (`destructiveHint` defaults to `true`). Verify the client applies those defaults, since clients differ in how strictly they honor them. **(MCP Spec)**
- Provide a rollback or undo path for every consequential tool, using snapshots, reversible operations, or retained drafts. A tool with no undo should carry the highest gate. **(CoSAI)**
- Use an independent guardrail model or control outside the primary agent’s context to detect intent drift and mismatched actions. The guardrail advises. It never authorizes. Authorization decisions rest on deterministic policy and verified identity, never on the judgment of any model, primary or guardrail. Calibrate the guardrail the way an evaluator is calibrated. Hold a labeled reference set of injected and clean traces, run the guardrail against it on a schedule, track false positive and false negative rates, and re-baseline whenever the guardrail model or its prompt changes, since a prompt change alone can move the false positive rate with nothing changed in the agent under test. LangSmith's Align Evaluator and Braintrust scorer alignment run this loop for quality judges against human-labeled examples, and the same loop applies to a security guardrail. **(CoSAI, Six Pillars)**
- Do not rely solely on human approval for safety. Approvers fatigue and start confirming by habit. Pair confirmation prompts with hard authorization boundaries, keep the prompt rate low enough that each one gets read, and make security-relevant prompts state the consequence plainly. Where the server can request confirmation itself through MCP elicitation, do so from the server, so the check does not depend on the client's implementation. **(CoSAI)**
- On trusted servers, set `readOnlyHint: true` on read-only tools and `destructiveHint: false` on additive tools so confirmation prompts fire only for calls that can delete or overwrite. Accurate hints are what keep the prompt rate low enough to be read. **(MCP Spec)**
- Re-anchor the original intent during long-running sessions and multi-step workflows.
- Treat context preconditioning as an attack vector: probabilistic interpretation of prior context means an actor can seed context history to steer the agent toward a specific unsafe outcome; constrain how much prior context can influence consequential decisions. **(NSA CSI)**
- Validate every tool result in three steps before it reaches the model. First, confirm the message is well-formed JSON-RPC 2.0 for the protocol version the request declared in `_meta`, accept every legal shape including `isError: true` and unstructured content, and reject anything malformed as a protocol error. Second, where the tool declares an `outputSchema`, validate `structuredContent` against it and withhold the result on mismatch as a server fault, and treat a tool with no `outputSchema` as conforming, since the schema is optional and a blanket reject-on-mismatch rule rejects conforming servers. Third, apply the content controls in this section regardless of the first two steps, because schema-valid content still carries prompt injection. The official SDKs perform the second step when a schema is declared. **(Six Pillars)**
- Inspect and filter each tool’s output before passing it to the next component in a chained pipeline — content length checks, disallowed-keyword scanning, and detection of injected prompts or code elements that could alter downstream behavior. **(NSA CSI)**
- Treat a server's sampling request, which asks the client to run model inference for the server, as untrusted input that shapes the prompt and parameters of that inference. Revision 2026-07-28 deprecates sampling under SEP-2577 and replaces server-initiated `sampling/createMessage` with the multi round-trip `input_required` result, while deprecated sampling keeps working for at least twelve months, so require the person's approval of each sampling request, log the prompt it carries, and refuse sampling from servers outside the trusted registry for the rest of the window. **(MCP Spec, Six Pillars)**
- Avoid giving one agent or credential simultaneous access to untrusted public content, sensitive private data, and unrestricted outbound actions.
- Break at least one leg of the private-data/untrusted-content/external-communications trifecta. Score the legs across every server the agent can reach together, since read access on one server and write access on another form a copy-out path that neither server sees on its own. mcp-scan and mcp-scrutiny report these cross-server capability pairs at scan time. Blocking the aggregate at runtime as a scope violation is a target state with no current gateway implementation. **(Six Pillars)**

### Detection and review

- Find workflows with no intent-alignment validation.
- Identify agents that can treat retrieved text as instructions.
- Identify “blind planning” paths that execute immediately after retrieving context.
- Find high-impact tools that execute in a single call with no preview stage, and consequential tools with no rollback path. **(CoSAI)**
- Find approval flows that fire often enough that approvers no longer read them. **(CoSAI)**
- Find clients that auto-approve calls on the strength of `readOnlyHint: true` or `destructiveHint: false` from servers outside the trusted registry, and clients that run unannotated tools without confirmation. **(MCP Spec)**
- Find clients that reject results from tools with no `outputSchema`, and clients that pass `structuredContent` to the model after it fails a declared schema. **(Six Pillars)**
- Find guardrail or judge models with no labeled reference set and no recorded false positive and false negative rates. **(Six Pillars)**
- Find servers that still issue sampling requests and clients that fulfill them without per-request approval. **(MCP Spec, Six Pillars)**
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
- Require mutual authentication between clients, agents, tools, and servers; use mTLS where appropriate. For workload-to-workload identity, a SPIFFE/SPIRE deployment issues and rotates the short-lived cryptographic identities (SVIDs) that mutual authentication needs, without long-lived certificates in configuration. **(CoSAI)**
- Validate every token server-side, on every request, for authenticity, validity, expiry, intended resource, and authorization for the specific action.
- Make authorization decisions server-side from verified facts, never from client-supplied identity, roles, or scopes.
- Classify each tool's destructiveness in the gateway or registry record and enforce from that record. The server's declared `destructiveHint` is an input to review, never the authorization fact, and a tool with no registry classification fails closed to destructive. **(MCP Spec)** Refuse gateway startup when a policy or classification record names a tool the backend does not advertise, so a renamed tool or a typo in the record cannot leave a tool running under no policy. openziti/mcp-gateway's `ValidateTools` fails startup on that case with the tool name in the error. Test it by renaming one tool in the record and confirming the gateway exits rather than serving. **(mcp-gateway)**
- Enforce scope checks for every tool and resource request.
- Use OAuth resource indicators to bind a token to its intended target server and prevent cross-resource replay.
- Validate the `iss` parameter on every authorization response before redeeming a code, as revision 2026-07-28 requires under RFC 9207, and treat client credentials as bound to the authorization server that minted them, so a credential registered with one issuer is refused by every other. Prefer Client ID Metadata Documents over Dynamic Client Registration, which the revision deprecates. **(MCP Spec)**
- Check that the `Mcp-Method` and `Mcp-Name` headers on each Streamable HTTP request match the JSON-RPC body, since gateways and policy engines authorize on the headers before parsing the body and a mismatch is a smuggling path. **(MCP Spec)**
- Never pass a client token through to a downstream service.
- Use token exchange or an OBO flow so an agent receives its own narrowly scoped downstream token that names the person as subject and the agent as actor. Send the agent's own actor credential with the exchange. RFC 8693 states that delegation is impossible with only a subject token, so an exchange that omits the actor token produces impersonation, and agent clients should be refused it. **(OBO)**
- Pre-authorize which agents may act for a person through the may_act claim in the person's token or an equivalent authorization-server policy, and have the authorization server refuse every other requester. **(OBO)**
- Check both authorities at the resource on every request. The action must be permitted for the person and permitted for the acting agent, and either check failing denies. Decide on the current actor's own identity and scope, since RFC 8693 treats prior actors in a nested act claim as informational, which makes the chain an audit record and not a permission. **(OBO)**
- When an MCP server or gateway exchanges a token it received from an upstream agent, it adds itself to the actor chain rather than replacing the agent, so the downstream API sees the person, the agent, and the server. **(OBO)**
- Record consent for an agent to act for a person as an expiring, revocable grant with grantor, scope, and review date, and revoke outstanding exchanged tokens within the published latency when the person's session ends, consent is withdrawn, or the agent identity is disabled. **(OBO)**
- Hold a refresh token or other standing grant for acting while the person is absent in the vault, used by the middleware only, bound to one agent and one person, with its own expiry and a distinct audit event on each use. Microsoft Entra issues the agent identity a refresh token for asynchronous scenarios, and it carries password-grade sensitivity. **(OBO)**
- Apply the same requirements across dialects. RFC 8693 carries the agent in actor_token and records it in the act claim. Microsoft Entra's agent OBO flow uses the RFC 7523 jwt-bearer grant with requested_token_use set to on_behalf_of, identifies the agent by its client assertion and fmi_path, and has no actor_token parameter. Amazon Bedrock AgentCore Identity drives either dialect from the credential provider configuration rather than from agent code. Where a dialect cannot place the agent in the token as a distinct claim, record the actor at the gateway and declare the gap in the inventory. **(OBO)**
- For an MCP server in another trust domain, use the MCP Enterprise-Managed Authorization extension, which has the client exchange its SSO identity assertion at the enterprise IdP for an Identity Assertion JWT Authorization Grant and present that grant to the server's authorization server under RFC 7523, so consent moves to IdP administrator policy and the client holds no per-server refresh token. Servers declare the extension in their authorization metadata and clients declare it in their capabilities. **(OBO)**
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
- Find OBO exchanges that send a subject token with no actor credential, and downstream tokens that name the agent as subject for an action taken for a person. **(OBO)**
- Find resources that check only the person's authority or only the agent's before acting on a delegated token. **(OBO)**
- Alert when one person's token is exchanged by two agents, when an exchange comes from an agent not pre-authorized for that person, when an exchanged token is used after consent or session revocation, and on the first consent pairing a person with an agent. **(OBO)**
- Review OAuth implementations for combined client/authorization-server roles, shared client IDs, redirect weaknesses, token passthrough, and missing resource binding.
- Test message handling for replay acceptance and duplicate re-execution of consequential operations. **(NSA CSI)**

### Immediate action

- Audit every downstream call made by agents and flag any flow that forwards the client’s token unchanged or exchanges it without the agent's actor token.

## MCP08 — Lack of Audit and Telemetry

### Preventive controls

- Record agent activity as structured, queryable, tamper-evident logs.
- State for each evidence type who relies on it, whether internal audit, an external auditor, a regulator, or a counterparty, what that party must trust to accept it, such as the producing host, the log pipeline, or key custody, and whether a party that trusts none of those can verify the record offline from its signatures alone. RFC 9334 and SLSA define the verifier and relying-party roles this statement follows. **(Six Pillars)**
- Capture every tool call with, at minimum:
  - verified identity;
  - tool name;
  - parameters or safe parameter metadata; and
  - timestamp.
- Where feasible, record cryptographic hashes of tool results and outputs to support forensic reconstruction. **(NSA CSI)**
- Record the person on whose behalf a call ran and the actor chain, and whether the person was present or the call ran on a standing grant, so a reviewer can separate what the person did from what an agent did in the person's name. Red Canary's 2026 Entra Agent ID detection walkthrough traced an agent identity sending mail to an external recipient on behalf of a user through delegated Mail.Send, and the activity was attributable only from the agent identity and the consent grant, not from the user's own activity log. **(OBO)**
- Log the consent grant that lets an agent act for a person as a security event with grantor, scope, and expiry. **(OBO)**
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
- Find records where an agent's OBO action appears only under the person's identity with no actor recorded. **(OBO)**
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
- Revalidate session and tenant ownership on every request; never trust cached or implicit session-to-user mappings. Revision 2026-07-28 removes the protocol session and `Mcp-Session-Id`, and a server that needs state across calls returns an explicit handle from a tool that the model passes back as an argument, so a handle for a file, a job, a resource, or saved state is a plain string that any caller who learns it can replay. Bind each handle at issue time to the person and agent that received it, check that binding on every use, and revoke it the way a stolen token is revoked. No MCP client or gateway does this today, so treat the binding as a target state and keep handles short-lived and unguessable until it exists. **(Six Pillars)**
- Avoid shared singleton state, shared context buffers, or shared vector stores unless isolation is explicitly enforced and tested.
- Classify and tag sensitive data as it enters context.
- Apply policy at ingestion to block, mask, minimize, or restrict sensitive content before it is stored.
- Minimize what a tool returns into context. Return only the fields the task needs, and redact PII and other sensitive values in the tool before the result reaches the model. What never enters context cannot be leaked, logged, or summarized onward. **(CoSAI)**
- Give every context item a time-to-live and enforce automatic purging.
- Treat persistent context as a governed data store with access controls, classification, retention, and deletion requirements.
- Prevent injected content in shared memory from becoming instructions in later sessions.
- Add automated tenant-isolation testing to CI, and run it through every proxy, bridge, and local HTTP shim in the path rather than against the server alone. openziti/mcp-gateway's `mcp-tools http` served every local agent from one remote MCP session until v0.1.11, so two agents behind the same shim shared backend state while the gateway behind it was isolating correctly. The test opens two client sessions through the same hop and asserts that the far side reports two distinct session identifiers and two separate backend processes. **(mcp-gateway)**

### Detection and review

- Inventory shared context buffers and vector stores used by multiple users, tenants, agents, or workflows.
- Check whether context persists across users or sessions.
- Find reused context or server-side state whose ownership is not revalidated on every request.
- Find state handles that any authenticated caller can replay regardless of who received them. **(Six Pillars)**
- Identify sensitive data that enters context without classification or tags.
- Find tools that return whole records, full rows, or raw API responses where a named subset of fields would serve. **(CoSAI)**
- Find context with no TTL, expiry, or automatic purge.
- Test timing- and concurrency-dependent cross-tenant leakage conditions.

### Immediate action

- Write and run a tenant-isolation test that accesses tenant A and then verifies tenant B can see none of tenant A’s data. Run it through each intermediary in the path, and add the test to CI for every build.

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

## Deployment Pattern Requirements **(CoSAI)**

The controls above apply to every deployment. Several of them are conditional on how the server is reached and who shares it. This section states which controls become mandatory under each pattern, so a reviewer can tell whether a control is missing or simply not applicable.

### All-local (one host, one user)

- Security rests on the posture of the host. A compromised host means a compromised server, and no MCP control changes that.
- Use `stdio` transport. It removes the network listener and with it DNS rebinding and cross-origin attacks (MCP04).
- Sandbox the server and run it as non-root so a poisoned tool cannot escalate on the host (MCP05).
- Put a stdio policy shim between the client and each local server so allow-list tool filtering and path rules apply before the server sees the call, with `tools.mode: allow` naming the read tools to expose and `policy.paths` binding each path argument to its roots (MCP02, MCP05). openziti/mcp-gateway's `aggregator` package implements both and serves on stdio through its embedding API, while the shipped `mcp-gateway run` binary serves only over zrok or Agora and `mcp-bridge` exposes a local server to the network rather than filtering it, so a local-only deployment needs a small wrapper around the package or an equivalent shim. **(mcp-gateway)**
- Appropriate for development and personal use. Not appropriate for anything that handles another person's data.

### Single-tenant remote (one organization, reached over the network)

- Authentication between client and server is mandatory, not optional, and all traffic is encrypted (MCP07).
- Store client credentials in secure storage such as the OS keychain or a secrets manager, never in configuration files (MCP01).
- A remote server that calls further services for the person performs its own OBO exchange with its own actor credential and never forwards the token it received (MCP07). **(OBO)**
- Enterprise clients enforce authenticated server discovery against an explicit allowlist. A server absent from the allowlist is unreachable, not merely unapproved (MCP09).
- HTTP streaming transport carries the full remote control set. Payload size limits and recursive-payload rejection, rate limiting on tool calls and transport requests, client and server authentication, mutual TLS, CORS protection, CSRF protection, and integrity checks against replay, spoofing, and poisoned responses (MCP07, MCP11).

### Multi-tenant remote (several organizations share the server)

- Everything in single-tenant remote, plus infrastructure-enforced tenant isolation with per-tenant encryption and role-based access control tested in CI (MCP10).
- Prefer the server hosted directly by the service provider that already holds the tenant data. A third-party intermediary adds a party to the data path without reducing the provider's access (MCP02).
- Provide remote attestation where the platform supports it, so a client can verify that the server is running the expected code before trusting it. Treat this as a target state rather than a baseline, since attestation support across MCP hosting is uneven.

## Consolidated Security Review Checklist

### Secrets and identity

- [ ] Secrets are stored in a vault and fetched only at runtime.
- [ ] No secret enters model context, prompts, configuration, environment variables, or logs.
- [ ] Logs redact token-like values.
- [ ] Tokens are short-lived, task-scoped, and revocable.
- [ ] Person tokens and refresh tokens are handled by middleware and never held by the agent runtime. **(OBO)**
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
- [ ] Signatures are verified in the client before a server is loaded. **(CoSAI)**
- [ ] Full schema content—including parameters and defaults—is scanned before model exposure.
- [ ] ANSI/display-control characters are stripped during review.
- [ ] Schema registries enforce RBAC and reviewed promotion.
- [ ] Tool-list and runtime schema changes trigger alerts.
- [ ] Capability or data-access changes on a connected server trigger renewed user consent. **(NSA CSI)**
- [ ] A changed tool definition is compared to its pinned baseline and shown to the person for acknowledgment before the tool runs again. **(Six Pillars)**
- [ ] Tool discovery pages to completion before filters apply, and a policy that names an unadvertised tool fails startup. **(mcp-gateway)**
- [ ] Exposed tools are namespaced by backend, the separator is barred from backend identifiers, and post-prefix collisions fail startup. **(mcp-gateway)**
- [ ] Every tool is single-purpose with explicit boundaries. **(CoSAI)**
- [ ] Tool annotations are signed and pinned with the schema, set to the worst-case argument path, and a lowered risk hint quarantines the tool. **(MCP Spec)**

### Supply chain and networking

- [ ] Every server and plugin has a complete SBOM.
- [ ] Dependencies and servers are pinned; no floating versions are used.
- [ ] Dependency scanning runs in CI.
- [ ] Third-party plugins are sandboxed.
- [ ] Unexpected outbound domains are blocked or alerted on.
- [ ] Gateway backend clients refuse redirects and environment proxies unless a backend opts in. **(mcp-gateway)**
- [ ] Outbound traffic flows through a filtering egress proxy or DLP pinned to approved resource URLs and methods. **(NSA CSI)**
- [ ] Maintenance status is verified; archived/unmaintained servers are not adopted, and MCP CVEs are formally tracked. **(NSA CSI)**
- [ ] Local services bind only to localhost and enforce authentication and origin checks.
- [ ] Local servers use `stdio`; remote servers carry the full HTTP transport control set. **(CoSAI)**

### Execution safety

- [ ] Untrusted values never enter shell command strings.
- [ ] Processes are launched with structured argument arrays.
- [ ] Option parsing is terminated with `--` where supported.
- [ ] Generated code is not executed without validation and authorization.
- [ ] File paths are normalized and constrained to approved roots.
- [ ] SQL queries are parameterized.
- [ ] Output encoding is context-aware for every sink, including HTML. **(CoSAI)**
- [ ] Commands and operations are allowlisted.
- [ ] Deserialization is schema-validated with hard code/data isolation. **(NSA CSI)**
- [ ] The gateway parses arguments once, rejects duplicate keys, and forwards the parsed object it checked. **(mcp-gateway)**
- [ ] Stdio servers start with a closed environment and path rules are enforced at the gateway. **(mcp-gateway)**
- [ ] Tool execution is confined by OS-level sandboxing (seccomp, AppArmor, SELinux, AppContainers). **(NSA CSI)**
- [ ] Agents do not run with unnecessary root or administrator privileges.

### Intent integrity

- [ ] The user’s objective is anchored in trusted system instructions.
- [ ] Retrieved content is explicitly marked and handled as untrusted data.
- [ ] Context items carry origin labels, and derived content inherits the most restrictive label of its inputs. **(Six Pillars)**
- [ ] System instructions and retrieved content remain structurally separate.
- [ ] Consequential actions require an intent-alignment check.
- [ ] High-impact actions use a two-stage commit with a preview and a bound confirmation. **(CoSAI)**
- [ ] Annotations only raise a gate. Hints from untrusted servers never skip confirmation, and unannotated tools are treated as destructive. **(MCP Spec)**
- [ ] Every consequential tool has a rollback or undo path. **(CoSAI)**
- [ ] An independent guardrail detects intent drift and never authorizes. **(CoSAI)**
- [ ] Human approval is paired with hard boundaries and kept at a rate approvers actually read. **(CoSAI)**
- [ ] Chained tool outputs are inspected before reaching the next component. **(NSA CSI)**
- [ ] Tool results pass envelope validation, declared-schema validation where an `outputSchema` exists, and content controls, in that order. **(Six Pillars)**
- [ ] The guardrail is calibrated against a labeled reference set with tracked false positive and false negative rates. **(Six Pillars)**
- [ ] Sampling requests require per-request approval and are refused from servers outside the trusted registry. **(MCP Spec, Six Pillars)**
- [ ] Every production agent has a current trifecta scorecard.

### Authentication and authorization

- [ ] Clients, agents, tools, and servers mutually authenticate.
- [ ] Every token is validated server-side on every request.
- [ ] Authorization is enforced server-side using verified identity and scope.
- [ ] Tool destructiveness is classified in the gateway or registry, not taken from the server's declared hint. **(MCP Spec)**
- [ ] Tokens are bound to their intended resource.
- [ ] Authorization responses are issuer-validated and client credentials are issuer-bound. **(MCP Spec)**
- [ ] Client tokens are never passed through downstream.
- [ ] Downstream access uses token exchange or OBO delegation that names the person as subject and the agent as actor, never impersonation. **(OBO)**
- [ ] OBO exchanges are pre-authorized per agent and person, scoped to the intersection of consent, registry entry, and task, and checked for both authorities at the resource. **(OBO)**
- [ ] Consent for an agent to act for a person is recorded, expiring, and revocable, and standing grants for acting while the person is absent are vaulted, bound to one agent and one person, and audited per use. **(OBO)**
- [ ] Sensitive messages carry payload signatures, expiry, and replay protection; consequential operations are idempotent. **(NSA CSI)**
- [ ] Logs correlate actions to verified user and agent identities.

### Audit and response

- [ ] Tool-call logs are structured, queryable, centralized, and tamper-evident.
- [ ] Every tool call records identity, tool, safe parameter metadata, and time.
- [ ] Records carry the person acted for and the actor chain, so an agent's OBO action is never shown as the person's own. **(OBO)**
- [ ] Relevant prompt/context changes and outbound events are observable.
- [ ] Sensitive log content is masked.
- [ ] MCP telemetry is integrated with the SIEM or XDR.
- [ ] Behavioral baselines and anomaly alerts are active.
- [ ] Protocol anomalies (malformed/repeated requests, authorization failures, RBAC violations) generate native alerts. **(NSA CSI)**
- [ ] Telemetry cannot be silently disabled or rewritten.
- [ ] Incident-response and telemetry drills are performed regularly.
- [ ] A safe state is defined per incident class, with a recovery gate before normal operation resumes. **(Six Pillars)**
- [ ] Each evidence type names its relying party, what that party must trust, and whether it verifies offline. **(Six Pillars)**

### Inventory and governance

- [ ] Every MCP server is registered, inventoried, reviewed, and assigned an owner.
- [ ] Unregistered servers fail deployment or access.
- [ ] Continuous discovery scans networks, repositories, hosts, and CI/CD.
- [ ] MCP-aware scanners run differential scans to catch dynamic port changes and modified services. **(NSA CSI)**
- [ ] Agents calling unknown endpoints trigger alerts.
- [ ] Secure-by-default deployment templates are available.
- [ ] Every server uses central identity through SSO or OIDC.
- [ ] Network segmentation limits the reach of unknown services.
- [ ] Each deployment is classified as all-local, single-tenant remote, or multi-tenant remote, and carries that pattern's mandatory controls. **(CoSAI)**

### Context and tenant isolation

- [ ] Context is ephemeral and scoped to one session.
- [ ] Tenant, user, agent, and workflow namespaces are infrastructure-enforced.
- [ ] Ownership is revalidated on every request.
- [ ] Sensitive context is classified and tagged at ingestion.
- [ ] Tool results are minimized to needed fields and redacted before entering context. **(CoSAI)**
- [ ] Context has enforced TTLs and automatic purging.
- [ ] Tenant-isolation tests run in CI and through every proxy hop. **(mcp-gateway)**

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
   - Per-user delegation as OBO, with the person as subject and the agent as actor. **(OBO)**
   - Every request tied to a verified identity with minimum privilege for minimum time.
2. **Isolation and lifecycle**
   - Tenant boundaries are tested, not assumed.
   - Workloads run in containers or equivalent isolation.
   - Services run as non-root and bind locally when remote exposure is unnecessary.
   - Context and credentials expire automatically.
   - Resource ceilings, rate limits, and timeouts bound every workload. **(NSA CSI)**
   - The deployment pattern is declared and its mandatory controls are present. **(CoSAI)**
3. **Trusted, controlled tooling**
   - Every server is inventoried.
   - Tools are vendor-built or independently security-reviewed.
   - Components are signed, pinned, provenance-verified, and continuously revalidated.
   - Tools are single-purpose and return only the fields the task needs. **(CoSAI)**
4. **Schema-driven validation**
   - Inputs and outputs are validated at the MCP boundary.
   - Tool results pass envelope validation always and declared-schema validation only where an `outputSchema` exists. **(Six Pillars)**
   - Retrieved text is always treated as data, never as a command.
   - Sensitive messages are signed, time-bound, and replay-protected; deserialization is schema-enforced. **(NSA CSI)**
5. **Hardened deployment and oversight**
   - Structured tool-call logs flow to a SIEM or XDR.
   - Behavioral baselines and anomaly alerts are active.
   - High-impact actions preview before they commit and can be rolled back after. **(CoSAI)**
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
   - Use OAuth 2.1, just-in-time tokens, and OBO delegation that names the person as subject and the agent as actor.
   - Tie every action to a real identity with minimum privilege for minimum time.
4. **Defense in depth**
   - Combine gateways, runtime policies, tool guardrails, network controls, and container isolation.
   - Design each layer to contain failures that pass through an earlier layer.
5. **Detection and response**
   - Maintain structured audit pipelines and anomaly detection.
   - Track and patch MCP-related CVEs as a formal, inventory-linked process. **(NSA CSI)**
   - Exercise compromise scenarios: determine who is paged, where the logs are, and how the kill switch is activated.
   - Define a safe state per incident class instead of one read-only fallback. For a confidentiality event, block reads and egress, since reads are the harm during exfiltration. For identity compromise, revoke credentials and sessions. For context poisoning, reset context and quarantine the source, since continued retrieval spreads the poison. End each with side-effect reconciliation and an explicit recovery gate before normal operation resumes. **(Six Pillars)**

## Immediate-Priority Checklist

1. Audit every MCP source: determine whether each server is vendor-built or from an unreviewed repository.
2. Move static API keys to a vault and scan logs for token-like values.
3. Inventory all agent write and delete capabilities and remove those that are unnecessary.
4. Validate inputs at the MCP boundary, validate tool results in the three-step order, and ensure retrieved text is never treated as a command.
5. Search for string-interpolated shell calls and replace them with structured process APIs.
6. Audit each agent’s effective production permissions—not only its documented permissions.
7. Confirm that client tokens are never passed through to downstream services and that every OBO exchange carries the agent's actor token. **(OBO)**
8. Enable structured tool-call logging with identity, tool, safe parameter metadata, and time.
9. Run the tenant-isolation test and add it to CI.
10. Build a trifecta scorecard for every production agent and workflow.
11. Set hard rate limits, timeouts, and recursion caps on every MCP server and verify runaway tasks are contained. **(NSA CSI)**
12. Enable message expiry, replay protection, and idempotent handling for consequential operations. **(NSA CSI)**
13. Add a preview stage and a rollback path to every high-impact tool, and confirm no authorization decision rests on a model's judgment. **(CoSAI)**

## References

1. Base document: consolidated MCP Top 10 security practices (MCP01–MCP10).
2. NSA. *Model Context Protocol (MCP): Security Design Considerations for AI-Driven Automation*. U/OO/6030316-26, May 2026.
3. Coalition for Secure AI. *Model Context Protocol (MCP) Security*. OASIS Open Project, Workstream 4 (Secure Design Patterns for Agentic Systems), approved 8 January 2026. https://www.coalitionforsecureai.org/wp-content/uploads/2026/03/model-context-protocol-security-1.pdf
4. Model Context Protocol. *Specification, revision 2025-11-25, Server Features, Tools* and `ToolAnnotations` schema. https://modelcontextprotocol.io/specification/2025-11-25/server/tools
5. Hungerford, Morrow, Chang. *Tool Annotations as Risk Vocabulary: What Hints Can and Can't Do*. Model Context Protocol Blog, 16 March 2026. https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/
6. Jones, Nadalin, Campbell, Bradley, Mortimore. *OAuth 2.0 Token Exchange*. RFC 8693, January 2020. https://datatracker.ietf.org/doc/rfc8693/
7. Microsoft. *Agent OAuth flows: On behalf of flow*. Microsoft Entra Agent ID documentation. https://learn.microsoft.com/en-us/entra/agent-id/agent-on-behalf-of-oauth-flow
8. Amazon Web Services. *On-behalf-of token exchange with AgentCore Identity*. Amazon Bedrock AgentCore developer guide. https://docs.amazonaws.cn/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html
9. Schwenkschuster, Kasselmann, Burgin, Jenkins, Campbell, Parecki. *OAuth Identity and Authorization Chaining Across Domains*. draft-ietf-oauth-identity-chaining-16, June 2026, work in progress. https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-chaining-16
10. Parecki, McGuinness, Campbell. *Identity Assertion JWT Authorization Grant*. draft-ietf-oauth-identity-assertion-authz-grant, revision 04, May 2026, work in progress. https://www.ietf.org/archive/id/draft-ietf-oauth-identity-assertion-authz-grant-02.html
11. Model Context Protocol. *Enterprise-Managed Authorization* extension, io.modelcontextprotocol/enterprise-managed-authorization, stable. https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization
12. Red Canary (now Zscaler). *Investigating suspicious AI workflows in Microsoft Entra Agent ID: Assistive agents*. 2026, lab-scenario detection walkthrough. https://redcanary.com/blog/threat-detection/entra-id-ai-workflows-assistive-agents/
13. Model Context Protocol. *MCP 2026-07-28 Specification* release post, 28 July 2026. https://blog.modelcontextprotocol.io/posts/2026-07-28/
14. Cloud Security Alliance AI Safety Working Group. *The Six Pillars of MCP Security*. Working draft, 2026.
15. Microsoft. *Security* (FIDES information-flow control middleware), Microsoft Agent Framework documentation. https://learn.microsoft.com/en-us/agent-framework/agents/security
16. APISec Research Labs. *Trust Me, Bro: Forging Security Labels in Microsoft's FIDES Middleware*. 2026. https://labs.apisec.ai/research/articles/trust-me-bro-forging-security-labels-fides
17. Microsoft. *Define Sensitivity Label Inheritance Requirements for AI Outputs*. Zero Trust Assessment, AI_080. https://microsoft.github.io/zerotrustassessment/docs/workshop-guidance/AI/AI_080
18. Invariant Labs. *mcp-scan*, tool pinning. https://github.com/invariantlabs-ai/mcp-scan
19. policylayer. *MCP rug pull*, MCP Security reference. https://policylayer.com/attacks/mcp-rug-pull
20. LangChain. *Improve LLM-as-judge evaluators using human feedback*. LangSmith documentation. https://docs.langchain.com/langsmith/improve-judge-evaluator-feedback
21. Braintrust. *LLM-as-a-judge*. Encyclopedia entry. https://braintrust.dev/encyclopedia/llm-as-a-judge
22. Birkholz, Thaler, Richardson, Smith, Pan. *Remote ATtestation procedureS (RATS) Architecture*. RFC 9334, January 2023. https://datatracker.ietf.org/doc/rfc9334/
23. OpenZiti. *mcp-gateway*, v0.1.11, commit 8f99623, 25 August 2026, Apache-2.0. `aggregator/policy.go`, `aggregator/environment.go`, `aggregator/namespace.go`, `gateway/session.go`, and `CHANGELOG.md`. https://github.com/openziti/mcp-gateway
