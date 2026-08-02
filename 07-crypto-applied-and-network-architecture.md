# Chapter 7 — Cryptography Applied and Network Architecture

The biggest chapter in the course. First half applies the crypto from chapter 2 to real systems (cryptosystems, block modes, certificates, PKI); second half is the network stack and everything that defends it.

---

# Part A — Applied Cryptography

## 7.1 Data Protection

The three states again, because everything downstream depends on which one you're protecting:

* **At rest** - stored. Disk, file, database encryption.
* **In transit** - moving over a network. TLS, IPsec, VPN.
* **In use** - in memory, being processed. Hardest, usually has to be plaintext.

### Data classification

**Public** → **Private/Internal** → **Sensitive/Confidential** → **Restricted/Critical**. Government: unclassified → classified → secret → top secret.

The purpose of classifying at all is that **you can't protect everything to the same standard**, and pretending otherwise means the important things get protected badly.

### Sovereignty and residency

**Data sovereignty** — data is subject to the laws of the country it physically sits in. Which makes cloud **region selection a legal decision**, not a latency one. GDPR restricts transfers out of the EU; some countries mandate local storage outright. Multinationals hit genuinely conflicting requirements (localisation here, lawful access demands there) and somebody has to make a documented decision about which obligation to breach.

### DLP

**Data Loss Prevention** inspects data in motion, at rest and in use for sensitive patterns (card numbers, national ID formats) and blocks or alerts on exfiltration. Known weaknesses: encrypted channels, steganography, and photographing the screen. So it's a control against carelessness and opportunism, not against a determined insider.

## 7.2 Cryptographic Methods

Quick map of the toolkit before the detail:

| Method | Key(s) | Reversible | Gives you |
|---|---|---|---|
| **Symmetric encryption** | One shared | Yes | Confidentiality |
| **Asymmetric encryption** | Public + private | Yes | Confidentiality, authenticity, non-repudiation |
| **Hashing** | None | **No** | Integrity |
| **HMAC** | One shared | No | Integrity + authenticity |
| **Digital signature** | Private to sign, public to verify | No | Integrity + authenticity + non-repudiation |

**Steganography, tokenization and masking** are the non-encryption protections (covered in 7.3 below).

### Obfuscation techniques

* **Steganography** - hiding the *existence* of the message, not just its content. Data in image pixel low-order bits, in audio, in whitespace, in protocol fields. Mostly relevant as an **exfiltration technique** — the traffic looks like holiday photos and it's your customer database. Hard for DLP to catch.
* **Tokenization** - replacing sensitive data with a substitute that has **no mathematical relationship** to the original, with the mapping held in a separate vault. `4532 1234 5678 9010` → `7a9f2b4c8e1d`. **This is the key difference from encryption:** encrypted data is still mathematically derived from the plaintext, so a stolen key exposes it. A token isn't derived from anything — there is no key that decrypts it. That's why tokenization takes systems **out of PCI-DSS scope** entirely; those systems genuinely never hold card data.
* **Masking** - obscuring part while keeping it usable. `4532 **** **** 9010`. **Static** masking permanently alters a copy (safe non-production datasets); **dynamic** masking leaves data intact and masks at query time based on who's asking.
* **Code obfuscation** - making binaries hard to read. **Not a security control**, a speed bump. Anything the CPU executes, an analyst can eventually read.

## 7.3 Symmetric Cryptosystems

One key for both directions. Fast, and it can't solve key distribution on its own.

### Block vs stream ciphers

* **Block cipher** - operates on fixed-size blocks (AES: 128 bits). Needs **padding** when data doesn't divide evenly.
* **Stream cipher** - operates a bit or byte at a time, XORing plaintext against a generated keystream. Fast, low latency, no padding. **Fatal if the keystream is ever reused** — XOR two ciphertexts encrypted under the same keystream and the keystream cancels out, leaving you with the XOR of two plaintexts, which is very solvable.

### The algorithms

| Cipher | Key size | Block | Status |
|---|---|---|---|
| **DES** | 56-bit | 64-bit | **Broken.** Brute-forceable in hours. |
| **3DES** | 112/168-bit effective | 64-bit | Deprecated. Slow, small block size. |
| **AES** | 128/192/256 | 128-bit | **The standard.** Hardware accelerated (AES-NI). |
| **Blowfish / Twofish** | variable | 64/128 | Fine, largely superseded. |
| **RC4** | variable | stream | **Broken.** Biased keystream. Prohibited in TLS. |
| **ChaCha20** | 256-bit | stream | Modern stream cipher. Excellent where AES-NI is absent (mobile). |

**AES** came from an open international competition where every candidate was publicly attacked for years — that public beating is exactly why it's trusted (Kerckhoffs, chapter 2.1).

Key size note: AES-128 is not "half as strong" as AES-256; it's 2^128 versus 2^256 operations, both far beyond brute force. AES-256 is chosen mainly for long-term and post-quantum margin (Grover's algorithm effectively halves symmetric key strength, making 256 → 128 equivalent, which is still fine).

## 7.4 Symmetric Block Modes

**The mode is as important as the cipher.** AES itself is sound; using it in the wrong mode destroys the security anyway. This is one of the clearest examples of "the algorithm is never the weak point" (chapter 2.3).

### ECB — Electronic Codebook

Each block encrypted independently with the same key.

**Never use it.** Identical plaintext blocks produce identical ciphertext blocks, so **structure in the data survives encryption**. The famous demonstration is encrypting a bitmap image in ECB mode — you can still clearly see the picture, because the flat regions of colour produce repeating identical blocks. The data is "encrypted" and the shape is fully visible.

That's the mode's whole lesson: encryption that leaks patterns isn't confidentiality.

### CBC — Cipher Block Chaining

Each plaintext block is XORed with the *previous* ciphertext block before encryption. The first block uses an **IV (Initialisation Vector)**.

* Identical plaintext blocks now produce different ciphertext — pattern problem solved.
* The **IV must be random and unpredictable**, and unique per message. A predictable IV enables chosen-plaintext attacks (this was BEAST).
* Sequential, so it can't be parallelised for encryption.
* Vulnerable to **padding oracle attacks** if the system reveals whether padding was valid — the attacker decrypts data byte by byte using only that yes/no signal. POODLE and Lucky13 are variants of this.

### CTR — Counter

Turns a block cipher into a stream cipher: encrypt a counter value and XOR the result with the plaintext.

* Parallelisable, random access, no padding needed.
* **The counter/nonce must never repeat under the same key.** Repeat it and you've reused a keystream, with the catastrophic consequence described above.

### GCM — Galois/Counter Mode

CTR mode **plus** an authentication tag. This is **AEAD — Authenticated Encryption with Associated Data**.

This is the mode you want, and the reason is worth stating properly:

**Encryption alone gives confidentiality but not integrity.** With CBC or CTR, an attacker who can't read your ciphertext can still *modify* it — flipping a bit in CTR ciphertext flips the corresponding plaintext bit predictably. The recipient decrypts it happily and acts on tampered data. AEAD modes attach an authentication tag so any modification is detected and decryption **fails loudly** rather than returning corrupted plaintext.

In my password manager I used **AES-256-GCM** for exactly this: a wrong master password or a tampered vault file surfaces as a GCM authentication tag failure, not as garbage plaintext that the application might half-process. And GCM shares CTR's constraint — **a fresh random nonce on every single encryption**, because nonce reuse under GCM doesn't just leak plaintext, it can leak the authentication key itself and let an attacker forge messages. The code comments in `crypto.py` say so explicitly so the constraint survives future edits.

**ChaCha20-Poly1305** is the other standard AEAD construction, preferred where AES hardware acceleration isn't available.

### Summary

| Mode | Parallel | Integrity | Verdict |
|---|---|---|---|
| ECB | Yes | No | **Never** |
| CBC | Decrypt only | No | Legacy; padding oracle risk |
| CTR | Yes | No | Fine, needs a MAC alongside |
| **GCM** | Yes | **Yes** | **Default choice** |

## 7.5 Asymmetric Cryptosystems

Two mathematically linked keys. What one encrypts, only the other decrypts.

* **Encrypt with public → decrypt with private** = confidentiality.
* **Sign with private → verify with public** = authenticity and non-repudiation.

### The algorithms

* **RSA** - based on the difficulty of factoring large semiprimes. 2048-bit minimum today, 3072/4096 for longer-term. Can do both encryption and signatures.
* **ECC (Elliptic Curve)** - based on the elliptic curve discrete logarithm problem. **Much smaller keys for equivalent strength** — 256-bit ECC ≈ 3072-bit RSA. Dominant on mobile and embedded because of size and speed. Curves: P-256, P-384, Curve25519.
* **DSA / ECDSA / EdDSA (Ed25519)** - signature-only algorithms. Ed25519 is the modern default (it's what `ssh-keygen -t ed25519` gives you, and what roundhound generates).
* **Diffie-Hellman / ECDHE** - key *exchange* only, not encryption. Derives a shared secret over a public channel.

### Hybrid encryption

Real systems use both. Asymmetric to authenticate and agree a secret; symmetric for the bulk data. That's TLS, and it's the trade of "expensive tool for the small hard problem, cheap tool for the large easy one".

### Perfect forward secrecy

**ECDHE** generates an ephemeral key pair per session, discarded afterward. If the server's long-term private key is stolen next year, previously recorded traffic **still can't be decrypted**, because those session keys never touched disk. Without PFS, one key compromise retroactively decrypts everything ever recorded — which is exactly the "harvest now, decrypt later" threat model.

### Post-quantum

Shor's algorithm breaks RSA and ECC outright once a sufficiently large quantum computer exists. NIST has standardised replacements (ML-KEM/Kyber for key encapsulation, ML-DSA/Dilithium for signatures). Relevant *now* for anything with a long secrecy lifetime, because the recording is happening today.

## 7.6 Understanding Digital Certificates

### The problem certificates solve

Asymmetric crypto has a hole: **if you hand me a public key claiming to be your bank's, how do I know it's actually the bank's?** Nothing in the mathematics binds a key to an identity. An attacker who substitutes their own public key can MITM the entire conversation and every cryptographic check still passes perfectly.

A **certificate** is the binding: a public key plus identity information, **signed by a trusted third party** that verified the identity.

### X.509 structure

The standard format. Contents:

* **Version**, **Serial number** (unique per CA, used for revocation)
* **Signature algorithm**
* **Issuer** - the CA's distinguished name
* **Validity period** - not before / not after
* **Subject** - who the certificate belongs to
* **Subject Public Key Info** - the actual public key
* **Extensions** - **SAN (Subject Alternative Name)**, Key Usage, Extended Key Usage, Basic Constraints (is this a CA cert?), CRL/OCSP distribution points
* **CA's digital signature** over all of the above

**SAN matters most in practice** — modern browsers ignore the Common Name entirely and validate against SAN. A certificate without the hostname in its SAN list fails, regardless of CN.

### Formats

* **PEM** - base64 text, `-----BEGIN CERTIFICATE-----`. Most common on Linux. `.pem`, `.crt`, `.cer`.
* **DER** - binary. Common on Windows/Java.
* **PKCS#12 / PFX** - bundle of certificate **plus private key**, password-protected. `.p12`, `.pfx`.
* **PKCS#7** - certificate chains, no private key. `.p7b`.
* **CSR** - the signing request. `.csr`.

## 7.7 Trust Models

How trust is established and propagated.

* **Hierarchical (the web PKI model)** - a root CA at the top, intermediate CAs beneath, end-entity certificates below that. Trust flows downward through the **chain of trust**. This is what browsers use.
* **Web of trust** - decentralised, peer-to-peer, no central authority. Individuals sign each other's keys, and you trust a key based on how many people you already trust have signed it. PGP/GPG uses this. Works in small technical communities, doesn't scale to the internet.
* **Bridge trust** - independent hierarchies connected by a bridge CA so separate organisations can trust each other's certificates without merging their PKIs.
* **Hybrid** - combinations of the above.

### Where the trust actually bottoms out

Your OS and browser ship with a **root store** — a preloaded list of root CAs they trust unconditionally. Everything else derives from that.

Worth sitting with for a second: **you are trusting whoever curated that list, and every CA on it.** There are hundreds. Any one of them can technically issue a valid certificate for any domain. A compromised or coerced CA breaks the model for everyone — DigiNotar was compromised in 2011, issued fraudulent Google certificates used to intercept Iranian users' traffic, and the company ceased to exist shortly after.

**Certificate Transparency** is the response: CAs must log every certificate issued to public append-only logs, so a domain owner can detect certificates issued for their domain that they never requested. Doesn't prevent misissuance; makes it visible.

**Certificate pinning** is the other response — an application hardcodes which certificate or CA it expects, refusing anything else. Strong, and operationally brittle (rotate the cert, break the app).

## 7.8 Public Key Infrastructure

The whole system for managing certificates.

### Components

* **Root CA** - top of the hierarchy, self-signed. **Kept offline**, in a physical safe, powered on only for ceremonies. Compromise is catastrophic and unrecoverable.
* **Intermediate/subordinate CA** - signed by the root, does the day-to-day issuing. Exists precisely so the root can stay offline, and so a compromised intermediate can be revoked without burning the root.
* **RA (Registration Authority)** - performs identity verification on the CA's behalf.
* **CRL / OCSP responders** - revocation infrastructure.
* **Certificate repository** - where issued certificates are published.

### Lifecycle

1. Generate a key pair locally. **The private key never leaves your machine.**
2. Create a **CSR** containing the public key and identity details.
3. CA validates identity to the required level.
4. CA signs and issues the certificate.
5. Install and use.
6. **Renew before expiry** — expired certificates are one of the most common self-inflicted outages there is.
7. Revoke if compromised.

### Validation levels

* **DV (Domain Validation)** - prove you control the domain. Automated, free, what Let's Encrypt issues. Proves control of the domain, **nothing about the organisation behind it**.
* **OV (Organization Validation)** - the organisation's identity is verified too.
* **EV (Extended Validation)** - heavy vetting. Browsers no longer give it special visual treatment, which considerably reduced the point.

### Revocation — the genuinely hard problem

A certificate is valid until it expires. So what if the private key is stolen on day 3 of a 90-day certificate?

* **CRL (Certificate Revocation List)** - CA publishes a list of revoked serials. Problems: the list grows large, clients cache it, so revocation propagates slowly.
* **OCSP** - client asks the CA in real time about one specific certificate. Faster, but it's a **privacy leak** (the CA learns every site you visit) and a **hard dependency** — if the OCSP responder is down, do you fail open or closed? Most browsers **fail open**, which substantially weakens the whole mechanism, because an attacker who can block OCSP can use a revoked certificate.
* **OCSP stapling** - the *server* periodically fetches a signed, timestamped OCSP response and staples it into the TLS handshake. Fixes both the privacy leak and the availability problem, because the client never contacts the CA at all.

**The practical answer to revocation being weak: short-lived certificates.** A 90-day (or 7-day) certificate limits exposure by expiry rather than by revocation. Same reasoning as roundhound — rotation puts a ceiling on how long a compromise stays useful, and it works without needing anyone to notice the compromise.

## 7.9 Certificate Types

* **Domain validated / Organization validated / Extended validation** - by vetting level, above.
* **Wildcard** (`*.example.com`) - covers all first-level subdomains. Convenient, and **one stolen private key compromises every subdomain**. Doesn't cover multi-level (`a.b.example.com`).
* **SAN / multi-domain (UCC)** - several specific names on one certificate.
* **Self-signed** - no CA involved. Fine internally where you control the trust store, rejected by browsers publicly because no third party vouches for it.
* **Code signing** - proves software authorship and integrity. Stolen code-signing keys are extremely valuable to attackers, since signed malware bypasses many controls (Stuxnet used stolen certificates).
* **Client certificates** - authenticate the *client* to the server. Mutual TLS.
* **Email (S/MIME)** - signing and encrypting mail.
* **Root / intermediate CA certificates** - carry the Basic Constraints extension marking them as CAs.

## 7.10 Touring Certificates

Practical inspection.

```bash
# view a certificate file
openssl x509 -in cert.pem -text -noout

# inspect a live server's certificate
openssl s_client -connect example.com:443 -servername example.com </dev/null \
  | openssl x509 -text -noout

# see the full chain and handshake details
openssl s_client -connect example.com:443 -showcerts

# check expiry quickly
openssl x509 -in cert.pem -noout -dates

# generate a key + CSR
openssl req -new -newkey rsa:4096 -nodes -keyout server.key -out server.csr

# self-signed cert for testing
openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365

# verify a chain
openssl verify -CAfile ca.pem cert.pem
```

### What to look at

Subject and SAN (does it actually cover the hostname), issuer and the chain up to a trusted root, validity dates, key size and signature algorithm (anything still signing with SHA-1 is a finding), key usage extensions, and CRL/OCSP endpoints.

In a browser: click the padlock → certificate details. Doing this on a few real sites is what makes the abstract chain concrete — you can see the leaf, the intermediate, and the root your OS already trusts.

---

# Part B — Network Architecture

## 7.11 The OSI Model

Seven layers. The value isn't memorisation, it's that **it gives you a place to stand when troubleshooting or classifying an attack** — "is this a layer 2 problem or a layer 7 problem" is genuinely how you narrow things down.

| # | Layer | Unit | Examples | Attacks | Devices |
|---|---|---|---|---|---|
| 7 | **Application** | Data | HTTP, DNS, SMTP, FTP | XSS, SQLi, phishing | WAF, proxy |
| 6 | **Presentation** | Data | TLS, encoding, compression | SSL stripping, downgrade | — |
| 5 | **Session** | Data | Session establishment, RPC | Session hijacking | — |
| 4 | **Transport** | Segment | TCP, UDP | SYN flood, port scanning | Firewall (stateful) |
| 3 | **Network** | Packet | IP, ICMP, routing | IP spoofing, Smurf, route poisoning | Router, L3 switch |
| 2 | **Data Link** | Frame | Ethernet, MAC, ARP, VLANs | **ARP poisoning, MAC flooding, VLAN hopping** | Switch, bridge |
| 1 | **Physical** | Bits | Cables, radio, voltage | Wiretapping, jamming, cutting | Hub, repeater, cable |

Mnemonic that stuck: **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way (1→7).

**TCP/IP model** collapses this to four: Network Access (1–2), Internet (3), Transport (4), Application (5–7).

### Why layer 2 matters disproportionately for security

Layer 2 protocols — ARP especially — were designed for small, trusted, physically-secured networks. **They have no authentication whatsoever.** Anything on the local segment can claim to be anything. That's not a bug in an implementation, it's the design, and it's why the next two sections exist.

## 7.12 ARP Cache Poisoning

### How ARP works

ARP maps IP addresses to MAC addresses on a local segment. Host A wants to reach 192.168.1.1, broadcasts "who has 192.168.1.1?", the owner replies "that's me, my MAC is xx:xx", A caches it.

### The flaw

**ARP has no authentication and hosts accept unsolicited replies.** Nothing verifies the answer. Anyone on the segment can send a gratuitous ARP saying "192.168.1.1 is at *my* MAC" and hosts will believe it and update their cache.

### The attack

Attacker tells the victim "I'm the gateway", and tells the gateway "I'm the victim". Both update their caches, and now all traffic between them flows through the attacker — a **MITM on a switched network**, which is otherwise hard to achieve (chapter 5.8).

From there: passive sniffing, session hijacking, SSL stripping, credential capture, or selective packet modification. It's also how an attacker gets around the fact that a switch only forwards traffic to its destination port.

```bash
# detection: two IPs sharing a MAC, or a gateway MAC that changed
arp -a
ip neigh
```

### Defences

* **Dynamic ARP Inspection (DAI)** on managed switches — validates ARP packets against a trusted DHCP snooping binding table and drops the ones that lie. **The real fix.**
* **Static ARP entries** for critical hosts. Doesn't scale.
* **Port security** limiting MACs per port.
* **Network segmentation** — poisoning only reaches the local broadcast domain, so smaller segments mean smaller blast radius.
* **Encryption everywhere** — MITM on TLS traffic gets ciphertext, and certificate validation fails on interception. This is why the "just click through the certificate warning" habit is genuinely dangerous, and why HSTS exists.

## 7.13 Other Layer 2 Attacks

* **MAC flooding** - flood the switch with fake source MACs until its CAM table fills. Many switches then **fail open** and flood all traffic to every port like a hub, enabling sniffing. Defence: **port security** limiting MAC addresses per port.
* **MAC spoofing** - changing your MAC to impersonate another device, bypassing MAC filtering (which is why MAC filtering is not a security control — it's an inventory convenience).
* **VLAN hopping** - escaping your assigned VLAN, defeating segmentation:
  * **Switch spoofing** — the attacker's port negotiates a trunk link via DTP, gaining access to all VLANs. Defence: disable DTP, explicitly configure access ports.
  * **Double tagging** — two 802.1Q tags; the first switch strips one and forwards to the native VLAN. Defence: change the native VLAN to an unused one and don't use VLAN 1.
* **STP attacks** - claiming to be the root bridge in Spanning Tree, redirecting traffic. Defence: BPDU Guard and Root Guard.
* **DHCP starvation and rogue DHCP** - exhaust the DHCP pool, then serve your own leases pointing clients at an attacker-controlled gateway and DNS. Defence: **DHCP snooping**, which is also what DAI depends on.
* **CDP/LLDP reconnaissance** - discovery protocols leaking device models, versions and topology. Disable on untrusted ports.

The pattern: **layer 2 trusts everything on the segment.** Every defence here is about adding the verification the protocols never had.

## 7.14 Network Architecture Planning

### Security zones

Segments grouped by trust level, with controlled paths between them.

* **Untrusted** - the internet.
* **Screened subnet / DMZ** - internet-facing services (web, mail gateway, reverse proxy). The point is that compromise there is **contained** and doesn't grant internal access. Traffic from DMZ *inward* is heavily restricted.
* **Trusted / internal** - corporate LAN.
* **Restricted** - highest-value assets. Databases, domain controllers, key material.
* **Management network** - out-of-band admin access, separated from data traffic.

The design assumption: **any given zone will eventually be compromised**, so make sure that isn't the whole game.

### Segmentation

* **Physical** - separate hardware. Strongest, least flexible.
* **VLANs** - logical separation on shared hardware. Cheap, and defeated by VLAN hopping if misconfigured.
* **Subnets + ACLs** - routing-level control.
* **Microsegmentation** - per-workload policy, typically software-defined. What zero trust needs.
* **Air gap** - complete physical isolation. Strongest, most painful, and defeated by removable media and humans (Stuxnet).

### Device placement

* **Inline vs tap/SPAN** - inline can **block** but is a failure point and adds latency; a tap only **observes** and can only alert. This is the prevention-versus-detection trade in physical form.
* **Fail-open vs fail-closed** - when the security device dies, does traffic flow (availability preserved, security lost) or stop (security preserved, availability lost)? **There's no universally correct answer, and it must be a deliberate documented decision** rather than whatever the vendor shipped. A hospital and a bank will rightly choose differently.

### Related

* **Jump server / bastion host** - a single hardened, heavily monitored entry point into a sensitive zone. All admin access funnels through it, so there's one place to enforce MFA and one place that logs everything.
* **Out-of-band management** - a separate path for administration that survives the production network being down or compromised.

## 7.15 Network Planning

Practical design considerations.

* **IP addressing and subnetting** - allocate deliberately so segmentation is expressible in ACLs. RFC1918 private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
* **Redundancy** - dual links, dual devices, dual paths. No single point of failure (chapter 1 availability).
* **Capacity planning** - bandwidth, device throughput, session tables. A security device that becomes a bottleneck gets bypassed by someone under pressure.
* **Documentation** - network diagrams, current ones. **A two-year-stale diagram actively misleads responders during an incident**, which is worse than none, and this is why change management (chapter 13) requires updating documentation.
* **IPv6** - frequently enabled by default and unmonitored, creating a parallel network path with no firewall rules. A real and commonly missed gap.
* **Zero trust overlay** - see next.

## 7.16 Zero Trust Network Access (ZTNA) 2.0

### The shift

The old **castle-and-moat** model builds a hard perimeter and broadly trusts everything inside it. That assumption is dead:

* Insider threats bypass the perimeter entirely.
* Cloud means there is no single "inside" — data is in AWS, mail in O365, CRM in Salesforce.
* Remote work puts users outside by default.
* Lateral movement means one phished laptop hands over a flat trusted network.

**Zero trust replaces it with "never trust, always verify."** No trusted network location. Every request is authenticated and authorized **on its own merits**, regardless of origin.

### Architecture

* **Control plane** — where decisions are made. The **Policy Engine** decides yes/no from policy and signals; the **Policy Administrator** issues or revokes the access token. Together they're the **Policy Decision Point (PDP)**.
* **Data plane** — where traffic flows. The **Policy Enforcement Point (PEP)** sits in front of the resource and actually permits or blocks.

The model that made it click: the PEP is the bouncer, the policy engine is the manager on the phone deciding whether you're on the list, and the bouncer calls the manager **every single time** — not once at the start of the night.

### ZTNA 2.0 specifically

Palo Alto's framing, and the improvements over first-generation ZTNA are the substantive part:

1. **Continuous trust verification** - trust is re-evaluated *during* the session, not just at connection. Device posture degrades mid-session → access is revoked mid-session.
2. **Continuous security inspection** - all traffic is inspected for threats throughout, not just authenticated once and then passed through blindly. First-gen ZTNA authenticated the connection and then acted as a dumb tunnel, so malware inside an authorised session went unexamined.
3. **Least privilege at the application layer** - access to *specific applications and functions*, not network-level access to a subnet. Old VPNs gave you an IP on the network and therefore everything reachable from it.
4. **Protects all applications** - including modern apps using dynamic ports, SaaS, and private apps, not just the legacy ones.
5. **Consistent data protection** - one DLP policy across all of it.

### Supporting concepts

* **Adaptive identity** - required verification scales with risk signals.
* **Policy-driven access control** - evaluated live from written policy, not a group membership set in 2019 nobody has reviewed.
* **Threat scope reduction** - the actual payoff, narrowing what any single compromised identity reaches.
* **Microsegmentation** - lateral movement blocked by default.

Zero trust is **least privilege applied to the network, taken to its conclusion.**

## 7.17 Load Balancing

Distributing traffic across multiple servers.

* **Layer 4** - routes on IP and port. Fast, protocol-agnostic.
* **Layer 7** - routes on application content (URL path, headers, cookies). Smarter, can terminate TLS and inspect.

### Algorithms

Round robin · weighted round robin · least connections · least response time · source IP hash (which gives stickiness naturally).

### Persistence / sticky sessions

Keeping a user on the same backend. Needed when session state lives on the server. Better solved by **externalising session state** (shared cache/database) so any backend can serve any request — statelessness scales better and fails better.

### Active/active vs active/passive

* **Active/active** - all nodes serving. Full capacity utilisation, and losing one means the rest absorb its load, so you must have headroom.
* **Active/passive** - standby node idle until failover. Wastes capacity, simpler and more predictable.

### Security relevance

* **Availability** control, which is a third of the triad.
* **DDoS absorption** at some scale.
* **TLS termination** at the balancer — centralises certificate management, and means traffic behind it is plaintext unless you re-encrypt. Whether that internal segment is trusted is a real decision, and under zero trust the answer is no, so you re-encrypt.
* **Health checks** remove failed backends automatically.
* It's also a **single point of failure** unless the balancer itself is redundant.

## 7.18 Securing Network Access

* **NAC (Network Access Control)** - checks device posture *before* granting network access: is AV installed and current, is the OS patched, is disk encryption on. Non-compliant devices go to a **quarantine VLAN** with remediation resources only.
  * **Agent-based** (persistent or dissolvable) vs **agentless**.
  * **Pre-admission** (checked before access) vs **post-admission** (checked continuously after).
* **802.1X** - port-based authentication; the device authenticates before it gets a usable connection at all. Supplicant / authenticator / authentication server (chapter 4.7).
* **Port security** - limit MACs per switch port, shut down or restrict on violation.
* **MAC filtering** - allow-list by MAC. **Trivially defeated by MAC spoofing.** Inventory convenience, not a security control.
* **Guest networks** - fully segregated from corporate, internet-only egress.
* **BYOD considerations** - you have security obligations over hardware you don't own, which limits what you can enforce.

## 7.19 Honeypots

Decoys that exist to waste attacker time and generate high-quality alerts.

* **Honeypot** - a single decoy system, deliberately attractive, no legitimate purpose.
* **Honeynet** - a whole fake network. More convincing, because one lonely server is suspicious while a network with fake traffic between fake machines is much harder to distinguish.
* **Honeyfile** - a decoy file. `passwords_final_v2.xlsx` in a share nobody has a reason to open.
* **Honeytoken** - a fake credential planted so that its *use* is the alarm. A fake AWS key in a config file — the instant anyone authenticates with it you know (a) someone read that file and (b) they're using what they found.

### Interaction levels

* **Low-interaction** - emulates services only. Safe, cheap, easy for a skilled attacker to fingerprint.
* **High-interaction** - real systems. Convincing, far more intelligence, and **it's a real system an attacker could actually take over and pivot from**, so it needs rigorous containment.

### Why the signal quality is the actual point

A normal detection system has a false positive problem — legitimate activity constantly resembles attack activity, and analysts drown.

**A honeypot has no legitimate users.** There is no valid reason for anyone to touch it. So *any* interaction is either an attacker or a serious misconfiguration. **The false positive rate is essentially zero.**

That's a fundamentally different class of alert from "this login looked slightly unusual", and it's why honeytokens in particular are cheap, easy, and disproportionately worth deploying.

**Legal note:** entrapment concerns are largely a red herring for private organisations (entrapment applies to law enforcement inducing crime), but a high-interaction honeypot used to attack a third party creates real liability. Containment isn't optional.

## 7.20 Static and Dynamic Code Analysis

Finding vulnerabilities in software rather than in the network.

* **SAST (Static)** - analyses **source code without running it**. Finds injection patterns, hardcoded secrets, unsafe functions. Runs early and cheaply in the pipeline, covers all code paths including rarely-executed ones. **High false positive rate**, because it lacks runtime context and can't tell whether a path is actually reachable.
* **DAST (Dynamic)** - tests the **running application** from outside, sending malicious input and observing responses. Black box. Finds genuinely exploitable issues with few false positives, and **only in the code paths it manages to reach** — so coverage is the weakness.
* **IAST (Interactive)** - instruments the running application from inside while it's exercised. Combines both perspectives.
* **SCA (Software Composition Analysis)** - scans **dependencies** against known-CVE databases. Given that most of a modern application is third-party code, this often finds more real risk than SAST on your own code. This is the "are we running a vulnerable log4j" question.
* **Fuzzing** - throwing malformed and random input at the application to trigger crashes. Excellent at finding memory-safety bugs (chapter 6.4).
* **Manual code review** - still catches logic and authorisation flaws that no tool understands, because "should this user be allowed to do this" isn't a pattern.

### The complementary point

SAST and DAST find different things and neither replaces the other. **SAST = early, broad, noisy. DAST = late, narrow, accurate.** Plus SCA, because your dependencies are the majority of your attack surface.

Secrets scanning belongs here too — hardcoded credentials in source and in **git history**, which is the specific gap leakhound addresses, since a secret removed in a later commit is still sitting in the object database.

## 7.21 Firewalls

Filter traffic against rules.

### Types by capability

* **Packet filtering (stateless)** - examines each packet independently against ACLs. Fast, dumb, can't tell a legitimate response from an unsolicited packet.
* **Stateful inspection** - tracks connection state, so return traffic for an established connection is permitted automatically. **The baseline expectation.**
* **Application layer / proxy firewall** - understands protocols, inspects payloads, terminates and rebuilds connections.
* **NGFW (Next-Generation Firewall)** - stateful plus application awareness, user identity awareness, integrated IPS, threat intelligence and TLS inspection. Can distinguish "Facebook" from "Facebook chat" rather than just seeing port 443.
* **WAF (Web Application Firewall)** - HTTP-specific, understands SQLi and XSS payloads. Sits in front of web applications (chapter 11).
* **UTM (Unified Threat Management)** - several functions in one appliance. Convenient for small organisations, single point of failure.

### By deployment

Network-based (perimeter or between zones), **host-based** (per machine, matters more under zero trust), cloud (security groups, NACLs, cloud-native firewalls), and virtual.

### Rules

Rules are evaluated **top-down, first match wins** — so order is functionally part of the policy, and a permissive rule near the top silently negates stricter rules below it.

Every rule set should end with **implicit deny**: anything not explicitly permitted is denied. This is what makes the policy safe by construction.

**Egress filtering is the underrated half.** Most organisations write careful inbound rules and permit essentially all outbound. But **C2, exfiltration and reverse shells are all outbound problems** (chapters 5.2, 6.6). Restricting outbound traffic to what's actually needed breaks a large fraction of post-compromise activity.

Rules need periodic review too — firewall rule sets accumulate temporary exceptions that became permanent.

## 7.22 Proxy Servers

An intermediary that makes requests on someone's behalf.

* **Forward proxy** - sits between internal clients and the internet. Used for content filtering, logging, caching, and hiding internal addressing. Clients are configured to use it (or it's transparent).
* **Reverse proxy** - sits in front of servers, facing the internet. Used for load balancing, TLS termination, caching, WAF functionality, and hiding backend topology. Clients don't know it's there.

The direction is the distinction: **forward proxies protect and control clients; reverse proxies protect and front servers.**

### Security value

* Central point for **logging** all outbound web traffic — one place to answer "what did this host talk to".
* **Content inspection and filtering** before content reaches the endpoint.
* **DLP** enforcement on uploads.
* Breaks direct client-to-internet connectivity, which is itself a control.
* Caching reduces bandwidth.

### TLS inspection

To inspect encrypted traffic, the proxy terminates TLS and re-encrypts using an internal CA certificate that clients trust — an **authorised MITM**.

Genuine tension here worth stating plainly: it's the only way to see threats inside encrypted traffic, and it means the proxy **sees everything in plaintext**, including employees' banking and health traffic. That's a privacy and legal question at least as much as a technical one. It also breaks certificate pinning in some applications, so exemption lists are always needed in practice.

## 7.23 Web Filtering

* **URL filtering** - block specific addresses.
* **Category-based filtering** - block by classification (gambling, adult, known malware hosting, newly-registered domains).
* **Reputation-based** - block by scored risk.
* **Content inspection** - examine the actual payload.
* **DNS filtering** - block resolution of malicious domains. **The cheapest high-value control in this list** — a large share of malware can't reach its C2 if the name doesn't resolve, and it applies network-wide with no endpoint agent. Quad9, Cisco Umbrella, Pi-hole.

### Deployment

* **Agent-based** - on the endpoint, so it works off-network too.
* **Centralised proxy** - all traffic through a filtering gateway. Doesn't protect devices off the corporate network.

Blocking newly-registered domains as a category is worth calling out — attacker infrastructure is very often days old, and legitimate business need for a domain registered yesterday is rare.

## 7.24 Network and Port Address Translation

### NAT

Translates private internal addresses to public ones at the network boundary.

* **Static NAT** - one-to-one permanent mapping. Used for internal servers needing a fixed public address.
* **Dynamic NAT** - one-to-one from a pool, assigned as needed.
* **PAT (Port Address Translation) / NAT overload** - **many-to-one**, distinguishing sessions by source port. This is what almost every home and office router does — hundreds of internal devices behind one public IP.

### Why it exists

Primarily **IPv4 address exhaustion**. There aren't enough public addresses, so private ranges get reused everywhere and translated at the edge.

### The security question

NAT provides **incidental** inbound protection: external hosts can't initiate connections to internal devices, because there's no mapping until an internal host starts one.

**But NAT is not a firewall**, and treating it as one is a genuine mistake:

* It does not inspect anything.
* It does nothing about outbound traffic — so C2, exfiltration and reverse shells are entirely unaffected.
* It does nothing about malicious content in permitted responses.
* Once an internal host initiates, the return path is wide open.

It's a side effect that happens to block unsolicited inbound, which is a small subset of what a firewall does.

It also **complicates security monitoring**: many internal hosts share one public IP, so external logs can't attribute activity to a specific device without correlating internal NAT translation logs. Worth keeping those logs for exactly that reason.

**IPv6** removes the address-scarcity reason for NAT entirely, which surprises people — every device can have a public address, and the security work moves to firewalling properly rather than relying on translation.

## 7.25 IP Security (IPsec)

A protocol suite securing traffic at **layer 3**, so it protects *everything* above it transparently — no application changes required.

### Protocols

* **AH (Authentication Header)** - integrity and authentication, **no encryption**. Rarely used alone, and it breaks through NAT because it authenticates the IP header that NAT rewrites.
* **ESP (Encapsulating Security Payload)** - encryption plus integrity and authentication. **This is what's actually used.**
* **IKE (Internet Key Exchange)** - negotiates the security association and handles key exchange. IKEv2 is current.

### Modes

* **Transport mode** - encrypts only the **payload**; original IP header stays visible. Used host-to-host.
* **Tunnel mode** - encrypts the **entire original packet** and wraps it in a new IP header. Used for site-to-site VPNs and gateway-to-gateway, and it hides the internal addressing.

### Security association

An SA is the negotiated agreement (algorithms, keys, lifetime) and it's **unidirectional** — a bidirectional connection needs two.

IPsec is the basis for most site-to-site VPNs and is what "IPsec VPN" means on enterprise gear.

## 7.26 SD-WAN and SASE

**SD-WAN (Software-Defined WAN)** - centrally-managed, policy-driven routing across multiple transport links (MPLS, broadband, LTE/5G). Chooses paths dynamically by application and link quality. Cheaper than MPLS everywhere, and it means branch traffic can go direct to the internet rather than backhauling — which is a security problem, since that traffic now bypasses the datacentre security stack.

**SASE (Secure Access Service Edge)** - converges SD-WAN with **cloud-delivered security** into one service: secure web gateway, CASB, ZTNA, FWaaS, DLP.

### The reasoning

Once your users and your applications are both outside the datacentre, **hauling all traffic back to a datacentre firewall to inspect it stops making sense** — you're adding latency to send traffic to a building neither endpoint is in. SASE moves inspection to points of presence near the user, so security follows the user rather than the office.

**SSE (Security Service Edge)** is the security half of SASE without the SD-WAN networking part.

## 7.27 Virtual Private Networks

An encrypted tunnel across an untrusted network.

### Types

* **Remote access VPN** - individual user to corporate network.
* **Site-to-site VPN** - permanently connecting two networks, typically IPsec tunnel mode.
* **Clientless VPN** - browser-based (SSL/TLS portal), no software install, limited to web apps.

### Full vs split tunnel

* **Full tunnel** - *all* traffic goes through the corporate network. Everything is inspected and policy applies universally, at the cost of bandwidth, latency and concentration.
* **Split tunnel** - corporate traffic through the tunnel, everything else direct. Faster and cheaper, and it means the endpoint is **simultaneously on the corporate network and the open internet** — a bridging risk, since a compromise arriving via the direct path has a live route inward.

### Protocols

* **IPsec** - layer 3, strong, standard for site-to-site.
* **SSL/TLS VPN (OpenVPN)** - runs over TCP 443, so it traverses restrictive firewalls easily.
* **WireGuard** - modern, very small codebase (a few thousand lines versus hundreds of thousands), fast, uses fixed modern cryptography with no negotiation — which removes downgrade attacks by design. Smaller codebase means smaller audit surface, and it's a good example of simplicity as a security property.
* **L2TP/IPsec**, **IKEv2/IPsec** - common on mobile, handles network changes well.
* **PPTP** - **broken, do not use.**

### VPN vs zero trust

Worth being clear about, because this is the current direction of travel: a traditional VPN gives you **an IP address on the network**, and therefore access to everything reachable from it — network-level access. **ZTNA gives per-application access** with continuous verification. That's why ZTNA is steadily replacing VPNs for remote access rather than sitting alongside them.

My own homelab uses Tailscale (WireGuard-based) over a tailnet, which is closer to the ZTNA model — device-level identity and explicit ACLs rather than "you're on the LAN now".

## 7.28 Intrusion Detection and Prevention Systems

* **IDS** - **detects and alerts.** Passive, typically out-of-band on a SPAN port or tap.
* **IPS** - **detects and blocks.** Inline, in the traffic path.

### The trade

An IPS false positive **blocks legitimate business traffic**. That's a real operational cost, and it's why plenty of organisations run some rule sets in detect-only mode and only enable blocking on high-confidence signatures. The choice isn't "IPS is better", it's a risk decision about which failure you'd rather have.

### By scope

* **NIDS/NIPS** - network-based, sees traffic across a segment, blind to encrypted payloads and to anything that doesn't cross its monitoring point.
* **HIDS/HIPS** - host-based, sees what happens *on* the machine including post-decryption, and only for that machine.

### Detection methods

* **Signature-based** - known patterns. No false positives on known threats, completely blind to novel ones. Needs constant updates.
* **Anomaly-based** - deviation from a learned baseline. Can catch novel attacks; generates far more false positives; requires a clean baseline period (and if the network was already compromised during baselining, the compromise becomes "normal").
* **Behaviour-based** - known-bad behaviours rather than known-bad bytes.
* **Heuristic** - rules of thumb and scoring.

Same IOC-versus-TTP trade from chapter 1.3 appearing yet again: signatures are brittle and precise, behaviour is durable and noisy.

### Placement

Outside the firewall sees everything including what gets blocked (noisy, useful for threat intelligence). Inside sees only what got through (actionable). Many deployments do both, for different purposes.

## 7.29 Lab — Snort IDS on Linux

```bash
sudo apt install snort

# check the config parses
sudo snort -T -c /etc/snort/snort.conf

# sniffer mode — just print packets
sudo snort -v -i eth0

# IDS mode with rules, logging alerts
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

### Rule structure

```
alert tcp any any -> $HOME_NET 22 (msg:"SSH connection attempt"; sid:1000001; rev:1;)
```

Broken down:
* `alert` — the **action** (alert / log / pass / drop / reject).
* `tcp` — protocol.
* `any any -> $HOME_NET 22` — source IP/port, direction, destination IP/port.
* `msg:` — what appears in the alert.
* `sid:` — unique rule ID. Local rules use 1000000+.
* `rev:` — revision number.

Custom rules go in `/etc/snort/rules/local.rules`.

### Rules I wrote to see it work

```
# detect a possible port scan — many SYNs from one source
alert tcp any any -> $HOME_NET any (msg:"Possible SYN scan"; flags:S; \
  threshold:type threshold, track by_src, count 20, seconds 10; sid:1000002; rev:1;)

# detect cleartext credentials on FTP
alert tcp any any -> $HOME_NET 21 (msg:"FTP login attempt"; content:"USER"; \
  nocase; sid:1000003; rev:1;)

# detect an ICMP ping sweep
alert icmp any any -> $HOME_NET any (msg:"ICMP ping detected"; \
  itype:8; sid:1000004; rev:1;)
```

Then triggering them from another machine with `nmap -sS`, an FTP login, and `ping` — and watching alerts appear in real time.

### What the lab actually teaches

Two things beyond the syntax:

1. **The `threshold` keyword is the whole art.** Without it, the SYN rule alerts on *every single* SYN packet, which is every legitimate connection on the network — thousands of alerts an hour, instantly useless. With `count 20, seconds 10` it fires on the *pattern* rather than the packet. That gap between "technically detecting" and "producing an actionable alert" is what tuning means, and it's why an untuned SIEM (chapter 5.12) drowns its analysts.

2. **You can only detect what crosses the sensor.** Traffic between two hosts on the same switch never reaches an IDS on a different port, which is exactly the SPAN/TAP placement problem from chapter 5.8. Deciding *where* the sensor sits is a bigger decision than which rules you load.

And the encryption limit applies here too: Snort can see that a TLS connection happened, to where, how big and how often. It cannot see inside it without TLS inspection.

---

## Chapter 7 — what I'd take away (is a lot...)

* The block **mode** matters as much as the cipher. ECB leaks structure; GCM gives you integrity as well as confidentiality, and that's why it's the default.
* Encryption without authentication lets an attacker modify ciphertext predictably. AEAD exists for that reason.
* Certificates solve the binding problem — nothing in the maths says whose key this is.
* Trust bottoms out in the root store your OS shipped with, and any CA on that list can issue for any domain. Certificate Transparency exists because of that.
* Revocation is weak; short-lived certificates are the practical answer.
* Layer 2 protocols have no authentication at all, which is why ARP poisoning works and why DAI and DHCP snooping exist.
* NAT is not a firewall. It blocks unsolicited inbound as a side effect and does nothing about outbound.
* Egress filtering is the consistently underrated control — C2, exfiltration and reverse shells are all outbound.
* IDS/IPS tuning is the actual work. An untuned rule is worse than no rule.
