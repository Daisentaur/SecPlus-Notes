# Chapter 14 — Incident Response and Forensics

## 14.1 Incident Response Overview

### Event vs incident

* **Event** - any observable occurrence. A login. A file write. Millions per day.
* **Alert** - an event flagged as worth attention.
* **Incident** - an event that **actually or potentially harms** confidentiality, integrity or availability.

The distinction matters because it determines what process kicks in. Most alerts are not incidents, and the triage step that decides is where analyst time goes.

### The phases (NIST SP 800-61)

NIST uses four phases; SANS splits them into six. Same content either way:

1. **Preparation**
2. **Detection and Analysis**
3. **Containment, Eradication and Recovery**
4. **Post-Incident Activity**

SANS: Preparation → Identification → Containment → Eradication → Recovery → Lessons Learned.

### Preparation carries most of the value

**Everything you do before an incident determines how the incident goes.** Once it starts, you use what you already have — you don't build tooling, write plans or establish contacts at 3am.

Preparation means: a written and tested IR plan · defined roles and an on-call rota · contact lists including legal, PR, insurance, law enforcement and your provider · logging and monitoring actually working (chapter 5.12) · **tested** backups · forensic tooling and clean media ready · playbooks for likely scenarios · and training.

### Roles

* **Incident response team / CSIRT** - the responders.
* **Incident commander** - runs the response and makes the calls. **One named person**, decided in advance, because response by committee at 3am fails.
* **Technical leads**, **communications lead**, **legal counsel**, **executive sponsor**.

### Detection and analysis

Sources: SIEM correlation, EDR alerts, IDS/IPS, user reports (still one of the most valuable channels), threat intelligence, and third-party notification — which is unfortunately how many organisations learn they were breached.

Analysis determines: what happened, scope, which systems and data are affected, whether it's still active, and how they got in.

**Triage and prioritisation** by functional impact, information impact and recoverability. Not everything can be first.

## 14.2 Incident Response Plans

### What the plan contains

* **Scope and definitions** - what counts as an incident, and severity levels.
* **Roles and responsibilities** - named, with deputies.
* **Contact information** - internal, external, out-of-hours, and **out-of-band** (see below).
* **Classification and severity criteria** - so escalation is consistent and doesn't depend on who picked up the phone.
* **Procedures per phase**.
* **Communication plan** - internal, customers, regulators, law enforcement, press.
* **Escalation thresholds**.
* **Evidence handling** requirements.
* **Recovery priorities** - the order things come back, agreed in advance.
* **Legal and regulatory obligations**, including notification deadlines.

### Playbooks

Scenario-specific procedures: ransomware, BEC, data exfiltration, DDoS, insider threat, lost device, web compromise. Playbooks turn general principles into concrete steps, which is what people can actually follow under pressure.

### Communication — the part that decides how the incident is judged

Not one of the technical phases, and in practice it determines the outcome as much as the technical response does.

* **Who notifies the regulator, within what deadline.** GDPR is **72 hours** from awareness. Others differ. Missing the deadline is a separate violation from the breach itself.
* **Who talks to customers**, and when. Too early and you're wrong; too late and you look like you concealed it.
* **Who talks to press** — one voice, briefed.
* **Whether and when to involve law enforcement.**
* **Internal communication** so staff aren't learning about it from the news.

**Out-of-band communication is the detail people miss.** If the corporate email or Slack is compromised, coordinating the response over it hands the attacker your playbook in real time. Agree an alternative channel in advance — a separate messaging app, phone tree, anything not dependent on the systems under investigation.

## 14.3 IRP Testing

A plan that's never been exercised **will** fail on contact, usually at something mundane like "nobody knows who can authorise taking production offline at 2am".

### Exercise types, in increasing cost and realism

* **Tabletop exercise** - discussion-based walkthrough of a scenario. Cheap, no disruption, and it reliably surfaces process gaps, unclear ownership and missing contacts. **The highest value per rupee spent**, and the one to do first.
* **Walkthrough / structured review** - step through the documented procedures in detail.
* **Simulation** - a realistic scenario exercised in a controlled way, with people actually doing things.
* **Parallel test** - recovery systems brought up alongside production without cutting over.
* **Full interruption / failover test** - actually switching to the secondary and running on it. Most realistic, highest risk, and the only way to know your DR site genuinely works.

### What testing actually surfaces

In practice it's rarely the technical steps that break. It's: outdated contact lists · nobody knowing who has authority to make a given decision · backups that were never restore-tested · a network diagram that's two years stale · missing tooling access · and no out-of-band communication channel.

### Frequency and follow-through

At least annually, plus after significant change. And **the findings have to be tracked to closure** — an exercise that produces a list nobody actions is theatre. Rotate scenarios so you're not rehearsing the same one indefinitely.

## 14.4 Threat Analysis and Mitigating Actions

### Containment

Stop the spread. **The judgement call:** containing too early destroys evidence and tips off the attacker; containing too late lets them spread further. There's no clean answer, which is exactly why the decision should belong to a named role decided in advance.

* **Short-term containment** - isolate the host, pull it off the network (but consider leaving it powered on to preserve memory), block the C2 address, disable the compromised account.
* **Long-term containment** - temporary fixes allowing business to continue while you prepare eradication.
* **Segmentation and isolation** - limit lateral movement.

**Isolate rather than power off** where you can — pulling the plug destroys everything in RAM, which is where fileless malware, encryption keys and network state live (see 14.5's order of volatility).

### Eradication

Remove the attacker and **all** their persistence. Missing one backdoor means they're back next week — which is why **full rebuild from known-good media is often chosen over cleaning**. You can be certain about a rebuild; you can only hope about a clean.

Also: patch the exploited vulnerability, reset compromised credentials (all of them, including service accounts and any credential the attacker could have reached), and remove created accounts, scheduled tasks and services.

### Recovery

Restore from clean backups · verify integrity · **validate the vulnerability that allowed entry is actually closed** before reconnecting · phased return to production · and **heightened monitoring afterwards**, because attackers frequently return, particularly if you eradicated imperfectly.

### Root cause analysis

The **actual underlying failure**, not the proximate one.

"The user clicked a link" is not a root cause. "We had no MFA, no email filtering, a flat network, and no egress control" is. The test is whether fixing your stated cause would actually have prevented it — and blaming a user almost never passes that test.

### Lessons learned

* Held soon after resolution, while detail is fresh.
* **Blameless.** This isn't a nicety — if people are punished, they hide information during the next incident, and you lose the thing that matters most. A culture where the person who clicked the link reports it in ten seconds is worth more than any control you can buy.
* Output: what happened, what worked, what didn't, and **specific tracked action items with owners**.
* Feed the findings back into detection rules, playbooks, controls and training.

### Threat hunting

Proactively looking for compromise **that no alert fired on**, working from hypotheses about attacker TTPs rather than waiting for detection. "If an attacker were living off the land here, what would that look like in my logs?"

This is where the TTP thinking from chapter 1.3 becomes an actual job, and it assumes something the alerting-only model doesn't: **that you are already compromised and haven't noticed.**

## 14.5 Digital Forensics

Evidence handling, for when the incident becomes a legal matter.

### Legal hold

A formal instruction to **preserve all potentially relevant data**, overriding normal retention and deletion schedules. Ignoring it is **spoliation** and carries legal consequences independent of the incident itself. Issued as soon as litigation is reasonably anticipated.

### Chain of custody

The unbroken documented record of **who had the evidence, when, and what they did with it**. Every transfer signed and timestamped.

**A break in the chain can make evidence inadmissible regardless of what it proves.** That's the whole reason the paperwork matters.

This is exactly why my VAPT pipeline stores **SHA-256 and MD5 hashes** with every uploaded report — a hash taken at acquisition and re-verified later proves the artefact wasn't altered in between. It converts "trust us" into something mathematically demonstrable.

### Acquisition

* Work from a **forensic image** — a bit-for-bit copy — **never the original**. Hash both and confirm they match.
* Use a **write blocker** so the acquisition process cannot modify the source.
* Document everything: time, method, tools, versions, who performed it.

### Order of volatility

Collect the most perishable first:

1. CPU registers, cache
2. **RAM** — running processes, network connections, encryption keys, fileless malware
3. Network state, routing/ARP tables, live connections
4. Running processes
5. Disk
6. Remote logging and monitoring data
7. Physical configuration, network topology
8. Archival media / backups

**Pulling the power to "preserve" a machine destroys everything above disk.** That's a genuinely common mistake, and it's why "isolate, don't power off" is the containment guidance in 14.4.

### Analysis

Disk analysis (deleted files, slack space, unallocated space, filesystem timestamps) · memory analysis (Volatility — processes, injected code, network connections, keys) · **timeline reconstruction** from filesystem, log and registry timestamps · log correlation · and network analysis from captures.

**Anti-forensics** to watch for: timestamp manipulation (timestomping), log deletion, secure wiping, encryption, and steganography.

### Reporting and preservation

Documented findings, methodology and conclusions, written to survive challenge by someone motivated to discredit them. Preserve evidence securely for as long as required.

### E-discovery

The legal process of identifying, preserving, collecting and producing electronic evidence for proceedings. Broader than incident forensics and frequently the reason retention and legal hold policies exist at all.

### Non-repudiation in forensics

Where chapter 2's non-repudiation earns its keep. **Hashes, digital signatures and timestamps** are what let you assert that evidence is what it was at collection time, to a third party who is actively motivated to argue otherwise. Logs alone are weak — the admin controls the logs. A signature is strong.

## 14.6 Business Continuity and Alternate Sites

### BCP vs DRP

* **Business Continuity Plan** - keeping the **business** operating during a disruption. Broader: people, processes, facilities, suppliers, communications.
* **Disaster Recovery Plan** - restoring **IT systems** specifically. A subset of BCP.

BCP asks "how do we keep serving customers"; DRP asks "how do we get the servers back". Both are needed, and organisations frequently write the DRP and skip the BCP.

### Alternate sites

| Site | Equipment | Data | Recovery time | Cost |
|---|---|---|---|---|
| **Hot** | Fully equipped, running | Replicated live | Minutes | Highest |
| **Warm** | Equipment and connectivity in place | Needs restoring | Hours to days | Medium |
| **Cold** | Space and power only | Nothing | Days to weeks | Lowest |
| **Mobile** | Trailer/container, deployable | Varies | Varies | Varies |
| **Cloud / DRaaS** | On-demand | Replicated | Minutes to hours | Pay-as-you-go |

**Reciprocal agreement** — two organisations agree to host each other. Cheap, and fraught: capacity may not exist when needed, confidentiality is awkward, and a regional disaster hits both of you.

**The choice is driven by RTO and RPO** (chapter 13.2), which are set by the BIA, which is set by business impact. That chain is the answer to "why are we spending this much on a standby site" — and it's why cloud DR has changed the economics substantially, since you no longer pay for idle hardware.

### Geographic dispersion

Far enough apart that a single event can't take both. **A DR site in the same flood plain, on the same power grid, or served by the same fibre path is decorative.** Also consider whether staff can physically reach it, and data sovereignty implications of the location (chapter 7.1).

### Backups, briefly

Full / incremental / differential / snapshots (chapter 6.7 for RAID, and RAID is **not** backup).

**3-2-1 rule** — three copies, two media types, one off-site. Modern amendment: **one offline or immutable**, because ransomware deliberately hunts and encrypts network-reachable backups.

**And the point that outranks all the rest: a backup you have never tested restoring is not a backup, it is a hope.** Untested backups fail at restore time constantly — incomplete sets, missing encryption keys, corrupted media, or a restore that takes four days when the business needed four hours.

### Resilience

High availability, redundancy, load balancing, clustering, failover · **capacity planning for people, technology and infrastructure** — and the one that gets forgotten is **people**, since a single person who understands the system is an availability risk exactly like a single server · UPS and generators (tested under load) · and **testing**, without which none of the above is known to work.

## 14.7 Lab — Autopsy Forensic Browser

Autopsy is the GUI front end to The Sleuth Kit — open-source disk forensics.

### Workflow

```bash
# acquire an image first, with a write blocker on the source
sudo dd if=/dev/sdb of=evidence.dd bs=4M status=progress conv=noerror,sync

# hash immediately, before anything touches it
sha256sum evidence.dd | tee evidence.dd.sha256

# better: use a forensic format that stores metadata and hashes internally
sudo ewfacquire /dev/sdb        # produces .E01
```

Then in Autopsy: create a case (case number, examiner name — this is chain of custody metadata) → add the image as a data source → select ingest modules → analyse.

### Ingest modules worth running

Recent Activity · Hash Lookup (against known-good and known-bad hash sets) · File Type Identification · **Extension Mismatch Detector** · Embedded File Extractor · **Keyword Search** · Email Parser · **Encryption Detection** · Interesting File Identifier.

### What you can recover

* **Deleted files** — recovered from unallocated space, because deleting removes the index pointer, not the data (chapter 1.10). Seeing this work is what makes that point concrete rather than theoretical.
* **File slack space** — remnants of previous files in the tail of a cluster.
* **Timeline** — a unified chronology from filesystem, log and registry timestamps. Usually the single most valuable artefact for reconstructing what happened.
* **Web history, downloads, bookmarks, cookies.**
* **Registry hives** (Windows) — USB device history, recently opened files, autostart entries, network connections.
* **Extension mismatches** — a file named `.jpg` whose header says it's an executable. Deliberate concealment, and trivially detectable once you look at content rather than name.
* **Keyword hits** across allocated and unallocated space.

### What the lab actually taught

1. **Deleted does not mean gone.** Recovering files I'd deleted from a test USB is the most direct demonstration of chapter 1.10 possible, and it's the reason `dd`-wiping and crypto-erase exist as separate practices from "delete".

2. **The timeline is where the story is.** Individual artefacts are weak on their own; the sequence is what constitutes evidence. Same structural insight as SIEM correlation (chapter 5.12) — no single event is the incident, the ordering is.

3. **Hashing at acquisition is the whole basis of admissibility.** Hash the image, re-hash after analysis, prove they match, prove you changed nothing. Without that step the analysis is just an assertion, however competent it was.

4. **Working from the original is a career-ending mistake.** Every action goes against the image. The write blocker and the copy aren't bureaucracy — they're what makes the findings defensible.

---

## Chapter 14 — what I'd take away

* Preparation carries almost all the value. During an incident you use what you already built.
* Isolate, don't power off — RAM holds fileless malware, keys and network state, and it's near the top of the order of volatility.
* Contain too early and you lose evidence; too late and it spreads. Decide who makes that call in advance.
* "The user clicked a link" is never a root cause.
* Lessons learned must be blameless, or people hide information during the next one.
* Out-of-band communication matters — running the response over compromised email hands the attacker your playbook.
* A break in chain of custody can void evidence regardless of what it proves.
* Deleted isn't gone; a backup that's never been restore-tested isn't a backup.
* RTO and RPO drive the alternate-site choice, and that chain traces all the way back to business impact.
