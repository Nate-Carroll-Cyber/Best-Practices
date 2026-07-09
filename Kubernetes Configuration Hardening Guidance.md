# Kubernetes Configuration Hardening Guidance

| Field | Value |
|---|---|
| Document owner | Platform Security / Cloud Infrastructure |
| Version | 2.0 |
| Last reviewed | 2026-07-09 |
| Review cadence | Every 6 months, and on major Kubernetes version upgrades |
| Applies to | Production Kubernetes clusters (managed and self-managed) |
| Primary references | CIS Kubernetes Benchmark, NSA/CISA Kubernetes Hardening Guidance, Kubernetes Pod Security Standards, MITRE ATT&CK for Containers |

Securing Kubernetes requires moving beyond default cluster settings, sample manifests, and permissive development workflows. Kubernetes is frequently deployed as the control layer for production applications, cloud credentials, service identities, secrets, data stores, CI/CD pipelines, and horizontally scalable compute. A single weak control can become a path from one compromised pod to the cluster, the underlying node, or the broader cloud environment.

Kubernetes should be treated as both:

1. **A control-plane platform**, because the API server, RBAC, admission controllers, kubelet, etcd, and cloud IAM integrations determine what users and workloads can do.
2. **A distributed workload execution runtime**, because pods, jobs, deployments, cronjobs, containers, service accounts, volumes, and service meshes run code and connect to sensitive systems.

In production, secure Kubernetes like a high-trust developer platform and runtime fabric, not like a single application server.

---

## Core Security Principle

> Assume that a compromised workload will attempt to use its service account, network position, mounted credentials, and Kubernetes API access to discover, escalate, persist, move laterally, collect data, and exfiltrate.

Kubernetes does not provide secure-by-default isolation for every production threat model. Namespace isolation, RBAC, NetworkPolicies, Pod Security Standards, admission controls, and service accounts are powerful, but only when configured consistently and continuously validated.

A secure Kubernetes environment needs controls across:

* Authentication and identity
* RBAC and authorization
* Admission control
* Workload configuration
* Network segmentation
* Secrets and credential management
* Node and kubelet hardening
* etcd and control-plane protection
* Runtime detection
* Audit logging
* CI/CD and image governance
* Cloud workload identity boundaries

---

## Scope and Shared Responsibility

Not every control in this document is actionable on every cluster. Responsibility depends on who operates the control plane.

| Component | Managed cluster (EKS / GKE / AKS) | Self-managed cluster (kubeadm, kops, bare metal) |
|---|---|---|
| API server flags, admission plugins | Partially provider-controlled; configure via provider options | Fully your responsibility |
| etcd hardening, TLS, encryption at rest | Provider-managed (verify provider settings) | Fully your responsibility |
| kubelet flags, node OS, kernel patching | Shared; managed node groups reduce burden | Fully your responsibility |
| NodeRestriction, control-plane audit config | Provider-managed; enable via provider audit options | Fully your responsibility |
| RBAC, NetworkPolicy, workloads, images, secrets | Fully your responsibility | Fully your responsibility |

Before applying control-plane sections (etcd, kubelet flags, NodeRestriction, audit policy), confirm whether your provider exposes the setting. On managed clusters, "verify the provider default and enable the provider-native equivalent" replaces "set this flag." Record which party owns each control so audits do not flag provider-managed items as gaps.

---

## How to Prioritize (Maturity Tiers)

This document is not a flat checklist where every item carries equal weight. Adopt controls in tiers so teams know where to start and auditors can measure progress.

**Tier 0 — Baseline (do first, non-negotiable for any production cluster):**

* Private/restricted API server, anonymous access disabled.
* Audit logging enabled and shipped off-cluster.
* Least-privilege RBAC; no wildcard or `cluster-admin` bindings for workloads.
* Pod Security Admission (or equivalent) enforcing at least `baseline`, targeting `restricted`.
* Dedicated service accounts; automount disabled unless required.
* Default-deny ingress NetworkPolicies.
* Secrets encrypted at rest.

**Tier 1 — Hardened (target state for regulated or internet-facing workloads):**

* Admission policy engine (Kyverno/Gatekeeper) with fail-closed webhooks.
* Default-deny egress plus explicit allow, cloud metadata blocked.
* External secret manager and projected (short-lived) service account tokens.
* Digest-pinned, signed, scanned images with admission-time verification.
* Runtime detection (Falco/eBPF) with response runbooks.
* Continuous compliance scanning and drift detection.

**Tier 2 — High-security / multi-tenant (hostile or high-value environments):**

* Workload sandboxing (gVisor/Kata), AppArmor/SELinux profiles.
* mTLS with authorization policies (service mesh).
* Hard multi-tenancy isolation, egress FQDN filtering, break-glass PAM.
* Attestation-based supply chain (SLSA provenance, SBOM verification).

Map each cluster to a target tier based on data sensitivity and exposure, and track the gap.

---

## Compliance and Framework Mapping

These controls align to established benchmarks. Map your implementation to the relevant framework so hardening is auditable and traceable.

| Framework | Relevance | How this document maps |
|---|---|---|
| CIS Kubernetes Benchmark | Config-level hardening of control plane, nodes, policies | Control-plane, kubelet, etcd, RBAC, and policy sections; run `kube-bench` for scored evidence |
| NSA/CISA Kubernetes Hardening Guidance | Authoritative hardening baseline | Namespace, network, admission, auth, and logging sections |
| Kubernetes Pod Security Standards | Workload privilege baselines | Admission control and workload security context sections (`baseline`/`restricted`) |
| MITRE ATT&CK for Containers | Threat-informed detection | Runtime detection and red-team validation sections |
| PCI DSS / SOC 2 / ISO 27001 / FedRAMP | Organizational compliance | Audit logging, secrets, access control, change management (CI/CD), and IR sections supply evidence |

Record, for each control, the framework reference it satisfies and the tool that produces evidence (e.g., `kube-bench`, kubescape, Trivy Operator, audit log export).

---

## Security Model and Logging Baseline

### Treat Kubernetes API Activity as the Control-Plane Source of Truth

Kubernetes attacks often leave traces in API activity before they leave traces in host logs. Enable and retain Kubernetes audit logs for all production clusters.

Audit logs should capture:

```text
timestamp
user.username
user.groups
verb
requestURI
sourceIPs
userAgent
objectRef.resource
objectRef.namespace
objectRef.name
authorization.k8s.io/decision
responseStatus.code
requestReceivedTimestamp
stageTimestamp
```

At minimum, the security team should be able to answer:

```text
Who made this request?
From where?
Using what user agent?
Against which resource?
Was the request allowed or denied?
Which RBAC rule allowed it?
Was it normal for that identity?
Was it correlated with cloud IAM authentication activity?
```

---

### Correlate Kubernetes, Cloud IAM, and Runtime Logs

Kubernetes authentication often happens outside the cluster through cloud IAM, SSO, OIDC, or another external identity provider. Kubernetes audit logs may show what happened after authentication, but not always the full authentication context.

Correlate:

* Kubernetes audit logs
* Cloud IAM logs
* Identity provider logs
* CI/CD logs
* Container runtime events
* Node process telemetry
* Network flow logs
* Secret manager audit logs
* Service mesh access logs

This correlation is essential for identifying suspicious source locations, unusual service account use, credential theft, and API activity that appears valid but is operationally abnormal.

---

### Protect Log Integrity and Time

Logs are only trustworthy if an attacker who compromises the cluster cannot alter or delete them, and only correlatable if timestamps across sources agree.

Log integrity requirements:

* Ship audit logs off-cluster to a SIEM or log store in near real time; do not rely on logs that live only on the control plane or nodes an attacker may control.
* Store logs in append-only / write-once (WORM) or otherwise tamper-evident storage, in an account or project separate from the workloads being logged.
* Restrict who can read and delete logs; alert on deletion, retention-policy changes, and gaps in the log stream.
* Define explicit retention periods (for example 90 days hot for investigation, 1 year or more cold for compliance) rather than an unspecified "retain."
* Verify the audit policy actually captures the required verbs and resources at `Metadata` or `Request` level; a misconfigured audit policy is a silent blind spot.

Time synchronization:

* Ensure NTP (or the cloud provider's time service) is enabled and healthy on all control-plane and worker nodes. Reliable correlation across Kubernetes, cloud IAM, identity provider, and runtime logs depends on synchronized clocks.
* Alert on clock drift and NTP failures; skewed timestamps break both investigations and time-based detections.

---

## Control Plane Hardening

### Private Control Plane Access

Do not expose the Kubernetes API server directly to the public internet unless there is a deliberate, documented, reviewed, and monitored reason.

Preferred access paths:

* Private endpoint
* VPN
* ZTNA
* Bastion host
* Corporate SSO
* Cloud IAM with conditional access
* IP allowlisting
* Short-lived administrator credentials

Restrict the API server to known administrative networks and automation systems.

---

### API Server Authentication

Use strong, centralized identity for API server access.

Recommended options:

* OIDC
* Cloud IAM integration
* SSO-backed authentication
* Short-lived credentials
* MFA for human users
* Per-user kubeconfigs
* Named service accounts for automation

Avoid:

* Shared kubeconfig files
* Long-lived admin certificates
* Generic admin users
* Static credentials in CI/CD
* Anonymous API server access

Every human administrator should be individually attributable in audit logs.

---

### API Server Authorization

Use RBAC with least privilege.

High-risk permissions to tightly restrict:

```text
*
bind
escalate
impersonate
create pods
create deployments
create jobs
create cronjobs
create rolebindings
create clusterrolebindings
get secrets
list secrets
watch secrets
pods/exec
pods/attach
pods/portforward
delete persistentvolumes
patch deployments
```

Use `kubectl auth can-i` during reviews:

```bash
kubectl auth can-i --list
kubectl auth can-i get secrets --all-namespaces
kubectl auth can-i create pods -n production
kubectl auth can-i create clusterrolebinding
kubectl auth can-i impersonate users
```

For service accounts:

```bash
kubectl auth can-i --as=system:serviceaccount:<namespace>:<serviceaccount> --list
```

---

### Detect RBAC Escalation Paths

Review ClusterRoles and Roles for dangerous verbs and broad scopes.

Risk patterns:

```text
verbs: ["*"]
resources: ["*"]
verbs: ["bind"]
verbs: ["escalate"]
verbs: ["impersonate"]
resources: ["secrets"]
resources: ["pods/exec"]
resources: ["rolebindings", "clusterrolebindings"]
```

A user who can create pods may be able to indirectly access mounted service account tokens, node-local resources, or cloud workload identities. Treat pod creation in sensitive namespaces as a powerful permission.

---

### Just-in-Time and Break-Glass Admin Access

Standing cluster-admin access is a large, always-on attack surface. Human administrators should hold minimal day-to-day privilege and elevate only when needed, through an auditable path.

Just-in-time (JIT) elevation:

* Grant high privilege on demand for a bounded window via an access-request/approval workflow (PAM tool or short-lived RoleBinding), not permanent bindings.
* Require approval and business justification for elevation to sensitive namespaces or cluster-scoped admin.
* Issue short-lived credentials for the session and revoke automatically on expiry.
* Log every elevation with requester, approver, scope, and duration.

Break-glass:

* Maintain a documented emergency-access procedure for when normal auth (SSO/OIDC) is unavailable.
* Store break-glass credentials sealed and offline (for example a vaulted local admin cert), usable only by explicit action.
* Fire a high-severity alert whenever break-glass credentials are used, and require post-use review and rotation.
* Test the procedure periodically so it works during an actual outage.

---

### etcd Hardening

etcd stores cluster state and may include sensitive objects such as Secrets. Protect it as a critical control-plane data store.

Recommended etcd controls:

* Keep etcd on private control-plane networks only.
* Do not expose etcd to worker nodes or general application networks.
* Require TLS for client-to-server communication.
* Require TLS for peer-to-peer communication.
* Require client certificate authentication.
* Use trusted CA validation.
* Encrypt backups.
* Restrict filesystem access to etcd data.
* Monitor etcd access and backup activity.
* Protect etcd snapshots like production secrets.

Client-to-server TLS flags to review:

```bash
--cert-file=<path>
--key-file=<path>
--client-cert-auth
--trusted-ca-file=<path>
```

Peer-to-peer TLS flags to review:

```bash
--peer-cert-file=<path>
--peer-key-file=<path>
--peer-client-cert-auth
--peer-trusted-ca-file=<path>
```

Avoid using auto-generated trust in production unless the operational risk is explicitly accepted and documented.

---

### Kubelet Hardening

The kubelet runs on every node and can expose APIs that reveal workload metadata or control pod execution. Harden kubelet configuration on every node.

Required settings:

```bash
--anonymous-auth=false
--read-only-port=0
```

Do not use:

```bash
--authorization=AlwaysAllow
```

Use webhook or another policy-backed authorization mode appropriate for the cluster.

Recommended kubelet controls:

* Disable anonymous kubelet access.
* Disable the read-only kubelet port.
* Restrict kubelet API access to control-plane components.
* Block kubelet ports from external networks.
* Require authenticated and authorized kubelet requests.
* Enable certificate rotation where supported.
* Monitor `pods/exec`, kubelet API access, and node-level process activity.

Common ports to review:

```text
10250  kubelet HTTPS API
10255  kubelet read-only port, should be disabled
```

---

### NodeRestriction Admission

Enable NodeRestriction so kubelets are limited to modifying their own node and associated pod objects.

This helps prevent a compromised node identity from modifying objects associated with other nodes.

Review whether NodeRestriction is enabled and whether node identities are correctly scoped.

---

## Namespace Architecture

### Use Namespaces as Security Boundaries, Not Just Organizational Labels

Create separate namespaces for:

* Production applications
* Staging applications
* Development workloads
* Platform services
* Security tooling
* CI/CD runners
* Data services
* Tenant or customer workloads where applicable

Avoid running production workloads in:

```text
default
kube-system
kube-public
kube-node-lease
```

Block or alert on application deployments to the `default` namespace.

---

### Namespace Labels for Policy Enforcement

If using admission policies scoped by namespace selectors, apply consistent labels to all governed namespaces.

Example:

```yaml
metadata:
  labels:
    policy: enforced
```

For Pod Security Admission, use explicit namespace labels where appropriate:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Do not rely on policy bindings that apply only to labeled namespaces unless you also monitor for unlabeled namespaces.

Detect:

```bash
kubectl get ns --show-labels
```

Flag namespaces missing required enforcement labels.

---

## Admission Control and Policy Enforcement

### Enforce Policy Before Runtime

Admission control is the best place to stop risky workload configurations before they become running pods.

Use one or more of:

* ValidatingAdmissionPolicy
* ValidatingAdmissionWebhook
* MutatingAdmissionWebhook
* Open Policy Agent Gatekeeper
* Kyverno
* Kubescape controls
* Cloud-provider admission policies
* ImagePolicyWebhook
* Pod Security Admission

Admission controls should enforce:

```text
No privileged containers
No privilege escalation
No hostPID
No hostIPC
No hostNetwork unless explicitly approved
No hostPath mounts unless explicitly approved
No container runtime socket mounts
No writable root filesystem unless explicitly approved
No untrusted registries
No mutable image tags in production
No missing CPU or memory limits
No automatic service account token mounting where not required
No dangerous Linux capabilities
No SSH servers inside containers
No public images in production without exception
No deployment to default namespace
```

---

### Make Admission Enforcement Reliable

A policy engine only protects the cluster while it is actually in the admission path. An unreachable, slow, or misconfigured webhook can silently disable every policy above. Treat the admission tier itself as a control that must be hardened and monitored.

Fail-closed for security-critical policies. A webhook with `failurePolicy: Ignore` admits every request when the webhook is down — an attacker who can disrupt the webhook (or simply exploit an outage) bypasses all enforcement.

```yaml
webhooks:
  - name: policy.example.com
    failurePolicy: Fail        # not Ignore for security-critical policies
    timeoutSeconds: 5
    matchPolicy: Equivalent    # avoid bypass via alternate API versions/subresources
    sideEffects: None
```

Watch for and eliminate these bypass paths:

```text
failurePolicy: Ignore on security-critical webhooks
namespaceSelector / objectSelector that unintentionally excludes namespaces
matchPolicy: Exact (allows bypass via alternate API group/version)
Webhook excluding kube-system while workloads can deploy there
Overly broad exemption lists (usernames, service accounts, namespaces)
Webhook pod without adequate replicas, PDB, or priority class
Expired or misissued webhook serving certificate
```

Operational requirements:

* Run policy-engine controllers highly available (multiple replicas, PodDisruptionBudget, a high PriorityClass).
* Alert when the webhook is unreachable, latency rises, or `failurePolicy` is changed to `Ignore`.
* Prefer in-tree `ValidatingAdmissionPolicy` (CEL) for core invariants, since it has no external webhook to disrupt, and use webhook engines for richer policy.
* Review exemption lists on a schedule and treat every exemption as a documented, time-bound exception.
* Test failure behavior: confirm that with the webhook unavailable, security-critical resources are rejected rather than admitted.

---

### Deny Privilege Escalation

Risky workload setting:

```yaml
securityContext:
  allowPrivilegeEscalation: true
```

Preferred workload setting:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

Example policy binding pattern:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: c0016-binding
spec:
  policyName: kubescape-c-0016-allow-privilege-escalation
  paramRef:
    name: basic-control-configuration
  matchResources:
    namespaceSelector:
      matchLabels:
        policy: enforced
```

Adapt API versions and policy names to your Kubernetes version and policy framework.

---

### Deny Forbidden Container Registries

Production workloads should pull images only from approved registries.

Example policy binding pattern:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: c0001-binding
spec:
  policyName: kubescape-c-0001-deny-forbidden-container-registries
  paramRef:
    name: basic-control-configuration
  matchResources:
    namespaceSelector:
      matchLabels:
        policy: enforced
```

Allowed registries should be explicit, for example:

```text
registry.example.com
123456789012.dkr.ecr.us-east-1.amazonaws.com
gcr.io/approved-project
ghcr.io/approved-org
```

Avoid unrestricted pulls from public registries in production.

---

### Deny Mutable Container Filesystems

Risky workload setting:

```yaml
securityContext:
  readOnlyRootFilesystem: false
```

Preferred workload setting:

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

Example policy binding pattern:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: c0017-binding
spec:
  policyName: kubescape-c-0017-deny-resources-with-mutable-container-filesystem
  paramRef:
    name: basic-control-configuration
  matchResources:
    namespaceSelector:
      matchLabels:
        policy: enforced
```

Use explicit writable mounts only where required:

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 128Mi
```

---

### Deny Host and Runtime Breakout Paths

Block or require explicit exceptions for:

```yaml
hostPID: true
hostIPC: true
hostNetwork: true
privileged: true
```

Block hostPath mounts unless explicitly required:

```yaml
volumes:
  - name: host
    hostPath:
      path: /
```

Block container runtime socket mounts:

```text
/var/run/docker.sock
/run/containerd/containerd.sock
/run/crio/crio.sock
```

Block dangerous capabilities:

```yaml
securityContext:
  capabilities:
    add:
      - SYS_ADMIN
      - NET_ADMIN
      - SYS_PTRACE
```

Preferred baseline:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

---

### Enforce Resource Limits

Every production workload should define CPU and memory requests and limits.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Use ResourceQuota and LimitRange per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-resource-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "100"
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-container-limits
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

This reduces the risk of resource abuse, accidental denial of service, and crypto-mining impact.

---

### Availability and Eviction Controls

Security controls are only effective while the components enforcing them stay running. During node pressure, an incident, or a noisy-neighbor workload, the scheduler evicts pods — and it should not evict your policy engine, ingress controller, CNI, or runtime sensors first.

Protect critical security and platform components:

* Assign a high `PriorityClass` to policy engines, admission webhooks, CNI, runtime detection agents, and ingress controllers so they are scheduled and survive eviction.
* Define `PodDisruptionBudget` for these components so voluntary disruptions (drains, upgrades) cannot take all replicas down at once.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: policy-engine-pdb
  namespace: security
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: policy-engine
```

* Use `ResourceQuota` and `LimitRange` (above) so a single namespace cannot exhaust node resources and trigger eviction of critical pods.
* Be deliberate with high/`system-cluster-critical` priority: reserve it for genuinely critical components, and gate who may assign it via admission policy, since a high priority class is itself a way to evict other workloads.

---

### Vet Operators, CRDs, and Helm Charts

Third-party operators, Custom Resource Definitions, and Helm charts are one of the most common ways broad privilege enters a cluster. Installing a chart or operator frequently creates cluster-wide CRDs and highly privileged ClusterRoles that outlive the person who ran `helm install`.

Before installing any operator or chart, review:

```text
ClusterRole / ClusterRoleBinding it creates (watch for verbs: ["*"], resources: ["*"])
Ability to read Secrets cluster-wide
Ability to create/modify RBAC (bind, escalate, impersonate)
Webhooks it registers and their failurePolicy
CRDs it installs and who can create those custom resources
Container securityContext (privileged, hostPath, hostNetwork)
Image source, digest pinning, and signature
Namespace scope vs. cluster scope (prefer namespaced operators)
```

Controls:

* Pin operator and chart versions by digest; do not track `latest`.
* Apply the same admission policies to operator-managed pods as to application pods; do not blanket-exempt operator namespaces.
* Gate who can create high-impact custom resources with RBAC, and audit CRD creation.
* Prefer namespaced operators over cluster-scoped where the project supports it.
* Track installed operators/CRDs in an inventory and re-review on upgrade.

Custom resources can be as powerful as core objects. A CRD that provisions cloud infrastructure or mints credentials deserves the same scrutiny as `pods/exec` or Secret access.

---

## Workload Security Context Baseline

Use a restrictive security context by default.

Example pod/container baseline:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hardened-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hardened-app
  template:
    metadata:
      labels:
        app: hardened-app
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.example.com/app@sha256:<digest>
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            privileged: false
            capabilities:
              drop:
                - ALL
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 128Mi
```

Use exceptions only where documented, reviewed, time-bound, and monitored.

---

### Mandatory Access Control and Kernel Isolation

`seccompProfile: RuntimeDefault` is a strong default but not the whole story. Layer mandatory access control (AppArmor or SELinux) and, for high-risk or multi-tenant workloads, kernel-level sandboxing.

AppArmor (Kubernetes 1.30+ uses a first-class field; older versions use annotations):

```yaml
securityContext:
  appArmorProfile:
    type: RuntimeDefault      # or Localhost with localhostProfile: <name>
```

SELinux, where the node enforces it:

```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

For custom seccomp profiles, use `type: Localhost` with a curated profile rather than `Unconfined`. Never set `seccompProfile.type: Unconfined` or `container.apparmor.security.beta.kubernetes.io/...: unconfined` in production.

Sandboxed runtimes for untrusted, multi-tenant, or high-value workloads:

* gVisor (`runsc`) — user-space kernel that intercepts syscalls.
* Kata Containers — lightweight VM-per-pod isolation.

Select them through a dedicated RuntimeClass:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
spec:
  runtimeClassName: gvisor
```

Reserve sandboxed runtimes for workloads that run untrusted code, handle sensitive tenant data, or sit at the edge of the trust boundary; they add overhead and are not needed for every pod.

---

## Image and Supply Chain Controls

### Use Trusted Images

Production images should come from:

* Private registries
* Approved vendor registries
* Official base image sources
* Internally built and scanned images

Avoid ambiguous or typo-prone image names.

Risk pattern:

```dockerfile
FROM ngnix:latest
```

Preferred pattern:

```dockerfile
FROM nginx@sha256:<digest>
```

---

### Pin Images by Digest

Avoid mutable tags in production:

```yaml
image: registry.example.com/app:latest
```

Prefer immutable digests:

```yaml
image: registry.example.com/app@sha256:<digest>
```

---

### Scan and Sign Images

Before deployment:

* Scan images for vulnerabilities.
* Scan images for malware or suspicious packages.
* Check license and provenance where required.
* Sign images at build time.
* Verify image signatures before deployment.
* Block noncompliant images through admission control.

Periodically rescan running images because new vulnerabilities appear after deployment.

---

### Verify Provenance and Attestations at Admission

Signing an image is only useful if something refuses to run unsigned or unattested images. Move from "we sign images" to "the cluster rejects images it cannot verify."

Generate and retain supply-chain metadata at build time:

* SBOM (CycloneDX or SPDX) for every image, stored and queryable so you can answer "which running images contain package X" when the next CVE lands.
* SLSA provenance attestation describing how and from what source the image was built.
* Signed attestations (for example with cosign/sigstore) covering the SBOM, vulnerability scan, and provenance.

Enforce verification in admission, not just in CI:

```text
Reject images without a valid signature from a trusted key/identity.
Reject images without required attestations (provenance, SBOM, scan).
Require keyless signing identities to match expected CI workflow (issuer + subject).
Bind trusted identities to specific repositories.
```

Example intent (Kyverno / sigstore policy-controller enforce signature and attestation before a pod is admitted):

```yaml
# Conceptual: verify a cosign signature and an SLSA provenance attestation
verifyImages:
  - imageReferences:
      - "registry.example.com/*"
    attestors:
      - entries:
          - keyless:
              issuer: "https://token.actions.githubusercontent.com"
              subject: "https://github.com/approved-org/*"
    attestations:
      - type: "https://slsa.dev/provenance/v1"
```

Adapt to your policy engine and signing setup. The control that matters is that an unsigned or unattested image never becomes a running pod.

---

### Use Minimal Base Images

Use minimized images where compatible:

* Distroless
* UBI Micro
* Chiselled Ubuntu
* Scratch / empty images

Reduce:

* Shells
* Package managers
* Compilers
* Downloaders
* Debug tools
* Unused libraries

If shells or package managers are required, document why.

---

## Service Accounts and Workload Identity

### Disable Default Token Mounting

For workloads that do not need Kubernetes API access:

```yaml
spec:
  automountServiceAccountToken: false
```

At the service account level:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
automountServiceAccountToken: false
```

---

### Use Dedicated Service Accounts

Do not use the default service account for production workloads.

Preferred:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-api
  namespace: production
automountServiceAccountToken: false
```

If API access is required, create narrowly scoped Roles and RoleBindings.

---

### Prefer Short-Lived, Bound Service Account Tokens

Legacy service account tokens stored in Secrets do not expire, are not audience-scoped, and remain valid if stolen until manually deleted. This is one of the most common real-world escalation and persistence paths. Prefer bound, projected tokens issued by the TokenRequest API, which are short-lived, audience-bound, and tied to the pod's lifecycle.

When a workload genuinely needs an API token, project it explicitly with an audience and a short expiration:

```yaml
volumes:
  - name: api-token
    projected:
      sources:
        - serviceAccountToken:
            path: token
            audience: my-service          # scope the token to its intended consumer
            expirationSeconds: 3600        # short-lived; kubelet auto-rotates
```

Guidance:

* Do not create long-lived Secret-based tokens (`kubernetes.io/service-account-token`) for new workloads. In modern clusters these are not auto-generated; do not reintroduce them.
* Audit for and remove legacy static token Secrets that are not required.
* Set explicit `audience` values so a token stolen from one service cannot authenticate to another.
* Keep automount disabled by default and project a token only for the specific container that needs it.
* Alert on creation of long-lived service account token Secrets and on `TokenRequest` calls with unusually long expirations.

---

### Avoid Static Cloud Credentials

Do not place cloud access keys in:

```text
Kubernetes Secrets
ConfigMaps
environment variables
container images
Helm values
Git repositories
CI/CD variables without controls
```

Prefer:

* Cloud workload identity
* IAM roles for service accounts
* Short-lived credentials
* Secret brokers
* External secret managers
* Just-in-time credential issuance

Ensure cloud roles can only be used by the intended Kubernetes service account and namespace.

---

### Monitor Workload Identity Use

Alert on cloud credentials assigned to Kubernetes workloads being used from:

* Outside the cluster
* Unexpected regions
* Unexpected IP addresses
* Unexpected user agents
* Non-workload systems
* Developer laptops
* CI/CD systems not expected to use them

Workload identity credentials should be used only by the target workload.

---

## Network Architecture and Segmentation

### Default-Deny Network Policy

Kubernetes often allows broad workload-to-workload communication unless NetworkPolicies and a supporting CNI enforce restrictions.

Apply default-deny policies per namespace.

Default deny ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Default deny egress:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

Then add explicit allow rules for required traffic.

---

### Allow DNS Explicitly

If default-deny egress is enabled, allow DNS to approved resolvers.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Adapt selectors to your DNS deployment.

---

### Restrict Access to Control-Plane and Node Services

Workloads should not generally be able to reach:

```text
Kubernetes API server
kubelet APIs
etcd
node-local admin ports
container runtime sockets
cloud metadata services
internal admin dashboards
CI/CD controllers
source control systems
secret managers unless explicitly required
```

Use:

* NetworkPolicies
* Security groups
* Firewall rules
* Service mesh authorization policies
* Egress gateways
* Cloud metadata protections

---

### Block Cloud Metadata Access by Default

Block pod access to metadata services unless explicitly required.

High-risk destinations:

```text
169.254.169.254
metadata.google.internal
```

Where metadata access is required, scope identity tightly and monitor use.

---

### Use mTLS for Service-to-Service Communication

Use mTLS where service identity and transport protection matter.

Options:

* Istio
* Linkerd
* Consul
* Cilium service mesh
* Application-level mTLS

mTLS should provide both:

* Authentication: the service is who it claims to be.
* Authorization: the service is allowed to talk to the target.

Do not treat encryption alone as authorization.

---

### Harden Ingress and External Exposure

Most of this document faces inward, but the most common real-world entry point is whatever the cluster exposes to the internet. Treat the external edge as a first-class attack surface.

Control what is exposed:

* Inventory every internet-reachable `Service` of type `LoadBalancer` and `NodePort`, and every Ingress/Gateway route. Alert on new external exposure.
* Prefer a single hardened ingress/gateway tier over many `LoadBalancer` Services; avoid `NodePort` for production external traffic.
* Do not expose management and platform endpoints publicly — API server, kubelet, etcd, metrics, tracing, and dashboards belong on private networks or behind ZTNA/VPN.

Harden the ingress/gateway tier:

* Terminate TLS with current protocols and managed certificates; redirect plaintext.
* Put a WAF and rate limiting in front of internet-facing HTTP routes.
* Keep the ingress controller patched, run it with a restrictive securityContext, and scope its RBAC (ingress controllers often read Secrets for TLS — limit to required namespaces).
* Validate and constrain ingress annotations; some controllers allow config snippets that can be abused for SSRF or auth bypass. Restrict who can create/modify Ingress objects.

Kubernetes Dashboard and similar UIs:

* The Kubernetes Dashboard has a history of being exposed without authentication and abused for full cluster takeover. Prefer not to deploy it in production.
* If it is required, never expose it publicly, never grant it a privileged/`cluster-admin` service account, require authenticated SSO access, and place it behind the private access paths used for the API server.
* The same rules apply to other in-cluster consoles (CI dashboards, tracing/metrics UIs, message-queue or database admin panels).

---

### Monitor Network Flows

Use network visibility tools such as:

* Cilium Hubble
* Service mesh telemetry
* VPC flow logs
* eBPF-based runtime sensors
* Cloud network logs

Alert on:

```text
New cross-namespace communication
Workload-to-control-plane communication
Unexpected outbound internet access
Outbound transfers from sensitive workloads
Connections to metadata services
Connections to unknown IPs
NetworkPolicy changes
```

---

## Secrets and Data Security

### Use External Secret Managers

Kubernetes Secrets are commonly overexposed through RBAC, environment variables, and broad service account permissions.

Use:

* HashiCorp Vault
* AWS Secrets Manager
* Azure Key Vault
* Google Secret Manager
* External Secrets Operator
* Sealed Secrets
* SOPS for GitOps workflows

Kubernetes Secrets should be encrypted at rest and tightly scoped.

---

### Avoid Secrets in ConfigMaps and Environment Variables

Do not store sensitive values in:

```text
ConfigMaps
plain environment variables
Helm values files
container image layers
application logs
flow or job definitions
Git repositories
```

Attackers frequently target Secrets, ConfigMaps, and environment variables during credential access and collection.

---

### Encrypt Secrets at Rest

Enable encryption at rest for Kubernetes Secrets.

Protect the encryption configuration and keys. Rotate keys according to policy.

Also encrypt:

* Persistent volumes
* etcd backups
* Cluster snapshots
* CI/CD artifact stores
* Container registry storage
* Secret manager storage

---

### Protect Persistent Volumes

Persistent volumes can contain application data, credentials, database files, logs, and user uploads.

Controls:

* Restrict PV/PVC listing.
* Restrict volume attachment visibility.
* Encrypt volumes at rest.
* Enforce storage class policies.
* Avoid broad hostPath use.
* Monitor unusual PV/PVC access.
* Backup securely.
* Test restore procedures.
* Alert on deletion attempts.

Sensitive APIs to monitor:

```text
/api/v1/persistentvolumes
/api/v1/namespaces/*/persistentvolumeclaims
/apis/storage.k8s.io/v1/volumeattachments
```

---

## Backup, Disaster Recovery, and Ransomware Resilience

Encryption and access control protect confidentiality, but recovery protects availability and integrity. A cluster hit by ransomware, a destructive insider, or a bad change needs backups an attacker cannot also destroy.

What to back up:

```text
etcd (or provider control-plane state)
Cluster resource manifests (namespaces, RBAC, CRDs, policies)
Persistent volume data
External secret manager state / key material
Container registry contents
```

Backup controls:

* Use a cluster-aware backup tool (for example Velero) to capture resource state and PV snapshots together, so a restore is consistent.
* Keep at least one copy offline or immutable (object-lock / WORM) in a separate account or project, so credentials stolen from the cluster cannot delete backups. This is the primary defense against ransomware.
* Encrypt all backups and snapshots (etcd, PVs, registry) and manage keys separately from the cluster.
* Restrict and log access to backups and snapshots; alert on deletion or retention changes.

Recovery requirements:

* Define RTO and RPO per cluster tier and size backup frequency to meet them.
* Test restores on a schedule — including full cluster rebuild and etcd restore — not just backup success. An untested backup is a hope, not a control.
* Maintain a documented rebuild-from-known-good runbook: provision fresh nodes, restore state, rotate all credentials, and redeploy from trusted images.
* For ransomware specifically, assume the live cluster and its online backups are compromised; recover from immutable copies and treat all secrets as burned.

---

## CI/CD and Deployment Governance

### Do Not Allow Manual Production Workload Creation by Default

Production deployment should normally happen through approved CI/CD.

Alert on manual creation of:

```text
pods
deployments
jobs
cronjobs
daemonsets
statefulsets
```

Sensitive request paths:

```text
/api/v1/namespaces/*/pods
/apis/apps/v1/namespaces/*/deployments
/apis/batch/v1/namespaces/*/jobs
/apis/batch/v1/namespaces/*/cronjobs
```

Manual emergency changes should be time-bound, approved, and reviewed after the incident.

---

### Protect CI/CD Configuration

Attackers target CI/CD because it can modify workloads, inject images, and access secrets.

Protect:

* Pipeline definitions
* GitHub Actions workflows
* GitLab CI files
* Jenkinsfiles
* Helm charts
* Kustomize overlays
* Terraform files
* Argo CD Applications
* Flux manifests
* Image build scripts

Review pull requests that modify:

```text
deployment image
serviceAccountName
securityContext
resources
volumes
volumeMounts
hostNetwork
hostPID
hostPath
container command/args
NetworkPolicies
RBAC objects
Secret references
```

---

## Continuous Compliance and Configuration Drift

Hardening is not a one-time event. Clusters drift as people make manual changes, upgrades reset defaults, and new workloads land. Point-in-time reviews and the red-team plan later in this document are necessary but insufficient — pair them with automated, recurring validation.

Continuous scanning:

* Run CIS benchmark scans on a schedule with `kube-bench` and track the scored result over time.
* Run posture scanning (kubescape, Trivy Operator, or a cloud-native equivalent) for misconfigurations, vulnerable images, and exposed workloads.
* Deploy an in-cluster scanner (for example Trivy Operator) to continuously rescan running images against updated vulnerability feeds, not just at deploy time.
* Feed results into the same dashboard/SIEM as detections, with owners and remediation SLAs by severity.

Drift detection:

* If you use GitOps (Argo CD, Flux), enable drift detection and either alert on or auto-revert out-of-band changes, so the desired state in Git remains the source of truth.
* Alert on live-cluster changes that do not correspond to a pipeline run: new or modified RBAC, NetworkPolicy, admission policy, PodSecurity labels, or webhook configuration.
* Periodically diff live RBAC and policy against the reviewed baseline and flag additions.

Version and patch lifecycle:

* Track the Kubernetes version against its supported/EOL window and upgrade before end of support; running an unsupported version means unpatched control-plane CVEs.
* Patch node OS and kernel on a defined cadence; subscribe to Kubernetes and CNI/runtime security advisories.
* Re-run baseline scans after every upgrade, since upgrades can reset flags and reintroduce defaults.

---

## Runtime Detection and Threat Monitoring

### API Indicators by ATT&CK Tactic

Use Kubernetes audit logs to detect attacker-like API activity.

| Tactic | What to watch | API and field indicators |
|---|---|---|
| Initial Access | Anonymous access, suspicious service account use, unexpected pod exec | `system:anonymous`, `system:unauthenticated`, `system:serviceaccount:*`, `/api/v1/namespaces/*/serviceaccounts`, `/api/v1/namespaces/*/pods/*/exec` |
| Persistence | Manual creation of pods, jobs, cronjobs, deployments | `/api/v1/namespaces/*/pods`, `/apis/batch/v1/namespaces/*/jobs`, `/apis/batch/v1/namespaces/*/cronjobs`, `/apis/apps/v1/namespaces/*/deployments` |
| Privilege Escalation | Default service account misuse, broad RBAC, `bind`, `escalate`, `impersonate` | `system:serviceaccount:default:default`, `/apis/rbac.authorization.k8s.io/v1/clusterroles`, `/apis/rbac.authorization.k8s.io/v1/namespaces/*/roles` |
| Credential Access | Secrets enumeration, kubeconfig/IAM access, secret manager access | `/api/v1/namespaces/*/secrets` |
| Discovery | Resource enumeration and bursts of forbidden requests | `list`, `get`, `forbid`, `/api/v1/nodes`, `/api/v1/namespaces`, `/clusterroles`, `/clusterrolebindings` |
| Collection | ConfigMaps, PVs, PVCs, volume attachments | `/api/v1/namespaces/*/configmaps`, `/api/v1/persistentvolumes`, `/persistentvolumeclaims`, `/volumeattachments` |
| Exfiltration | NetworkPolicy changes, compression/transfer tooling, outbound connections | `/apis/networking.k8s.io/v1/namespaces/*/networkpolicies` |
| Lateral Movement | Suspicious pod creation, pod exec, cloud IAM use from unusual location | `/api/v1/namespaces/*/pods`, `/api/v1/namespaces/*/pods/*/exec`, `system:serviceaccount:default:default` |
| Impact | Workloads without limits, patch/delete activity, PV deletion | `patch`, `delete`, `/deployments`, `/pods`, `/api/v1/persistentvolumes` |

---

### Runtime Process Monitoring

Audit logs alone are not enough. Use runtime telemetry on nodes and containers.

Alert on unexpected execution of:

```text
sh
bash
dash
zsh
curl
wget
nc
ncat
socat
python
python3
perl
ruby
chmod
chown
tar
zip
gzip
7za
rar
openssl
ssh
scp
kubectl
crictl
docker
ctr
```

High-risk patterns:

```text
Interactive shell inside production container
Download-and-execute behavior
Compression followed by outbound network connection
Unexpected package installation
Writes to /tmp, /var/tmp, /dev/shm followed by execution
Access to service account token files
Access to cloud credential files
Access to mounted secrets outside expected process
```

---

### Example Falco-Style Detection Ideas

```yaml
- rule: Shell Spawned In Kubernetes Workload
  desc: Detect shell execution inside a container
  condition: >
    spawned_process and
    container and
    proc.name in (sh, bash, dash, zsh)
  output: >
    Shell spawned in container
    namespace=%k8s.ns.name pod=%k8s.pod.name container=%container.name
    parent=%proc.pname child=%proc.name cmdline=%proc.cmdline user=%user.name
  priority: HIGH
```

```yaml
- rule: Downloader Executed In Container
  desc: Detect curl or wget execution inside a container
  condition: >
    spawned_process and
    container and
    proc.name in (curl, wget)
  output: >
    Downloader executed in container
    namespace=%k8s.ns.name pod=%k8s.pod.name container=%container.name
    cmdline=%proc.cmdline
  priority: HIGH
```

```yaml
- rule: Kubernetes Service Account Token Read
  desc: Detect reads of mounted service account tokens
  condition: >
    open_read and
    container and
    fd.name contains "/var/run/secrets/kubernetes.io/serviceaccount/token"
  output: >
    Service account token read
    namespace=%k8s.ns.name pod=%k8s.pod.name container=%container.name
    process=%proc.name cmdline=%proc.cmdline
  priority: MEDIUM
```

---

### From Detection to Response

A list of "alert on X" only reduces risk if each alert has a severity, an owner, and a defined action. Without that, high-signal detections drown in noise and nothing happens when they fire.

Assign every detection a severity and a routing/response expectation:

| Severity | Example signals | Response expectation |
|---|---|---|
| Critical | `system:anonymous` allowed, privileged pod created, RBAC wildcard/`escalate` added, PV delete, workload identity used outside cluster | Page on-call immediately; begin containment runbook |
| High | Shell/exec in production container, downloader executed, secret enumeration, NetworkPolicy modified, admission webhook set to Ignore | Investigate within minutes; escalate if confirmed |
| Medium | Service account token read, manual workload creation, cross-namespace traffic, missing resource limits | Triage same day; tune or confirm expected |
| Low / informational | New image tag, autoscaling change | Aggregate and review in trend dashboards |

Operational requirements:

* Route alerts into the SOC/on-call system (not just a dashboard), with clear ownership per namespace or workload.
* Attach a short runbook to each high/critical detection linking the signal to concrete actions (isolate pod, revoke token, cordon node, preserve logs) — see the Incident Response Checklist.
* Tune aggressively against a known-good baseline to control false positives; alert fatigue is itself a security failure.
* Track detection coverage against the ATT&CK tactics table above and close gaps.
* Periodically fire test events (see the Red-Team Validation Plan) to confirm detections still trigger and route correctly.

---

## Red-Team Validation Plan

Use authorized testing to verify that hardening controls work and detections fire. Tests should be scoped, non-destructive, and performed in approved environments.

### Initial Access Validation

Validate:

```text
Anonymous API access is denied.
Unauthenticated groups cannot interact with API resources.
Service account tokens do not work outside expected context.
Unexpected source locations trigger alerts.
pod exec is restricted and monitored.
```

Evidence:

```text
Audit decision
User identity
Source IP
User agent
Request URI
Detection or alert result
```

---

### Persistence Validation

Validate whether an unauthorized identity can create:

```text
pod
deployment
job
cronjob
service account token
role binding
cluster role binding
```

Test with benign canary workloads only.

Finding condition:

```text
Manual production workload creation succeeds outside CI/CD without alerting.
```

---

### Privilege Escalation Validation

Validate:

```text
Default service accounts have no unexpected privileges.
No roles permit wildcard permissions without exception.
No roles permit bind, escalate, or impersonate without exception.
Risky pod configurations are denied by admission.
Container escape primitives are blocked.
```

Use:

```bash
kubectl auth can-i --as=system:serviceaccount:<namespace>:default --list
kubectl auth can-i --as=<user> create clusterrolebinding
kubectl auth can-i --as=<user> impersonate users
```

---

### Credential Access Validation

Validate:

```text
Low-privileged users cannot list or get secrets.
Service accounts can access only required secrets.
ConfigMaps do not contain credentials.
Secret manager access is logged and scoped.
Kubeconfig and cloud credential file access is monitored.
```

Use:

```bash
kubectl auth can-i get secrets --all-namespaces
kubectl auth can-i list secrets --all-namespaces
```

---

### Discovery Validation

Validate that enumeration is limited and visible.

Test:

```text
list namespaces
list nodes
list pods
list services
list roles
list rolebindings
list clusterroles
list clusterrolebindings
list configmaps
```

Finding condition:

```text
Low-privileged identity can broadly enumerate cluster resources.
```

Detection condition:

```text
Bursts of forbidden API calls alert.
High failure-to-success ratios are flagged.
```

---

### Collection Validation

Validate access to sensitive resource metadata and data.

Test:

```text
ConfigMap access
PV/PVC enumeration
VolumeAttachment enumeration
Node filesystem access through workload configuration
Database access through workload credentials
Third-party service access through stored credentials
```

Finding condition:

```text
Non-admin identity can collect sensitive configuration or storage metadata.
```

---

### Exfiltration Validation

Use canary data only.

Validate:

```text
Outbound egress is restricted.
Compression tooling execution is detected.
Outbound transfer tooling is detected.
Unexpected software installation is detected.
NetworkPolicy changes are detected.
Runtime telemetry identifies source pod, namespace, process, and destination.
```

Finding condition:

```text
A workload can send arbitrary outbound data without detection or policy control.
```

---

### Lateral Movement Validation

Validate:

```text
Cross-namespace network traffic is blocked by default.
Pods cannot reach cloud metadata services unless required.
Pods cannot reach kubelet, API server, or etcd unless required.
Workload identity credentials cannot be used outside the intended workload.
pod exec is tightly controlled.
Default service account use is detected.
```

Finding condition:

```text
A compromised pod can move to unrelated services, namespaces, nodes, or cloud IAM permissions.
```

---

### Impact Validation

Use safe simulations only.

Validate:

```text
Workloads without CPU/memory limits are denied or alerted.
Unexpected CPU spikes alert.
Autoscaling anomalies alert.
Patch permissions on workloads are least-privilege.
Delete permissions on persistent volumes are restricted.
Destructive delete attempts generate high-severity alerts.
```

Do not test destructive deletion against production resources.

---

## Indicators of Compromise to Monitor

Monitor for:

```text
system:anonymous with authorization decision allow
system:unauthenticated group activity
system:serviceaccount:default:default activity
Service account activity from outside the cluster
Unexpected /pods/*/exec
Manual pod, deployment, job, or cronjob creation
Public image used in production
Privileged pod creation
hostPath mount creation
hostPID or hostNetwork usage
Wildcard RBAC permissions added
bind, escalate, or impersonate verbs added
Secrets enumeration
ConfigMap enumeration
PV/PVC/VolumeAttachment enumeration
NetworkPolicy modification
Patch operations against deployments
Delete operations against persistent volumes
Workloads without CPU or memory limits
Unexpected compression tools in containers
Unexpected curl, wget, nc, or bash execution
Outbound connections to unknown IPs
Cloud IAM credential use from suspicious locations
Large database reads, deletes, or updates
```

---

## Incident Response Checklist

### Containment

```text
[ ] Disable or revoke suspected kubeconfigs.
[ ] Revoke compromised service account tokens where applicable.
[ ] Remove public API server exposure if present.
[ ] Restrict control-plane access to trusted networks.
[ ] Isolate affected namespaces.
[ ] Quarantine suspicious workloads.
[ ] Block suspicious egress.
[ ] Preserve Kubernetes audit logs.
[ ] Preserve cloud IAM logs.
[ ] Preserve node and runtime telemetry.
[ ] Snapshot affected nodes and volumes where appropriate.
```

---

### Investigation

```text
[ ] Identify initial identity used.
[ ] Correlate identity provider logs with Kubernetes audit logs.
[ ] Review source IPs and user agents.
[ ] Review pod exec activity.
[ ] Review manual pod, deployment, job, and cronjob creation.
[ ] Review service account usage.
[ ] Review RBAC changes.
[ ] Review Secrets, ConfigMaps, PVs, PVCs, and VolumeAttachments access.
[ ] Review NetworkPolicy changes.
[ ] Review runtime process execution.
[ ] Review outbound network connections.
[ ] Review cloud IAM use from workload identities.
[ ] Review CI/CD changes.
[ ] Review image provenance for suspicious workloads.
```

---

### Eradication and Recovery

```text
[ ] Remove unauthorized workloads.
[ ] Remove unauthorized RBAC bindings.
[ ] Remove unauthorized service accounts.
[ ] Rotate exposed Kubernetes Secrets.
[ ] Rotate cloud credentials and workload identity bindings.
[ ] Rotate database credentials.
[ ] Rotate CI/CD credentials.
[ ] Rebuild affected nodes from known-good images.
[ ] Redeploy workloads from trusted images.
[ ] Apply missing admission controls.
[ ] Apply missing NetworkPolicies.
[ ] Patch vulnerable images and nodes.
[ ] Validate audit logging and runtime detections.
[ ] Conduct post-incident RBAC and namespace review.
```

---

## Production Deployment Checklist

### Control Plane

```text
[ ] API server is private or restricted to trusted networks.
[ ] API server uses centralized authentication.
[ ] Anonymous API access is disabled.
[ ] Shared kubeconfigs are prohibited.
[ ] Audit logging is enabled, shipped off-cluster, and tamper-evident.
[ ] Audit retention periods are explicitly defined.
[ ] NTP/time sync is healthy on all nodes.
[ ] Just-in-time elevation is used instead of standing cluster-admin.
[ ] A tested break-glass procedure exists and its use alerts.
[ ] etcd is private.
[ ] etcd requires TLS and client certificate authentication.
[ ] etcd backups are encrypted and access-controlled.
[ ] kubelet anonymous auth is disabled.
[ ] kubelet read-only port is disabled.
[ ] kubelet authorization is not AlwaysAllow.
[ ] NodeRestriction is enabled.
[ ] Shared-responsibility ownership is recorded per control (managed vs self-managed).
```

---

### RBAC and Identity

```text
[ ] Default service accounts have no elevated permissions.
[ ] Production workloads use dedicated service accounts.
[ ] Service account token automounting is disabled unless required.
[ ] Wildcard RBAC permissions are reviewed and minimized.
[ ] bind, escalate, and impersonate verbs are restricted.
[ ] Secret access is least-privilege.
[ ] pods/exec access is restricted.
[ ] RoleBinding and ClusterRoleBinding creation is restricted.
[ ] Workload identities are scoped to intended namespaces and service accounts.
[ ] Static cloud credentials are not stored in manifests.
[ ] Projected, short-lived, audience-scoped tokens are used instead of static token Secrets.
[ ] Legacy long-lived service account token Secrets are audited and removed.
```

---

### Admission and Workload Policy

```text
[ ] Pod Security Admission or equivalent is enforced.
[ ] Privileged containers are blocked.
[ ] Privilege escalation is blocked.
[ ] hostPID, hostIPC, and hostNetwork are blocked unless approved.
[ ] hostPath mounts are blocked unless approved.
[ ] Runtime socket mounts are blocked.
[ ] Dangerous Linux capabilities are blocked.
[ ] Containers run as non-root.
[ ] Root filesystems are read-only where possible.
[ ] Resource requests and limits are required.
[ ] Approved registries are enforced.
[ ] Mutable tags are restricted in production.
[ ] Public images require exception approval.
[ ] Security-critical admission webhooks are fail-closed and highly available.
[ ] Admission exemption lists are reviewed and time-bound.
[ ] seccomp RuntimeDefault plus AppArmor/SELinux profiles are applied; Unconfined is blocked.
[ ] Sandboxed runtime (gVisor/Kata) is used for untrusted or multi-tenant workloads.
[ ] Operators, CRDs, and Helm charts are vetted for privilege before install.
[ ] Critical security/platform components have PriorityClass and PodDisruptionBudget.
```

---

### Network

```text
[ ] Default-deny ingress policies are applied.
[ ] Default-deny egress policies are applied where feasible.
[ ] Required traffic is explicitly allowed.
[ ] Workloads cannot reach etcd.
[ ] Workloads cannot reach kubelet APIs.
[ ] Workloads cannot reach cloud metadata endpoints unless required.
[ ] Cross-namespace traffic is restricted.
[ ] Service-to-service mTLS is implemented where required.
[ ] NetworkPolicy changes are monitored.
[ ] Network flow visibility is available.
[ ] Internet-exposed LoadBalancer/NodePort Services and Ingress routes are inventoried.
[ ] Ingress/gateway tier terminates TLS and sits behind a WAF and rate limiting.
[ ] Management UIs (Kubernetes Dashboard, metrics, tracing) are not publicly exposed.
```

---

### Secrets and Data

```text
[ ] Kubernetes Secrets are encrypted at rest.
[ ] External secret manager is used for high-value secrets.
[ ] Secrets are not stored in ConfigMaps.
[ ] Secrets are not stored in plain environment variables unless explicitly accepted.
[ ] Secret access is logged.
[ ] Secret access patterns are monitored.
[ ] PVs and PVCs are encrypted and access-controlled.
[ ] VolumeAttachment access is restricted.
[ ] Database credentials are scoped and rotated.
[ ] Canary credentials are deployed where useful.
```

---

### Runtime Monitoring

```text
[ ] Container runtime telemetry is enabled.
[ ] Node process monitoring is enabled.
[ ] Alerts exist for shell execution inside containers.
[ ] Alerts exist for curl, wget, nc, and suspicious download behavior.
[ ] Alerts exist for compression followed by outbound transfer.
[ ] Alerts exist for pod exec activity.
[ ] Alerts exist for public image use in production.
[ ] Alerts exist for workloads without resource limits.
[ ] Alerts exist for unusual CPU and autoscaling spikes.
[ ] Alerts exist for cloud IAM use from suspicious locations.
[ ] Every detection has a severity, owner, and response runbook.
[ ] Alerts route to the SOC/on-call system, not just a dashboard.
[ ] Detections are periodically test-fired to confirm they trigger and route.
```

---

### CI/CD and Supply Chain

```text
[ ] Production deployment happens through approved CI/CD.
[ ] Manual production workload creation is restricted or alerted.
[ ] Images are scanned before deployment.
[ ] Running images are rescanned periodically.
[ ] Images are signed and verified where supported.
[ ] Image signatures and attestations are verified at admission, not just in CI.
[ ] SBOMs are generated and retained for every image.
[ ] SLSA provenance attestations are produced and required.
[ ] IaC is scanned before deployment.
[ ] CI/CD changes to security-sensitive fields require review.
[ ] Public repositories and third-party packages are verified.
[ ] Vulnerability feeds are monitored.
```

---

### Backup and Disaster Recovery

```text
[ ] Cluster resource state and PV data are backed up together (e.g., Velero).
[ ] At least one backup copy is offline or immutable (object-lock/WORM) in a separate account.
[ ] Backups and snapshots are encrypted with separately managed keys.
[ ] Backup access is restricted and deletion is alerted.
[ ] RTO and RPO are defined per cluster tier.
[ ] Restores (including etcd and full rebuild) are tested on a schedule.
[ ] A rebuild-from-known-good runbook with credential rotation exists.
```

---

### Continuous Compliance and Lifecycle

```text
[ ] CIS benchmark scans (kube-bench) run on a schedule with tracked results.
[ ] Posture scanning (kubescape/Trivy Operator or equivalent) runs continuously.
[ ] Configuration drift is detected and alerted (or auto-reverted via GitOps).
[ ] Live RBAC and policy are periodically diffed against the reviewed baseline.
[ ] Kubernetes version is within its supported/EOL window.
[ ] Node OS and kernel patching follows a defined cadence.
[ ] Baseline scans are re-run after every cluster upgrade.
[ ] Controls are mapped to CIS/NSA/compliance frameworks with evidence sources.
[ ] Cluster target tier (baseline/hardened/high-security) is assigned and gap-tracked.
```

---

## Summary

The most important Kubernetes hardening controls are:

1. Keep the API server, etcd, and kubelet interfaces private and strongly authenticated.
2. Disable anonymous access and kubelet read-only access.
3. Enforce least-privilege RBAC and restrict `bind`, `escalate`, `impersonate`, wildcard permissions, `pods/exec`, and Secrets access.
4. Use dedicated service accounts and disable automatic token mounting unless required.
5. Enforce admission policies for privileged containers, privilege escalation, host access, mutable filesystems, public registries, and missing resource limits.
6. Use private, trusted, scanned, signed, and digest-pinned images, and verify signatures and attestations at admission.
7. Apply namespace isolation, default-deny NetworkPolicies, and harden the internet-facing ingress edge.
8. Block workload access to cloud metadata, kubelet APIs, etcd, and unnecessary internal services.
9. Use external secret managers, short-lived projected tokens, and encrypt Secrets, etcd, volumes, and backups.
10. Keep admission enforcement itself reliable — fail-closed, highly available, and monitored.
11. Monitor Kubernetes audit logs and runtime telemetry together, with tamper-evident off-cluster storage and synchronized time.
12. Give every detection a severity, owner, and response runbook so alerts drive action.
13. Detect attacker-like API patterns across initial access, persistence, privilege escalation, credential access, discovery, collection, exfiltration, lateral movement, and impact.
14. Maintain immutable, tested backups and a rebuild-from-known-good recovery plan.
15. Validate controls continuously (scanning, drift detection, version lifecycle) and through safe, authorized red-team testing.

Adapt scope to whether the cluster is managed or self-managed, prioritize by maturity tier, and map controls to recognized frameworks so hardening stays auditable.

Kubernetes should be secured as a control plane, a distributed runtime, and a cloud identity boundary. A normal container hardening baseline is not enough.
