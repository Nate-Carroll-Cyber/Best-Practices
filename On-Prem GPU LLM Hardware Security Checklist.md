# On-Prem GPU LLM Hardware Security Checklist

Scope: hardware, firmware, and platform-layer controls for on-prem GPU inference clusters (H100/H200-class, SXM/HGX chassis), extended with regulatory obligations (§9) and model-weight, data-pipeline, and agentic controls drawn from joint agency guidance (§10–12). Application-layer prompt/RAG filtering remains out of scope. Items marked ⚠ correct or downgrade claims from the original source material.

## 1. Physical & Chassis

- [ ] Rack GPU servers in access-controlled, monitored space; log physical access
- [ ] Chassis intrusion detection enabled in BMC where supported
- [ ] Asset inventory records GPU UUIDs, serials, and firmware versions per node

## 2. Boot Chain & Firmware

- [ ] UEFI Secure Boot enabled in server SBIOS; only signed NVIDIA kernel modules (`nvidia.ko`) load
- [ ] GPU VBIOS and GSP firmware versions pinned and tracked; updates only from NVIDIA-signed packages
- [ ] Hardware vendor advisories (GPU, CPU, memory) monitored and patched promptly, preferably consumed via the Common Security Advisory Framework (CSAF) [2]
- [ ] Host measured boot via platform TPM 2.0 (covers host boot chain; distinct from GPU attestation — see §4)
- [ ] Firmware update process gated: change control, hash verification, rollback plan
- [ ] Default administrative credentials (BMC, DGX OS `admin/admin`-style accounts) rotated before first network attach

## 3. Out-of-Band Management (BMC/IPMI/Redfish)

- [ ] BMC on isolated management VLAN; no internet exposure, no corporate-subnet exposure
- [ ] BMC reachable only via dedicated jump host or hardware VPN
- [ ] IPMI-over-LAN disabled if unused; Redfish with TLS + unique credentials otherwise
- [ ] BMC firmware patched on the same cadence as host firmware
- [ ] BMC event logs forwarded to SIEM

## 4. Confidential Computing / TEE (Hopper CC mode)

- [ ] CC-On mode enabled where the threat model requires protection from a compromised host OS/hypervisor
- [ ] CPU TEE paired: AMD SEV-SNP or Intel TDX enabled on host; CPU↔GPU transfers pass through encrypted bounce buffers
- [ ] ⚠ GPU attestation performed via NVIDIA's on-die root of trust: SPDM session, signed measurement report, verified with nvtrust (local verifier) or NVIDIA Remote Attestation Service (NRAS). The server TPM does not attest the GPU.
- [ ] Composite attestation (CPU TEE + GPU) gated before model weights are decrypted/loaded
- [ ] GPU certificate revocation checked via NVIDIA OCSP
- [ ] ⚠ NVLink: on Hopper, CC mode does not encrypt NVLink; multi-GPU NVLink encryption arrives with Blackwell. Validate multi-GPU CC topology against the driver release notes for your stack before assuming coverage.
- [ ] Accept documented CC-mode performance overhead in capacity planning

## 5. Partitioning & Multi-Tenancy

- [ ] MIG used for hard partitioning when sharing a GPU across trust boundaries
- [ ] ⚠ MIG instances have no NVLink peer-to-peer — isolation comes from disabling P2P, not "routing NVLink domains"
- [ ] SR-IOV / NVIDIA vGPU for VM-level sharing instead of software-only wrappers
- [ ] No time-slicing (sequential GPU reuse without scrub guarantees) across tenants of different trust levels
- [ ] Kubernetes: NVIDIA GPU Operator + device plugin with exclusive GPU assignment per pod unless MIG is in use

## 6. VRAM Hygiene

- [ ] Treat residual HBM contents (weights, KV cache, prompts) as sensitive-at-rest until scrubbed
- [ ] ⚠ `InitializeAndSanitizeOnAlloc=1` from the source material is not a documented NVIDIA driver parameter — do not deploy it. Verify actual module parameters on your driver build via `cat /proc/driver/nvidia/params` / `modinfo`. `NVreg_InitializeSystemMemoryAllocations` covers driver system-memory allocations, not VRAM.
- [ ] ⚠ preStop hooks that allocate `torch.zeros` across VRAM are best-effort only: they do not run on OOM kill, node crash, or force-delete, and the CUDA allocator cannot guarantee coverage of fragmented or driver-reserved regions. Defense-in-depth at most; never the primary control.
- [ ] Primary controls, in order of assurance: (1) CC mode — TEE teardown triggers hardware scrub of enclave memory; (2) MIG/vGPU hard partitions; (3) empirically validated driver scrub-on-alloc behavior for your specific driver version (test with a residual-read PoC, don't assume)
- [ ] GPU reset (`nvidia-smi --gpu-reset`) or node drain-and-scrub in the decommission/reassignment workflow
- [ ] RMA/disposal process treats GPUs and storage as data-bearing devices; sanitize per NIST SP 800-88 (cryptographic erase, block erase, or overwrite) before repurpose or decommission [4]

## 7. Host & Network Isolation

- [ ] GPU inference nodes microsegmented from corporate network; default-deny east-west
- [ ] Egress from inference hosts blocked by default; explicit allowlist for model registry / package mirrors only
- [ ] Model weights on encrypted-at-rest volumes, mounted read-only into inference containers
- [ ] Weight integrity verified (hash/signature) at load time
- [ ] Inference engines (vLLM, TensorRT-LLM, Ollama) run as non-root, dropped capabilities, seccomp profile applied
- [ ] No privileged containers on GPU nodes; NVIDIA Container Toolkit configured without `--privileged`
- [ ] mTLS for all machine-to-machine paths touching the inference tier

## 8. Monitoring & Audit

- [ ] DCGM (or equivalent) telemetry: utilization, ECC error counts, XID errors, thermal anomalies
- [ ] Alert on unexpected GPU process launches, driver reload events, firmware version drift
- [ ] BMC, host, and container logs correlated in SIEM
- [ ] ECC enabled and error trends reviewed (silent corruption + tamper indicator)
- [ ] Periodic re-attestation for CC-mode fleets, not just at provisioning
- [ ] User and entity behavior analytics (UEBA) applied to detect insider threats against model assets [2]
- [ ] Logs written to immutable backup storage; treat logs as sensitive assets with their own CIA controls [1][2]

## 9. Regulatory & Compliance (EU AI Act / GDPR)

Applies when the system serves EU users or processes EU personal data. Adapted from cloud-GPU source material to on-prem; provider-facing items (DPA, EU-region node selection) dropped or restated.

### Pre-Deployment

- [ ] Classify the system against EU AI Act risk tiers (prohibited / Annex III high-risk / limited / minimal); document the rationale
- [ ] ⚠ Determine your role under the Act — internal on-prem deployment typically makes the org a *deployer*; developing, substantially modifying, or marketing the system under your own name makes you a *provider*. Obligations differ materially
- [ ] Determine GDPR applicability for training data and inference inputs; run a DPIA where required
- [ ] On-prem satisfies EU data-residency by construction — still document processing locations; DPAs only needed where a processor (e.g., managed-service vendor) touches the data
- [ ] Define retention policy for inference logs before go-live

### Infrastructure

- [ ] Inference server (vLLM / SGLang / Triton) deployed with request logging enabled
- [ ] Logs routed to persistent, access-controlled, EU-resident storage
- [ ] Kubernetes RBAC scoped for the GPU deployment (overlaps §5, §7)
- [ ] Model version pinning and rollback capability in place
- [ ] Deployment locations and applicable certifications documented

### High-Risk Systems Only

- [ ] ⚠ Timeline: Annex III high-risk obligations apply from **2 Dec 2027** (embedded Annex I: 2 Aug 2028) per the Digital Omnibus — not Aug 2026 as the source assumed
- [ ] Technical documentation per Annex IV (provider obligation)
- [ ] Human oversight measures designed in per Art. 14; oversight endpoints in the inference API are one implementation
- [ ] Conformity assessment — internal control (Annex VI) or notified body (Annex VII) depending on system type
- [ ] ⚠ Registration in the EU database for high-risk AI systems (Art. 49) before placing on market / putting into service — not "the regulatory framework" generically
- [ ] Post-market monitoring plan and serious-incident reporting process (Arts. 72–73)

### Ongoing

- [ ] Monitor output quality; flag drift
- [ ] Log retention ≥ 6 months (Art. 26(6), deployer obligation); technical documentation retained 10 years (Art. 18, provider obligation) — both verified against the Act
- [ ] Re-assess classification and role on model weight or configuration changes (substantial modification can convert deployer → provider)
- [ ] Track European AI Office guidance and harmonised standards as they land
- [ ] Art. 50 transparency obligations (chatbot disclosure, machine-readable marking of synthetic content) apply from **2 Aug 2026** regardless of risk tier — already live; systems on the market before that date get until 2 Dec 2026 for Art. 50(2) marking

## 10. Model Weight Protection

Source: [1], [2]. Weights are the highest-value single asset in the deployment.

- [ ] Weight access limited to a small privileged set with two-person control (TPC) and two-person integrity (TPI)
- [ ] Weights encrypted at rest; keys held in an HSM with on-demand decryption
- [ ] Hashes and encrypted archival copies of each model release stored in a tamper-proof location; keys and encrypted data never co-located
- [ ] Weight storage aggressively isolated — protected vault, highly restricted zone / dedicated enclave, or HSM-backed
- [ ] Unneeded hardware communication capabilities disabled on weight-storage hosts; emanation/side-channel protections where the threat model warrants
- [ ] Third-party model files treated as untrusted code: prefer safetensors, scan and inspect imported models in a secure development zone before any production placement
- [ ] All model artifacts (weights, checkpoints, configs, prompts) in version control with signatures; consuming systems validate hashes at load
- [ ] Inference APIs return the minimum data required for the task, limiting model-inversion and extraction surface
- [ ] Automated rollback to last-known-good model version with human-in-the-loop failsafe

## 11. Data Pipeline Security

Source: [4]. Applies to training data, fine-tuning data, and RAG/knowledge-base corpora.

- [ ] Data provenance tracked in an append-only, cryptographically signed ledger; each revision signed by the person making the change
- [ ] Datasets digitally signed (quantum-resistant standards — FIPS 204/205); hashes verified at ingest and before every parameter-modifying run (training, fine-tuning, RLHF)
- [ ] Storage on FIPS 140-3-validated cryptographic modules; AES-256 at rest, TLS in transit
- [ ] Web-derived corpora treated as an active attack surface: split-view and frontrunning poisoning are demonstrated, low-cost (≈$60–$1,000) attacks — hash checks at download, consensus verification, trusted-snapshot sourcing
- [ ] Anomaly detection and sanitization passes run before every training or fine-tuning cycle
- [ ] AI output classified at the same level as its input data
- [ ] Drift-vs-poisoning triage documented: gradual degradation → data drift; abrupt, dimension-specific shifts → investigate as compromise

## 12. Agentic Extensions

Source: [3]. Applies when models drive agents with tool or system access.

- [ ] Each agent constructed as a distinct cryptographic principal (unique keys/certs); no static or shared secrets
- [ ] Inter-agent and agent-to-service calls authenticated via mTLS; identities bound to roles in a trusted registry, periodically reconciled against the live agent set — unregistered identities denied
- [ ] Least privilege with just-in-time, ephemeral credentials for privileged actions; runtime authorization at a centralized policy decision point per request, not at startup
- [ ] Tool use restricted to a verified allowlist of tools and versions; tool descriptions standardized (no persuasive language), tool responses validated before ingestion
- [ ] Agents prohibited from modifying their own privileges or delegating without explicit expiry timers and recorded grant chains
- [ ] Human approval gates on high-impact or irreversible actions (network egress, deletion of records, system resets); log-deletion requests quarantined pending human review
- [ ] Agent enclaves segmented with no write access to logs; isolation limits blast radius of a compromised agent
- [ ] Monitoring covers goal drift, privilege drift, and impersonation; agent self-reports cross-validated against independent system logs
- [ ] Progressive deployment: start low-risk/low-autonomy, expand scope only on continuous-evaluation evidence

## Acronym Guide

- **ACSC** — Australian Cyber Security Centre
- **AES** — Advanced Encryption Standard
- **AI** — Artificial Intelligence
- **AISC** — Artificial Intelligence Security Center (NSA)
- **API** — Application Programming Interface
- **ASD** — Australian Signals Directorate
- **BMC** — Baseboard Management Controller
- **CC** — Confidential Computing
- **CCCS** — Canadian Centre for Cyber Security
- **CIA** — Confidentiality, Integrity, Availability
- **CISA** — Cybersecurity and Infrastructure Security Agency
- **CPU** — Central Processing Unit
- **CSAF** — Common Security Advisory Framework
- **CUDA** — Compute Unified Device Architecture
- **DCGM** — Data Center GPU Manager (NVIDIA)
- **DPA** — Data Processing Agreement
- **DPIA** — Data Protection Impact Assessment
- **ECC** — Error-Correcting Code
- **FBI** — Federal Bureau of Investigation
- **FIPS** — Federal Information Processing Standards
- **GDPR** — General Data Protection Regulation
- **GPU** — Graphics Processing Unit
- **GSP** — GPU System Processor
- **HBM** — High Bandwidth Memory
- **HGX** — NVIDIA multi-GPU baseboard platform designation (not an expansion)
- **HSM** — Hardware Security Module
- **IPMI** — Intelligent Platform Management Interface
- **KV cache** — Key-Value cache
- **LAN** — Local Area Network
- **LLM** — Large Language Model
- **MIG** — Multi-Instance GPU
- **mTLS** — mutual Transport Layer Security
- **NCSC-NZ / NCSC-UK** — National Cyber Security Centre (New Zealand / United Kingdom)
- **NIST** — National Institute of Standards and Technology
- **NRAS** — NVIDIA Remote Attestation Service
- **NSA** — National Security Agency
- **NVLink** — NVIDIA high-speed GPU interconnect (brand name, not an acronym)
- **OCSP** — Online Certificate Status Protocol
- **OOM** — Out Of Memory
- **OS** — Operating System
- **P2P** — Peer-to-Peer
- **PoC** — Proof of Concept
- **RAG** — Retrieval-Augmented Generation
- **RBAC** — Role-Based Access Control
- **RLHF** — Reinforcement Learning from Human Feedback
- **RMA** — Return Merchandise Authorization
- **SBIOS** — System Basic Input/Output System
- **seccomp** — secure computing mode (Linux kernel facility)
- **SEV-SNP** — Secure Encrypted Virtualization – Secure Nested Paging (AMD)
- **SIEM** — Security Information and Event Management
- **SP** — Special Publication (NIST document series)
- **SPDM** — Security Protocol and Data Model
- **SR-IOV** — Single Root I/O Virtualization
- **SXM** — NVIDIA socketed GPU module form factor (not officially expanded)
- **TDX** — Trust Domain Extensions (Intel)
- **TEE** — Trusted Execution Environment
- **TLS** — Transport Layer Security
- **TPC** — Two-Person Control
- **TPI** — Two-Person Integrity
- **TPM** — Trusted Platform Module
- **UEBA** — User and Entity Behavior Analytics
- **UEFI** — Unified Extensible Firmware Interface
- **UUID** — Universally Unique Identifier
- **VBIOS** — Video Basic Input/Output System
- **vGPU** — virtual GPU
- **VLAN** — Virtual Local Area Network
- **VPN** — Virtual Private Network
- **VRAM** — Video Random-Access Memory
- **XID** — NVIDIA driver error event code (identifier, not an expansion)

## References

1. NCSC-UK, CISA, NSA, FBI, and international partners. *Guidelines for Secure AI System Development*. Nov 2023.
2. NSA AISC, CISA, FBI, ACSC, CCCS, NCSC-NZ, NCSC-UK. *Deploying AI Systems Securely: Best Practices for Deploying Secure and Resilient AI Systems*. U/OO/143395-24, Apr 2024.
3. ASD ACSC, CISA, NSA, Canadian Cyber Centre, NCSC-NZ, NCSC-UK. *Careful Adoption of Agentic AI Services*. Apr 2026.
4. NSA AISC, CISA, FBI, ASD ACSC, NCSC-NZ, NCSC-UK. *AI Data Security: Best Practices for Securing Data Used to Train & Operate AI Systems*. U/OO/157249-25, May 2025.
