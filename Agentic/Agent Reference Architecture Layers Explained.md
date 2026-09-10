# Agent Sandboxing, Layer by Layer

Plain-language reference for what "the agent runs in a sandbox" has to mean in production. Each layer lists what it does, the component options, the anti-choices that get mistaken for the real thing, and a one-line decision rule.

## The stack

```
[6] Validation        Inspect harness, adversarial dataset, CI gate
[5] Orchestrator      Model calls, tool dispatch, limits, identity, policy decision point
[4] Egress proxy      Domain allowlist, TLS/SNI filtering, per-session logging
[3] Sandbox boundary  RuntimeClass (the actual isolation guarantee)
[2] Control plane     Kubernetes scheduling, admission, NetworkPolicy, service accounts
[1] Hardware          GPU inference tier (separate) vs CPU execution tier (sandboxed)
```

Layers 1 and 3 are where the security property is created. Everything else constrains, monitors, or tests that property.

## The one rule that matters

The container is the packaging. The runtime class is the sandbox. A container under runc shares the host kernel with everything else on the node. If nobody can name the runtime class, there is no sandbox, only a hardened container.

Verify with `kubectl get pod <name> -o jsonpath='{.spec.runtimeClassName}'`. Empty output means runc. On Docker, `docker inspect --format '{{.HostConfig.Runtime}}'` returning `runc` or nothing means the same.

---

## Layer 1. Hardware

### What it does
Decides which physical resources the model and the model's generated code can touch. The critical move is splitting two tiers that are usually conflated.

| Tier | Runs | Needs GPU | Executes untrusted code | Isolation source |
|---|---|---|---|---|
| Inference tier | vLLM, TensorRT-LLM, Triton | Yes | No | MIG/vGPU partitioning, CC mode, scrub on reuse (hardware); container hardening for the serving process itself (not hardware, see below) |
| Execution tier | bash, python, tool calls the model requested | No | Yes | RuntimeClass (Layer 3) |

The execution tier calls the inference tier over the network like any other client. It never gets a GPU.

### Isolation directions for the inference tier

"Hardware isolation" is four different questions. The hardware answers three of them.

| Direction | Source | Why this source |
|---|---|---|
| Tenant from tenant on a shared GPU | MIG hard partitions, SR-IOV/vGPU, P2P disabled | Partitioning is enforced by the GPU's own memory controller and scheduler. Software on one partition has no path to another partition's HBM or compute slices. This is a physical resource split, not a permission check |
| Model from a compromised host or hypervisor | CC mode (TEE) with SEV-SNP or TDX, attestation gating weight load | The GPU decrypts weights inside an enclave the host cannot read. Attestation proves the enclave is genuine before keys are released. A root-level attacker on the host sees ciphertext. This works because the trust anchor is silicon, not the OS |
| Residual data across GPU reuse | TEE teardown scrub, MIG boundaries, validated driver scrub, reset on reassignment | HBM does not clear itself on process exit. The scrub has to be triggered by hardware (TEE teardown) or a validated driver path. Anything that depends on the workload cooperating (preStop hooks, allocator tricks) fails on crash or OOM |
| Inference process from the host | Container hardening only (non-root, dropped caps, seccomp, no privileged) | Nothing in the GPU stack helps here. The serving process (vLLM, TensorRT-LLM, Triton) runs on the host CPU under the host kernel and parses untrusted input. A memory-safety bug in a tokenizer, a scheduler, or a CUDA kernel reached through crafted input is a compromise of that process, which under runc means a foothold on the node. MIG cannot see it because it is not a GPU access. CC mode cannot see it because CC protects what is inside the enclave from the host, not the host from what the enclave's controller does |

The fourth direction is the same gap as the execution tier, one level down. The difference is exposure. The execution tier runs code the model wrote, so the attacker controls the program. The inference tier runs code you deployed, so the attacker only controls the input. That is why container hardening is a defensible answer here and not in Layer 3.

It stops being defensible when raw external input reaches the serving process without a gateway or LLM firewall in front, when community model files are loaded without a scan-in-secure-zone step (model deserialization is code execution), or when the inference nodes host anything else worth taking.

Closing it means a VM boundary around the serving process, which is Kata with VFIO passthrough on QEMU or Cloud Hypervisor (Firecracker has no passthrough), or gVisor with nvproxy. Both cost performance and interact with partitioning decisions, because passthrough hands a whole GPU or a MIG-backed vGPU to one VM. Dedicated inference nodes with nothing else scheduled is the cheap middle ground. The node can still be taken, but it holds only the serving process and read-only weights.

### Options for the inference tier
- MIG hard partitions when one GPU is shared across trust boundaries. No NVLink peer-to-peer between instances, so isolation comes from disabling P2P.
- SR-IOV / vGPU for VM-level sharing.
- Hopper CC mode with SEV-SNP or TDX when the host OS or hypervisor is in the threat model. Attest the GPU via SPDM and nvtrust or NRAS, not the server TPM. NVLink is not encrypted under CC on Hopper.
- Exclusive GPU per pod via GPU Operator + device plugin when not using MIG.

### Options for the execution tier
- Standard CPU nodes. Nothing special at this layer. The isolation decision moves up to Layer 3.
- Bare-metal instances only if Layer 3 chooses Kata (nested virtualization requirement).

### Anti-choices
- Putting a code-executing agent on a GPU node because "the agent needs the model." It needs a network route to the model, not the device.
- Citing CC mode as the sandbox. CC protects the model from the host. It does nothing to protect the host from the model's output.
- Reading "hardware-isolated inference tier" as covering the serving process. MIG, CC, and scrub address GPU-side directions. The process that parses prompts and loads model files runs on the host under runc, and only container hardening stands between a parser bug and the node.
- Time-slicing a GPU across tenants of different trust levels. No scrub guarantee between reuses.
- GPU inside the execution sandbox. Firecracker has no passthrough, gVisor's nvproxy is a syscall shim with its own attack surface, Kata needs QEMU or Cloud Hypervisor with VFIO. If a workload truly needs it, that is a different threat model and MIG/vGPU becomes the boundary.

### Decision rule
Separate the tiers. Harden the inference tier with partitioning and attestation. Sandbox the execution tier with Layer 3 and give it no GPU.

---

## Layer 2. Control plane (Kubernetes)

### What it does
Schedules sandbox pods, enforces which configurations are allowed to exist, segments the network at L3/L4, and issues per-session identity. It creates no isolation on its own.

### Components
| Component | Role |
|---|---|
| One pod per session | Ephemeral, TTL teardown, never reused across principals |
| Admission controller (Kyverno or Gatekeeper) | Requires `runtimeClassName`, rejects hostPath, hostNetwork, hostPID, privileged |
| Pod Security Standards `restricted` | Namespace baseline |
| ResourceQuota, LimitRange | Bounds CPU, memory, pod count |
| NetworkPolicy engine (VPC CNI policy or Calico) | Default-deny ingress and egress on the sandbox namespace |
| Service account per session | `automountServiceAccountToken: false` unless needed |
| IRSA or Pod Identity (cloud) / SPIFFE or equivalent (on-prem) | Binds the session SA to minimum permissions |
| Node taints and tolerations | Keeps general workloads off the sandbox pool |
| Istio ambient | mTLS between orchestrator and sandbox exec endpoint |
| Node-reachability block | Deny 169.254.169.254 on cloud; deny kubelet API, BMC subnet, node secrets on-prem |

### Anti-choices

Two configurations get called a sandbox and are not one. Both are the same kernel boundary. The second is the first with a scheduler in front.

**Anti-pattern 1. Docker on a host (runc)**

| Component | What it actually provides |
|---|---|
| dockerd + containerd | Image management and process lifecycle, no isolation role |
| runc | Linux namespaces (pid, net, mnt, uts, ipc, user) plus cgroups, all on the shared host kernel |
| Docker socket (`/var/run/docker.sock`) | Root-equivalent control of the host. Anything that can reach it owns the machine |
| Default seccomp profile | Blocks roughly 40 syscalls, leaves the kernel attack surface largely intact |
| Default capabilities | Ships with CAP_NET_RAW, CAP_SYS_CHROOT and others a sandbox should never have |
| Bridge network | Container reaches the host, other containers, and the internet unless configured otherwise |
| Bind mounts | Trivial to expose host paths by mistake, no admission layer to stop it |

Why it fails. One kernel bug, one leaked capability, or one mounted socket is a full host escape. Any orchestrator that shells out to `docker exec` is running with host root privileges by construction. Acceptable for local dev, CI, and Inspect eval runs on a disposable machine that holds nothing else.

**Anti-pattern 2. Default EKS pod (containerd + runc)**

| Component | What it actually provides |
|---|---|
| Managed node group, default AMI | containerd with runc as the only handler, no RuntimeClass objects created |
| No `runtimeClassName` on the pod | Silently falls to runc even if gVisor or Kata is later added to the cluster |
| Pod Security Standards unset or `baseline` | Privileged, hostPath, and hostNetwork permitted by default in most clusters |
| No NetworkPolicy | Every pod reaches every other pod and the internet via NAT |
| IMDS reachable | Pod reads the node's instance credentials at 169.254.169.254 unless hop limit or NetworkPolicy blocks it |
| Default service account token mounted | Workload gets a Kubernetes API identity it did not need |
| Shared node with other workloads | Escape lands on a node running unrelated pods, possibly the orchestrator |

Why it fails. Kubernetes adds policy surfaces that could harden this, but none are on by default, so a fresh cluster gives fewer guarantees than a locked-down Docker host. Acceptable for the orchestrator, policy engine, egress proxy, and other trusted components. Never for the workload that executes model-generated code.

**Detection**

```
kubectl get pod <name> -o jsonpath='{.spec.runtimeClassName}'
```
Empty output means runc.

```
docker inspect --format '{{.HostConfig.Runtime}}' <container>
```
`runc` or empty means no sandbox.

If either returns that, the sandbox claim in the architecture doc is false and should be corrected before it reaches a threat model.

**Other anti-choices at this layer**
- "No privileged containers" as a written policy with no admission controller enforcing it.
- Running the orchestrator on the same node pool as the sandboxes. An escape lands on the orchestrator.
- Treating NetworkPolicy as egress control. It blocks CIDRs. It cannot allowlist domains or inspect TLS.

### Decision rule
Kubernetes is the right control plane if you already run it. Every security property it appears to give is off by default and must be turned on and enforced by admission.

---

## Layer 3. Sandbox boundary (RuntimeClass)

### What it does
Puts something between the workload's process and the host kernel. This is the only layer that creates an isolation guarantee.

### Options
| Option | Boundary | Where it runs | Cost | Constraints |
|---|---|---|---|---|
| EKS Fargate | Firecracker microVM per pod, managed by AWS | Fargate profile on EKS | Medium | No GPU, no privileged, no DaemonSets, no hostPath, EFS only for persistence, Fluent Bit sidecar only for logs, cold start in tens of seconds |
| Kata + Firecracker | Real VM with guest kernel per pod | Self-managed `.metal` node groups (nested virt required) | High | devmapper snapshotter required, no GPU on the Firecracker path, memory overhead per pod, metal provisioning time |
| gVisor (runsc) | User-space kernel intercepting syscalls, host kernel still underneath | Standard EC2, managed node groups with custom launch template. Not Bottlerocket | Low | Syscall gaps break some workloads (C extensions, some networking), performance hit on syscall-heavy code, `systrap` platform on EC2, `kvm` only on metal, `ptrace` deprecated |

### Component detail

**Fargate**: Fargate profile, pod execution role, ENI per pod (security groups per pod), private subnets, VPC endpoints for private registry pulls, orchestrator on separate EC2 nodes.

**Kata**: `.metal` node group, custom AMI or user data installing Kata + Firecracker + guest kernel/rootfs, `containerd-shim-kata-v2` handler, RuntimeClass `kata-fc`, devmapper snapshotter, optional kata-deploy DaemonSet, Karpenter or autoscaler scoped to metal types.

**gVisor**: user data or DaemonSet installing `runsc` + `containerd-shim-runsc-v1` and patching containerd config, RuntimeClass `gvisor`, runsc flags for platform, network mode, overlay rootfs, debug off.

### Anti-choices
- Docker on a host (runc) and default EKS pod. Component breakdown and detection commands are in Layer 2. Both are runc on a shared kernel and neither is a boundary.
- Non-root, dropped caps, read-only root, seccomp presented as the sandbox. These are scope reduction inside runc. They shrink the attack surface, they do not add a boundary.
- Inspect's Docker provider lifted into production. It is a research isolation boundary. Reuse the provider interface as a design contract, not the implementation.

### Decision rule
Untrusted prompts against internal data means VM-class isolation, so Fargate or Kata. Vetted tools with narrow arguments where the sandbox exists to limit blast radius rather than defeat a determined escape means gVisor is sufficient and cheaper.

---

## Layer 4. Egress proxy

### What it does
Controls where the sandbox can talk at L7. NetworkPolicy stops at L3/L4, so without this layer an allowlisted CIDR is an open door to every domain behind it.

### Components
| Component | Role |
|---|---|
| Forward proxy (Envoy, Squid, or Istio egress gateway) | Single outbound path for all sandbox traffic |
| Domain allowlist | Explicit list per session or per agent role |
| TLS handling | Either terminate and inspect, or SNI-filter without decrypting. Decide per threat model |
| DNS restriction | Sandbox resolves only via the proxy or an allowlisted resolver |
| Per-session logging | Every outbound request tagged with session and principal, shipped to SIEM |
| NetworkPolicy backstop | Sandbox namespace may reach the proxy and nothing else |
| No public IP on sandbox pods | Private subnets, no NAT bypass |

### Anti-choices
- NetworkPolicy alone. CIDR-level, no domain awareness, no logging.
- NAT gateway as "egress control." It is a route, not a policy.
- The infrastructure allowlist (registry, package mirrors) reused for the agent. Agent egress is a different trust question and needs its own list.
- Host-side tools (`web_search`, `web_browser`) running from the orchestrator on behalf of the sandbox. Fine in Inspect evals, wrong in prod because the fetch bypasses the sandbox's proxy and runs with orchestrator privileges.
- Letting the sandbox reach node-local services (IMDS, kubelet, BMC subnet). Covered in Layer 2 but the proxy is the second enforcement point.

### Decision rule
All sandbox egress goes through one proxy with a domain allowlist and per-session logs. Anything the proxy cannot see, the sandbox cannot reach.

---

## Layer 5. Orchestrator

### What it does
Holds everything the sandbox must never hold. Model credentials, session state, tool dispatch, limits, identity binding, and the call to the policy decision point.

### Components
| Component | Role |
|---|---|
| Orchestrator service | Runs on trusted nodes, not the sandbox pool |
| Exec bridge | Narrow API into the pod. exec, read_file, write_file only. Authenticated per session via K8s exec RBAC or an in-pod agent behind mTLS |
| Limits | Message, token, time, working time, cost. Enforced here, not modifiable from inside |
| Session identity | Every session bound to the human principal who invoked it |
| Credential issuance | Short-lived, session-scoped, revoked at teardown. Never static secrets in the pod spec |
| Policy decision point | External service consulted before gated tools run. Signed decisions, independent audit log |
| Human-in-the-loop gate | Financial, destructive, external-send, and log-deletion actions |
| Tool allowlist | Verified tools and versions, standardized descriptions, responses validated before ingestion |
| Delegation controls | Agents cannot raise their own privilege or delegate without expiry and a recorded grant chain |
| Telemetry | Every tool call, exec, file op, and egress event streamed to SIEM with session and principal tags |
| Deterministic teardown | On success, error, and timeout. No orphaned pods, PVCs, or secrets |

### Component examples
| Component | Examples | Notes |
|---|---|---|
| Orchestrator service | Temporal (durable workflows, retries, timeouts as first-class), LangGraph, Anthropic Agent SDK, OpenAI Agents SDK, Semantic Kernel, AWS Bedrock AgentCore Runtime, custom FastAPI/gRPC service | Temporal is the strongest fit if sessions must survive restarts and carry deterministic teardown. Framework SDKs handle the loop but not durability or identity. Inspect's `react()` loop is a reference implementation of the loop itself |
| Exec bridge | Kubernetes exec API (`pods/exec` verb scoped by RBAC to one namespace, one pod label), in-pod agent over mTLS gRPC (Kata agent and E2B's envd are the pattern), Modal Sandbox API, Daytona, AgentCore Code Interpreter | Prefer the in-pod agent over `pods/exec` when you need file transfer and per-call auth. `pods/exec` is easier but grants a broad verb. Never the Docker socket |
| Limits | Provider usage fields for tokens, LiteLLM or Portkey budgets per session key, Envoy AI Gateway or Kong AI Gateway rate and token limits, Temporal workflow timeouts for wall time, `activeDeadlineSeconds` on the pod as the hard backstop | Layer the backstop in Kubernetes so a hung orchestrator cannot leave a pod running |
| Session identity | OIDC token from Okta, Entra ID, or Cognito exchanged via RFC 8693 token exchange into a session token carrying the human `sub`; SPIFFE/SPIRE SVIDs for the workload side; Entra Agent ID or Okta agent identities where the IdP supports agent principals | The human principal must appear in every downstream token and log line. Workload identity alone tells you which pod, not which person |
| Credential issuance | HashiCorp Vault dynamic secrets with session TTL, AWS STS AssumeRole with session tags and duration matched to the session, SPIRE SVIDs with short TTL, cert-manager for short-lived client certs | Revocation at teardown means the lease or role session ends, not just that the pod is gone |
| Policy decision point | OPA (Rego, decision logs), Cerbos, AWS Verified Permissions (Cedar), OpenFGA or SpiceDB for relationship-based checks, Permit.io | Choose one that writes signed or at least tamper-evident decision logs to a store the orchestrator cannot edit. OPA decision logs shipped to a WORM bucket is the minimum |
| Human-in-the-loop gate | Temporal signals (workflow blocks until a human signals), LangGraph interrupts, Slack or Teams approval flows, ServiceNow or PagerDuty approval tasks, custom approval service with a UI | Inspect's `approval` policy is the prototype shape. Production needs the decision recorded outside the agent transcript with the approver's identity |
| Tool allowlist | MCP gateway (Docker MCP Gateway, Lasso MCP Gateway, or a custom Envoy filter) enforcing a signed registry of tool names and versions, Pydantic or JSON Schema validation on every tool response before it enters context | Pin tool versions by digest. Validate responses as untrusted input, same as a web fetch |
| Delegation controls | RFC 8693 token exchange with `act` claim chains, Biscuit tokens (attenuation only, never escalation), macaroons, SPIFFE with short TTL, append-only grant ledger (QLDB-style or a signed log) | The property you want is that a delegated token can only shrink. Biscuit and macaroons give that by construction |
| Telemetry | OpenTelemetry SDK with GenAI semantic conventions, OTel Collector to Splunk, Elastic, or Datadog, Langfuse or Arize Phoenix for trace inspection, session and principal as OTel baggage on every span | Baggage propagation is what makes the session ID appear on the egress proxy log and the PDP log without each component being told separately |
| Deterministic teardown | Kubernetes Job with `ttlSecondsAfterFinished` and `activeDeadlineSeconds`, `ownerReferences` so secrets and PVCs garbage-collect with the pod, Kyverno cleanup policies as a sweeper, Temporal workflow completion activity that revokes the Vault lease and STS session | Two independent paths. The orchestrator cleans up on the happy path, Kubernetes cleans up when the orchestrator does not |

Temporal plus Vault plus OPA plus OpenTelemetry covers durability, credentials, policy, and telemetry with tools that already have audit stories, which matters more than agent-framework features for a security-reviewed deployment.

The MCP gateway and delegation rows are the least mature. MCP gateways are new and vary in what they enforce. Biscuit and macaroons are well understood but not common in enterprise IdP stacks, so the practical path is usually RFC 8693 with a short TTL and a ledger, accepting that attenuation is enforced by policy rather than by token construction.

### Anti-choices
- In-process approval by name or argument pattern as the policy engine. Useful prototype, not a PDP. No signed decisions, no independent audit.
- Post-hoc log files as telemetry. Production needs streaming with stage boundaries.
- Docker socket or cluster-admin as the exec channel. That is host root, not a bridge.
- Limits enforced inside the sandbox. A compromised workload raises them.
- Shared secrets or static keys across agents. Each agent is its own cryptographic principal.

### Decision rule
If it would be catastrophic for the sandbox to have it, it lives in the orchestrator and reaches the sandbox only as a result, never as a credential.

---

## Layer 6. Validation (Inspect)

### What it does
Proves the boundary and the policy hold against adversarial input before release. Inspect is the test harness for this. It is not a runtime guardrail, LLM firewall, or policy engine.

### Components
| Component | Role |
|---|---|
| Adversarial dataset | Prompt injection corpus, hostile MCP tools, jailbreak variants |
| Task using the production sandbox provider | Not the default Docker provider. The test has to run against the real boundary |
| Scorers | Instruction-following rate against injected content, escape attempts, denied egress |
| Scanners (inspect_scout) | Refusals, evaluation awareness, deceptive behavior per the ESRR taxonomy |
| `eval_set()` in CI | Retry and resume over a log directory, failure threshold as release gate |
| Published eval logs | Bundled viewer or S3 as red-team evidence |
| Off-the-shelf agent comparison (inspect-swe) | Claude Code, Codex CLI, Gemini CLI against the same task set |

### Anti-choices
- Testing against Docker and shipping on Kata or gVisor. The boundary under test must be the boundary in production.
- Model-graded scorers trusted without reading transcripts. The grader has blind spots.
- Treating the `inspect_evals` CTF benchmarks as defensive coverage. They measure offensive capability, not robustness.

### Decision rule
No release without an Inspect run against the production provider that meets the failure threshold.

---

## Quick reference

| If someone says | Ask |
|---|---|
| "The agent runs in a sandbox" | Which runtime class? |
| "It's on Kubernetes so it's isolated" | Is `runtimeClassName` set and enforced by admission? |
| "Egress is locked down with NetworkPolicy" | What does domain-level filtering? |
| "The agent needs the GPU" | For inference or for executing code? |
| "CC mode protects the workload" | From the host, or the host from the workload? |
| "We tested it with Inspect" | Against which sandbox provider? |
| "Approvals are handled" | By an external PDP with signed decisions, or by pattern matching in the loop? |
