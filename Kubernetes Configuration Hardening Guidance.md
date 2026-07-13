# Kubernetes Configuration Hardening Guidance

| Field | Value |
|---|---|
| Document owner | Platform Security / Cloud Infrastructure |
| Version | 2.1 |
| Last reviewed | 2026-07-13 |
| Review cadence | Every 6 months and on major Kubernetes, CNI, runtime, or cloud-provider upgrades |
| Applies to | Production Kubernetes clusters, managed and self-managed |
| Primary references | CIS Kubernetes Benchmarks, NSA/CISA Kubernetes Hardening Guidance, Kubernetes security documentation, MITRE ATT&CK for Containers |

## Purpose

Kubernetes is both a high-trust control plane and a distributed workload runtime. It commonly controls application code, service identities, cloud permissions, secrets, data stores, CI/CD automation, and horizontally scalable compute. A single weak control can provide a path from a compromised workload to a node, the cluster control plane, or the surrounding cloud environment.

This guidance establishes a risk-based production baseline. It is not a substitute for:

- A benchmark matched to the exact Kubernetes distribution and version.
- Cloud-provider documentation for managed control-plane settings.
- Threat modeling for the applications, tenants, and data hosted by the cluster.
- Testing controls in a non-production environment before enforcement.

Version-sensitive examples must be validated with server-side dry run, schema validation, and policy tests against every supported cluster minor version.

## Core security principle

> Assume a compromised workload will use its service account, network position, mounted data, local node services, and cloud identity to discover, escalate, persist, move laterally, collect data, and exfiltrate.

Namespaces, RBAC, NetworkPolicy, Pod Security Admission, workload identity, and admission policy provide useful controls only when configured consistently and continuously tested. Containers normally share the host kernel and are not equivalent to virtual machines as an isolation boundary.

## Scope and shared responsibility

Responsibility depends on who operates the control plane and worker nodes.

| Component | Managed cluster | Self-managed cluster |
|---|---|---|
| API server flags and admission plugins | Provider-controlled or exposed through provider settings | Operator responsibility |
| etcd TLS, access, backup, and at-rest encryption | Usually provider-managed; verify documented behavior and options | Operator responsibility |
| Audit configuration and export | Enable through provider options | Configure policy, backend, export, and retention |
| Kubelet, node OS, kernel, and runtime | Shared; responsibility varies by node type | Operator responsibility |
| RBAC, namespaces, workloads, images, secrets, and most network policy | Customer responsibility | Operator responsibility |
| Cloud IAM and workload identity | Shared with cloud provider | Customer responsibility where used |

For every control, record:

```text
control owner
implementation location
applicable cluster types and versions
evidence source
exception owner and expiration
last test date
```

Do not translate a provider-managed control into an unsupported command-line flag. Verify the provider default and configure the provider-native equivalent.

## Prioritization tiers

### Tier 0 — production baseline

- Restrict API server network access and disable anonymous API access unless a reviewed compatibility requirement exists.
- Enable Kubernetes audit logging and export it off-cluster.
- Enforce least-privilege RBAC; prohibit workload `cluster-admin` and unjustified wildcards.
- Enforce Pod Security Admission `baseline` at minimum and move application namespaces toward `restricted`.
- Use dedicated service accounts; disable token automounting where API credentials are unnecessary.
- Apply default-deny ingress policy with a CNI verified to enforce it.
- Encrypt Kubernetes Secrets at rest and protect encryption keys.
- Patch supported cluster, node, runtime, CNI, ingress, and policy-engine versions.

### Tier 1 — hardened

- Apply default-deny egress where feasible and explicitly allow required destinations.
- Block or tightly constrain cloud metadata and node-local administrative endpoints.
- Enforce workload invariants with stable in-process admission policy or a highly available policy engine.
- Use short-lived workload identities and service account tokens.
- Pin images by digest; scan, sign, and verify them at admission.
- Use external secret management with a documented delivery model.
- Continuously scan configuration and running images and detect drift.
- Deploy runtime and network telemetry with owned response playbooks.

### Tier 2 — high-security or hostile multi-tenancy

- Use sandboxed runtimes, dedicated nodes, virtual control planes, or separate clusters according to the threat model.
- Enforce AppArmor or SELinux profiles in addition to seccomp.
- Use workload-authenticated mTLS with authorization policy where justified.
- Apply controlled egress gateways and Layer 7 or FQDN-aware policy where required.
- Require verified provenance, SBOMs, and attestations.
- Use just-in-time privileged access and tested break-glass controls.

Namespaces alone do not provide hostile-tenant isolation. When tenants do not trust one another, evaluate dedicated nodes, sandboxed runtimes, virtual control planes, or separate clusters.

## Authentication and API access

### Restrict control-plane exposure

Prefer private API endpoints. If public reachability is required, document the reason and combine:

- Narrow source restrictions.
- Strong centralized identity.
- MFA for people.
- Short-lived credentials.
- Conditional access where supported.
- Rate limiting, monitoring, and alerting.

Do not expose etcd, kubelet APIs, controller or scheduler diagnostics, or administrative dashboards to the public internet.

### Use attributable identities

Prefer OIDC, cloud IAM integration, or another centralized identity provider for people. Use individual identities and short-lived credentials. Prohibit shared kubeconfigs, generic administrator accounts, long-lived admin certificates, and static credentials in CI/CD unless a documented exception exists.

Each human action must be attributable in audit logs. Automation must use a named non-human identity with narrowly scoped permissions and an owner.

### Use least-privilege RBAC

Review these permissions as escalation-capable or sensitive:

```text
wildcard verbs or resources
bind
escalate
impersonate
create or update RoleBinding and ClusterRoleBinding
create workloads in privileged namespaces
create pods that can use privileged service accounts or sensitive volumes
get, list, or watch Secrets
pods/exec
pods/attach
pods/portforward
create or modify admission configuration
create or modify RuntimeClass, PriorityClass, nodes, or CSRs
delete persistent volumes or backups
```

Creating a Pod can be equivalent to using the Pod's service account and any accessible Secret, volume, node feature, or workload identity. Treat workload creation in sensitive namespaces as a powerful permission.

Review access with explicit identities:

```bash
kubectl auth can-i --list
kubectl auth can-i get secrets --all-namespaces
kubectl auth can-i create pods -n production
kubectl auth can-i create clusterrolebindings
kubectl auth can-i impersonate users
kubectl auth can-i \
  --as=system:serviceaccount:<namespace>:<serviceaccount> \
  --list
```

Do not use `kubectl auth can-i` as the only access review. Analyze effective permissions, aggregation, group membership, impersonation, admission constraints, and escalation paths.

### Just-in-time and break-glass access

- Keep day-to-day human privileges minimal.
- Grant elevated access for a defined scope and duration through an approval workflow.
- Record requester, approver, justification, scope, start, and expiry.
- Revoke automatically.
- Maintain a separately protected emergency credential for identity-provider outages.
- Alert immediately on break-glass use and rotate or reseal the credential after use.
- Test both elevation and emergency procedures on a schedule.

## Control-plane and node hardening

### etcd

For self-managed etcd:

- Permit access only from required control-plane components and administrative recovery systems.
- Require mutually authenticated TLS for client and peer traffic.
- Validate trusted CAs; protect private keys.
- Restrict filesystem access to data and configuration.
- Encrypt and access-control snapshots.
- Monitor access, snapshot creation, deletion, and restore activity.
- Test restore procedures.

Treat etcd read access as highly sensitive and write access as cluster compromise. Managed-cluster users must verify the provider's controls and backup model.

### Kubelet

Prefer the structured kubelet configuration rather than relying on command-line flags:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
readOnlyPort: 0
rotateCertificates: true
```

The effective configuration is version- and distribution-dependent. Verify it on each node rather than assuming the configuration file was loaded.

Also:

- Restrict TCP 10250 to required control-plane and administrative sources.
- Never use `AlwaysAllow` authorization in production.
- Protect kubelet client and serving certificates.
- Verify certificate rotation and CSR approval behavior.
- Monitor kubelet access and node-level execution.
- Do not expose the deprecated unauthenticated read-only service.

### Node authorization and NodeRestriction

Use the Node authorizer together with the NodeRestriction admission plugin. Ensure kubelets authenticate in the `system:nodes` group with usernames in the form `system:node:<nodeName>`. NodeRestriction is not a substitute for correct node identity, Node authorizer configuration, node network controls, or node credential protection.

### Node operating system and runtime

- Use minimal, immutable or replaceable node images where practical.
- Patch the OS, kernel, container runtime, CNI, CSI, and node agents on a defined cadence.
- Minimize direct SSH; use audited access and rebuild nodes after suspected compromise.
- Disable unused services and kernel modules.
- Protect runtime sockets and host paths from workloads.
- Separate sensitive and untrusted workloads with taints, affinity, dedicated pools, and stronger isolation where required.

## Namespace and tenancy architecture

Namespaces are logical management and policy scopes, not standalone security boundaries. Use them to separate environments, teams, applications, and tenants so RBAC, quota, Pod Security Admission, network policy, and ownership can be applied precisely.

Avoid application workloads in:

```text
default
kube-system
kube-public
kube-node-lease
```

Create namespaces through a controlled workflow that applies required labels and baseline resources atomically. Monitor for missing or modified labels.

Example Pod Security Admission labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Pin Pod Security Standard versions when reproducibility across Kubernetes upgrades is required. Test policy changes in `warn` and `audit` modes before enforcement.

For hostile multi-tenancy, assess cluster-scoped resources, shared nodes, kubelet identities, DNS visibility, storage, webhooks, CRDs, and control-plane denial-of-service risk. Use stronger isolation when namespace-scoped controls do not satisfy the threat model.

## Admission control

### Enforcement strategy

Use Pod Security Admission for standardized Pod controls. Use stable `ValidatingAdmissionPolicy` with CEL for in-process invariants when it can express the rule. Use external policy engines for controls requiring richer context, mutation, image verification, or external data.

Common production controls include:

- Deny privileged containers and privilege escalation.
- Deny host PID, IPC, and network access unless approved.
- Deny unsafe host paths and runtime socket mounts.
- Require non-root execution and approved seccomp/AppArmor settings.
- Drop Linux capabilities by default and allow only reviewed additions.
- Require approved image registries, immutable digests, and signature verification.
- Disable service account token automounting unless required.
- Require ownership metadata and an approved namespace.
- Enforce resource policy appropriate to the workload class.

Controls such as detecting SSH servers, malware, or packages inside an image cannot be inferred reliably from a PodSpec alone. Enforce those through image build, scanning, attestations, and admission-time verification.

### ValidatingAdmissionPolicy example

The following complete example uses the stable API and denies Pods whose regular containers do not explicitly disable privilege escalation. Test it against the supported cluster versions before enforcement.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-no-privilege-escalation.example.com
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: >-
        object.spec.containers.all(c,
          has(c.securityContext) &&
          has(c.securityContext.allowPrivilegeEscalation) &&
          c.securityContext.allowPrivilegeEscalation == false)
      message: "Every container must set securityContext.allowPrivilegeEscalation to false"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-no-privilege-escalation.example.com
spec:
  policyName: require-no-privilege-escalation.example.com
  validationActions: [Deny, Audit]
  matchResources:
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: [kube-system]
```

This example covers only regular containers. A production policy must deliberately address init and ephemeral containers, exemptions, controller-generated Pods, and system namespaces.

### External admission webhook reliability

For security-critical validating webhooks:

- Use `failurePolicy: Fail` only after testing outage and recovery behavior.
- Use `matchPolicy: Equivalent` unless a reviewed reason requires otherwise.
- Keep timeouts short and observable.
- Scope rules and match conditions narrowly.
- Run multiple replicas across failure domains with adequate priority and disruption protection.
- Protect serving certificates and monitor expiry.
- Do not create a dependency cycle in which the webhook must approve the resources needed to restore itself.
- Maintain an audited break-glass recovery path.
- Test API-server and webhook version skew during upgrades.
- Review every user, group, namespace, and object exemption; make exceptions time-bound.

Do not copy a webhook fragment without supplying a valid `clientConfig`, CA trust, service, rules, and tested recovery design.

## Workload baseline

Use a restrictive baseline while allowing application-specific UID and storage requirements:

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
      serviceAccountName: hardened-app
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
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
              drop: ["ALL"]
            appArmorProfile:
              type: RuntimeDefault
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 512Mi
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 128Mi
```

Notes:

- Use a verified digest in a real manifest.
- Set a numeric UID/GID only when it is compatible with the image and storage model.
- Avoid hard-coded SELinux MCS categories unless the platform deliberately manages category allocation.
- A read-only root filesystem normally requires explicit writable mounts.
- Require resource requests. Define memory and CPU limit policy by workload class; omitting CPU limits can be appropriate when quota and capacity controls prevent abuse and latency-sensitive workloads would otherwise be throttled.
- Verify AppArmor is enabled on eligible nodes. Explicit `RuntimeDefault` can prevent scheduling or admission on nodes without AppArmor support.

### Resource governance

Use `ResourceQuota`, `LimitRange`, admission policy, and capacity monitoring together. The goals are fair scheduling, bounded memory use, controlled object counts, and resistance to accidental or malicious exhaustion.

Example namespace quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.memory: 40Gi
    pods: "100"
```

Do not apply one universal CPU limit ratio to every workload. Document approved profiles for latency-sensitive services, batch jobs, platform agents, and untrusted code.

### Mandatory access control and sandboxing

Use `seccompProfile: RuntimeDefault` as a baseline. Add AppArmor or SELinux according to node support. Block `Unconfined` in production unless a time-bound exception exists.

For untrusted or high-value workloads, configure a sandboxed runtime on eligible nodes and reference it from the workload PodSpec:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sandboxed-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sandboxed-app
  template:
    metadata:
      labels:
        app: sandboxed-app
    spec:
      runtimeClassName: gvisor
      containers:
        - name: app
          image: registry.example.com/app@sha256:<digest>
```

The `runsc` handler must exist on every eligible node, or the RuntimeClass must define scheduling constraints. Restrict writes to RuntimeClass objects to cluster administrators.

## Service accounts and workload identity

- Use one dedicated service account per workload or trust domain.
- Do not bind application service accounts to `cluster-admin`.
- Set `automountServiceAccountToken: false` where Kubernetes API credentials are unnecessary.
- Prefer cloud workload identity and other short-lived credential mechanisms over static cloud access keys.
- Bind cloud identities to the intended cluster, namespace, and service account, including issuer and audience conditions.

When a workload needs a non-default audience or mount location, project a bounded token explicitly:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-client
  namespace: production
spec:
  serviceAccountName: api-client
  automountServiceAccountToken: false
  containers:
    - name: client
      image: registry.example.com/client@sha256:<digest>
      volumeMounts:
        - name: api-token
          mountPath: /var/run/identity
          readOnly: true
  volumes:
    - name: api-token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              audience: my-service
              expirationSeconds: 3600
```

Only containers mounting the volume can read it. Do not use `subPath` when token rotation is required. The recipient must validate the expected audience.

Avoid manually created long-lived `kubernetes.io/service-account-token` Secrets. Inventory existing legacy tokens, validate ownership, and remove or rotate unnecessary tokens.

## Network segmentation

### Verify enforcement before relying on policy

The NetworkPolicy API has no effect unless the installed CNI implements it. Test both allowed and denied paths after installation and upgrades.

Standard NetworkPolicy is primarily Layer 3/4 and has implementation-dependent behavior for `hostNetwork` Pods, node traffic, service address translation, and some control-plane paths. It does not provide portable FQDN policy, forced egress gateways, TLS identity, or Layer 7 authorization. Use CNI-specific host policies, node firewalls, cloud security groups, egress gateways, service meshes, or proxies where those controls are required.

### Default deny

Apply ingress default deny to production namespaces. Apply egress default deny after dependencies are inventoried and operational recovery paths are tested.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

Add explicit allow policies for required traffic.

### DNS allow policy

The following example limits DNS to Pods with the common CoreDNS label in `kube-system`. Verify labels and architecture in the actual cluster; NodeLocal DNS and managed resolvers may need a different rule.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cluster-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Test name resolution and verify that traffic to non-DNS Pods in `kube-system` remains denied.

### Sensitive destinations

Block by default or explicitly authorize:

```text
cloud metadata services
kubelet APIs
etcd
node and runtime administration ports
Kubernetes API access for workloads that do not need it
secret managers
CI/CD and source control systems
internal administrative dashboards
unexpected internet destinations
```

Metadata protection is provider-specific. Use workload identity, least-privilege node identities, provider metadata protections, hop limits or headers where applicable, and network controls. Do not assume a standard NetworkPolicy alone blocks every metadata or node-local path.

### Ingress and gateways

- Inventory every Ingress, Gateway, `LoadBalancer`, `NodePort`, and externally routable IP.
- Prefer a small number of hardened ingress or gateway tiers.
- Use TLS, authentication, rate limiting, and WAF controls where appropriate for the protocol and risk.
- Restrict dangerous controller-specific annotations and configuration snippets.
- Scope ingress-controller RBAC and Secret access.
- Keep management interfaces private and behind authenticated administrative access.

## Secrets and data

### Secret delivery models

An external secret manager does not automatically mean a secret never enters Kubernetes. Document which model is used:

1. **Synchronized Kubernetes Secret:** an operator copies the value into a Secret. Kubernetes RBAC and at-rest encryption remain critical, and rotation depends on synchronization.
2. **CSI or file mount:** the value is mounted from an external provider and may avoid persistence in etcd, depending on the driver and configuration.
3. **Brokered short-lived credential:** the workload receives an ephemeral credential through workload identity. Prefer this for cloud and database access where supported.

Do not store long-lived cloud access keys in manifests, images, ConfigMaps, Helm values, Git, or unprotected CI variables. If compatibility requires a Kubernetes Secret, use encryption at rest, narrow RBAC, rotation, audit, and a migration plan.

Environment variables are easy to consume but are static for the process lifetime and may leak through diagnostics, crash reports, or child processes. Prefer mounted files or brokered credentials when rotation and reduced exposure matter.

### Encryption and key management

- Encrypt Kubernetes API data at rest with an appropriate provider or KMS integration.
- Protect and rotate encryption keys.
- Verify existing objects have been rewritten after enabling or changing encryption.
- Encrypt persistent volumes, snapshots, registry storage, artifacts, and backups.
- Separate key administration from data administration where required.

### Persistent storage

- Restrict PV, PVC, StorageClass, CSI, and VolumeAttachment permissions.
- Avoid `hostPath` for application data.
- Use approved encrypted storage classes.
- Define reclaim, snapshot, backup, and deletion policies.
- Prevent cross-tenant reuse of residual data.
- Monitor unusual attachment, snapshot, restore, and deletion activity.

## Image and software supply chain

- Use trusted, maintained base images and remove unnecessary tools.
- Pin deployed images by digest.
- Scan source, dependencies, manifests, and images before deployment.
- Rescan running images as vulnerability intelligence changes.
- Generate and retain SPDX or CycloneDX SBOMs.
- Produce provenance and other required attestations.
- Sign images and attestations with protected keys or tightly scoped keyless identities.
- Verify signatures, issuer, subject, repository binding, digest, and attestations at admission.
- Define vulnerability exception criteria, owner, expiry, and compensating controls.

Registry allowlisting is not sufficient by itself: an approved registry can contain an untrusted image. Combine registry controls with digest pinning and cryptographic verification.

For Helm charts, operators, and CRDs:

- Pin a reviewed chart or operator version.
- Verify provenance or the OCI artifact digest where supported.
- Render manifests and inspect cluster-scoped RBAC, Secrets access, webhooks, CRDs, host access, and images before installation.
- Prefer namespaced operation where supported.
- Inventory CRDs and operators and re-review them on upgrade.
- Avoid blanket policy exemptions for operator namespaces.

## Availability of security controls

Security controls must remain available during maintenance and pressure events.

- Run admission engines and gateways with multiple replicas across failure domains.
- Use PodDisruptionBudgets for voluntary disruptions.
- Assign PriorityClasses deliberately and restrict who can use high priority.
- Reserve resources for node and cluster-critical components.
- Monitor webhook latency, rejection rate, timeouts, and certificate expiry.
- Test failure modes, upgrades, and disaster recovery.

A PodDisruptionBudget does not protect against involuntary disruption or guarantee service availability. Priority can also cause preemption and must be governed as a sensitive capability.

## Audit logging and detection

### Audit baseline

Capture enough metadata to identify:

```text
timestamp and audit stage
user and groups
source IPs
user agent
verb
request URI
object resource, namespace, and name
authorization decision and reason when available
response status
request and stage timestamps
```

Use `Metadata` broadly and raise selected sensitive resources to `Request` or `RequestResponse` only after assessing secret and personal-data leakage. Never log Secret bodies indiscriminately.

Ship logs off-cluster to access-controlled, tamper-evident storage. Define retention, monitor gaps and deletion, and synchronize time across Kubernetes, cloud, identity, CI/CD, node, runtime, network, and secret-manager sources.

### High-value audit detections

- Allowed anonymous requests.
- Bursts of denied requests: authorization decision `forbid` or HTTP 403, not an API verb named `forbid`.
- New wildcard, `bind`, `escalate`, or `impersonate` RBAC.
- Unexpected Pod creation, `pods/exec`, attach, or port-forward.
- Secret enumeration or TokenRequest anomalies.
- Admission, Pod Security labels, audit policy, or NetworkPolicy changes.
- Privileged Pods, host access, runtime sockets, or unapproved RuntimeClasses.
- Persistent volume, snapshot, or backup deletion.
- Workload identity use from unexpected sources.

### Runtime detection

Shells, downloaders, compression tools, package managers, token reads, and ConfigMap access can be legitimate. Treat them as contextual signals rather than universal incidents.

Raise severity when combined with factors such as:

- A production workload that has no administrative function.
- Interactive execution through `pods/exec`.
- A new or unusual parent process, user, destination, or binary.
- Token access followed by API discovery.
- Compression followed by unusual egress.
- Execution from writable temporary paths.
- A workload identity used outside its expected environment.

Every detection needs an owner, severity, routing target, tuning method, evidence fields, and response playbook. Test detections periodically with safe canary activity.

## CI/CD and configuration governance

- Deploy production changes through approved, attributable automation.
- Restrict or alert on out-of-band changes.
- Require review for RBAC, admission, Pod Security labels, service accounts, workload identity, images, commands, security contexts, volumes, networking, and external exposure.
- Protect pipeline definitions, runners, deployment credentials, signing identities, and artifact stores.
- Separate build and deployment permissions.
- Detect drift between reviewed desired state and the live cluster.
- Use server-side dry run, schema validation, policy tests, and staging rollout before enforcement.

Emergency changes must be time-bound, recorded, reviewed afterward, and reconciled into the declared configuration.

## Backup, recovery, and ransomware resilience

Back up the resources required to rebuild service, including:

```text
control-plane state or provider-supported equivalent
cluster manifests, CRDs, RBAC, and policies
persistent data
external secret-manager and key-management dependencies
registry artifacts and deployment configuration
```

- Keep at least one immutable or offline copy in a separate security boundary.
- Encrypt backups with separately managed keys.
- Restrict and log backup, snapshot, retention, and deletion actions.
- Define RTO and RPO by cluster tier.
- Test application-consistent restore, control-plane recovery where applicable, and full rebuild.
- Maintain a known-good rebuild procedure that includes credential rotation and trusted images.
- Treat all credentials reachable from a compromised cluster as exposed until assessed and rotated.

## Continuous assurance

- Run the CIS benchmark matched to the Kubernetes distribution and version. Do not treat an unmatched generic profile as authoritative evidence.
- Run posture and vulnerability scanning continuously.
- Track findings with owner and remediation SLA.
- Compare live RBAC, admission, namespace labels, and network policy with the reviewed baseline.
- Track Kubernetes and component support windows and upgrade before end of support.
- Re-run functional security tests after upgrades, not only configuration scans.
- Validate provider-managed controls and record provider evidence.
- Reassess the assigned maturity tier when exposure, tenants, or data sensitivity changes.

## Safe validation plan

Perform authorized tests with canary identities and data in an approved environment.

Validate that:

- Anonymous and unauthorized API requests are denied and logged.
- Low-privilege identities cannot escalate through Pods, RBAC, impersonation, or service accounts.
- Risky Pod configurations and unverified images are rejected.
- Default service accounts have no unintended privileges.
- Network denies are enforced by the actual CNI, including tested node, host-network, DNS, metadata, and control-plane paths.
- Secret and storage access is least-privilege and observable.
- Workload identities cannot be reused outside their intended context.
- Destructive actions generate high-severity alerts without testing deletion against production data.
- Admission controls recover safely from outages.
- Backup restoration meets RTO and RPO.

Record the identity, source, request, decision, response, alert, owner, and remediation for every test.

## Incident response checklist

### Containment

```text
[ ] Revoke suspected user, kubeconfig, token, and cloud credentials.
[ ] Restrict API and administrative access to trusted sources.
[ ] Isolate affected workloads, namespaces, nodes, and egress as appropriate.
[ ] Preserve audit, identity, cloud, node, runtime, and network evidence.
[ ] Snapshot affected systems where forensically appropriate.
[ ] Protect backups and prevent attacker access to recovery systems.
```

### Investigation

```text
[ ] Identify the initial identity, source, and authentication path.
[ ] Review Pod creation, exec, attach, port-forward, and ephemeral containers.
[ ] Review RBAC, service account, token, CSR, admission, and namespace-label changes.
[ ] Review Secret, ConfigMap, storage, backup, and secret-manager access.
[ ] Review images, provenance, CI/CD, runtime processes, and outbound connections.
[ ] Determine node, tenant, cloud account, and data blast radius.
```

### Eradication and recovery

```text
[ ] Remove unauthorized workloads, identities, bindings, and persistence.
[ ] Rotate exposed Kubernetes, cloud, database, CI/CD, and signing credentials.
[ ] Rebuild affected nodes and redeploy from reviewed images and configuration.
[ ] Restore from immutable known-good backups when integrity is uncertain.
[ ] Close policy and detection gaps.
[ ] Validate recovery, logging, alerting, and workload identity boundaries.
[ ] Complete a post-incident review and track corrective actions.
```

## Production checklist

### Control plane and nodes

```text
[ ] API access is private or narrowly restricted.
[ ] Central identity, MFA, attribution, and short-lived credentials are used.
[ ] Anonymous API access is disabled or formally excepted.
[ ] Audit logs are exported, protected, retained, and tested.
[ ] etcd and backups are private, authenticated, encrypted, and recoverable.
[ ] Kubelet anonymous access and read-only service are disabled.
[ ] Kubelet webhook authorization and certificate rotation are verified.
[ ] Node authorizer and NodeRestriction are correctly configured where operator-controlled.
[ ] Nodes, runtime, CNI, CSI, and ingress components are supported and patched.
[ ] Managed-provider ownership and evidence are recorded.
```

### Identity and admission

```text
[ ] RBAC escalation paths and wildcards are reviewed.
[ ] Workloads use dedicated least-privilege service accounts.
[ ] Token automounting is disabled unless required.
[ ] Long-lived service account token Secrets are absent or excepted.
[ ] Workload identity is bound to the intended namespace and service account.
[ ] Pod Security Admission is enforced at the approved level.
[ ] Privilege, host access, dangerous capabilities, and unconfined profiles are denied.
[ ] Stable admission APIs are used and manifests are version-tested.
[ ] Webhooks are scoped, highly available, observable, and recoverable.
[ ] Admission exemptions are owned and time-bound.
```

### Workloads, supply chain, and resources

```text
[ ] Containers run non-root with least privilege and RuntimeDefault seccomp.
[ ] Read-only root filesystems and MAC profiles are used where compatible.
[ ] Resource requests, quotas, and workload-class limit policies are enforced.
[ ] Images are digest-pinned, scanned, signed, and verified at admission.
[ ] SBOM and provenance requirements are enforced by risk tier.
[ ] Operators, charts, CRDs, and RuntimeClasses are reviewed and inventoried.
[ ] Sandboxed runtimes or stronger isolation protect untrusted workloads.
```

### Network, secrets, and data

```text
[ ] NetworkPolicy enforcement is functionally tested with the installed CNI.
[ ] Default-deny ingress and approved egress controls are applied.
[ ] DNS policy targets only approved resolvers and matches the cluster architecture.
[ ] Host-network, node, metadata, kubelet, API, and etcd paths have compensating controls.
[ ] External exposure is inventoried and management interfaces remain private.
[ ] Secret delivery models and residual storage are documented.
[ ] Secrets and API data are encrypted at rest with protected keys.
[ ] Static cloud credentials are eliminated or formally excepted and rotated.
[ ] Persistent data, snapshots, and attachments are encrypted and access-controlled.
```

### Detection, delivery, and recovery

```text
[ ] Audit, cloud, identity, runtime, and network telemetry are correlated.
[ ] Detections are contextual, tuned, owned, routed, and test-fired.
[ ] Production delivery occurs through protected, attributable automation.
[ ] Drift and out-of-band changes are detected.
[ ] Version-specific CIS and posture checks run continuously.
[ ] Immutable backups exist in a separate security boundary.
[ ] Restore and full rebuild procedures meet tested RTO and RPO.
[ ] Just-in-time and break-glass access are tested and audited.
```

## Authoritative references

- [Kubernetes: Security Checklist](https://kubernetes.io/docs/concepts/security/security-checklist/)
- [Kubernetes: Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [Kubernetes: Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes: Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Kubernetes: RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Kubernetes: Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes: Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes: Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Kubernetes: RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [Kubernetes: Kubelet Configuration API](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [NSA/CISA: Kubernetes Hardening Guidance, version 1.2](https://www.nsa.gov/Press-Room/Digital-Media-Center/Document-Gallery/igphoto/2003066362/)
- [CIS Kubernetes Benchmarks](https://www.cisecurity.org/benchmark/kubernetes)
- [MITRE ATT&CK: Containers Matrix](https://attack.mitre.org/matrices/enterprise/containers/)

## Summary

The highest-value controls are strong attributable identity, least-privilege RBAC, restricted control-plane and kubelet exposure, enforced workload policy, short-lived workload credentials, verified software supply chain, tested network segmentation, protected secrets and storage, correlated audit and runtime telemetry, and immutable tested recovery.

Treat Kubernetes as a control plane, workload runtime, node fleet, software supply chain, and cloud identity boundary. Validate controls against the exact distribution, version, CNI, runtime, and provider rather than relying on generic manifests alone.
