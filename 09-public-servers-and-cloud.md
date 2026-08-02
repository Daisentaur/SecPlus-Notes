# Chapter 9 — Securing Public Servers and the Cloud

## 9.1 Defending a Public Server

A server exposed to the internet is fundamentally different from an internal one: **it is under continuous automated attack from the moment it gets an address.** Not targeted — just constant background scanning by botnets looking for anything vulnerable. Standing up a fresh server and watching the SSH logs fill with login attempts within minutes is the fastest way to internalise this.

### The layered approach

1. **Minimise attack surface** — run only what's needed. Every listening service is something that can be attacked, and a service that isn't running can never be exploited by any vulnerability discovered in it in future.
2. **Patch aggressively** — public servers are the least forgiving place to be behind.
3. **Harden the configuration** — CIS benchmark, no defaults, no verbose errors.
4. **Authenticate strongly** — keys not passwords for SSH, MFA everywhere else.
5. **Place it correctly** — in a DMZ/screened subnet, so compromise doesn't grant internal access.
6. **Filter both directions** — inbound to only the needed ports, and **outbound**, because that's where C2 and exfiltration live.
7. **Monitor** — ship logs off-host in real time, because an attacker with root deletes local logs.
8. **Assume compromise** — plan for it, don't just try to prevent it.

The mindset that matters: **a public server should be treated as semi-trusted at best.** Design so that its full compromise is survivable rather than catastrophic.

## 9.2 Common Attacks and Mitigations

| Attack | Mitigation |
|---|---|
| Brute force on SSH/RDP | Key-based auth, disable password auth, fail2ban, non-standard port (noise reduction only), VPN/bastion in front |
| Exploitation of unpatched services | Patch management, minimise services, vulnerability scanning |
| Web application attacks (SQLi, XSS) | Parameterised queries, output encoding, WAF, input validation |
| DDoS | CDN, upstream scrubbing, rate limiting, autoscaling |
| Credential theft | MFA, least privilege, no shared accounts |
| Malicious file upload | Type and content validation, store outside the web root, never execute uploads, scan them |
| Directory traversal | Canonicalise paths, reject `../`, chroot/containerise |
| Information disclosure | Suppress verbose errors and version banners, remove default pages |
| Man in the middle | TLS everywhere, HSTS, valid certificates |
| Privilege escalation | Least privilege, run services as unprivileged users, patch the kernel |

### Fail2ban

Worth its own mention as the practical answer to automated brute force:

```bash
sudo apt install fail2ban
# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 3
bantime = 3600
findtime = 600
```

It watches logs for failure patterns and adds firewall rules to block the source. Cheap and effective against the automated background noise. It does **not** stop a targeted attacker with a botnet of source addresses, and it introduces a small DoS risk of its own if someone can forge failures from an address you need.

### Defence in depth in practice

No single control holds. A public web server realistically wants: cloud firewall/security group → CDN/WAF → reverse proxy → hardened OS with host firewall → application running as an unprivileged user → database on a separate host reachable only from the app tier. Each layer assumes the one in front of it will eventually fail.

## 9.3 DDoS Attacks in the Real World

### Categories

* **Volumetric** - saturate the bandwidth. Measured in Gbps/Tbps. UDP floods, ICMP floods, **amplification**.
* **Protocol** - exhaust connection state. Measured in packets per second. SYN floods, Ping of Death, fragmented packet attacks.
* **Application layer (L7)** - exhaust server resources with expensive-looking legitimate requests. Measured in requests per second. HTTP floods, Slowloris. **The hardest to filter**, because each request looks valid — you can't drop it on shape alone, you have to understand intent.

### Amplification and reflection

The technique behind the largest attacks. Send a **small spoofed request** (source address = victim) to a third-party service that replies with something much larger, and the reply goes to the victim.

* **DNS** - ~50× amplification.
* **NTP** monlist - ~500×.
* **memcached** - ~50,000×. Drove the 1.7 Tbps attacks in 2018.

The attacker spends a trickle of bandwidth and the victim receives a flood. **This is only possible because of IP source address spoofing**, which is why **BCP38 / ingress filtering** by ISPs (dropping packets with source addresses that couldn't legitimately come from that network) is the structural fix. Adoption is incomplete, which is why the problem persists.

**Smurf attack** is the classic ICMP version — spoof the victim's address, ping a broadcast address, every host on that network replies to the victim.

### Real-world cases worth knowing

* **Dyn (2016)** - Mirai botnet of compromised IoT devices (default credentials, chapter 6.2) took down a major DNS provider, and with it Twitter, Netflix, Reddit and GitHub. **The lesson: attacking a shared dependency takes down everyone who depends on it**, without attacking any of them directly.
* **GitHub (2018)** - 1.35 Tbps via memcached amplification, mitigated in under 10 minutes by rerouting through a scrubbing service.
* **AWS (2020)** - 2.3 Tbps, CLDAP reflection.

### Mitigation

* **CDN / anycast** - distribute traffic across many points of presence so no single one is saturated. The most effective volumetric defence available to most organisations.
* **Scrubbing services** - reroute traffic through a provider that filters and forwards only clean traffic (Cloudflare, Akamai, AWS Shield).
* **Rate limiting** per source and per endpoint.
* **SYN cookies** for SYN floods — don't allocate state until the handshake completes.
* **Autoscaling** to absorb load. **Watch the bill**, though — "economic denial of sustainability" is a real outcome where you stay up and the invoice does the damage instead.
* **Blackholing** as a last resort: drop *all* traffic to the targeted address. You complete the attacker's goal to protect everything else. Sometimes the right call.
* **Have a plan and the provider contacts in advance.** Sorting out DDoS response during an attack is too late.

## 9.4 Hypervisors and Virtual Machines

### Types

* **Type 1 (bare metal)** - runs directly on hardware. ESXi, Hyper-V, KVM, Xen. Used in datacentres. **Smaller attack surface**, since there's no general-purpose host OS underneath.
* **Type 2 (hosted)** - runs on top of a normal OS. VirtualBox, VMware Workstation. Used on desktops. **Inherits every vulnerability of the host OS** beneath it.

### Security properties

* **Isolation** - each VM has its own kernel and virtualised hardware. Strong separation, which is what makes multi-tenant cloud possible at all.
* **Snapshots** - point-in-time state. Excellent for recovery and for malware analysis (detonate, observe, revert). Security caveat: **reverting a snapshot can undo patches and re-expose fixed vulnerabilities**, and snapshots contain memory, so they may hold secrets.
* **Templates and golden images** - consistent, hardened builds. A vulnerability baked into the template propagates to everything built from it.

### Threats

* **VM escape** - breaking out of a guest into the hypervisor or another guest. **Always critical**, because it breaks the isolation the entire multi-tenant model depends on. Rare and extremely valuable.
* **Resource reuse** - memory or storage handed to a new VM still containing the previous tenant's data. Cloud providers zero memory between allocations for exactly this reason.
* **VM sprawl** - unmanaged VMs nobody is tracking or patching. This is asset management (chapter 1) failing at scale, and it's the most common real problem here by a wide margin — far more damage comes from forgotten VMs than from escapes.
* **Hypervisor vulnerabilities** - patch the hypervisor, which people forget because it's "infrastructure".
* **Side channels between tenants** - Spectre/Meltdown-class issues let one VM infer data from another via shared CPU cache.

## 9.5 Containers and Software-Defined Networking

### Containers

Containers **share the host kernel** rather than virtualising hardware. That's why they start in milliseconds and use a fraction of the resources — and why their isolation is weaker than a VM's.

**The security consequence stated plainly: a kernel vulnerability is a container escape.** A VM escape needs a hypervisor bug; a container escape needs a kernel bug, and there are far more of those.

Isolation comes from kernel features, not from hardware:
* **Namespaces** - separate views of PIDs, network, mounts, users, hostname.
* **cgroups** - resource limits.
* **Capabilities** - fine-grained privilege instead of all-or-nothing root.
* **Seccomp** - restricting available syscalls.
* **AppArmor / SELinux** - mandatory access control on the container.

### Container security work

* **Image provenance** - where did this base image come from? Pull from trusted registries, pin digests not tags, sign images.
* **Image scanning** - scan layers for vulnerable packages before deploy and continuously after, since new CVEs appear against images you already shipped.
* **Don't run as root inside the container.** Default in many images, and it removes a layer of defence for free if you fix it.
* **Read-only root filesystem** where possible.
* **Drop capabilities**, avoid `--privileged` (which effectively disables the isolation entirely).
* **Never mount the Docker socket** into a container — it's equivalent to giving it root on the host.
* **Secrets management** - environment variables are visible via `/proc` and in `docker inspect`. Use a secrets manager or mounted tmpfs.
* **Minimal base images** - distroless or Alpine. Less installed means less to exploit and fewer CVEs to triage.
* **The layer problem** - a secret added in one layer and deleted in a later one **is still in the image**. Anyone pulling it can extract it. This is precisely the same lesson as git history and leakhound: removing the reference is not removing the data.

### Orchestration

Kubernetes adds its own surface: RBAC, network policies (default is **all pods can talk to all pods**, which is the opposite of segmentation), secrets (base64-encoded, not encrypted, unless you enable encryption at rest), admission controllers, and the API server as a very high-value target.

### SDN

**Software-Defined Networking** separates the **control plane** (routing decisions) from the **data plane** (forwarding), with centralised programmable control.

Note this is the same architectural split as zero trust (chapter 7.16) — decision-making centralised and separated from enforcement.

Security benefits: policy defined centrally and applied consistently, **microsegmentation at scale**, rapid isolation of a compromised workload, and network config as code (versionable, reviewable, testable).

Security risks: **the controller is a single high-value target** — compromise it and you control the entire network. Plus a programmable network means a misconfiguration propagates everywhere instantly.

## 9.6 Cloud Deployment Models

* **Public cloud** - shared multi-tenant infrastructure from a provider. Cheap, elastic, and you accept the provider's controls and multi-tenancy.
* **Private cloud** - dedicated to one organisation, on-prem or hosted. Control and compliance, at capital cost and without elastic scale.
* **Hybrid cloud** - both, integrated. The common enterprise reality, and the messiest security case: **two security models, and the seam between them is where things get missed.** Identity federation, consistent policy and encrypted interconnects all have to be built deliberately.
* **Community cloud** - shared by organisations with common requirements (healthcare consortium, government agencies). Shared cost, aligned compliance.
* **Multi-cloud** - multiple providers. Avoids lock-in and improves resilience, at the cost of needing expertise in every provider's distinct security model — which in practice often *reduces* security because nobody knows all of them well.

### Related concepts

* **Cloud service provider (CSP)** and **MSP/MSSP** (managed service providers) — outsourcing operations, which is third-party risk (chapter 13).
* **Edge computing** - processing near the data source, for latency and bandwidth. Distributed physical infrastructure means weaker physical security by nature.
* **Fog computing** - an intermediate layer between edge and cloud.

## 9.7 Cloud Service Models

* **IaaS (Infrastructure as a Service)** - virtual machines, storage, networking. AWS EC2, Azure VMs. You manage OS and up.
* **PaaS (Platform as a Service)** - managed runtime. Heroku, App Engine, Elastic Beanstalk. You manage the application and data.
* **SaaS (Software as a Service)** - complete application. O365, Salesforce, Workday. You manage configuration and data.
* **FaaS / Serverless** - functions executed on demand. Lambda, Cloud Functions. No servers to manage; your surface becomes function code, permissions, and triggers. **Over-permissive function roles are the common failure.**
* **XaaS** - anything as a service.

### The shared responsibility model

**The single most important concept in cloud security**, and the source of most cloud breaches when it's misunderstood.

**The provider secures *the* cloud. You secure what you put *in* it.**

| Layer | On-prem | IaaS | PaaS | SaaS |
|---|---|---|---|---|
| Data & access | **You** | **You** | **You** | **You** |
| Application | You | You | You | Provider |
| Runtime / middleware | You | You | Provider | Provider |
| OS | You | **You** | Provider | Provider |
| Virtualization | You | Provider | Provider | Provider |
| Hardware / physical | You | Provider | Provider | Provider |

**The rule that holds at every level: your data, your identities and your configuration are always yours.**

AWS will never be at fault for your public S3 bucket. That's not a technicality — misconfiguration is the leading cause of cloud incidents, and it sits squarely in the customer column at every service model. "It's SaaS, they secure it" is false: every over-shared file and over-privileged account is yours.

## 9.8 Securing the Cloud

### Identity

**Identity is the new perimeter in cloud.** There's no network edge to defend, so IAM *is* the control plane.

* **Least privilege on IAM roles and policies.** The most common and most damaging cloud misconfiguration.
* **No long-lived access keys** where a role can be assumed instead. Use instance/workload identity.
* **MFA on all human accounts**, hardware keys for privileged ones.
* **Never use the root account** for daily work; lock it away with MFA.
* **CIEM (Cloud Infrastructure Entitlement Management)** - discovers and right-sizes excessive permissions. "This service account has admin and uses three API calls."

### Data

* Encrypt at rest (provider-managed keys, customer-managed keys, or bring-your-own — the choice determines who can technically decrypt it).
* Encrypt in transit, TLS everywhere including internal.
* **No public storage buckets** unless genuinely intended, and audited continuously.
* **Data residency and sovereignty** drive region selection as a legal decision (chapter 7.1).
* Versioning and immutable/object-lock backups against ransomware.

### Network

* **Security groups and NACLs** — implicit deny, minimum necessary.
* **VPC design** — private subnets for anything not needing direct internet access, NAT gateways for outbound.
* **Private endpoints** so traffic to provider services never traverses the internet.
* **No management interfaces exposed publicly.** SSH and RDP open to `0.0.0.0/0` is a recurring finding.

### Posture and monitoring

* **CSPM (Cloud Security Posture Management)** - continuously checks for misconfigurations. Given that misconfiguration is the top cause of incidents, this is high value.
* **CWPP (Cloud Workload Protection Platform)** - runtime protection for VMs, containers and serverless.
* **CNAPP** - CSPM + CWPP + container security combined.
* **CASB (Cloud Access Security Broker)** - sits between users and cloud services, giving visibility into shadow IT, DLP and access control.
* **SSPM** - the same idea for SaaS configuration.
* **Logging** - CloudTrail/Activity Log **enabled in all regions** (attackers deliberately operate in regions you don't monitor), delivered to a separate account, immutable.

### Governance

* **IaC (Terraform, CloudFormation)** so infrastructure is versioned, reviewable and repeatable — with scanning of the templates themselves, and vigilance about hardcoded secrets in them.
* **Guard rails** — service control policies and admission controls that make insecure configurations impossible to deploy rather than merely detectable afterward.
* **Tagging and cost/asset visibility**, because untracked resources are unpatched resources.

### Recurring failure patterns

Publicly exposed storage · over-permissive IAM · exposed management interfaces · secrets in code and IaC · logging disabled or unmonitored regions · unpatched images · abandoned resources still running and billing.

Almost all of it is **configuration, not exploitation**, which is why CSPM and guard rails matter more in cloud than traditional vulnerability management does.

## 9.9 Lab — Docker Containers

```bash
# run something
docker run -d --name web -p 8080:80 nginx

# inspect
docker ps
docker inspect web
docker logs web
docker exec -it web /bin/bash

# see the layers — and what's in them
docker history nginx

# scan an image for known vulnerabilities
docker scout cves nginx
trivy image nginx
```

### Hardening a container

```bash
docker run -d \
  --name web \
  --user 1000:1000 \              # don't run as root
  --read-only \                   # immutable root filesystem
  --cap-drop=ALL \                # drop all capabilities
  --cap-add=NET_BIND_SERVICE \    # add back only what's needed
  --security-opt=no-new-privileges \
  --memory=512m --cpus=1 \        # resource limits (DoS containment)
  -p 8080:8080 \
  myapp:1.2.3                     # pinned version, never :latest
```

A minimal Dockerfile with the security decisions visible:

```dockerfile
FROM node:20-alpine                 # minimal base = fewer CVEs
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev               # no dev dependencies in production
COPY . .
RUN addgroup -S app && adduser -S app -G app
USER app                            # drop root before running
EXPOSE 8080
CMD ["node", "server.js"]
```

### What the lab actually demonstrated

1. **`docker history` shows every layer, and deleted files still exist in earlier ones.** Adding a secret in one layer and `RUN rm`-ing it in the next leaves it fully extractable from the image. Same lesson as git objects — **removing the reference is not removing the data**, which is the entire premise of leakhound.

2. **Default is root.** Most images run as root unless you change it, and combined with a mounted volume that's a straightforward path to affecting the host. `--user` is one flag and it's frequently skipped.

3. **`--privileged` disables the isolation.** Worth trying once in a lab to see how completely — it's not "slightly more access", it's effectively root on the host.

4. **Scanning a popular official image returns a long CVE list.** That's the SCA point from chapter 7.20 made concrete: most of what you deploy is other people's code, and that's where most of your vulnerability count lives.

---

## Chapter 9 — what I'd take away

* A public server is under continuous automated attack from the moment it has an address. Design for compromise, not just against it.
* Amplification attacks exist because IP source spoofing is still possible; ingress filtering is the structural fix.
* Attacking a shared dependency (Dyn) takes down everyone downstream without touching them.
* Type 1 hypervisors have less attack surface than Type 2; VM escape breaks the whole multi-tenant model.
* Containers share the host kernel — a kernel bug is an escape, and that's the core difference from VMs.
* Deleted files persist in earlier image layers, exactly like git objects.
* Shared responsibility: your data, identities and configuration are yours at **every** service model, SaaS included.
* Cloud incidents are overwhelmingly misconfiguration, not exploitation — so posture management beats patch-chasing there.
