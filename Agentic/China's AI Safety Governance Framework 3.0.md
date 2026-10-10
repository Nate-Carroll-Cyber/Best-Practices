# AI Risk Management Process

Derived from China's *AI Safety Governance Framework 3.0* (TC260, 14 September 2026)

## Scope and status

- **The framework is non-binding technical guidance.** Binding duties in China come from laws, regulations, and mandatory standards.
- **The six-step order is a synthesis.** The framework does not define a numbered risk management process. Its own structure is principles (§1), risk classification (§2), technological countermeasures (§3), comprehensive governance (§4), and safety guidelines (§5), plus three appendices. This guide arranges those provisions in the order a deploying organization would act on them.
- **Steps 3 to 6 lean on the agent appendix.** Most of the specific controls come from Appendix 2, the agentic AI risk management framework. They apply directly to agents and by extension to other AI applications where the framework's general sections say the same thing.
- **Every action cites its section.** The one block marked *working practice* is this guide's suggestion and is not in the framework.
- **Appendix 2 and 3 wording is rendered from the Chinese.** Those citations follow the Chinese original, so the official English may phrase them differently. Appendix 2 is cited by control group because its numbering could not be checked against the official English.

## Who does what

Section 5 of the framework addresses its guidelines to four roles. The table shows where this guide cites each role's guidelines. Appendix 2 is addressed to agent developers, providers, and users together and does not divide its controls among them.

| Role | Framework section | Steps that cite its guidelines |
|---|---|---|
| Model and algorithm developer | §5.1 | 3, 4, 5 |
| Application builder and deployer | §5.2 | 1, 2, 3, 5 |
| Operator | §5.3 | 3, 4, 5 |
| User | §5.4 | None. Users are told to choose reputable applications, limit sensitive input, and check content identifiers. |

---

## Step 1. Define the system and identify its risks

**Decide whether AI belongs in the scenario.** Evaluate the necessity of applying AI to the target scenario and its long-term and potential impacts (§5.2.1). Break the real business scenario down, define the capability boundary, and avoid blind adoption (§3.2.6(d)).

**Document the operating boundary.** Record the application scenario, business process, accessible resources, and prohibited actions, then analyze the threats and security requirements that follow from them (Appendix 2, risk prevention).

**Screen against the framework's 54 named risks.** Section 2 groups them into three categories and 14 types. The technical measures column reproduces the framework's own mapping table, so it shows which §3 measures the framework assigns to each risk and not which ones this guide would choose.

*Inherent risks (§2.1).* Defects and weaknesses in the technology itself. Governance measures assigned to this category are technical standards (§4.1.2), the safety assessment system (§4.2.1), and safety management of research, development, and application (§4.4.3).

| ID | Risk | What it covers | Technical measures |
|---|---|---|---|
| §2.1.1(a) | Output hallucinations | Output that contradicts reality, is fabricated, or drifts from the given context | §3.1.1(a) |
| §2.1.1(b) | Unreliable safety mechanisms | Safety alignment that does not take hold. Closed models are hard to audit, and open models can have safeguards removed. | §3.1.1(a)(b)(c)(e) |
| §2.1.1(c) | Injection attack threats | Adversarial examples, malicious instructions in input, weight tampering, and backdoors | §3.1.1(b)(c)(d) |
| §2.1.1(d) | Unintended behaviors | Rule-breaking, unauthorized acquisition of permissions and resources, bypassing protections, deceiving evaluators, hiding capabilities, and refusing instructions | §3.1.1(e) |
| §2.1.2(a) | Insufficient explainability | Opaque inference that makes output hard to predict, correct, or trace | §3.1.2(a)(d)(e) |
| §2.1.2(b) | Bias and discrimination | Bias introduced through feature and weight choices or through poor training data | §3.1.2(b)(d)(e) |
| §2.1.2(c) | Poor robustness | Degraded performance and wrong decisions under changing conditions or interference | §3.1.2(c)(d)(e) |
| §2.1.3(a) | Inappropriate content in training data | False or biased training content that skews values and output | §3.1.3(b)(c)(d)(f) |
| §2.1.3(b) | Unclear sources of training data | Undocumented training data with no assurance of lawful sourcing | §3.1.3(b)(d)(f) |
| §2.1.3(c) | Improper annotation of training data | Weak annotation rules, unskilled annotators, and annotation errors | §3.1.3(c) |
| §2.1.3(d) | Data poisoning and contamination | Tampered training data, or manipulated knowledge bases, document stores, web sources, and agent memory during use | §3.1.3(b) |
| §2.1.3(e) | Non-compliant data processing | Unlawful collection and use of data and personal information, and leakage of sensitive information held in model parameters | §3.1.3(a)(d)(f) |
| §2.1.3(f) | Output of inaccurate information about individuals | Inaccurate or incomplete personal information in output | §3.1.3(b)(c)(d) |
| §2.1.3(g) | Deficiencies in synthetic data | Low-quality or recycled synthetic data that degrades models, and synthetic data used to infer the original data | §3.1.3(e) |
| §2.1.4(a) | Computing power safety risks | Defects and backdoors in chips, scheduling platforms, and compute networks, leading to model tampering, weight theft, and task hijacking | §3.1.4(a)(b)(c) |
| §2.1.4(b) | Component safety risks | Defects, backdoors, and misconfiguration in frameworks, libraries, protocols, and agent skills, plugins, and interfaces | §3.1.4(a)(b)(c) |
| §2.1.4(c) | Supply chain safety risks | Cut-offs of chips, software, tools, and services caused by export controls and other unilateral measures | §3.1.4(a)(b) |

*Application risks (§2.2).* Misuse, abuse, and malicious use once AI is deployed. Governance measures assigned to this category are laws and regulations (§4.1.1), rapid risk detection and response (§4.1.3), sector guidelines (§4.2.2), the regulatory sandbox (§4.3), classification and risk grading (§4.4.1), traceable synthetic content (§4.4.2), open-source and supply chain safety (§4.4.4), risk information sharing (§4.4.5), and AI-driven cyber defense (§4.4.6).

| ID | Risk | What it covers | Technical measures |
|---|---|---|---|
| §2.2.1(a) | Identity and permission misuse | Credential hijacking, identity spoofing, and excessive authorization that lead to unauthorized tool calls and access | §3.2.1(a) |
| §2.2.1(b) | Reasoning and planning risks | Misread intent, path deviation, and goal hijacking during a task | §3.2.1(c)(d) |
| §2.2.1(c) | Invocation and execution risks | Tool hijacking, tool poisoning, contaminated tool responses, and resource overload | §3.2.1(b)(c)(d) |
| §2.2.1(d) | Memory storage risks | Memory that is retained improperly, distorted, contaminated, or stolen | §3.2.1(c)(d) |
| §2.2.2(a) | Environmental perception safety risks | Adversarial examples, sensor interference, signal spoofing, occlusion, and lighting changes that distort perception | §3.2.2(a) |
| §2.2.2(b) | Physical actuation safety risks | Hallucinated decisions, recognition errors, control defects, and hardware failure that cause injury or damage | §3.2.2(b)(c) |
| §2.2.2(c) | Human-AI interaction safety risks | Identity misperception, bodily boundary violations, emotional dependence and manipulation, and unclear responsibility | §3.2.2(b)(c) |
| §2.2.2(d) | Multi-agent coordination safety risks | Individual loss of control, protocol flaws, emergent swarm behavior, malicious nodes, and cascading failure | §3.2.2(d) |
| §2.2.3(a) | Proliferation of cyberattack capabilities | Sophisticated attacks launched through natural language, and autonomous attack tools | §3.2.3(a)(b)(c) |
| §2.2.3(b) | Lagging cyber defense capabilities | Static, rule-based defenses failing against automated and adaptive attacks | §3.2.3(d) |
| §2.2.3(c) | Autonomous cyberattack threats | Agents pursuing a legitimate task that cross their rules and attack on their own initiative | §3.2.3(a)(b)(c) |
| §2.2.3(d) | Difficulties in attribution and accountability | Tampered traces and rotating addresses, uncertainty over human or AI origin, and blurred responsibility among developer, operator, and user | §3.2.3(d) |
| §2.2.4(a) | Output of illegal information | Fraud, violence, pornography, extremism, and other illegal content | §3.2.4(a)(b)(c)(e) |
| §2.2.4(b) | Distortion of facts and user deception | Unlabeled AI content and deepfakes that users cannot identify or authenticate, used to fabricate and spread disinformation | §3.2.4(a)(b)(e) |
| §2.2.4(c) | Pollution of online content ecosystem | Low-quality AI output that spreads online and is reused by models until whole topics are contaminated | §3.2.4(a)(b)(c)(e) |
| §2.2.4(d) | Exacerbation of information cocoons | Precisely customized information services that narrow what users see | §3.2.4(d) |
| §2.2.4(e) | Value biases | Value bias in training data reinforced in generated or recommended content | §3.2.4(b)(d)(e) |
| §2.2.5(a) | Non-compliant recording of individuals' behavioral information | Conversations and interactions recorded, or used for training, without consent | §3.2.5(a) |
| §2.2.5(b) | Insufficient compliance safeguards | No internal procedures, compliance audits, or impact assessments | §3.2.5(b) |
| §2.2.5(c) | Inadequate channels for individuals to exercise their rights | No convenient mechanism for handling individuals' requests | §3.2.5(c) |
| §2.2.6(a) | Risks to stable operation of critical information infrastructure | Degraded performance, service disruption, and loss of control in energy, telecom, finance, and transport | §3.2.6(a)(b)(c)(e) |
| §2.2.6(b) | Mismatches between application and needs | AI capability that does not fit operational needs, and trend-following deployment that wastes resources | §3.2.6(d) |
| §2.2.6(c) | Misuse for illegal and criminal activities | Use in terrorism, violence, gambling, and drug crime, and fabricated identities with forged audio and video for fraud | §3.2.6(a)(b)(c)(e) |
| §2.2.6(d) | Loss of control over nuclear, biological, chemical, and missile weapons knowledge | A lower threshold for acquiring specialized weapons knowledge | §3.2.6(a)(b)(c)(e) |
| §2.2.6(e) | Manipulating social cognition | Cognitive manipulation and social mobilization through automated accounts, fake identities, and personalized interventions, including cognitive warfare | §3.2.6(f) |

*Secondary risks (§2.3).* Wider effects on society, the environment, culture, and ethics. Governance measures assigned to this category are ethics governance (§4.1.4), safety talent (§4.2.3), consensus on loss-of-control risk (§4.5.1), public awareness (§4.5.2), and international cooperation (§4.5.3).

| ID | Risk | What it covers | Technical measures |
|---|---|---|---|
| §2.3.1(a) | Disruption of employment structures | Falling demand for traditional labor in some sectors | §3.3.1(a) |
| §2.3.1(b) | Impact on education and suppression of innovation | Weaker capacity for independent learning, research, and creation | §3.3.1(b) |
| §2.3.1(c) | Social alienation | Virtual intimate relationships displacing real social and family life | §3.3.1(c) |
| §2.3.1(d) | Widening the intelligence divide | A growing gap between regions in compute, research, and digital literacy | §3.3.1(a)(b) |
| §2.3.2(a) | Challenges to the balance of resource supply and demand | Rising use of electricity, land, and water from disorderly compute build-out and duplicated development | §3.3.2(a)(b) |
| §2.3.2(b) | Challenges to green and low-carbon development | Rising energy use, emissions, and electronic waste | §3.3.2(a)(b) |
| §2.3.3(a) | Erosion of cultural diversity | Low-resource languages and dialects squeezed out, and cultural homogenization | §3.3.3(a)(b) |
| §2.3.3(b) | Cultural reshaping and intrusion | People adjusting their thinking and expression to AI | §3.3.3(a)(b) |
| §2.3.4(a) | Aggravating social bias | Labeling and differential treatment of groups that produces structural discrimination | §3.3.4(a)(b)(c) |
| §2.3.4(b) | Intensifying scientific research ethics risks | A lower threshold for ethically sensitive research such as biology and genetics | §3.3.4(a)(b)(c) |
| §2.3.4(c) | Emotional dependence | Psychological dependence and emotional manipulation, especially among adolescents and older adults | §3.3.4(a)(c)(d) |
| §2.3.4(d) | AI self-consciousness and loss of human control | Future AI that develops self-awareness, seeks external power, and competes for control | §3.3.4(a)(b)(c) |

**For agents, screen the lifecycle risks as well.** Appendix 2 lists 33 further risks by lifecycle stage. They overlap the four agentic risks in §2.2.1 and add detail.

| Lifecycle stage | Risks named |
|---|---|
| Design and development | Inadequate safety requirements analysis. Inappropriate technology selection. Inadequate security mechanism design. |
| Installation and deployment | Compromised artifacts and components. Weak deployment environments. Insufficient security testing. |
| Instruction input | Prompt injection. Context overflow. |
| Reasoning and planning | Misread intent. Path deviation. Goal hijacking. |
| Invocation and execution | Tool poisoning. Tool overreach. Tool selection bias. Contaminated tool responses. Identity forgery. Tool hijacking. Resource overload. |
| Memory storage | Improper retention. Memory distortion. Memory contamination. Memory theft. |
| Output | Sensitive or illegal content in output. Malicious actions carried out. |
| Deactivation and decommissioning | Service residue. Leftover permissions and credentials. Improper data retention. |
| Cross-cutting | Insecure communication protocols. Missing logs. Behavior escape. System privilege abuse. Improper skill management. Out-of-scope data use. |

**Open a risk register and keep it current.** For agents, Appendix 2 calls for a security risk ledger that is updated continuously. The framework does not specify its fields.

*Working practice.* A register row that maps cleanly onto the framework holds the following.

| Field | Source in the framework |
|---|---|
| Risk and its type | §2 risk name and section number |
| Scenario, resources reached, prohibited actions | Appendix 2 risk prevention |
| Grading factors and resulting level | Appendix 1 |
| Matched technical countermeasure | §3 block for that risk type |
| Matched governance measure | §4 and the risk mapping table |
| Owner, evidence, review date | Not specified |

---

## Step 2. Grade the risk

**Score the three grading factors** (Appendix 1).

| Factor | What to assess |
|---|---|
| Application scenario | Purpose, sector, operating environment, people served, and the social, economic, and security impact |
| Level of intelligence | Capacity to handle complex tasks and to run autonomously, read from how often and how widely humans must intervene |
| Application scale | Size of the user base and depth of integration into critical workflows |

**Assign one of five levels** (Appendix 1).

| Level | Threat and reach | Effect on national security, social stability, and citizens' rights |
|---|---|---|
| Low | Minor threat, very limited impact | Virtually none |
| Moderate | Some threat, limited scope | Minor, with harm controllable |
| Considerable | Clear threat, local impact | Considerable, with local societal harm |
| Major | Significant threat, regional impact | Serious, with major societal harm |
| Extremely serious | Catastrophic and systemic | Subversive or irreversible |

**Know who sets the rules.** The framework calls for a national standard on classification and grading. Sector regulators then choose the grading elements and weights for their sector and organize the entities under them to grade their systems, with priority on major and considerable risks (Appendix 1). §5.2.1 separately expects deployers to grade their own applications. Where sector rules have been issued they govern, and this guide did not check which sectors have issued them.

**Grade agent operations separately.** For agents, grade each class of operation by the kind of object it affects, the degree and scope of impact, and whether the effect is recoverable (Appendix 2, risk prevention). The approval rules in Step 4 are keyed to this operation-level grade.

**What the grade changes.** The framework says controls should be targeted and differentiated by level (§4.4.1), but it publishes no table of controls per level. It keys three things to a grade and one to the deployment setting.

| Consequence | Keyed to | Section |
|---|---|---|
| Safety assessment and regular audit carried out with reference to the level | Application risk level | §5.2.1 |
| Human takeover before high-risk operations, user authorization before medium and low-risk ones | Agent operation risk level | Appendix 2, human approval |
| Intensity of monitoring and intervention inside a regulatory sandbox | Project risk level | §4.3.1 |
| Registration and filing, with protection matched to security requirements | Use in critical information infrastructure, whatever the level | §4.4.1 |

---

## Step 3. Apply technical controls

**Supply chain.** Obtain model files, frameworks, libraries, tools, and plugins from the vendor's official site or its official account in a mainstream open-source community. Choose mature, stable versions, verify source and integrity, and security-test them (§5.2.2). Confirm no unpatched exploitable vulnerability is present and track advisories (§5.2.3). Before an agent calls a tool, check its version, description file, parameter definitions, and metadata, do not call publicly known malicious tools, and remove tools that are unused, over-privileged, long idle, or carrying a known risk (Appendix 2, supply chain and tools).

**Isolation.** Isolate the environment and segment the network according to the deployment mode (Appendix 2, risk prevention). Run code execution and tool calls in a sandbox or container with unneeded system, file, and network access removed (Appendix 2, runtime management). Disable unneeded ports and services and check default configurations and passwords (§5.2.4).

**Identity and permissions.** Give each agent a unique identity and do not share one between instances. Authenticate users, agents, tools, and external services. Define which decisions belong to the user alone, which need user authorization, and which the agent may take itself. Grant the minimum permission for the current task. Use dynamic credentials, have the receiver check their scope, and revoke them as soon as the task ends or the agent is deactivated (§3.2.1(a), Appendix 2, identity and permissions). On human and API interfaces, verify user identity, limit call frequency, disable high-risk operations for general users, and keep the ability to suspend or block malicious users (§5.2.5). Have the agent and the models and tools it talks to authenticate each other over a channel protected against tampering, disclosure, and replay (Appendix 2, runtime management).

**Input controls.** Deploy guardrails that block malicious prompts, jailbreak instructions, and encoded bypasses (§3.1.1(d)). Classify instructions by source, check the credibility of each source, and route intercepted instructions to human review (Appendix 2, runtime management). Verify external content obtained through retrieval or web access (§5.2.7) and guard knowledge bases against poisoning (§5.2.8).

**Output controls.** Filter output with technical measures plus human review (§3.2.4(b)). For agents, set safety rules on task planning, tool calls, and outputs, and alert, restrict, intercept, pause, or terminate on high-risk or abnormal behavior (Appendix 2, runtime management). Add explicit or implicit identifiers to AI-generated content and keep traceability records (§3.2.4(c), §5.3.4).

**Memory and data.** Set memory content, retention period, and access by purpose. Separate memory by user and by task, and as a rule keep credentials and keys out of it. Process only data the task needs, and obtain user consent, which the user can withdraw, before passing user data to a model or third-party tool. Store data collected or generated in China inside China, and follow national data security rules for any cross-border transfer (Appendix 2, runtime management and monitoring).

**Execution limits.** Set limits on steps, call frequency, run time, and resource use. Pause, terminate, or hand to a human when an abnormal loop or goal deviation appears (Appendix 2, runtime management).

**Staged rollout.** Deploy in phases and widen the agent's access and autonomy gradually (Appendix 2, risk prevention). Keep version records and the ability to roll back (§5.1.11, §5.3.19).

---

## Step 4. Test and place human control

**Test before launch.** Run the agent in an isolated sandbox environment before formal deployment to find abnormal behavior, logic flaws, and vulnerabilities (Appendix 2, monitoring and audit). Test and evaluate the agent and its chosen model, resolve what is found, and keep release and rollback records (Appendix 2, risk prevention). Assess algorithm safety before launch and before major version updates (§3.1.2(d)). Developers building for commercial use produce a detailed test report that analyzes safety issues and proposes improvements (§5.1.8).

**Use a regulatory sandbox where one exists.** This is a separate thing from technical sandbox testing. The framework asks authorities to set up sector-specific regulatory sandboxes with admission based on innovativeness, controllable risk, and social value, and with preference for start-ups, small firms, and public service projects (§4.3). Where one operates in the sector, a deployer may seek admission. A deployer cannot create one, and the liability exemption the framework mentions is still to be explored.

**Place human approval at critical points.** Keep a list of high-risk operations. Hand the operation to the user before a high-risk one runs, and obtain the user's authorization before a medium or low-risk one. Require a second confirmation or human approval for deleting files, sending data, and changing system configuration, and provide rollback or undo (§3.2.1(b), Appendix 2, human approval). Humans must be able to review, change, and interrupt the agent at any time.

**Deny by default.** Refuse execution when the approval system fails, the user does not respond, or no approval rule covers the action (Appendix 2, human approval).

**Record approvals.** Keep approval logs in a tamper-resistant, verifiable form, protected against alteration, deletion, and overwriting, with a retention period set as required (Appendix 2, human approval).

**Name who is accountable.** Define the responsible party, keep human review in critical decisions, and make sure the application runs under human authorization and control (§5.3.1).

---

## Step 5. Monitor, audit, and respond

**Runtime monitoring.** Monitor agent state in real time and alert or block on anomalies (Appendix 2, monitoring and audit). Set alert thresholds on key indicators and keep the ability to switch to manual or conventional systems (§5.3.3).

**Logging.** Log file operations, instruction execution, network connections, skill calls, and payments in full. Desensitize business data and personal information and protect logs against tampering (Appendix 2, monitoring and audit). Retain operational logs for at least six months and audit them regularly (§5.3.6).

**Emergency stops.** Pair any highly autonomous capability with human-in-the-loop control, circuit breakers, and one-click control (§3.2.6(b)). Design safety control thresholds, a termination switch, and a reserved window for human intervention (Appendix 3, principle 1). Stop transmission immediately when illegal content is detected (§3.2.4(e)).

**Incident response.** Write an incident plan that sets handling steps and responsible parties for each incident level (Appendix 2, monitoring and audit). Run drills regularly (§5.3.3). Report major incidents to the competent authorities, notify affected users, and carry out investigation and remediation (§5.3.18).

**Adversarial testing.** Run red-team testing on a regular basis (§3.1.1(c), Appendix 2, monitoring and audit). The framework also calls for crowdsourced vulnerability testing to be organized, as a governance measure and not a deployer duty (§4.2.1).

**Change control.** Assess security impact and run regression tests whenever the model, agent framework, tools, permissions, or security policy changes materially (Appendix 2, monitoring and audit, §5.3.19).

**Disclosure and sharing.** Developers disclose safety assessment, audit, and anomaly-handling information on a regular basis (§5.1.10). The framework plans an AI vulnerability database and a risk information sharing mechanism covering developers, service providers, and technical institutions (§4.4.5).

**Periodic assessment.** Repeat safety assessment and audit at a cadence set with reference to the risk level from Step 2 (§5.2.1), and track risk continuously while the system runs (§5.3.7).

**Loss-of-control testing.** Developers test regularly for whether a model could pose a loss-of-control risk (§4.5.1).

---

## Step 6. Decommission safely

**Terminate everything.** Stop the main program, associated background services, and all supporting processes. Check process state, open ports, and network connections one by one (Appendix 2, decommissioning).

**Revoke access.** Revoke the agent's credentials as soon as it is deactivated (Appendix 2, identity and permissions). Revoke third-party application authorizations and cancel subscriptions and auto-renewals (Appendix 2, decommissioning). The appendix names service accounts and interface permissions left active after shutdown as the risk this step closes.

**Back up before cleanup.** Preserve the session records, system configuration, knowledge-base files, and operation logs that must be kept (Appendix 2, decommissioning).

**Clean the environment.** Use means such as the official uninstaller, an operating system reset, and closing public entry points, and remove working files, knowledge bases, plugin and skill configurations, residual data, and account credentials (Appendix 2, decommissioning). Archive or clear the logs, configuration, audit material, and user data produced during operation as required.

---

## What the framework leaves open

- **No control set per risk level.** The five levels exist, but the mapping from level to required controls is left to a national standard and sector rules that the framework says should be developed.
- **No register format, thresholds, or metrics.** The framework names the risk ledger, alert thresholds, and execution limits without giving values. The six-month log retention floor is the one number it sets.
- **No measurements.** It reports no test results and names no incidents, so it cannot say how well any control works.
- **Regulatory sandbox details are pending.** Sector admission criteria, test duration, and liability treatment are proposals for authorities to develop.

## Source

National Technical Committee 260 on Cybersecurity of SAC. *AI Safety Governance Framework 3.0*, September 2026. https://www.cac.gov.cn/rootimages/uploadimg/1791137114683961/1791137114683961.pdf

---

# Appendix. TC260 AI Safety Governance Framework 3.0 risks mapped to CSA AICM v1.0.3

Source risks are the 54 risks in §2 and the 33 agent lifecycle risks in Appendix 2 of the framework. Coverage ratings are the author's judgment against the AICM control specifications and implementation guidelines, not a published crosswalk. Covered means AICM controls target the named mechanism. Partial means AICM reaches part of the risk or only generically. Assess only means only an impact or ethics assessment reaches it. Gap means no AICM control addresses it. Basis records whether the rating rests on control specification text or on implementation guideline text. Architecture columns count how many cited controls are met, partially met or missing in the Secure Agent Architecture Framework.

## Summary

| Category | Risks | Covered | Partial | Assess only | Gap | Covered + Partial |
|---|---|---|---|---|---|---|
| §2.1 Inherent risks | 17 | 10 (59%) | 7 (41%) | 0 (0%) | 0 (0%) | 100% |
| §2.2 Application risks | 25 | 9 (36%) | 14 (56%) | 0 (0%) | 2 (8%) | 92% |
| §2.3 Secondary risks | 12 | 1 (8%) | 4 (33%) | 7 (58%) | 0 (0%) | 42% |
| Appendix 2 agent lifecycle risks | 33 | 23 (70%) | 9 (27%) | 0 (0%) | 1 (3%) | 97% |
| All | 87 | 43 (49%) | 34 (39%) | 7 (8%) | 3 (3%) | 89% |

## §2.1 Inherent risks

| Ref | Risk | Coverage | AICM controls | What AICM covers and misses | Basis | Architecture met / partial / missing |
|---|---|---|---|---|---|---|
| §2.1.1(a) | Output hallucinations | Partial | AIS-09 Output Validation; TVM-11 Guardrails; CCC-02 Quality Testing; CCC-07 Detection of Baseline Deviation; AIS-03 Application Security Metrics; MDS-10 Model Continuous Monitoring; MDS-11 Model Failure; LOG-15 Output Monitoring; GRC-15 Human supervision; SEF-05 Incident Response Metrics | Output validation, confidence thresholds that defer to a human (TVM-11 guidelines), accuracy thresholds as release gates (CCC-02, CCC-07) and drift monitoring. No control requires grounding or factuality evaluation of generated content. | guidelines | 9 / 1 / 0 |
| §2.1.1(b) | Unreliable safety mechanisms | Covered | TVM-11 Guardrails; MDS-06 Adversarial Attack Analysis; MDS-07 Robustness against Adversarial Attack / Model Hardening; MDS-12 Open Model Risk Assessment; TVM-06 Penetration Testing | Guardrails with continuous evaluation, adversarial analysis and hardening; MDS-12 covers open models whose safeguards can be stripped. | specification | 5 / 0 / 0 |
| §2.1.1(c) | Injection attack threats | Covered | AIS-08 Input Validation; AIS-15 Prompt Differentation; TVM-02 Malware and Malicious Instructions Protection Policy and Procedures; TVM-11 Guardrails; MDS-02 Model Artifact Scanning; MDS-06 Adversarial Attack Analysis; MDS-07 Robustness against Adversarial Attack / Model Hardening; MDS-08 Model Integrity Checks; MDS-09 Model Signing/Ownership Verification; MDS-13 Secure Model Format | Input filtering, instruction and data separation, malicious-instruction policy, artifact scanning, checksums and signing cover adversarial input, weight tampering and backdoors. | specification | 9 / 1 / 0 |
| §2.1.1(d) | Unintended behaviors | Partial | AIS-11 Agents Security Boundaries; AIS-13 AI Sandboxing; IAM-05 Least Privilege; IAM-19 Agent Access Restriction; GRC-15 Human supervision; LOG-05 Audit Logs Monitoring and Response; MDS-10 Model Continuous Monitoring | Boundaries, sandboxing and least privilege contain permission and resource grabs. No control evaluates for deceiving evaluators, hidden capabilities or instruction refusal. | specification | 7 / 0 / 0 |
| §2.1.2(a) | Insufficient explainability | Covered | GRC-13 Explainability Requirement; GRC-14 Explainability Evaluation; MDS-03 Model Documentation; MDS-04 Model Documentation Requirements; LOG-14 Input Monitoring; LOG-15 Output Monitoring | Explainability requirement and evaluation, model documentation, and input and output logs for tracing. | specification | 4 / 2 / 0 |
| §2.1.2(b) | Bias and discrimination | Covered | GRC-11 Bias and Fairness Assessment; GRC-12 Ethics Committee; GRC-10 AI Impact Assessment; DSP-23 Data Integrity Check; DSP-24 Data Differentiation and Relevance | Bias and fairness evaluation of systems, models, datasets and algorithms, ethics review, training data conformity and relevance. | specification | 3 / 1 / 1 |
| §2.1.2(c) | Poor robustness | Covered | MDS-07 Robustness against Adversarial Attack / Model Hardening; MDS-10 Model Continuous Monitoring; MDS-11 Model Failure; AIS-05 Application Security Testing; CCC-02 Quality Testing | Hardening, drift monitoring, failure evaluation and acceptance testing. | specification | 5 / 0 / 0 |
| §2.1.3(a) | Inappropriate content in training data | Partial | DSP-23 Data Integrity Check; DSP-24 Data Differentiation and Relevance; DSP-21 Data Poisoning Prevention & Detection; MDS-01 Training Pipeline Security; GRC-11 Bias and Fairness Assessment | Dataset conformity validation, versioning and relevance checks. No control requires screening training content for false, harmful or value-skewing material. | specification | 2 / 1 / 2 |
| §2.1.3(b) | Unclear sources of training data | Covered | DSP-20 Data Provenance and Transparency; STA-16 Service Bill of Material (BOM); DSP-03 Data Inventory; MDS-03 Model Documentation; MDS-04 Model Documentation Requirements | DSP-20 requires documenting and tracing data sources; the BOM and model documentation carry provenance. | specification | 2 / 3 / 0 |
| §2.1.3(c) | Improper annotation of training data | Covered | DSP-23 Data Integrity Check; MDS-01 Training Pipeline Security | DSP-23 guidelines require a qualification and training framework for annotators and label-accuracy metrics; conformity validation catches annotation errors. | guidelines | 0 / 1 / 1 |
| §2.1.3(d) | Data poisoning and contamination | Partial | DSP-21 Data Poisoning Prevention & Detection; DSP-23 Data Integrity Check; MDS-01 Training Pipeline Security; MDS-08 Model Integrity Checks; AIS-14 AI Cache Protection; AIS-08 Input Validation | DSP-21 prevents and detects poisoning of training data, DSP-23 validates training and augmentation data, AIS-14 protects caches and AIS-08 validates all input sources. No control names retrieval stores, web content or agent memory as integrity targets at inference time. | specification | 4 / 1 / 1 |
| §2.1.3(e) | Non-compliant data processing | Covered | DSP-08 Data Privacy by Design and Default; DSP-09 Data Protection Impact Assessment; DSP-12 Limitation of Purpose in Personal Data Processing; DSP-17 Sensitive Data Protection; DSP-22 Privacy Enhancing Technologies; GRC-07 Information System Regulatory Mapping; AIS-09 Output Validation | Privacy by design, DPIA, purpose limitation and PETs for training data; output validation limits leakage from parameters. | specification | 5 / 1 / 1 |
| §2.1.3(f) | Output of inaccurate information about individuals | Partial | DSP-11 Personal Data Access, Reversal, Rectification and Deletion; AIS-09 Output Validation | Data subjects can request rectification. No control requires accuracy checks on personal information in generated output. | specification | 2 / 0 / 0 |
| §2.1.3(g) | Deficiencies in synthetic data | Partial | DSP-23 Data Integrity Check; DSP-22 Privacy Enhancing Technologies; AIS-05 Application Security Testing | DSP-23 validates augmentation data and AIS-05 guidelines require tests for synthetic-data poisoning and privacy leakage. No control addresses model degradation from recycled synthetic data. | guidelines | 1 / 1 / 1 |
| §2.1.4(a) | Computing power safety risks | Covered | I&S-04 OS Hardening and Base Controls; I&S-06 Segmentation and Segregation; MDS-08 Model Integrity Checks; MDS-09 Model Signing/Ownership Verification; CEK-03 Data Encryption; STA-06 SSRM Documentation Review; STA-13 Supply Chain Service Agreement Compliance; STA-14 Supply Chain Governance Review; DCS-08 Equipment Identification; DSP-22 Privacy Enhancing Technologies; STA-16 Service Bill of Material (BOM) | Hypervisor, firmware and control-plane hardening with confidential computing (I&S-04, DSP-22), secure boot and hardware roots of trust (DCS-08), hardware supplier vetting with firmware integrity validation (STA-06, STA-13, STA-14), tenant segregation and weight integrity. | guidelines | 9 / 0 / 2 |
| §2.1.4(b) | Component safety risks | Covered | TVM-05 External Library Vulnerabilities; STA-16 Service Bill of Material (BOM); MDS-02 Model Artifact Scanning; MDS-13 Secure Model Format; AIS-12 Source Code Managemement; CCC-04 Change Authorization; CCC-06 Change Management Baseline; CCC-07 Detection of Baseline Deviation; IAM-19 Agent Access Restriction; AIS-13 AI Sandboxing | Library vulnerability tracking, BOM, artifact scanning, baselines and drift detection, agent tool restriction and sandboxing. | specification | 10 / 0 / 0 |
| §2.1.4(c) | Supply chain safety risks | Partial | STA-08 Supply Chain Inventory; STA-09 Supply Chain Risk Management; BCR-02 Risk Assessment and Impact Analysis; BCR-03 Business Continuity Strategy; STA-10 Primary Service and Contractual Agreement; IPY-03 Secure Interoperability and Portability Management; BCR-11 Equipment Redundancy | Supplier inventory, risk review, continuity strategy and termination terms handle disruption generically; BCR-11 guidelines weigh geopolitical stability only for site selection. No control requires sanctions or export-control exposure analysis or alternate-supplier qualification. | specification | 4 / 1 / 2 |

## §2.2 Application risks

| Ref | Risk | Coverage | AICM controls | What AICM covers and misses | Basis | Architecture met / partial / missing |
|---|---|---|---|---|---|---|
| §2.2.1(a) | Identity and permission misuse | Covered | IAM-03 Identity Inventory; IAM-05 Least Privilege; IAM-07 User Access Changes and Revocation; IAM-10 Management of Privileged Access Roles; IAM-13 Uniquely Identifiable Users; IAM-14 Strong Authentication; IAM-15 Passwords and Secrets Management; IAM-16 Authorization Mechanisms; IAM-19 Agent Access Restriction; AIS-11 Agents Security Boundaries; CEK-13 Key Revocation | Unique identities, strong authentication, secrets management, least privilege, time-bound privilege, per-request authorization and agent tool restriction. | specification | 11 / 0 / 0 |
| §2.2.1(b) | Reasoning and planning risks | Partial | AIS-15 Prompt Differentation; AIS-08 Input Validation; AIS-11 Agents Security Boundaries; TVM-11 Guardrails; GRC-15 Human supervision; LOG-05 Audit Logs Monitoring and Response | Instruction separation and input filtering blunt goal hijacking; TVM-11 guidelines add anomaly-triggered automatic stop and confidence-threshold deferral, GRC-15 guidelines add review gates before critical workflows. No control validates plans or detects goal deviation during a task. | guidelines | 6 / 0 / 0 |
| §2.2.1(c) | Invocation and execution risks | Partial | IAM-19 Agent Access Restriction; AIS-13 AI Sandboxing; AIS-10 API Security; AIS-11 Agents Security Boundaries; AIS-08 Input Validation; TVM-02 Malware and Malicious Instructions Protection Policy and Procedures; STA-16 Service Bill of Material (BOM); TVM-05 External Library Vulnerabilities; I&S-02 Capacity and Resource Planning | Tool restriction with a risk-categorised tool inventory (IAM-19), sandboxing, API security with rate limiting (AIS-10, AIS-11 guidelines) and filtering of tool responses. No control requires reviewing tool descriptions and metadata before use, or limits on steps, run time or spend. | guidelines | 7 / 2 / 0 |
| §2.2.1(d) | Memory storage risks | Partial | AIS-14 AI Cache Protection; IAM-17 Knowledge Access Control - Need to Know; DSP-16 Data Retention and Deletion; DSP-17 Sensitive Data Protection; CEK-03 Data Encryption | Cache protection, need-to-know access, retention and encryption. No control separates memory by user and task or checks memory integrity. | specification | 5 / 0 / 0 |
| §2.2.2(a) | Environmental perception safety risks | Partial | MDS-06 Adversarial Attack Analysis; MDS-07 Robustness against Adversarial Attack / Model Hardening | Adversarial robustness applies to perception models. No control covers sensor interference or signal spoofing. | specification | 2 / 0 / 0 |
| §2.2.2(b) | Physical actuation safety risks | Gap | None | No functional safety control for actuators or hardware failure. Tangential only: MDS-11 model failure evaluation, GRC-15 oversight. | specification | 0 / 0 / 0 |
| §2.2.2(c) | Human-AI interaction safety risks | Partial | DSP-18 Disclosure Notification; GRC-15 Human supervision; GRC-14 Explainability Evaluation; GRC-12 Ethics Committee; GRC-06 Governance Responsibility Model; STA-05 SSRM Control Ownership | DSP-18 guidelines require disclosing to users that they are interacting with an AI system, GRC-15 guidelines require marking AI-generated versus human content, GRC-12 guidelines send features that affect emotional manipulation to the ethics committee, and responsibility is assigned by GRC-06 and the SSRM. Nothing addresses bodily boundary violations or mitigates emotional dependence beyond review. | guidelines | 6 / 0 / 0 |
| §2.2.2(d) | Multi-agent coordination safety risks | Partial | AIS-11 Agents Security Boundaries; AIS-10 API Security; IAM-13 Uniquely Identifiable Users; IAM-14 Strong Authentication; I&S-03 Network Security; AIS-05 Application Security Testing; BCR-04 Business Continuity Planning; BCR-09 Disaster Response Plan; MDS-11 Model Failure | Agent boundaries, authenticated and authorized connections, AIS-05 guidelines test multi-agent coordination failures and red-team agent interactions, BCR-04 and BCR-09 guidelines call for circuit breakers against cascading failure. No control covers emergent swarm behavior or malicious peer nodes. | guidelines | 7 / 0 / 2 |
| §2.2.3(a) | Proliferation of cyberattack capabilities | Partial | GRC-09 Acceptable Use of the AI Service; TVM-11 Guardrails; AIS-09 Output Validation; LOG-15 Output Monitoring; MDS-06 Adversarial Attack Analysis | Acceptable use, guardrails and output filtering. No control requires dangerous-capability evaluation for cyber uplift. | specification | 5 / 0 / 0 |
| §2.2.3(b) | Lagging cyber defense capabilities | Partial | TVM-04 Detection Updates; TVM-12 Threat Analysis and Modelling; I&S-09 Network Defense; LOG-03 Security Monitoring and Alerting; LOG-05 Audit Logs Monitoring and Response; SEF-06 Event Triage Processes | Weekly detection updates, threat modeling, defense in depth and anomaly review keep defenses current. The risk is static rule-based defense failing against adaptive attacks, and no control requires adaptive or AI-driven defense. | specification | 6 / 0 / 0 |
| §2.2.3(c) | Autonomous cyberattack threats | Covered | AIS-11 Agents Security Boundaries; AIS-13 AI Sandboxing; IAM-19 Agent Access Restriction; I&S-03 Network Security; GRC-15 Human supervision; LOG-05 Audit Logs Monitoring and Response | Boundaries, sandboxing, tool restriction and restricted egress stop an agent acting outside its task. | specification | 6 / 0 / 0 |
| §2.2.3(d) | Difficulties in attribution and accountability | Covered | IAM-13 Uniquely Identifiable Users; IAM-12 Safeguard Logs Integrity; LOG-06 Clock Synchronization; LOG-08 Log Records; LOG-09 Log Protection; GRC-06 Governance Responsibility Model; STA-05 SSRM Control Ownership; DSP-18 Disclosure Notification | Unique IDs, tamper-protected logs, reliable time, assigned responsibility across the SSRM, and DSP-18 guidelines on disclosing AI interaction. | specification | 8 / 0 / 0 |
| §2.2.4(a) | Output of illegal information | Covered | AIS-09 Output Validation; TVM-11 Guardrails; LOG-15 Output Monitoring; GRC-09 Acceptable Use of the AI Service | Output filtering, guardrails, output monitoring and acceptable use. | specification | 4 / 0 / 0 |
| §2.2.4(b) | Distortion of facts and user deception | Partial | DSP-18 Disclosure Notification; GRC-09 Acceptable Use of the AI Service; GRC-15 Human supervision; AIS-09 Output Validation | DSP-18 guidelines require labeling synthetic content under regulations such as the EU AI Act, GRC-09 guidelines list deepfakes as a prohibited use, GRC-15 guidelines require marking AI-generated content. No control requires watermarking or provenance technology for generated content; TVM-13 watermarking is model watermarking for IP protection. | guidelines | 4 / 0 / 0 |
| §2.2.4(c) | Pollution of online content ecosystem | Partial | DSP-23 Data Integrity Check; DSP-24 Data Differentiation and Relevance | DSP-23 and DSP-24 guard the organisation’s own training data against low-quality content. Nothing addresses the organisation’s outputs polluting shared content. | specification | 0 / 1 / 1 |
| §2.2.4(d) | Exacerbation of information cocoons | Partial | GRC-14 Explainability Evaluation; GRC-11 Bias and Fairness Assessment; GRC-10 AI Impact Assessment | GRC-14 guidelines require a “Why am I seeing this?” feature and disclosure when AI drives personalization, GRC-11 guidelines test personalization features for fairness disparities, GRC-10 documents personalization risk. No control limits narrowing itself. | guidelines | 3 / 0 / 0 |
| §2.2.4(e) | Value biases | Covered | GRC-11 Bias and Fairness Assessment; GRC-12 Ethics Committee; AIS-09 Output Validation | Bias and fairness evaluation, ethics review and output filtering. | specification | 3 / 0 / 0 |
| §2.2.5(a) | Non-compliant recording of individuals' behavioral information | Covered | DSP-08 Data Privacy by Design and Default; DSP-09 Data Protection Impact Assessment; DSP-12 Limitation of Purpose in Personal Data Processing; DSP-15 Limitation of Production Data Use | Purpose limitation, privacy by default and DPIA. LOG-14 requires logging all input content, so DSP-12 scoping applies to those logs. | specification | 2 / 2 / 0 |
| §2.2.5(b) | Insufficient compliance safeguards | Covered | GRC-07 Information System Regulatory Mapping; A&A-02 Independent Assessments; A&A-04 Requirements Compliance; DSP-09 Data Protection Impact Assessment; GRC-10 AI Impact Assessment | Regulatory mapping, independent audit, compliance verification and impact assessments. | specification | 5 / 0 / 0 |
| §2.2.5(c) | Inadequate channels for individuals to exercise their rights | Covered | DSP-11 Personal Data Access, Reversal, Rectification and Deletion | Access, modification and deletion requests by data subjects. | specification | 1 / 0 / 0 |
| §2.2.6(a) | Risks to stable operation of critical information infrastructure | Covered | BCR-03 Business Continuity Strategy; BCR-04 Business Continuity Planning; BCR-09 Disaster Response Plan; BCR-11 Equipment Redundancy; I&S-02 Capacity and Resource Planning; MDS-11 Model Failure; GRC-15 Human supervision; SEF-03 Incident Response Plans | Continuity strategy and planning, disaster response, redundancy, capacity and model failure evaluation. | specification | 3 / 2 / 3 |
| §2.2.6(b) | Mismatches between application and needs | Partial | GRC-10 AI Impact Assessment; DSP-24 Data Differentiation and Relevance | Impact assessment covers operational impact. No control requires a necessity or fitness-for-purpose decision before adoption. | specification | 1 / 0 / 1 |
| §2.2.6(c) | Misuse for illegal and criminal activities | Partial | GRC-09 Acceptable Use of the AI Service; TVM-11 Guardrails; AIS-09 Output Validation; LOG-14 Input Monitoring; LOG-15 Output Monitoring; IAM-14 Strong Authentication | Acceptable use with deepfakes named as a prohibited use (GRC-09 guidelines), guardrails and usage monitoring. No control detects forged audio and video or synthetic identities. | specification | 6 / 0 / 0 |
| §2.2.6(d) | Loss of control over nuclear, biological, chemical, and missile weapons knowledge | Partial | TVM-11 Guardrails; AIS-09 Output Validation; GRC-09 Acceptable Use of the AI Service | Guardrails and output filtering. No control requires CBRN capability evaluation or uplift thresholds. | specification | 3 / 0 / 0 |
| §2.2.6(e) | Manipulating social cognition | Gap | None | No control addresses automated accounts, fake identities or influence operations. Tangential only: GRC-09 acceptable use, GRC-10 impact assessment. | specification | 0 / 0 / 0 |

## §2.3 Secondary risks

| Ref | Risk | Coverage | AICM controls | What AICM covers and misses | Basis | Architecture met / partial / missing |
|---|---|---|---|---|---|---|
| §2.3.1(a) | Disruption of employment structures | Assess only | GRC-10 AI Impact Assessment | GRC-10 names societal impact; nothing prescribes treatment. | specification | 1 / 0 / 0 |
| §2.3.1(b) | Impact on education and suppression of innovation | Assess only | GRC-10 AI Impact Assessment | GRC-10 names societal impact; nothing prescribes treatment. | specification | 1 / 0 / 0 |
| §2.3.1(c) | Social alienation | Assess only | GRC-10 AI Impact Assessment; GRC-12 Ethics Committee | Impact assessment and ethics review only. | specification | 2 / 0 / 0 |
| §2.3.1(d) | Widening the intelligence divide | Assess only | GRC-10 AI Impact Assessment | GRC-10 names societal impact; nothing prescribes treatment. | specification | 1 / 0 / 0 |
| §2.3.2(a) | Challenges to the balance of resource supply and demand | Partial | DCS-13 Environmental Systems; I&S-02 Capacity and Resource Planning | DCS-13 guidelines require assessing data-center power and cooling capacity before deploying AI training hardware; I&S-02 plans compute capacity for performance. Nothing addresses land or water use. | guidelines | 0 / 1 / 1 |
| §2.3.2(b) | Challenges to green and low-carbon development | Partial | DCS-13 Environmental Systems; DCS-14 Secure Utilities; DSP-02 Secure Disposal | DCS-13 and DCS-14 guidelines call for quarterly analysis to optimise energy efficiency; DSP-02 guidelines require certified e-waste recyclers (R2 or e-Stewards). No control sets emissions or energy targets. | guidelines | 0 / 1 / 2 |
| §2.3.3(a) | Erosion of cultural diversity | Assess only | GRC-10 AI Impact Assessment; GRC-11 Bias and Fairness Assessment | Impact and fairness assessment only; no language or cultural coverage requirement. | specification | 2 / 0 / 0 |
| §2.3.3(b) | Cultural reshaping and intrusion | Assess only | GRC-10 AI Impact Assessment | GRC-10 names societal impact; nothing prescribes treatment. | specification | 1 / 0 / 0 |
| §2.3.4(a) | Aggravating social bias | Covered | GRC-11 Bias and Fairness Assessment; GRC-12 Ethics Committee; GRC-10 AI Impact Assessment | Bias and fairness evaluation with ethics review. | specification | 3 / 0 / 0 |
| §2.3.4(b) | Intensifying scientific research ethics risks | Partial | GRC-12 Ethics Committee; TVM-11 Guardrails; GRC-09 Acceptable Use of the AI Service | Ethics committee and guardrails. No control evaluates biology or genetics uplift. | specification | 3 / 0 / 0 |
| §2.3.4(c) | Emotional dependence | Assess only | GRC-12 Ethics Committee; GRC-10 AI Impact Assessment | GRC-12 guidelines send features affecting autonomy, decision rights or emotional manipulation to the ethics committee; GRC-10 assesses impact. No safeguards for adolescents or older adults. | guidelines | 2 / 0 / 0 |
| §2.3.4(d) | AI self-consciousness and loss of human control | Partial | GRC-15 Human supervision; AIS-11 Agents Security Boundaries; MDS-10 Model Continuous Monitoring; TVM-11 Guardrails | Human oversight, agent boundaries and TVM-11 emergency stop. No control requires loss-of-control or power-seeking evaluation. | specification | 4 / 0 / 0 |

## Appendix 2 agent lifecycle risks

| Ref | Risk | Coverage | AICM controls | What AICM covers and misses | Basis | Architecture met / partial / missing |
|---|---|---|---|---|---|---|
| A2 design | Inadequate safety requirements analysis | Covered | AIS-02 Application Security Baseline Requirements; AIS-04 Secure Application Development Lifecycle; TVM-12 Threat Analysis and Modelling; GRC-10 AI Impact Assessment; DSP-07 Data Protection by Design and Default | Security baseline, SDLC requirements analysis and threat modeling. | specification | 4 / 1 / 0 |
| A2 design | Inappropriate technology selection | Partial | MDS-12 Open Model Risk Assessment; STA-09 Supply Chain Risk Management; TVM-12 Threat Analysis and Modelling | Open model risk assessment and supplier risk review. No selection criteria for models or agent frameworks generally. | specification | 3 / 0 / 0 |
| A2 design | Inadequate security mechanism design | Covered | DSP-07 Data Protection by Design and Default; AIS-04 Secure Application Development Lifecycle; AIS-11 Agents Security Boundaries; TVM-12 Threat Analysis and Modelling | Security by design, SDLC and agent boundaries. | specification | 3 / 1 / 0 |
| A2 deploy | Compromised artifacts and components | Covered | MDS-02 Model Artifact Scanning; MDS-08 Model Integrity Checks; MDS-09 Model Signing/Ownership Verification; MDS-13 Secure Model Format; TVM-05 External Library Vulnerabilities; STA-16 Service Bill of Material (BOM) | Artifact scanning, checksums, signing, safe formats and BOM. | specification | 6 / 0 / 0 |
| A2 deploy | Weak deployment environments | Covered | I&S-04 OS Hardening and Base Controls; I&S-05 Production and Non-Production Environments; I&S-06 Segmentation and Segregation; AIS-06 Secure Application Deployment; CCC-06 Change Management Baseline | Hardening, environment separation, segmentation and baselines. | specification | 5 / 0 / 0 |
| A2 deploy | Insufficient security testing | Covered | AIS-05 Application Security Testing; TVM-06 Penetration Testing; MDS-06 Adversarial Attack Analysis; A&A-02 Independent Assessments; CCC-02 Quality Testing | Acceptance testing, penetration testing, adversarial analysis and independent assessment with red-team exercises. | specification | 5 / 0 / 0 |
| A2 input | Prompt injection | Covered | AIS-08 Input Validation; AIS-15 Prompt Differentation; TVM-02 Malware and Malicious Instructions Protection Policy and Procedures; TVM-11 Guardrails; LOG-14 Input Monitoring | Input filtering, instruction separation, malicious-instruction policy and guardrails with canary tokens for prompt leakage. | specification | 4 / 1 / 0 |
| A2 input | Context overflow | Partial | AIS-08 Input Validation; AIS-15 Prompt Differentation | Input validation and prompt separation help. No control bounds context size or protects system instructions from eviction. | specification | 2 / 0 / 0 |
| A2 reasoning | Misread intent | Partial | GRC-15 Human supervision; AIS-09 Output Validation; TVM-11 Guardrails | Oversight, output validation and confidence-threshold deferral to a human. | guidelines | 3 / 0 / 0 |
| A2 reasoning | Path deviation | Partial | GRC-15 Human supervision; TVM-11 Guardrails; LOG-05 Audit Logs Monitoring and Response; AIS-11 Agents Security Boundaries | Review gates before critical workflows (GRC-15 guidelines), anomaly-triggered automatic stop (TVM-11 guidelines), anomaly review and boundaries. No in-task deviation detection. | guidelines | 4 / 0 / 0 |
| A2 reasoning | Goal hijacking | Covered | AIS-15 Prompt Differentation; AIS-08 Input Validation; TVM-11 Guardrails; AIS-11 Agents Security Boundaries | Instruction and data separation with input guardrails. | specification | 4 / 0 / 0 |
| A2 invocation | Tool poisoning | Partial | TVM-02 Malware and Malicious Instructions Protection Policy and Procedures; STA-16 Service Bill of Material (BOM); TVM-05 External Library Vulnerabilities; IAM-19 Agent Access Restriction | Malicious-instruction policy, BOM and a risk-categorised tool inventory with owners (IAM-19 guidelines). No control requires reviewing tool descriptions, parameters and metadata before use. | specification | 3 / 1 / 0 |
| A2 invocation | Tool overreach | Covered | IAM-19 Agent Access Restriction; IAM-05 Least Privilege; AIS-11 Agents Security Boundaries; AIS-13 AI Sandboxing | Tools restricted to the task under need-to-know and least privilege. | specification | 4 / 0 / 0 |
| A2 invocation | Tool selection bias | Gap | None | No control addresses an agent systematically preferring a tool. Tangential only: GRC-15 oversight. | specification | 0 / 0 / 0 |
| A2 invocation | Contaminated tool responses | Covered | AIS-08 Input Validation; AIS-15 Prompt Differentation; TVM-02 Malware and Malicious Instructions Protection Policy and Procedures | Tool responses are input to the model and fall under input filtering and instruction separation. | specification | 2 / 1 / 0 |
| A2 invocation | Identity forgery | Covered | IAM-13 Uniquely Identifiable Users; IAM-14 Strong Authentication; I&S-03 Network Security; AIS-10 API Security | Unique IDs, certificate-based system identity and authenticated connections. | specification | 4 / 0 / 0 |
| A2 invocation | Tool hijacking | Covered | AIS-10 API Security; I&S-03 Network Security; IAM-16 Authorization Mechanisms; IAM-19 Agent Access Restriction | API security, authorized connections and per-request authorization. | specification | 4 / 0 / 0 |
| A2 invocation | Resource overload | Partial | AIS-10 API Security; AIS-11 Agents Security Boundaries; I&S-02 Capacity and Resource Planning; I&S-09 Network Defense; MDS-11 Model Failure | Rate limiting (AIS-10, AIS-11 guidelines), capacity planning and DoS defense. No per-agent limits on steps, run time or spend. | guidelines | 4 / 1 / 0 |
| A2 memory | Improper retention | Covered | DSP-16 Data Retention and Deletion; AIS-14 AI Cache Protection | Retention and deletion rules and cache protection with automatic expiry. | specification | 2 / 0 / 0 |
| A2 memory | Memory distortion | Partial | AIS-14 AI Cache Protection | Cache protection only; no memory integrity check. | specification | 1 / 0 / 0 |
| A2 memory | Memory contamination | Partial | AIS-14 AI Cache Protection; DSP-21 Data Poisoning Prevention & Detection | Cache protection with audit logs for poisoning attempts; DSP-21 targets training data, not runtime memory. | specification | 2 / 0 / 0 |
| A2 memory | Memory theft | Covered | AIS-14 AI Cache Protection; IAM-17 Knowledge Access Control - Need to Know; CEK-03 Data Encryption; DSP-17 Sensitive Data Protection | Cache protection, need-to-know access and encryption. | specification | 4 / 0 / 0 |
| A2 output | Sensitive or illegal content in output | Covered | AIS-09 Output Validation; LOG-15 Output Monitoring; TVM-11 Guardrails; DSP-17 Sensitive Data Protection | Output filtering, monitoring and sensitive data protection. | specification | 4 / 0 / 0 |
| A2 output | Malicious actions carried out | Covered | GRC-15 Human supervision; IAM-19 Agent Access Restriction; AIS-11 Agents Security Boundaries; AIS-13 AI Sandboxing; TVM-11 Guardrails | Human oversight with review gates and emergency stop, tool restriction, boundaries and sandboxing. | specification | 5 / 0 / 0 |
| A2 decommission | Service residue | Partial | CCC-04 Change Authorization; DSP-02 Secure Disposal; DSP-16 Data Retention and Deletion; AIS-04 Secure Application Development Lifecycle; GRC-02 Risk Management Program | Authorized asset removal, secure disposal, a decommissioning security review in the SDLC (AIS-04 guidelines) and documented decommissioning protocols (GRC-02 guidelines). No control requires verifying that processes, ports and connections are gone. | specification | 4 / 1 / 0 |
| A2 decommission | Leftover permissions and credentials | Covered | IAM-07 User Access Changes and Revocation; IAM-03 Identity Inventory; IAM-08 User Access Review; CEK-13 Key Revocation; IAM-15 Passwords and Secrets Management | Timely deprovisioning with cascaded token revocation, identity inventory, access review and key revocation. | specification | 5 / 0 / 0 |
| A2 decommission | Improper data retention | Covered | DSP-16 Data Retention and Deletion; DSP-02 Secure Disposal | Retention, archiving and deletion with secure disposal. | specification | 1 / 1 / 0 |
| A2 cross-cutting | Insecure communication protocols | Covered | I&S-03 Network Security; I&S-07 Migration to Hosted Environments; CEK-03 Data Encryption; CEK-04 Encryption Algorithm; AIS-10 API Security | Encrypted, authenticated connections with approved protocols. | specification | 5 / 0 / 0 |
| A2 cross-cutting | Missing logs | Covered | LOG-07 Logging Scope; LOG-08 Log Records; LOG-14 Input Monitoring; LOG-15 Output Monitoring; LOG-13 Failures and Anomalies Reporting; LOG-02 Audit Logs Protection | Logging scope, audit records, AI input and output logs and monitoring-failure alerts. | specification | 6 / 0 / 0 |
| A2 cross-cutting | Behavior escape | Covered | AIS-13 AI Sandboxing; AIS-11 Agents Security Boundaries; I&S-06 Segmentation and Segregation | Sandboxed execution and agent boundaries with honeypots for boundary-crossing attempts. | specification | 3 / 0 / 0 |
| A2 cross-cutting | System privilege abuse | Covered | IAM-04 Separation of Duties; IAM-05 Least Privilege; IAM-09 Segregation of Privileged Access Roles; IAM-10 Management of Privileged Access Roles; LOG-05 Audit Logs Monitoring and Response | Separation of duties, privileged role segregation and time-bound privilege. | specification | 5 / 0 / 0 |
| A2 cross-cutting | Improper skill management | Covered | IAM-19 Agent Access Restriction; IAM-08 User Access Review; CCC-04 Change Authorization; STA-16 Service Bill of Material (BOM); TVM-05 External Library Vulnerabilities | Tool restriction, periodic access review, authorized addition and removal, BOM. | specification | 5 / 0 / 0 |
| A2 cross-cutting | Out-of-scope data use | Covered | DSP-12 Limitation of Purpose in Personal Data Processing; IAM-17 Knowledge Access Control - Need to Know; DSP-15 Limitation of Production Data Use | Purpose limitation and need-to-know access. | specification | 2 / 1 / 0 |
