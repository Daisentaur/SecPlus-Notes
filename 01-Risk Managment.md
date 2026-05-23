
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

### Assessing a an example 

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





















