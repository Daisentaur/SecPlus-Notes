
# 01-Defining Business Risk

## Foundational Definitions

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

# 02- Threat Actors

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

## Telling them apart 

* **Organised Crime v/s APT** - both are similar in regards of being quiet, professional and skilled. they're different because they have different motivations, APTs normally work in the interest of a nations government, while organised crime often motivated by money. also a matter of time invested into targets, APTs spend way more time than organised crime on one target, and just lurk around for ages before making an actual move.
* **The chosen target is also reveals information** - some targets have no monetary value really, but surveying them or shutting them down can cause damage to a countries system. that points to APTs. similarly other targets can reveal other actors.
* **Insiders are specially Dangerous** because they can bypass any perimeter defenses setup by the company with no skills or techniques, direct access to internal networks and assets.


# 03-Threat Intelligence 

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
	 Once an actor is in your machine they'd want to control it remotely, send it commands and pull data from it etc. For this an attacker can own a C2 server(command-and-control) server on the internet. the malware on the machine talks to the C2 server to receive orders.
	 A question arises here, who calls who ? in a real case the malware keeps periodically asking the C2 is they have any instructions for them. Now this periodic asking is the beaconing every 60 minutes in the example statement that makes this behavior from the attacker stand out.

# 04-Risk Management Concepts

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




















