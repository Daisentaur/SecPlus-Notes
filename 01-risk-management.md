# Chapter 1 — Risk Management

## 1.1 Defining Business Risk

### Asset
the thing of value we're protecting (the customer database, the server, the reputation)
### Threats
Anything or anyone that can cause harm to our assets. A potential cause of harm.(a ransomware crew, a careless employee, a flood in the data center).
### Vulnerabilities 
A weakness in the asset or its protections (unpatched software, weak password policy).
### Risk
Risk = Likelihood × Impact. the likelihood that a threat exploits a vulnerability, times the impact on the asset.

### Vulnerability v/s Exploit
Now to not confuse the two and use the terms interchangeably it's important to understand the basic level difference between them right now.
A vulnerability is the weakness itself. the error or flaw in the system that already exists.
An exploit on the other hand is the weaponized tool or technique to take advantage of said weakness.

### Assessing an example 

`Your homelab has a server exposed to the internet running an old version of some service with a known critical vulnerability. The server hosts nothing but a static webpage you made for fun — no data, no credentials, nothing connected to your real network.`

Let's analyse all 4 of our terms here and make a judgment of low or high risk.

**Asset** - The asset here is nothing but the data on the static site hosted on the server, and the servers IP. since it's explicitly stated that the server is isolated we don't have to worry about *what all it's connected to.*

(A bit of a tangent - What i mentioned above "what all it's connected to" is a separate type of condition you need to think of in cases where the vulnerable server might be connected to something else of value even though there isn't anything on it itself.  "what else can this thing reach from here?" is a very valid question to ask, _a low-value asset can become high-risk purely because of what it gives access to_ and is called thinking about **Lateral Movement** and **Blast Radius**.)

**Threat** - The threat is anyone aware of this vulnerability and using an exploit targeted for it to find anything vulnerable on the network

**Vulnerability** - The initial flaw in the service's old version.

**Risk** - Since we stated Risk = Likelihood × Impact lets look at both the terms. The likelihood of something like this being hit is high. The reason being, since the vulnerability is public knowledge there is a high chance there are threat actors who've made automated scripts that run looking for this kind of an vulnerable server on a network and scrape whatever they can find inside. The impact of this is near zero, since we've stated it just hosts a static site and nothing else the asset getting in someone else hand dose not impact it all that much. so the final verdict, **high likelihood x near zero impact = low risk**.

## 1.2 Threat Actors

Firstly, Every Threat Actor is a Threat but not every Threat is a Threat Actor.

The main difference between the two is that a Threat actor is a person or a group of people with actual intent to use vulnerabilities and exploits in their targets systems to harm,spy,encrypt etc. their assets.

We can categorize these threat actors, we require to do this based on a few attributes listed below. Because the way you deal with a bored teenager who found a script on the internet is different compared to against a government.

* **Motivation/Intent** - the reason behind why they do something that they do. ex - money, ideology,espionage,revenge,chaos,ego.
* **sophistication/capability** - how skilled is the threat actor? a script-kiddie using someone else's tools v/s state sponsored hackers writing custom zero-days.
* **Resources/Funding** - a hobbyist not using any money, and organization with a budget or a nation-state with pretty much unlimited resources and money.
* **Internal v/s External** - are they outside trying to get in, or already inside (an employee)?

Some common named categories are - 

* **Unskilled attacker** - also known as a script-kiddie. low skill, uses tools others built. motivated by curiosity or ego.
* **Hacktivist** - driven by ideology/politics, no real monetary goals. Defaces and harms things to make a point.
* **Insider threat** - someone inside with legitimate access.Upset or bribed or coerced employee. dangerous because they're already inside and bypass your perimeter defenses entirely.
* **Organised Crime** - professional, financially motivated, well-resourced. ransomware-as-a-service crews are included here. they run it like a business. 
* **Nation-state/APT** - as sophisticated and dangerous a threat can get. the top-dog. Government-backed, effectively unlimited resources, extreme patience, custom tooling.APT stands for Advanced Persistent Threat. The persistent part their truly defines them, they can stay put in a network for years just observing.

### Telling them apart 

* **Organised Crime v/s APT** - both are similar in regards of being quiet, professional and skilled. they're different because they have different motivations, APTs normally work in the interest of a nations government, while organised crime often motivated by money. also a matter of time invested into targets, APTs spend way more time than organised crime on one target, and just lurk around for ages before making an actual move.
* **The chosen target is also reveals information** - some targets have no monetary value really, but surveying them or shutting them down can cause damage to a countries system. that points to APTs. similarly other targets can reveal other actors.
* **Insiders are specially Dangerous** because they can bypass any perimeter defenses setup by the company with no skills or techniques, direct access to internal networks and assets.

## 1.3 Threat Intelligence 

In one line, threat intelligence is information about threats that is actionable. So much data in the world, even about attacks and threats but filtering all that noise to get valuable usable information for yourself by collecting and analyzing the data correctly so you can use it to adjust your defenses accordingly is the point of threat intelligence.

### Common Sources 
* **OSINT (Open Source Intelligence)** - freely and readily available public information. Something anyone can access. Includes things like social media, public government records, security blogs, news, vendor reports etc. They're cheap and abundant but because of that abundance you need to filter noise and verify information always.
* **Closed / proprietary / commercial feeds** - A paid vendor for curated, analyzed intel. ex - CrowdStrike,Mandiant. higher quality but as they are a service they obviously cost money.
* **ISACs ( Information Sharing and Analysis Centers)** -  industry specific sharing groups. Members share threat data with each other because one target can reveal possible next targets and this data can help prevent industry wide damage. Finance, Banks, Healthcare every industry has their own. 
* **The dark web/underground forums** - monitoring platforms where actors actually trade tools, sell data etc. can reveal a lot of information about recent targets etc.
* **Government Sources** - some govt. agencies roll out notices, alerts and indicators.

### Two approaches to threat intelligence 
Now there are two major types of "approaches" threat intelligence systems are based on. Since we want to look for suspicious patterns as a way of filtering noise and finding relevant information the things we focus on change how we work.

* **IOC (indicator of Compromise)** - a specific, concrete artifact that signals an attack. malicious IP address, a file hash of a known malware, suspicious domain name. these are atoms of threat intel, this is what you feed in detection systems to say "if you see this flag that shit immediately" .
* **TTPs(Tactics,Techniques and Procedures)** - this is more of a behavioral observation. Instead of actual items that can been compromised, patterns of how an actor attacks is noticed. example - "something just created a scheduled task _and then_ started beaconing at a fixed interval." we can see two segments here, (a) scheduled task creation - a way to avoid getting logged out of the machine upon reboot and (b) beaconing at fixed intervals - Probably a C2 server (command and control server) that the machine is pinging. the fixed intervals make it recognizable cause actual human pings are random. so even if the IP it pings change or the scheduled task created changes the pattern can be looked for and th attacked can be found even if they ditch their technical infrastructure. This method is based on detecting methodology.

**An IOC is brittle. A TTP is durable.** An actor can change their infra at any moment and continue with their attacks. the actual items they've used doesn't matter as much as why they used it. so flagging just the items and how why and how often puts IOC based systems at a disadvantage of being broken sooner than TTP based systems.

### Diving Into an Example

I'm going to try and dive into a TTPs approach for a particular pattern to show how to actually technically flag methodology.
We'll use the previous example, "something just created a scheduled task _and then_ started beaconing at a fixed interval."

In this threat pattern there are two things

(a) Persistence -- "created a scheduled task" 
	If you've broken into the machine, from the attackers POV you want to stay there. your connection initially is fragile a simple reboot or power off might kick you off the machine and you'll have to break in all over again. to avoid this a "run on start up" type program can be made by the actor to regain connection to machine. This is persistence, they want to stay their.
	 How to actually checked for a "run on startup" program that the actor might've added to the system ?
	 1. **Point-in-time comparison** - take a know good baseline of what scheduled tasks should exist on the machine. and periodically just check the current list against that list, and flag new items in that list.
	 2. **Event based detection** - the OS _itself_ generates a log event the instant a scheduled task is created. so instead of checking against a list periodically just subscribe to a change in that log, so any newly created scheduled task can be flagged immediately no delay in flagging due to when the checks happen.

	  
(b) Communication to a C2 -- "Beaconing to its command server every 60 minutes" 
	 Once an actor is in your machine they'd want to control it remotely, send it commands and pull data from it etc. For this an actor can own a C2 server(command-and-control) server on the internet. the malware on the machine talks to the C2 server to receive orders.
	 A question arises here, who calls who ? in a real case the malware keeps periodically asking the C2 is they have any instructions for them. Now this periodic asking is the beaconing every 60 minutes in the example statement that makes this behavior from the attacker stand out.

## 1.4 Risk Management Concepts

Once a risk is assessed it a decision needs to be made about what is to be done about said risk. This decision largely depends on the size of the risk which we can understand using the risk = likelihood x impact.

This risk assessment is an important step before this decision. There are four foundational answers depending on different risk size's and variation in company ideology.

* **Mitigate** - reduce likelihood/impact. 
* **Transfer** - shift the consequence to someone else. like insurance or external vendors
* **Avoid** - stop doing the risky activity entirely. stop flying if you're worried about plane crashes type.
* **Accept** - consciously,formally choose to live with it. understand the impact of it and choose not to do anything about it.

How this can vary by risk size.
* **Tiny risk that's expensive to fix?** accept it
* **Catastrophic risk you can't reduce?** avoid it entirely
* **Moderate risk with cheap fix?** mitigate

This isn't a sure shot list to follow just generally how things end up being looked at. A human brain needs to assess the risk and decide what to do out of these 4 always whenever something comes up. 

### Two things worth remembering 

Since this is a short chapter, other than the 4 actual terms and what they mean two more things that should be kept in mind.

1. Acceptance of a situation should always be **deliberate,documented,authorized decision. it should not come from forgetting about it**. 
2. in **Transfer** the **only the financial impact is moved** to someone else. the likelihood and the rest of the impact stays the same. insurance does not mean security. 
3. 3. When you mitigate a risk, you almost never reduce it to zero. **Whatever's left over after your controls is called residual risk**,risk that remains after you've done the mitigation. 

## 1.5 Security Controls

A smaller topic about the categorization of how security is implemented after it is decided that it must be. There are 2 axis that are followed to make a 2-axis framework of _category_ (what kind of thing) + _type_ (what it does/when)  

### First axis 
* **Technical** - also called logical. implemented in actual tech. MFA, audit logs, antivirus, firewalls, encryption etc.
* **Managerial/Administrative** - policies, procedures, risk assessments , security training. These are rules and human processes that can be bettered by making sure people in the company know safe practices.
* **Operational** - controls executed by people in the day to day. guards doing rounds, analyst reviewing audit logs etc.
* **Physical** - real world barriers. locks, doors, badge readers, cameras etc.

### Second Axis 
more timing related to an attack

* **Preventive** - stops the bad thing before it happens. 
* **Detective** - notices bad thing while or after it happens, doesn't prevent it.
* **Corrective** - fixes thing after the bad thing happened.


some more second axis that aren't that 
- **Deterrent** - discourages an attacker from trying (visible cameras, "guard dog" signage, warning banners).
- **Compensating** - a stopgap when you can't deploy the ideal control (e.g., extra monitoring on a legacy system you can't patch).
- **Directive** - tells people what to do (a policy saying "thou shalt encrypt").

One thing to note, all security control measures fall on both these axes, examples -

- A security-awareness training session. = **managerial + preventive**
- A backup you restore from after ransomware encrypts your files. = **technical + corrective**
- A CCTV camera in the server room. = **physical + detective**
- Encryption of data at rest. = **technical + preventive**
- A security guard checking IDs at the door = **operational + preventive**

and companies stack many measures like these together as  normal security practice, nothing is stand alone. Since if it did, one break in the shield would mean direct access to everything. so many control measures varied across the axes are important.

## 1.6 Risk Assessments and Treatments

The previous sections covered what risk *is* and what the four responses are. This is the process wrapped around them.

### When assessments happen

* **Ad hoc** - done because something triggered it. A new threat surfaced, a vendor got breached, someone asked.
* **Recurring** - on a schedule. Quarterly, annually. The point of scheduling it is that it happens even when nobody feels like it.
* **One-time** - for a specific event. Evaluating a company before acquiring them, or assessing a new product before launch.
* **Continuous** - always running, automated. Cloud posture tools do this.

The trade here is the usual one. Ad hoc is responsive but has gaps. Recurring is thorough but it's a snapshot that goes stale the day after. Continuous catches drift and is expensive to build.

### Risk appetite and risk tolerance

Two terms that sound identical and aren't.

* **Risk appetite** - how much risk the organisation is *willing to take on purpose*, in pursuit of its goals. Described as **expansionary** (aggressive, will accept a lot to grow fast — startups), **conservative** (avoids risk, moves slowly — banks, hospitals), or **neutral**.
* **Risk tolerance** - the acceptable *variation* around that appetite. How far past your stated appetite you'll let something drift before someone has to act.

Appetite is the target, tolerance is the allowed wobble. And crucially **this is a business decision, not a security decision.** Security's job is to state the risk accurately. The board decides how much of it to swallow. That distinction matters because a security team that unilaterally decides the appetite has overstepped, and one that never states the risk clearly has failed.

### The risk register

The central document where every identified risk lives. Not a formality — it's the thing that makes risk management something other than vibes. It typically holds -

* A description of the risk.
* The **risk owner** — a named human, not a team. Unowned risks don't get treated.
* Likelihood and impact ratings.
* The **inherent risk** (before controls), the chosen treatment, and the **residual risk** (after controls).
* **KRIs (Key Risk Indicators)** — measurable signals that tell you the risk is growing. Rising failed login volumes as a KRI for credential attacks.
* **Risk threshold** — the level at which it has to be escalated.
* Status and review date.

The register also solves an organisational problem more than a technical one — it makes accepted risk **visible**. A risk that was consciously accepted by a named executive two years ago is very different from one everyone quietly forgot about, and without a register those two look identical.

### Treatments

The four from 1.4 — mitigate, transfer, avoid, accept — with the additions worth knowing:

* **Risk reporting** - communicating the register upward in language the board acts on. Not CVSS scores; money and business impact.
* **Residual risk** must be formally accepted by someone with authority to accept it. If nobody signs, the risk is unowned.
* **Exception with an expiry date.** If a risk is accepted because fixing it isn't feasible right now, the acceptance should have a review date attached. Otherwise "temporarily accepted" quietly becomes permanent, which is how legacy systems accumulate.

## 1.7 Quantitative Risk Assessments

Putting actual money on risk. Slower and data-hungry, and it produces numbers that a CFO can act on, which qualitative ratings can't.

### The formulas

* **AV (Asset Value)** — what the asset is worth.
* **EF (Exposure Factor)** — the percentage of that value lost if the event happens. Expressed as a decimal.
* **SLE (Single Loss Expectancy) = AV × EF** — the cost of one occurrence.
* **ARO (Annualized Rate of Occurrence)** — how many times per year you expect it. Once every 20 years = 0.05.
* **ALE (Annualized Loss Expectancy) = SLE × ARO** — the expected cost per year.

### Working an example

`A server worth ₹10,00,000. A fire would destroy about half its value. Fires in this building happen roughly once every 20 years. A suppression system costs ₹40,000/year and would cut the exposure factor to 10%.`

* AV = ₹10,00,000
* EF = 0.5
* **SLE** = 10,00,000 × 0.5 = **₹5,00,000**
* ARO = 1/20 = 0.05
* **ALE** = 5,00,000 × 0.05 = **₹25,000 per year**

Now the control. With suppression, EF drops to 0.1, so SLE becomes ₹1,00,000 and ALE becomes ₹5,000/year.

* Risk reduced: 25,000 − 5,000 = **₹20,000/year of benefit**
* Control cost: **₹40,000/year**
* **Net: −₹20,000. The control loses money.**

So on pure numbers you don't buy it. And this is exactly where quantitative assessment earns its place — it lets you say "no" to a control with a defensible reason, and equally it lets you say "yes" and *prove* it when the numbers run the other way.

### Where it breaks down

Being honest about the limitations, because the formulas look more authoritative than they are -

1. **The inputs are guesses.** ARO especially. How do you calculate the annual rate of occurrence for "nation-state actor targets us"? You don't. You make it up and then the output inherits your made-up number wearing a suit.
2. **Intangibles don't fit.** Reputational damage, loss of customer trust, regulatory attention, staff morale. These are real costs that resist being priced, so they get left out, and anything left out of the model gets treated as zero.
3. **It's built on averages.** ALE says ₹25,000/year. It does *not* say you'll lose ₹25,000 this year. You'll lose ₹0 for nineteen years and ₹5,00,000 once, and if that once lands in a bad quarter the average was cold comfort.

The genuinely useful thing about the exercise isn't the final number, it's that it forces you to **state your assumptions explicitly** so someone else can argue with them.

## 1.8 Qualitative Risk Assessments

Descriptive ratings instead of currency. High/medium/low, or 1–5 scales, usually plotted on a matrix.

### The matrix

Likelihood on one axis, impact on the other, typically 5×5. Each risk gets placed, and where it lands determines the priority — the top-right corner (high likelihood, high impact) gets attention first, the bottom-left gets accepted.

|  | Negligible | Minor | Moderate | Major | Severe |
|---|---|---|---|---|---|
| **Almost certain** | Medium | High | High | Critical | Critical |
| **Likely** | Low | Medium | High | High | Critical |
| **Possible** | Low | Medium | Medium | High | High |
| **Unlikely** | Low | Low | Medium | Medium | High |
| **Rare** | Low | Low | Low | Medium | Medium |

### Why it's used more than quantitative

Faster, needs no financial data, and works when you genuinely can't price something. Most real-world risk registers are qualitative for these reasons alone.

### The weaknesses

* **Subjective.** My "moderate" and your "moderate" aren't the same, and neither of us can prove it. Partly fixed by writing definitions for each rating level ("Major = >₹50 lakh loss OR >24hr outage OR regulatory notification required") so people are at least calibrated to the same scale.
* **You can't do arithmetic on it.** "High" minus a control isn't a number. You can't sum your risks, can't compare against a control's cost, can't show a trend properly.
* **Everything drifts to the middle.** People avoid the extremes, so a 5×5 matrix ends up with most things clustered in the middle rows and no useful prioritisation. Some organisations use a 4×4 with no middle option specifically to force a decision.

**In practice both are used together** — qualitative to triage the whole register quickly, then quantitative on the handful of top risks where a real spending decision has to be justified.

## 1.9 Security and the Information Life Cycle

Data has a lifespan, and the controls that protect it change at each stage. Thinking about it as a life cycle is what stops you protecting data beautifully in production and leaving it in plaintext in a backup nobody thought about.

### The stages

1. **Creation / collection** — data comes into existence or is gathered. Security starts here: was collection lawful, was consent obtained, and — the question people skip — **did we need to collect this at all?** Data minimisation is the cheapest control that exists, because data you never collected cannot leak.
2. **Classification** — labelling it by sensitivity so downstream controls know how to treat it. Ideally applied at creation, because retrofitting classification across an existing estate is miserable.
3. **Storage** — at rest. Encryption, access control, backups.
4. **Use / processing** — in memory, being operated on. The hardest state to protect since it usually has to be plaintext to be useful.
5. **Sharing / transmission** — in transit. TLS, and the governance question of who it's being shared with and under what agreement.
6. **Archival** — moved to long-term storage. Still needs protecting, and this is where things quietly rot: the archive is on old media, in an old format, encrypted with a key nobody can find anymore.
7. **Destruction** — securely disposed of at end of life. Covered next.

### Data classification levels

Commercial: **Public** → **Private/Internal** → **Sensitive/Confidential** → **Restricted/Critical**.

Government: **Unclassified** → **Classified** → **Secret** → **Top Secret**.

The point of classifying at all is that **you cannot protect everything to the same standard**, and pretending you can means you protect the important things badly. Classification is how you decide where the budget goes.

### Roles

* **Data owner** — accountable, senior, usually a business figure. Sets classification and approves access.
* **Data controller** — determines the purpose and means of processing (GDPR term).
* **Data processor** — processes on the controller's behalf. Usually a vendor.
* **Data custodian / steward** — implements and runs the controls day to day. Usually IT.
* **Data Protection Officer (DPO)** — accountable for privacy compliance.

The legally important bit: **the controller holds the obligation even when a processor does the work.** Outsourcing the processing does not outsource the accountability — which is the same lesson as "transfer only moves financial impact" from 1.4.

### Retention

Keep data as long as required and **not one day longer**. Over-retention is a genuine liability, not a neutral default:

* Data you still hold can still be breached.
* It's discoverable in litigation.
* It's in scope for deletion requests you now have to honour.
* Regulators treat unnecessary retention as a failure by itself.

The counterweight is **legal hold**, which overrides retention schedules and freezes deletion when litigation is anticipated. Deleting under legal hold is spoliation and carries consequences separate from whatever the case was about.

## 1.10 Data Destruction

End of the life cycle. The goal is that data is unrecoverable — and "unrecoverable" is a higher bar than most people's instinct.

### Why deleting doesn't delete

Deleting a file removes the pointer in the filesystem index. **The blocks holding the data are untouched**, just marked as free. Until something overwrites them the data is fully recoverable with tools anyone can download. Formatting a drive largely does the same thing — rewrites the index, leaves the data.

This is the same reasoning as git objects in leakhound: removing the *reference* to data is not removing the data.

### Methods

* **Overwriting / wiping** — write new data over every block. One pass of zeros is sufficient on modern magnetic drives; the old multi-pass Gutmann patterns were for encoding schemes that don't exist anymore. Works on HDDs.
* **Degaussing** — a strong magnetic field scrambles the platters. Destroys the drive permanently as a side effect (it wipes servo tracks too). **Useless on SSDs and flash**, which store charge, not magnetism.
* **Shredding / pulverizing** — physically destroying the media into fragments. The particle size needs to be small enough that a chip can't be read; for SSDs the required size is much smaller than for HDDs, since a single surviving flash chip holds recoverable data.
* **Incineration** — burning it.
* **Crypto-erase / crypto-shredding** — the data was encrypted at rest, so you destroy the key and the ciphertext becomes permanently meaningless. Instant, and it's the practical answer for things you physically cannot reach — cloud storage, SSDs with wear levelling, and data spread across backup sets.
* **Third-party disposal** — outsourcing it, which introduces the vendor risk you were trying to avoid, so you demand a **certificate of destruction** as evidence.

### The SSD problem

Worth its own note because it catches people. SSDs use **wear levelling** — the controller deliberately spreads writes across cells to even out wear, and it maintains spare capacity you cannot address. So when you overwrite a file, the controller may write to entirely different physical cells and leave the originals holding your data in an area the OS cannot reach.

**Overwriting is therefore unreliable on SSDs.** The options that actually work are the drive's own **ATA Secure Erase** command, **crypto-erase**, or physical destruction. This is the single strongest practical argument for encrypting drives from day one — it makes disposal a solved problem later.

### Certificate of destruction

Documented evidence that destruction happened: what was destroyed, serial numbers, method, date, who did it. Needed for compliance, and needed for your own protection when a regulator later asks where a specific drive went.

## 1.11 Lab — Wiping Disks with `dd`

The course lab. `dd` is the low-level copy tool that reads and writes raw blocks, ignoring the filesystem entirely — which is exactly what wiping requires.

```bash
# identify the target device first — get this wrong and you destroy the wrong disk
lsblk

# overwrite the whole device with zeros
sudo dd if=/dev/zero of=/dev/sdX bs=4M status=progress

# overwrite with random data instead (slower; matters less than people think)
sudo dd if=/dev/urandom of=/dev/sdX bs=4M status=progress
```

* `if=` input file — `/dev/zero` gives an endless stream of null bytes, `/dev/urandom` gives random bytes.
* `of=` output file — the **device**, not a partition, if you want the whole disk.
* `bs=4M` block size, much faster than the tiny default.
* `status=progress` so it tells you what it's doing instead of sitting silent for an hour.

**The obvious warning:** `dd` is sometimes called "disk destroyer" for a reason. It does exactly what you tell it with no confirmation, no undo. Getting `/dev/sda` when you meant `/dev/sdb` wipes your operating system. Check `lsblk` twice.

**And the caveat that matters:** all of this applies to spinning disks. On an SSD, `dd` writes through the wear-levelling layer and cannot guarantee it reached every physical cell. Use `hdparm --security-erase` or crypto-erase there instead.

---

## Chapter 1 — what I'd take away

* Risk = likelihood × impact, and the impact half is where the intuitive answer is usually wrong.
* Every threat actor is a threat; not every threat is a threat actor. Categorise by motivation, capability, resources, and internal/external.
* IOCs are brittle, TTPs are durable. Detect methodology, not artefacts.
* Four treatments: mitigate, transfer, avoid, accept. Acceptance must be deliberate, documented and owned.
* Quantitative gives you numbers to argue with; qualitative gives you speed. The formulas are only as good as the ARO you invented.
* Deleting isn't destroying, and on SSDs overwriting isn't destroying either.
