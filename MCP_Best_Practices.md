# MCP Security Best Practices

This guide consolidates the actionable security practices in the supplied text. It covers MCP01 through MCP05 and removes case-study narrative and repetition while retaining detection checks, preventive controls, and immediate actions.

> **Source limitation:** The supplied text ends mid-sentence during the MCP05 remediation section (after “Sandbox l…”). Only complete recommendations present in the source are included.

## Core Principles

1. **Keep credentials out of the model context.** The model does not need to see a credential to use a tool. Put credential-handling middleware between the model and the protected service.
2. **Grant only temporary, task-specific authority.** Use least privilege and just-in-time, short-lived, revocable tokens instead of standing permissions.
3. **Treat tool metadata and configuration as executable-risk input.** Names, descriptions, parameters, defaults, schemas, manifests, and project settings can influence agent behavior and must be verified before use.
4. **Verify every component in the trusted path.** Sign, pin, inventory, scan, and sandbox schemas, plugins, servers, SDKs, connectors, and dependencies.
5. **Never turn untrusted text into executable strings.** Use structured process APIs, parameterized queries, validated paths, and allowlisted operations.
6. **Constrain external communication.** Limit the channels through which credentials and private data can leave, and monitor unexpected outbound connections.
7. **Make changes and actions attributable.** Log permission changes and preserve per-agent action attribution so incidents can be investigated and contained.

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
- Do not run agents with root, administrator, or unrestricted `sudo` privileges for convenience.
- Apply the same injection defenses to metadata and protocol fields—such as OAuth authorization URLs—as to conventional user input.

### Detection and review

- Search for shell commands assembled through concatenation or interpolation.
- Trace whether user input, model output, or remote responses flow into executable command strings.
- Find uses of `exec`, `eval`, and `shell: true` that can receive untrusted content.
- Identify generated code that is executed automatically.
- Review file operations for unsanitized or unconstrained paths.
- Find SQL statements built through string interpolation.
- Review protocol handlers, including OAuth flows, for values inserted into shell commands.

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

### Tools and schemas

- [ ] Schemas and manifests are signed, pinned, version-bound, and traceable to known provenance.
- [ ] Full schema content—including parameters and defaults—is scanned before model exposure.
- [ ] ANSI/display-control characters are stripped during review.
- [ ] Schema registries enforce RBAC and reviewed promotion.
- [ ] Tool-list and runtime schema changes trigger alerts.

### Supply chain and networking

- [ ] Every server and plugin has a complete SBOM.
- [ ] Dependencies and servers are pinned; no floating versions are used.
- [ ] Dependency scanning runs in CI.
- [ ] Third-party plugins are sandboxed.
- [ ] Unexpected outbound domains are blocked or alerted on.
- [ ] Local services bind only to localhost and enforce authentication and origin checks.

### Execution safety

- [ ] Untrusted values never enter shell command strings.
- [ ] Processes are launched with structured argument arrays.
- [ ] Option parsing is terminated with `--` where supported.
- [ ] Generated code is not executed without validation and authorization.
- [ ] File paths are normalized and constrained to approved roots.
- [ ] SQL queries are parameterized.
- [ ] Commands and operations are allowlisted.
- [ ] Agents do not run with unnecessary root or administrator privileges.

## Suggested Order of Implementation

1. Move secrets to a vault and keep them outside model context and logs.
2. Replace standing access with least-privilege, just-in-time tokens.
3. Pin and sign tool schemas; scan all metadata before model exposure.
4. Inventory and pin all MCP servers and dependencies; generate SBOMs and sandbox plugins.
5. Remove shell-string execution paths and parameterize commands and queries.
6. Separate environments, restrict outbound communication, and add per-agent attribution and alerts.