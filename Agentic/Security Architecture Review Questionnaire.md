# Agent Execution Platform, Security Architecture Review Questionnaire

## How to use this

Ten questions. Each one targets a claim that is commonly made about agent deployments and commonly untrue. The reviewer asks the question, records the answer verbatim, requests the listed evidence, runs the verification, and scores the item. The questionnaire is complete when every item has a score and every score below 3 has an entry in the findings register.

Interview answers are recorded but never scored on their own. Only evidence and verification results move the score.

## Scoring

| Score | Meaning |
|---|---|
| 0 | Control absent, or the answer describes one of the listed red-flag patterns |
| 1 | Control claimed, no evidence produced |
| 2 | Evidence produced, verification not run or failed |
| 3 | Verification passed on a representative production configuration |
| 4 | Verification passed and the control is enforced by admission, policy, or infrastructure the workload cannot modify |

Items 1, 2, 3, 7 and 8 are gating. A score below 3 on any gating item blocks production approval regardless of the total.

## Reviewer inputs

Before the session, obtain: the architecture diagram with trust boundaries, the sandbox namespace manifests and admission policies, the egress proxy configuration, one full session transcript with correlation IDs, one approval record, the Inspect (or equivalent) task file and one eval log, the tool registry or manifest store, and read access to a representative non-production cluster.

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
- Where are eval logs published and who reads them

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
- List of gated action classes (tool calls, memory writes, retrieval, compaction, subagent start, tool registration, publication to run-readable locations)

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
- Can the decision log corroborate the transcript during a compromise window

**Verification.** Pull one approval record, the matching PDP decision entry, and the matching exec bridge log by correlation ID. Confirm three separate authors. Replay the approval against a modified action and confirm rejection. Stop the PDP in staging and confirm every pending action resolves to deny and an alert fires.

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
- Latch behavior for privileged, destructive, financial, and externally visible action classes
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

**Pass criteria.** Digest pinning, full-schema scan before context, drift blocks connection, capability change triggers re-approval, responses validated, registration gates deployment, connections to unregistered endpoints alert. Client token is exchanged for an audience-scoped downstream token, never forwarded.

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

## Findings register

| ID | Item | Score | Finding | Severity | Owner | Remediation | Retest date |
|---|---|---|---|---|---|---|---|
| F-01 | | | | | | | |
| F-02 | | | | | | | |

Severity guide. Critical for any gating item at 0 or 1. High for any gating item at 2, or any non-gating item at 0. Medium for non-gating items at 1 or 2. Low for a 3 that lacks infrastructure enforcement.

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

Production approval requires every gating item at 3 or above and no Critical findings open.

Reviewer: ___ Date: ___ System: ___ Version reviewed: ___
Accountable owner: ___ Sign-off: ___
