
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

(A bit of a tangent - What i mentioned above "what all it's connected to" is a separate type of condition you need to think of in cases where the vulnerable server might be connected to something else of value even though there isn't anything on it itself.  "what else can this thing reach from here?" is a very valid question to ask and is called thinking about **Lateral Movement** and **Blast Radius**.)

**Threat** - The threat is anyone aware of this vulnerability and using an exploit targeted for it to find anything vulnerable on the network

**Vulnerability** - The initial flaw in the service's old version.

**Risk** - Since we stated Risk = Likelihood × Impact lets look at both the terms. The likelihood of something like this being hit is high. The reason being, since the vulnerability is public knowledge there is a high chance there are threat actors who've made automated scripts that run looking for this kind of an vulnerable server on a network and scrape whatever they can find inside. The impact of this is near zero, since we've stated it just hosts a static site and nothing else the asset getting in someone else hand dose not impact it all that much. so the final verdict, **high likelihood x near zero impact = low risk**.

# 02-Threat Actors






















