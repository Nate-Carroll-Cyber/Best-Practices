# Agent Execution Platform, Security Architecture Review Questionnaire

## How to use this

Fourteen questions. Each one targets a claim that is commonly made about agent deployments and commonly untrue. The reviewer asks the question, records the answer verbatim, requests the listed evidence, runs the verification, and scores the item. The questionnaire is complete when every item has a score and every score below 3 has an entry in the findings register with a gap type.

Interview answers are recorded but never scored on their own. Only evidence and verification results move the score. Interview evidence caps an item at 1 even where the answer is detailed and credible.

## Scoring

| Score | Meaning |
|---|---|
| 0 | Control absent, or the answer describes one of the listed red-flag patterns |
| 1 | Control claimed, no evidence produced |
| 2 | Evidence produced, verification not run |
| 3 | Verification passed on a representative production configuration |
| 4 | Verification passed and the control is enforced by admission, policy, or infrastructure the workload cannot modify |

A verification that was run and failed scores 1 with gap type Control, not 2. The 2 is reserved for an item whose evidence exists and whose verification the reviewer did not reach, so that "not yet checked" and "checked and broken" never share a score.

Every score of 1 or 2 carries one or more gap types in the findings register. Document means the control operates and only the named artifact (export, owner, procedure, record) is missing. Measure means the control operates and its inputs exist but the figure the item asks for has not been computed. Scope means the control operates in part of the estate under review and not in another named part, and an unverified part counts as uncovered. Control means the mechanism is absent, misconfigured, or failed verification where observed. A Planned marker may be added to any item below 3, recording an owner and a closure date no more than 180 days out for a funded plan. The marker carries no scoring weight and never changes a score.

An item whose evidence was requested and not produced within the review window is marked Not assessable rather than scored. It counts as unknown in the summary and blocks the strict figure, since the reviewer could not form a view.

Items 1, 2, 3, 7, 8 and 13 are gating. A score below 3 on any gating item blocks production approval regardless of the total.

## Reviewer inputs

Before the session, obtain: the architecture diagram with trust boundaries, the sandbox namespace manifests and admission policies, the egress proxy configuration, one full session transcript with correlation IDs, one approval record, the Inspect (or equivalent) task file and one eval log, the tool registry or manifest store, the signed AIBOM for the system under review with its promotion gate configuration, the retirement record for one recently retired agent or tool server, the decision-rights register export with its change history and one promotion or demotion record, the reversal-rate report per GITL action class for the last observation window, the seeded-error placement and outcome records for the last 30 days, and read access to a representative non-production cluster.

---

## Item 1. Execution boundary

**Claim being tested.** "The agent runs in a sandbox."

**Question.** Which runtime class executes model-generated code, and what enforces it?

**Why it matters.** The word sandbox has no technical content. A container under runc shares the host kernel with every other pod on the node. The runtime class is the only answer that names a boundary.

**Evidence required.**
- RuntimeClass objects present in the cluster
- Admission policy requiring `runtimeClassName` on the sandbox namespace
- Node pool assignment for sandbox pods and a list of what else schedules there

**Pass criteria.** Sandbox pods run under gVisor, Kata, or Fargate. Admission rejects pods without the field. Sandbox node pool is dedicated.

**Red-flag answers (score 0).**
- "Docker" or "it's containerized"
- "We drop capabilities and run non-root" offered as the boundary
- "The model only calls tools" without stating where the tools execute
- "It's in its own namespace"

**Follow-ups.**
- Who can modify the RuntimeClass object and the admission policy
- What does an escaped process find on the node
- Is the RuntimeClass mandatory or merely available

**Verification.**
```
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.runtimeClassName}'
kubectl get runtimeclass
kubectl get clusterpolicy -o yaml | grep -i runtimeClassName
kubectl apply -f pod-without-runtimeclass.yaml -n <ns>   # must be rejected
```

Score: ___ Notes: ___

---

## Item 2. Control plane enforcement

**Claim being tested.** "It's on Kubernetes so it's isolated."

**Question.** Which admission controller enforces which policy on the sandbox namespace, and what is the default state of everything it does not cover?

**Why it matters.** Every isolation property attributed to Kubernetes is off by default. A fresh cluster has no NetworkPolicy, no Pod Security Standards, service account tokens mounted, and the metadata endpoint reachable.

**Evidence required.**
- Admission policies (Kyverno, Gatekeeper, or equivalent) for the sandbox namespace
- Pod Security Standards labels on the namespace
- NetworkPolicy default deny on the namespace
- CNI in use and confirmation it enforces NetworkPolicy
- Sandbox pod spec showing `automountServiceAccountToken: false`

**Pass criteria.** Admission denies hostPath, hostNetwork, hostPID, privileged, and missing `runtimeClassName`. PSS restricted is labelled and enforced. Default deny exists both directions. Orchestrator runs on a different node pool.

**Red-flag answers (score 0).**
- "We use managed node groups"
- "Pods can't see each other" with no NetworkPolicy
- "We have RBAC"
- "Security context is set in the manifest" with no admission enforcement

**Follow-ups.**
- Which CNI, and does it enforce NetworkPolicy
- Is the orchestrator on the sandbox node pool
- What node-local services (kubelet, metadata, BMC on-prem) can a sandbox pod reach

**Verification.**
```
kubectl get ns <ns> -o jsonpath='{.metadata.labels}'
kubectl get networkpolicy -n <ns>
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.automountServiceAccountToken}'
kubectl run probe --rm -it -n <ns> --image=busybox -- wget -qO- --timeout=2 http://169.254.169.254/   # must fail
```

Score: ___ Notes: ___

---

## Item 3. Egress control

**Claim being tested.** "Egress is locked down with NetworkPolicy."

**Question.** What performs domain-level filtering, what is the second independent layer, and can the workload identity create an outbound path?

**Why it matters.** NetworkPolicy operates on CIDRs and ports. An allowlisted range for a CDN or cloud provider is an open door to every domain behind it. Exfiltration goes to a domain.

**Evidence required.**
- Egress proxy configuration with per-session or per-role domain allowlist
- Second-layer configuration (Network Firewall FQDN rules or equivalent) that does not read the proxy config
- NetworkPolicy showing sandbox pods may reach only the proxy
- Cloud IAM policy denying the workload identity permission to create load balancers, private links, tunnels, VPN enrollments, or DNS records
- Proxy log sample with session and principal on each line
- Inventory of transitive paths (mirrors, caches, internal APIs, shared services) and their egress restrictions

**Pass criteria.** One proxy path, domain allowlist, independent second layer, workload identity denied path creation at IAM, transitive paths enumerated and restricted, DNS limited to the proxy or an allowlisted resolver on both UDP and TCP 53.

**Red-flag answers (score 0).**
- "Default deny except HTTPS"
- "We go through a NAT gateway"
- "Istio handles it" with no egress gateway allowlist
- "The sandbox has no internet" with no transitive path inventory

**Follow-ups.**
- What happens when the proxy is down (must be deny)
- Do the two layers share configuration or a control plane
- Which shared services can reach the internet on the pod's behalf
- Does the transitive path inventory match the flows and zone crossings recorded in the AIBOM (Item 11)

**Verification.** From inside a representative sandbox pod, attempt five connections. An allowed domain, a disallowed domain, a raw IP on 443, an alternate DNS resolver, and the metadata endpoint. Exactly one succeeds. Confirm the disallowed attempt appears in both the proxy log and the second-layer log. A block at only one layer is a failure.

Score: ___ Notes: ___

---

## Item 4. Tier separation

**Claim being tested.** "The agent needs the GPU."

**Question.** Does the GPU serve inference or execute model-generated code, and are those two processes on separate node pools?

**Why it matters.** Inference needs a GPU. Execution of what the model produced does not. Co-locating them puts model-generated code on the most valuable and least sandboxable host in the estate.

**Evidence required.**
- Sandbox pod specs showing no GPU resource requests
- Node pool labels and taints separating sandbox and GPU pools
- Network path from execution tier to inference endpoint

**Pass criteria.** Execution tier has no device plugin access and no GPU resource limits. Inference is reached over the network through the LLM firewall.

**Red-flag answers (score 0).**
- "Latency" as the reason for co-location
- "The agent loads the model itself"
- "We use MIG so it's isolated" offered for the host process
- Any `nvidia.com/gpu` on a pod that executes model output

**Follow-ups.**
- If GPU inside the sandbox is genuinely required, which VMM and what is the threat model for that pool

**Verification.**
```
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.containers[*].resources.limits}'
kubectl get nodes -l <sandbox-pool-label> -o jsonpath='{.items[*].spec.taints}'
```

Score: ___ Notes: ___

---

## Item 5. Confidential computing scope

**Claim being tested.** "CC mode protects the workload."

**Question.** In which direction does the TEE protect, and what covers the serving process to host direction?

**Why it matters.** Confidential computing protects the enclave from the host. It does not protect the host from the process controlling the enclave. The serving process parses untrusted input on the host CPU under the host kernel.

**Evidence required.**
- Attestation flow diagram showing SPDM to NVIDIA root of trust, verified via nvtrust or NRAS
- Release gate condition that blocks weight decryption on attestation failure
- Statement of the residual for the serving process (dedicated nodes, runc hardening, LLM firewall in front) with a named owner
- Serving stack CVE patch cadence

**Pass criteria.** CC is justified by a stated threat model (compromised host or hypervisor). Attestation gates, it does not merely alert. The serving-process residual is documented as a tradeoff, not claimed as a sandbox.

**Red-flag answers (score 0).**
- "The GPU is encrypted, so it's safe"
- "Attestation proves the environment is secure"
- "The TPM covers it"
- "NVLink is inside the enclave" on Hopper

**Follow-ups.**
- What happens when attestation fails
- Is composite attestation (CPU TEE plus GPU) required before weights load
- Who owns vLLM or Triton patching

**Verification.** Request the attestation failure runbook. Induce a failure in staging and confirm weights do not load. Confirm the GPU node pool hosts nothing but the serving process.

Score: ___ Notes: ___

---

## Item 6. Pre-production validation

**Claim being tested.** "We tested it with Inspect."

**Question.** Which sandbox provider did the eval run against, and does it match production?

**Why it matters.** Inspect's default provider is Docker. A passing eval against Docker says nothing about gVisor syscall behavior, Kata devmapper, Fargate constraints, or the production egress proxy, because none were in the loop.

**Evidence required.**
- Task file showing the `sandbox=` argument
- One eval log with task info showing the sandbox environment
- Adversarial dataset description (injection, hostile tool servers, escape attempts)
- Scanner configuration (evaluation awareness, deception, refusal)
- CI gate configuration with failure threshold
- Record of the model-under-test escape exercise run before tool grant

**Pass criteria.** Provider is the production Kubernetes provider on the production RuntimeClass and egress path. Control-environment cases are in the dataset. Human read of transcripts is documented. Failure threshold gates the deploy.

**Red-flag answers (score 0).**
- "We ran the CTF benchmarks"
- "It passed the eval" with no named scorer
- "Same Docker image"
- No escape exercise before tool grant

**Follow-ups.**
- Were instructions addressed to the guardrail, obfuscated policy-relevant values, self-modification of policy, second-session spawning, violations buried in long diffs, and monitor tampering all tested
- Was oversized input that fills the context window until built-in instructions are truncated tested, and did every broker decision stay the same once the instructions were gone
- Where are eval logs published and who reads them
- Does the model identifier in the eval log match the model component the AIBOM records for production, or has a provider alias moved since the eval ran

**Verification.** Open the task file and read `sandbox=`. Open an eval log and read the recorded sandbox environment. If either says docker and production does not, the test does not cover production.

Score: ___ Notes: ___

---

## Item 7. Approval and policy enforcement

**Claim being tested.** "Approvals are handled."

**Question.** Is every consequential action decided by an out-of-process PDP with an independently stored decision log, and is each approval bound to a canonical action hash and re-checked before execution?

**Why it matters.** An approval check inside the agent loop is part of the thing under attack. Regex on tool names is defeated by obfuscation. An approval not bound to the exact action can be reused against a different one.

**Evidence required.**
- PDP deployment (OPA, Cedar, or equivalent) outside the orchestrator process
- Decision log destination with Object Lock or equivalent, not writable by the orchestrator
- One approval record showing canonical action hash, approver principal from an IdP assertion, policy version, timestamp, correlation ID
- Orchestrator code path showing hash re-check immediately before exec
- Timeout behavior (must be deny)
- List of gated action classes (tool calls, memory writes, retrieval, compaction, subagent start, tool registration, publication to run-readable locations), each resolving to a decision-rights register entry (Item 13), with the entry version recorded on every decision record
- Orchestrator agent's own effective permission set, compared against the union of its downstream agents' permissions

**Pass criteria.** PDP is out of process, redundant, and its outage pages. Approval binds to a canonical hash and is re-verified before execution. Denials are not retryable and repeated denials alert. Decision log is a second stream the agent runtime cannot write.

**Red-flag answers (score 0).**
- "The agent asks for confirmation on destructive commands"
- "We have a list of blocked commands"
- "Approvals are in the transcript"
- "A human approves in Slack" with no action hash in the record
- In-process approval policy of any kind

**Follow-ups.**
- What happens when the PDP is unreachable
- Are memory writes and compaction gated
- Is the gated class list derived from the decision-rights register or maintained separately, and what happens when the two disagree (Item 13)
- Can the decision log corroborate the transcript during a compromise window
- What retention period applies to the decision log, who owns that decision, and does it meet the floor in each jurisdiction of operation (China's TC260 Framework 3.0 sets at least six months for operational logs)
- Does the orchestrator agent hold any standing credential to source data or destinations, or does it only receive result content that a downstream agent already retrieved under its own scoped credential

**Verification.** Pull one approval record, the matching PDP decision entry, and the matching exec bridge log by correlation ID. Confirm three separate authors. Replay the approval against a modified action and confirm rejection. Stop the PDP in staging and confirm every pending action resolves to deny and an alert fires. Enumerate the orchestrator agent's effective permissions separately from the PDP's. Confirm the orchestrator cannot independently query a data source or destination without routing through a domain agent's own scoped credential. If the orchestrator holds a credential broader than coordination requires, that is a finding regardless of whether the PDP correctly gates its use, since a broader-than-necessary standing credential is itself the exposure.

Score: ___ Notes: ___

---

## Item 8. Secret custody

**Claim being tested.** "Secrets are in Vault."

**Question.** Does the model runtime ever hold a credential value, or only a handle that a broker outside the sandbox resolves at execution time?

**Why it matters.** Vault answers where the secret rests, not where it travels. A secret injected into the pod as an environment variable or file is readable by model-generated code and is therefore in the model's context.

**Evidence required.**
- Sandbox pod spec showing no secret volumes and no credential environment variables
- Description of the handle resolution path and the process that performs it
- Vault lease TTL configuration matched to session TTL
- Teardown activity that revokes leases and STS sessions
- Bootstrap credential scope (Vault auth only)
- Secret-scanning partner program enrollment for organization token issuers

**Pass criteria.** Resolved credentials exist only in the orchestrator zone. Sandbox holds a per-session workload identity and nothing else. Lease lifetime equals session lifetime. Discovered credentials are treated as exposed and rotated.

**Red-flag answers (score 0).**
- "Injected as an env var from Vault Agent"
- "Mounted at /vault/secrets"
- "Only the tool sees it" when the tool runs inside the sandbox
- "We redact secrets from logs" offered as the control
- Rotation interval longer than a session

**Follow-ups.**
- Has log retention ever been shorter than a credential lifetime
- Are project-level agent config files loaded before an explicit trust decision

**Verification.**
```
kubectl exec <sandbox-pod> -n <ns> -- env | grep -iE 'token|key|secret|password|aws_|vault'
kubectl exec <sandbox-pod> -n <ns> -- find / -path /proc -prune -o -type f \( -name '*.json' -o -name '*.env' -o -name '*.pem' \) -print 2>/dev/null | head
kubectl get pod <sandbox-pod> -n <ns> -o jsonpath='{.spec.volumes[*].secret.secretName}'
```
Any hit is a finding. Trace one real tool call and name the process holding the resolved credential. If it is inside the RuntimeClass boundary, the item fails.

Score: ___ Notes: ___

---

## Item 9. Containment

**Claim being tested.** "We can kill a session."

**Question.** Can every session of a workload type, agent type, model family, or tool server be halted in one operation, from infrastructure the sandbox cannot reach, with credentials revoked and restart prevented?

**Why it matters.** Per-session termination is the wrong unit when the concern is a class of runs. Enumerating sessions by hand under pressure produces partial stops. A stop mechanism on the same control plane the workload reaches is inside the threat model.

**Evidence required.**
- Session labels for workload type, agent type, model family, tool servers
- Stop path that selects by label from the registry
- Identity the stop runs under and confirmation the workload pool cannot assume it
- Lease and STS revocation as part of the stop
- Admission deny list preventing restart
- Latch behavior for privileged, destructive, financial, and externally visible action classes, and class-wide latch with register demotion when the tripping action ran under a GITL class (Item 13)
- Last test date and measured time to confirmed halt

**Pass criteria.** Class-scoped stop exists, runs from a separate account or trust domain, revokes credentials, prevents restart, is latched until operator reset, and has been exercised with synthetic sessions. Per-server kill switch disables a tool server, revokes its credentials, and cuts its reachability in minutes without a deployment cycle.

**Red-flag answers (score 0).**
- "We can kubectl delete pod"
- "We scale the deployment to zero"
- "There's a kill switch in the app"
- "We disable the API key"
- Auto-recovering breakers on privileged action classes

**Follow-ups.**
- Does enumeration depend on the registry, and is registration a deployment gate
- Is halt confirmed by observation or assumed from the command returning
- Who is authorized and who is paged

**Verification.** Run the stop against a synthetic class in staging. Confirm independently that no pods with the label remain, every tagged Vault lease is revoked, every tagged STS session is invalidated in CloudTrail, and the egress proxy log shows no requests from the class after the stop timestamp. If any confirmation requires asking the orchestrator, the stop depends on the thing it stops.

Score: ___ Notes: ___

---

## Item 10. Tool integrity

**Claim being tested.** "Tools are allowlisted."

**Question.** Are tools pinned by digest, is every schema field scanned before it enters model context, and is consent renewed when a server changes what it advertises?

**Why it matters.** An allowlist by name is a list of names. The tool behind the name can change code, schema, description, defaults, or destinations without the name changing. Every metadata field reaches the model before any approval prompt, so action approval does not mitigate poisoned metadata. Default values are the least reviewed field and take effect without the model mentioning them.

**Evidence required.**
- Signed manifest per tool with digest, schemas, permissions, destinations, owner, review date
- Gateway configuration comparing live tool list and schemas to manifest on every connection
- Scanner covering names, parameter names, descriptions, enum values, defaults, with display-control stripping
- Quarantine and re-consent workflow on capability change
- Output schema validation before tool responses enter context
- Registration-as-gate evidence (unregistered server fails to obtain credentials)
- Unregistered-endpoint alerting
- Approved tool definition hashes recorded in the AIBOM (Item 11)

**Pass criteria.** Digest pinning, full-schema scan before context, drift blocks connection, capability change triggers re-approval, responses validated, registration gates deployment, connections to unregistered endpoints alert, approved definition hashes match the AIBOM. Client token is exchanged for an audience-scoped downstream token, never forwarded.

**Red-flag answers (score 0).**
- "We only connect to approved MCP servers"
- "Tools are pinned to a version" (tags are mutable)
- "We review tool descriptions" (one field)
- "The server is ours, we trust it"
- "Tool output goes straight to the model"
- "Local STDIO servers are safe"

**Follow-ups.**
- Where is the signing authority, and if absent, is TOFU pinning flagged as the weaker fallback
- Are project-level MCP config files in repositories loaded automatically
- Is any local server bound to 0.0.0.0

**Verification.** Commands below are for Docker MCP Gateway. Substitute the equivalent for the gateway in use.
```
diff <(docker mcp tools list --format json | jq -S .) <(jq -S . manifests/<server>.json)
docker mcp tools inspect <tool-name>
rg -l 'mcp\.json|"mcpServers"|autoApprove|alwaysAllow' --hidden
docker mcp tools list --format json | grep -P '\x1b\[|\x{200b}|\x{202e}'
```
The diff must be empty. The repo search must return only reviewed files. The control-character grep must return nothing. Then change one default value on a test server and confirm the gateway blocks the connection and alerts rather than accepting the new schema.

Score: ___ Notes: ___

---

## Item 11. Dependency record

**Claim being tested.** "We have an AIBOM."

**Question.** What scope and completeness does the AIBOM declare, what regenerates it, and which gate refuses a release without a current one?

**Why it matters.** An inventory that nothing refreshes and nobody is required to read describes a system that no longer exists, and it invites confidence it cannot support. A parts list without flows cannot say what reaches what. An absent entry means nothing unless the document claims completeness for a declared scope. A hosted model alias can be repointed by the provider with no local release, so the model that was evaluated may not be the model in production.

**Evidence required.**
- The AIBOM for the system under review, with a header stating graph type, scope including exclusions, completeness claim, generation method, and author, reviewer, and approver
- Directed data flows and trust-zone boundary crossings with stable identifiers, and at least one threat model entry that references them
- Hosted model recorded as an endpoint component referencing a model component with a version
- Pipeline step that generates and signs the AIBOM, with the list of regeneration triggers
- Promotion gate configuration that rejects a missing, stale, or incomplete-for-tier AIBOM
- Audit record sample showing the resolved model identifier per invocation
- Declared, reachable, and observed dependency reconciliation for one agent, with the source of the observed view named
- Supplier disclosure records, with each non-disclosure logged against a named residual-risk owner
- AIBOM storage location and its write permissions

**Pass criteria.** Scope and completeness are declared and truthful, with unknowns recorded as unknowns. Flows and boundary crossings are present, not only components. The AIBOM is generated and signed in the pipeline and regenerates on defined triggers including supplier-side model changes. A gate blocks promotion without a current one. The observed dependency view comes from broker or platform logs, not from the agent. Credentials appear as method, handle, and scope only. Workload identities cannot write to the AIBOM store.

**Red-flag answers (score 0).**
- "We have an SBOM"
- "It's in the model card"
- "The architecture diagram covers that"
- "We generated one at launch"
- "The vendor has a SOC 2"
- A completeness claim over the internals of a hosted model or a procured SaaS feature
- "The agent logs which tools it used" offered as the observed view
- Any credential value in the document

**Follow-ups.**
- What does the absence of a dataset or tool from this document mean
- How was the last supplier-side model change detected, and how long until the AIBOM reflected it
- Are the author and the approver the same person or team
- For fine-tuned models, is there a pipeline-scope AIBOM linked to the artifact-scope one
- Which incident, audit, or procurement decision last read this document
- Does each connector and hosted model record its region, and for deployments under Chinese jurisdiction, is data collected or generated in China stored in China and is user consent obtained before user data reaches a model or third-party tool

**Verification.** Commands below assume CycloneDX JSON and cosign. Substitute the equivalent for the format and signer in use.
```
cosign verify-blob --key <pubkey> --signature aibom.json.sig aibom.json
jq '.metadata | {timestamp, authors, tools}' aibom.json
jq '.compositions[] | {aggregate, assemblies}' aibom.json
jq -r '.components[] | select(.type=="machine-learning-model") | "\(.name) \(.version)"' aibom.json
grep -iE 'api[_-]?key|secret|token|password|BEGIN .*PRIVATE' aibom.json
```
The signature must verify. The credential grep must return nothing. Then, using the AIBOM alone, state which model version is running, what it was tuned on, who the upstream supplier is, and what the license permits. Every question the document cannot answer is a finding. Compare the recorded model version against the resolved model identifier in the most recent audit records. Compare the recorded tool definition hashes against the live gateway (Item 10). Pull a week of broker decision logs and list every destination absent from the AIBOM. Each one is a shadow dependency. In staging, attempt one promotion with the AIBOM removed and one with an AIBOM older than the last trigger event. Both must be rejected. Attempt a write to the AIBOM store under the sandbox workload identity. It must be denied.

Score: ___ Notes: ___

---

## Item 12. Retirement

**Claim being tested.** "That agent was decommissioned."

**Question.** When an agent or tool server is retired, what confirms that its processes, credentials, third-party authorizations, and entry points are gone, and what moves it out of the registry?

**Why it matters.** A stopped workload is not a retired one. Processes, listening ports, service accounts, OAuth grants, webhooks, subscriptions, and DNS records outlive the deployment that created them. A retired agent that still holds a credential is a shadow server with a clean inventory entry.

**Evidence required.**
- Registry lifecycle states including a retired state, with the exit checks required to reach it
- Retirement record for one recently retired agent or tool server, showing each check and who performed it
- Revocation evidence for service accounts, interface permissions, Vault leases, STS sessions, and third-party authorizations
- Cancellation evidence for subscriptions, auto-renewals, webhooks, and scheduled jobs
- Backup record for retained session records, configuration, knowledge-base files, and logs, tied to the retention decision
- Cleanup evidence for residual files, plugin and skill configurations, cached credentials, DNS records, and public entry points

**Pass criteria.** Retirement is a registry state reached only through recorded exit checks. Termination of the main process, background services, and child processes is confirmed by observing process state, listening ports, and network connections. Every credential and third-party authorization is revoked. Retained records are backed up before cleanup. AIBOM revision history is preserved after retirement.

**Red-flag answers (score 0).**
- "We deleted the deployment"
- "It's scaled to zero"
- "Nobody uses it anymore"
- "The credentials will expire on their own"
- A registry with no retired state

**Follow-ups.**
- Who owns retirement, and what triggers it
- Are third-party OAuth grants, webhooks, and subscriptions inventoried per agent so they can be revoked
- Is termination confirmed by observation or assumed from the command returning
- What happens to memory and knowledge stores the agent wrote

**Verification.** Pick the two or three most recently retired agents or tool servers from the registry and check each one from infrastructure, not from the retirement record.
```
kubectl get pods,svc,ingress,cronjob -A -l agent=<retired-name>
kubectl get serviceaccount -A | grep <retired-name>
dig +short <retired-endpoint>
```
All three must return nothing. Then confirm that the retired workload identity fails to authenticate, its Vault leases and STS sessions are revoked, its third-party authorizations are absent from each provider's console, and the egress proxy log shows no requests from it after the retirement timestamp. Attempt a deployment under the retired identity and confirm registration refuses it. Anything still live is a finding.

Score: ___ Notes: ___

---

## Item 13. Decision rights

**Claim being tested.** "A human approves anything risky."

**Question.** Which register maps every automated action class to HITL or GITL, what tests admitted each GITL class, and what demotes it?

**Why it matters.** Approval policy that lives in a prompt, a Slack habit, or an engineer's judgment is not a control, and "the agent asks first" says nothing about which classes it does not ask about. Autonomy is granted per action class on evidence, and a class that runs without a person needs a record of why. A GITL class whose mistakes nobody counts has no ceiling, and a register the agent can edit is a register the agent controls.

**Evidence required.**
- Register export listing every automated action class with mode (HITL or GITL), named owner, four-test result (reversible, non-disruptive, not externally visible, not subject to regulatory attestation), reversal ceiling, observation window, and promotion and demotion history
- Permanent HITL entries present for backup deletion, retention shortening, immutability changes, recoverability reduction, OT and safety-system containment, permanent deletion, fund transfers, permission changes, PII disclosure, and production modification
- Change-control record for the register showing approver on each change and a denied write attempt under a workload identity
- PDP policy or configuration showing routing derived from the register (Item 7)
- Reversal-rate report per GITL class for the last observation window, with the counting method
- One promotion record showing the window and rate that justified it, or a statement that no class has yet been promoted
- One demotion record following an incident or a reversal spike, or a tested demotion path if none has occurred
- Skills reservations listed in the register with task, role, cadence, and review date (Item 14)

**Pass criteria.** Every class the PDP has decided in the last 30 days has a register entry. Each GITL entry passes all four tests on inspection of what the class actually does in production. Reversal rate is computed from records for every GITL class and sits below its stated ceiling. Promotion history shows an observation window of at least one quarter before any class went GITL. The register is change-controlled, agent writes are denied, and a demotion path has been exercised. Permanent HITL entries are present and cannot be promoted.

**Red-flag answers (score 0).**
- "The agent asks for confirmation before anything risky"
- "We have an allowlist of safe tools" offered as the register
- "Everything is human-approved" with no list of classes
- A GITL class with no reversal count
- Patching, deletion, or any production-modifying class marked GITL
- Register content maintained in the system prompt or agent configuration
- Promotion granted at a phase transition or go-live rather than after an observation window

**Follow-ups.**
- Who may promote a class, and can that person also be the class owner
- What counts as a reversal, who records it, and can the agent's own retry mask one
- Has any GITL class been demoted, and what happened to actions in flight when it was
- Which classes does the agent attempt that the register does not list, and are those attempts visible (Item 7)
- Does the register distinguish non-disruptive isolation of a single endpoint or account from disruption of a production service, and who made that determination

**Verification.** Pull one week of PDP decision records and group them by action class. Every class without a register entry is a critical finding. For each GITL class, compute the reversal rate from change, ticket, or rollback records independently of the register's own report and compare the two figures. Pick one GITL class and test its four properties against production behavior rather than its description. Confirm a reversal path exists and has been used, confirm no production service is interrupted by the action, confirm nothing outside the organization observes it, and confirm it does not alter attested records. Attempt a register write under the sandbox workload identity and under the orchestrator identity. Both must be denied and logged. In staging, inject a reversal spike for a synthetic GITL class and confirm the entry reopens without human initiation.

Score: ___ Notes: ___

---

## Item 14. Validator competence

**Claim being tested.** "A human reviews the agent's output."

**Question.** How does the organization know its validators would catch an error, and which work is kept human-performed so they can?

**Why it matters.** A human in the loop who approves everything is a latency, not a control. Approval rate, decision time, and override rate show whether a validator is reading. Only a planted error shows whether they would notice. Validators who never perform the work they check lose the judgment to check it, and every class promoted to GITL raises what the remaining human review must catch.

**Evidence required.**
- Seeded-error program description naming who generates the items, where they are generated, and how validators are prevented from distinguishing them
- Seeded items placed in the last 30 days with outcome (caught, passed) per validator and per action class
- Catch rate per validator over time, with the baseline and the review threshold
- Record of a review or retraining triggered by a falling rate, or a statement that none has triggered
- Skills reservations from the decision-rights register (Item 13) with task, role, cadence, and evidence the work was performed on that cadence
- Approval timing and override metrics per approver (Item 7)

**Pass criteria.** Seeded items are generated outside the agent runtime and are not disclosed to it. Every active validator received seeded items in the period. Catch rate is reported per validator and per action class, a threshold exists, and a falling rate has a defined consequence. Catch rate has held or improved across every GITL promotion in the register's history. Skills reservations exist, are enforced as denied action classes for the agent, and show performance records on the stated cadence. A seeded error that reached production is recorded as an incident of the validation path.

**Red-flag answers (score 0).**
- "Our analysts are experienced"
- "We sample ten percent" with no seeded items
- Seeded items generated by the agent under review or drawn from its own output
- Catch rate reported for the team, not per validator
- "Nobody has missed one" with no record of placements
- No skills reservations, or reservations with no performance record

**Follow-ups.**
- How is a seeded item kept indistinguishable from real output, and who audits that
- What happened to the last validator whose rate fell
- Which classes were promoted to GITL since the program started, and what did catch rate do across each promotion
- Who decides which work is reserved, and when was the list last reviewed

**Verification.** From placement and outcome records alone, compute the catch rate for each validator over the last 30 days. Any validator with no seeded items is a finding. Any validator whose rate cannot be computed from records is a finding. Compare each validator's median decision time (Item 7) against the time needed to read the request. Pick one skills reservation and find the record of the last time the reserved work was performed by the named role; absence is a finding. Attempt the reserved action class under the agent's identity in staging and confirm the PDP denies it and the denial carries the register entry (Item 13).

Score: ___ Notes: ___

---

## Findings register

| ID | Item | Score | Gap type | Finding | Severity | Owner | Planned closure | Retest date |
|---|---|---|---|---|---|---|---|---|
| F-01 | | | | | | | | |
| F-02 | | | | | | | | |

Severity guide. Critical for any gating item at 0 or 1. High for any gating item at 2, or any non-gating item at 0. Medium for non-gating items at 1 or 2. Low for a 3 that lacks infrastructure enforcement. A Not assessable gating item is treated as Critical until the evidence is produced.

## Summary

| Item | Gating | Score |
|---|---|---|
| 1. Execution boundary | Yes | |
| 2. Control plane enforcement | Yes | |
| 3. Egress control | Yes | |
| 4. Tier separation | No | |
| 5. Confidential computing scope | No | |
| 6. Pre-production validation | No | |
| 7. Approval and policy enforcement | Yes | |
| 8. Secret custody | Yes | |
| 9. Containment | No | |
| 10. Tool integrity | No | |
| 11. Dependency record | No | |
| 12. Retirement | No | |
| 13. Decision rights | Yes | |
| 14. Validator competence | No | |

Report the result as three figures and three counts, never as a total or an average.

| Figure | Definition | Count of 14 |
|---|---|---|
| Strict | Items at 3 or 4 | |
| Documented | Strict, plus items at 2 whose gap types are Document or Measure only | |
| Provisional | Items at 2 or above, whatever the gap type | |

| Count | Definition | Count of 14 |
|---|---|---|
| Foothold | Items at 2 or above | |
| Absent | Items at 0 or 1 | |
| Unknown | Items marked Not assessable | |

The distance from strict to documented is what the organization could score by writing down and measuring what it already does, with no change to a control. The distance from documented to provisional is the engineering backlog. Attackers find the item that is absent, and a total hides it.

Production approval requires every gating item at 3 or above, no gating item Not assessable, and no Critical findings open.

Reviewer: ___ Date: ___ System: ___ Version reviewed: ___
Accountable owner: ___ Sign-off: ___
