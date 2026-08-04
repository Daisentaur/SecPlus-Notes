# Chapter 13 — Business Security and Governance

## 13.1 Introduction to Business Security

The shift in this chapter: everything so far has been technical controls. This is the **management layer that decides which controls exist, who owns them, and how anyone knows they're working.**

Worth taking seriously rather than treating as paperwork, because a security team without governance has no mandate, no budget and no authority — it just has opinions.

### Governance

The structure that sets direction and holds people accountable. Boards, steering committees, defined roles, and the documents below.

### The document hierarchy

These get used interchangeably in speech and they are not interchangeable:

* **Policy** - high-level statement of intent and requirement. **Mandatory.** Approved at executive level, changes rarely. *"All data classified confidential or above must be encrypted at rest."*
* **Standard** - the specific measurable requirement implementing the policy. **Mandatory.** *"Encryption at rest uses AES-256."*
* **Procedure** - step-by-step instructions. **Mandatory.** *"To enable BitLocker: ..."*
* **Guideline** - recommended practice. **Optional.** Advice, not requirement.

**Policy says what and why · standard says to what specification · procedure says how · guideline suggests.**

A policy without procedures never gets implemented. Procedures without policy have no authority behind them. Both failure modes are common.

### Common policies

AUP (acceptable use) · information security · business continuity · disaster recovery · incident response · SDLC · change management · password · remote access · data classification and retention.

### External considerations

Regulatory, legal, industry and geographic (local / regional / national / global). Multinationals face genuinely conflicting requirements — data localisation in one jurisdiction versus lawful access demands in another — and someone has to make a **documented decision about which obligation to breach**.

### Monitoring and revision

Policies decay. Annual review as standard, plus review triggered by significant change or by an incident.

## 13.2 Business Impact Analysis

Determining what a disruption actually costs, so recovery investment can be justified and prioritised.

### The metrics

* **RTO (Recovery Time Objective)** - how long you can be down. **A time target.**
* **RPO (Recovery Point Objective)** - how much data you can afford to lose, measured in time. **A data target.** An RPO of 1 hour means backups at least hourly.
* **MTTR (Mean Time To Repair)** - average time to restore after a failure.
* **MTBF (Mean Time Between Failures)** - average time between failures. A reliability measure.
* **MTTF (Mean Time To Failure)** - for non-repairable items.

**RTO and RPO are the two people mix up. RTO is about the clock; RPO is about the data.** Both are set by the business based on impact, and together they determine whether you need a hot site or a cold one, and how often backups run. A 15-minute RPO and a 1-hour RTO is a completely different (and vastly more expensive) architecture from 24 hours of each.

### The process

1. **Identify critical business functions** — what does the organisation actually have to be able to do.
2. **Map dependencies** — systems, data, people, suppliers, facilities. This step consistently surfaces surprises, usually a single undocumented system everything quietly relies on.
3. **Determine impact of disruption** over time — financial, operational, reputational, regulatory, safety. Impact is rarely linear; an hour of outage may be trivial and a day catastrophic.
4. **Set RTO and RPO** per function.
5. **Identify the recovery requirements** to meet them.
6. **Prioritise** — you cannot recover everything simultaneously, so a recovery *order* has to be agreed in advance, when nobody is panicking.

### Related

* **Single point of failure** identification — including **people**. One person who understands the system is an availability risk exactly like one server.
* **Mission essential functions** and **critical systems** identification.
* The quantitative maths (SLE, ARO, ALE) from chapter 1.7 feeds the justification.

## 13.3 Data Types and Roles

### Data types

* **Regulated data** - subject to specific legal requirements.
* **PII (Personally Identifiable Information)** - identifies an individual.
* **PHI (Protected Health Information)** - health data; HIPAA obligations in the US.
* **Financial information** - PCI-DSS scope for card data.
* **Intellectual property** and **trade secrets**.
* **Legal information** - privileged, subject to hold.
* **Human-readable vs non-human-readable** - the latter (binary formats, embeddings) still counts as the data it represents, which people forget.

### Classification

**Public** → **Private/Internal** → **Sensitive/Confidential** → **Restricted/Critical**. Government: unclassified → classified → secret → top secret.

The purpose is triage: **you cannot protect everything equally**, so classification tells you where the money goes.

### Roles

* **Data owner** - accountable. Senior, usually a business figure. Sets classification, approves access.
* **Data controller** - determines the **purposes and means** of processing (GDPR term).
* **Data processor** - processes on the controller's behalf. A vendor.
* **Data custodian / steward** - implements and operates the controls day to day. Usually IT.
* **DPO (Data Protection Officer)** - accountable for privacy compliance. Mandatory under GDPR in defined circumstances, and required to be independent.

**The legally important point: the controller holds the obligation even when a processor does the work.** Outsourcing processing does not outsource accountability — the same structure as "transfer moves only financial impact" from chapter 1.4.

### Privacy concepts

* **Data subject** - the living individual the data concerns.
* **Right to be forgotten** - the subject can demand erasure. Operationally brutal, because it means locating and deleting one individual's data across every system, log and backup. **The strongest practical argument for not retaining data you don't need.**
* **Data sovereignty** - data is subject to the laws of the country it physically sits in, making cloud region selection a legal decision.
* **Consent and purpose limitation** - collected for a stated purpose, not reused for another without fresh basis.
* **Data minimisation** - the cheapest control there is. **Data never collected cannot leak.**

## 13.4 Personnel Risk and Policies

People are simultaneously the largest attack surface and the control that catches what tooling misses.

### Pre-employment

* **Background checks** - proportionate to the role's access.
* **NDA** - confidentiality obligations, signed before access.
* **Onboarding** - provisioning, policy acknowledgement, initial security training.

### During employment

* **Acceptable Use Policy** - what's permitted on company systems.
* **Security awareness training** - initial and **recurring**, because it decays fast. Phishing simulations, role-specific content, and reporting-channel familiarity.
* **Separation of duties** - no single person completes a sensitive process end to end. Requester ≠ approver. Defends against fraud *and* honest mistakes.
* **Job rotation** - surfaces fraud dependent on one person permanently controlling a process, and exposes single points of knowledge.
* **Mandatory vacation** - the same logic. Schemes needing daily maintenance break when the person is away for two weeks.
* **Least privilege** and periodic **access reviews** to catch privilege creep.
* **Clean desk policy** - physical information hygiene.

### Offboarding

* **Immediate de-provisioning**, triggered by HR rather than by someone remembering. **The step that fails most often** — orphaned accounts are a standard pentest finding and a standard breach cause.
* Return of assets and badges.
* Exit interview and NDA reminder.
* Disable rather than delete initially, for audit and data recovery.

### Insider threat

* **Malicious** - disgruntled, bribed, coerced, or recruited.
* **Negligent** - careless. **Far more common**, and responsible for more incidents.
* **Compromised** - a legitimate user whose account an attacker controls, which looks identical to insider activity from the logs.

Indicators: unusual access patterns, bulk downloads, off-hours activity, attempts to reach data outside their role, and — the non-technical ones that matter — disgruntlement, financial pressure, and an announced resignation.

**Detection relies on UEBA (User and Entity Behaviour Analytics)** — baselining normal behaviour per identity and flagging deviation. Which is the anomaly-detection trade from chapter 7.28 again: catches novel behaviour, generates false positives, needs a clean baseline.

## 13.5 Attestation

A **formal, signed statement that something is true**, with the signer's accountability behind it.

Types:
* **Self-attestation** - the organisation asserts its own compliance. Cheapest, weakest, and it still carries weight because someone signed it.
* **Third-party attestation** - an independent assessor states it. What SOC 2 reports are.
* **Regulatory attestation** - a formal declaration to a regulator.

### Why it matters

**It converts a claim into accountability.** An unsigned assertion that controls are working carries no consequence if it turns out to be false. A signed attestation from a named officer does — including personal legal exposure in some regimes (SOX being the clear example, where executives sign for financial reporting controls).

That's the whole mechanism: governance works by attaching a name to a claim.

## 13.6 Internal Audits and Assessments

Performed by the organisation's own audit function.

* **Compliance focus** - are we meeting our own policies and applicable regulations.
* **Internal committees** and audit boards oversee it.
* **Self-assessments** - a team evaluating its own controls. Useful for continuous improvement, and the weakest form of evidence because it's marking your own homework.

### The independence problem

**An internal audit function reporting to the people it audits is decoration.** For it to mean anything, it has to report to the board or an audit committee, not to the IT or operations leadership whose work it examines. That structural detail is the difference between real assurance and a formality — and it's the first thing worth checking about any internal audit function.

### Value

Cheaper and more frequent than external audits · builds familiarity with the environment · catches issues *before* an external auditor does, which is far less expensive · and prepares the organisation for external assessment.

## 13.7 External Audits and Assessments

Performed by an independent third party.

* **Regulatory requirement** in many industries — financial services, healthcare, payment processing.
* **Examinations** and **assessments** against a defined standard.
* **Independent third-party assessment** carries weight *because* of the independence.

### Common frameworks

* **SOC 1** - controls over financial reporting.
* **SOC 2** - security, availability, processing integrity, confidentiality, privacy. **Type I** is design at a point in time; **Type II** is operating effectiveness over a period (usually 6–12 months) and is much more meaningful.
* **ISO 27001** - certifiable ISMS standard.
* **PCI-DSS** - payment card data. Contractual rather than legal, and enforced by the card brands.
* **HIPAA**, **FedRAMP**, **GDPR** audits.

### Practical points

* External auditors need evidence, not assertions — logs, screenshots, configuration exports, tickets. Organisations that don't collect evidence continuously end up in a scramble before every audit, which is what GRC platforms (Vanta, Drata) exist to automate.
* **Findings** are ranked, and remediation is tracked to closure.
* **Compliance is not security.** You can pass an audit and be insecure, and you can be genuinely secure and fail an audit on documentation. They overlap; they're not the same thing. Worth holding onto, because the incentive in a compliance-driven organisation drifts toward satisfying the auditor rather than reducing risk.

## 13.8 Third-Party Risk Management

**You inherit the security posture of everyone you depend on**, usually with far less visibility into them than into yourself.

SolarWinds, Kaseya and MOVEit were all third-party compromises that hit thousands of downstream organisations who had done nothing wrong themselves.

### Vendor selection

* **Due diligence** *before* signing, because you have leverage then and none afterwards.
* **Conflict of interest** checks.
* Evidence review: SOC 2 reports, ISO certificates, pentest summaries, security questionnaires.

### Vendor assessment

* **Penetration testing** of the vendor, where contractually permitted.
* **Right-to-audit clause** — negotiated into the contract up front.
* **Evidence of internal audits**.
* **Independent assessments**.
* **Supply chain analysis** — extending to your vendor's vendors (**fourth-party risk**). You depend on things you've never heard of.

### Vendor monitoring

**Continuous, not once at onboarding.** Posture changes, and an annual questionnaire is a snapshot of a moment that's already passed. Continuous monitoring services score vendors on externally observable signals.

### The security terms worth fighting for in the contract

* **Breach notification obligations and timelines** — specific hours, not "promptly".
* Right to audit.
* Data handling, location and encryption requirements.
* Subcontractor restrictions and notification.
* Return or destruction of data at termination.
* Liability and indemnification.

All of these are cheap to obtain before signing and effectively impossible after.

## 13.9 Agreement Types

The acronyms, which get thrown around constantly:

* **SLA (Service Level Agreement)** - measurable performance commitments — uptime, response times — and the penalties for missing them.
* **MOU (Memorandum of Understanding)** - statement of intent. Typically **non-binding**.
* **MOA (Memorandum of Agreement)** - more formal than an MOU, generally binding.
* **MSA (Master Service Agreement)** - the overarching contract governing the relationship, so individual work packages don't renegotiate terms each time.
* **SOW (Statement of Work) / WO (Work Order)** - the specific deliverables, timeline and cost under an MSA.
* **NDA (Non-Disclosure Agreement)** - confidentiality. Can be one-way or mutual.
* **BPA (Business Partner Agreement)** - terms between partners in a joint venture.
* **AUP (Acceptable Use Policy)** - can be a contractual term for service users, as well as an internal policy.

### The one that matters most day to day

**The SLA**, because it's where availability commitments become enforceable. And the detail people miss: **an SLA penalty rarely compensates for the actual loss.** A 99.9% uptime SLA that refunds 10% of a month's fee does not cover what a day of downtime cost your business. That's chapter 1.4's transfer lesson again — **you moved a fraction of the financial impact, not the risk**.

## 13.10 Change Management

Not obviously security, and absolutely security: **a large share of real outages and newly-introduced vulnerabilities come from changes, not attacks.** Someone pushed a config, opened a firewall rule "temporarily", upgraded a library. Change management stops the environment drifting into a state nobody chose.

### The process

* **Approval process** - someone other than the implementer signs off. Separation of duties applied to change.
* **Change Advisory Board (CAB)** - the group approving anything significant.
* **Ownership** - a named person responsible, not "the platform team".
* **Stakeholders** - who else is affected. Consistently underestimated, because the person making the change usually only knows their own system.
* **Impact analysis** - what breaks if this goes wrong.
* **Test results** - it worked somewhere that wasn't production.
* **Backout plan** - how to undo it. **Non-negotiable. A change you cannot reverse is a change you should not make.**
* **Maintenance window** - timed so the blast radius of a failure is smallest.
* **SOP** - the documented standard procedure, so it's done identically every time.

### Change types

* **Standard** - pre-approved, low risk, routine. Follows a known procedure.
* **Normal** - goes through full assessment and approval.
* **Emergency** - expedited for an urgent issue, with **retrospective** review. The category most abused, since labelling something an emergency skips the controls — so emergency change rates are themselves worth monitoring.

## 13.11 Technical Change Management

The parts that actually bite engineers.

* **Allow lists / deny lists** - a change to what software may run must be reflected in the allow list, or the change breaks the application.
* **Restricted activities** - some changes are simply off-limits outside a window.
* **Downtime** - planned, versus the unplanned kind you get by skipping the process.
* **Service and application restarts** - the change isn't live until the thing restarts, and restarts are where surprises live (a config that was broken for weeks only fails at restart).
* **Legacy applications** - nobody wants to touch them, so they never get changed, so they never get patched. **Change management failure by avoidance**, and it's how organisations accumulate unpatchable systems.
* **Dependencies** - upgrading A breaks B. The software supply chain problem in miniature.

### Documentation

Updating diagrams, runbooks and policies **after** the change. Sounds like bureaucracy and it's the difference between incident response working and not — **during an incident, a two-year-stale network diagram actively misleads the responders**, which is worse than having none.

### Version control

Not just for code. **Infrastructure as code, firewall configs, policies and pipelines should all be versioned**, because then "what changed, when, and who approved it" is a question with an actual answer. That single property makes both incident response and audit dramatically easier.

## 13.12 Automation and Orchestration — What It Is

* **Automation** - a single task performed without human intervention. Creating a user account.
* **Orchestration** - coordinating multiple automated tasks into a workflow. Onboarding: create the account, assign groups, provision a mailbox, enrol the device, notify the manager.

Automation is the step; orchestration is the sequence.

## 13.13 Benefits of Automation and Orchestration

* **Efficiency and time saving** - the obvious one, and the least interesting.
* **Enforcing baselines** - the configuration is applied identically every time. **This is the security benefit that matters most**, because it removes human inconsistency, and inconsistency is where a lot of risk lives.
* **Standard infrastructure configurations** - no snowflake servers.
* **Scaling securely** - security keeps pace with growth rather than falling behind it.
* **Employee retention** - automating tedious repetitive work genuinely helps keep analysts, and **burnout is a security risk** because tired analysts miss things.
* **Reaction time** - machine-speed containment. Automated isolation of a compromised host happens in seconds rather than after someone reads the alert.
* **Workforce multiplier** - a small team covering far more ground.

## 13.14 Use Cases of Automation and Orchestration

* **User provisioning and de-provisioning** - which directly fixes the offboarding failure from 13.4. SCIM (chapter 4.8) is this.
* **Resource provisioning** - infrastructure as code.
* **Guard rails** - automated policy enforcement preventing insecure configurations from being deployed at all, rather than detecting them afterwards.
* **Security group management**.
* **Ticket creation and escalation**.
* **Escalation workflows** for alerts.
* **Enabling/disabling services and access**.
* **Continuous integration and testing** - SAST/DAST/SCA in the pipeline (chapter 7.20).
* **Integrations and APIs** between security tools.
* **SOAR playbooks** - enrich an alert with threat intelligence, open a ticket, block the IP, isolate the host, notify the analyst.

## 13.15 Other Considerations of Automation and Orchestration

Being honest about the downsides, because automation is not free:

* **Complexity** - the automation becomes a system that itself needs securing. **And its credentials are usually highly privileged** — compromise the automation platform and you have compromised everything it manages. That makes it a very high-value target.
* **Cost** - tooling, and the engineering time to build and maintain it.
* **Single point of failure** - if the orchestration platform is down, everything it does stops.
* **Technical debt** - scripts written quickly, never documented, and the author leaves.
* **Ongoing supportability** - automation breaks silently when the things it touches change. **Silent failure of a security control is worse than no control, because you believe you're covered.** This is the one worth watching hardest.

## 13.16 Putting It All Together

The layers stack:

**Governance** sets direction (policies, standards, roles) → **Risk management** identifies and prioritises what matters (chapter 1) → **Controls** implement the decisions (technical, managerial, operational, physical) → **Automation** applies them consistently at scale → **Monitoring** confirms they're working → **Audit and attestation** provide independent assurance → **Incident response** handles the failures (chapter 14) → and the lessons feed back into governance.

The loop is the point. A programme that only goes one direction produces policies nobody follows and controls nobody validates.

### What ties it together

* **Everything traces to risk.** A control that doesn't reduce an identified risk is spending without justification.
* **Accountability cannot be delegated.** You outsource work, never responsibility.
* **Documentation is what makes any of it reviewable**, transferable, and defensible.
* **Compliance is a floor, not a ceiling.**

## 13.17 The NIST Frameworks

NIST publications are the common reference vocabulary across the industry, and they're free, which is why they're everywhere.

### NIST Cybersecurity Framework (CSF)

Voluntary, risk-based, widely adopted. **CSF 2.0 (2024)** added **Govern** as a sixth function, sitting across the others.

* **Govern** *(2.0)* - strategy, roles, policy, oversight, supply chain risk.
* **Identify** - assets, risks, governance context. *You can't protect what you don't know exists.*
* **Protect** - controls: access control, awareness, data security, maintenance.
* **Detect** - monitoring, anomaly detection.
* **Respond** - incident response, communications, mitigation.
* **Recover** - restoration, improvements, communications.

Also **Tiers** (1 Partial → 4 Adaptive) describing maturity, and **Profiles** describing current versus target state — which is exactly a **gap analysis** (chapter 1) expressed in NIST vocabulary.

### NIST SP 800-53

The comprehensive control catalogue — over a thousand controls across 20 families, used by US federal agencies and their contractors. The reference you pull specific control language from.

### NIST SP 800-171

Protecting **Controlled Unclassified Information (CUI)** in non-federal systems. Matters to any contractor in the US federal supply chain.

### NIST SP 800-37 — Risk Management Framework (RMF)

Prepare → Categorize → Select → Implement → Assess → Authorize → Monitor.

### NIST SP 800-61 — Incident Handling Guide

The four-phase IR lifecycle used in chapter 14.

### NIST SP 800-63 — Digital Identity Guidelines

The source of the modern password guidance: **length over complexity, check against breach corpora, and no forced periodic rotation without evidence of compromise** (chapter 2.4). Worth knowing by name, because "NIST says so" is what actually shifts organisational password policy off the 90-day rotation habit.

### NIST AI RMF 1.0

Govern, Map, Measure, Manage — the AI-specific risk vocabulary, structured to parallel the CSF. Relevant to anything deploying agentic systems, including my own VAPT pipeline.

### Others worth recognising

**ISO 27001/27002** (certifiable ISMS), **CIS Controls** (prioritised, prescriptive, arguably the most practical starting point for a small organisation), **COBIT** (IT governance), **MITRE ATT&CK** (adversary behaviour), **MITRE ATLAS** (adversarial ML), **Google SAIF** (secure AI), and **ISO/IEC 42001** (AI management systems).

**How they relate:** NIST CSF and ISO 27001 give you structure; CIS Controls give you a prioritised to-do list; ATT&CK gives you the adversary's vocabulary; the regulations tell you what's mandatory. They're complementary rather than competing, and organisations typically map between several.

---

## Chapter 13 — what I'd take away

* Policy / standard / procedure / guideline are distinct. Policy without procedure never happens; procedure without policy has no authority.
* RTO is the clock, RPO is the data. They set the architecture and the cost.
* Data minimisation is the cheapest control — data never collected cannot leak.
* The controller keeps the obligation even when a processor does the work.
* An internal audit function reporting to the people it audits is decoration.
* Compliance is a floor, not a ceiling — and the incentive drifts toward satisfying auditors rather than reducing risk.
* Contract security terms are cheap before signing and impossible after.
* Automation's biggest risk is that it holds privileged credentials and fails silently.
* NIST CSF 2.0 added Govern; SP 800-63 is why forced password rotation is out of favour.
