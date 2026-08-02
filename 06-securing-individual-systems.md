# Chapter 6 — Securing Individual Systems

## 6.1 Malware

Software written to do harm. The taxonomy matters because the categories imply different detection and different response.

### The types

* **Virus** - attaches to a legitimate file or program and needs **user action** to spread. Requires a host.
* **Worm** - **self-propagating** across networks with no user action at all. That single difference is why worms cause the fastest and widest damage — WannaCry and Conficker spread at machine speed, not human speed.
* **Trojan** - disguised as something legitimate. No self-replication; it relies entirely on the user installing it willingly. A **RAT (Remote Access Trojan)** is the variant giving interactive remote control.
* **Ransomware** - encrypts data and demands payment. Modern crews use **double extortion**: exfiltrate first, *then* encrypt, so good backups no longer save you from the leak threat. That change broke the "just restore from backup" answer that used to be sufficient. Some now add a third layer — DDoSing you or contacting your customers directly.
* **Spyware** - covert surveillance. **Keyloggers** are the specific subtype (chapter 3.3).
* **Rootkit** - hides deep in the system to conceal itself and other malware. Kernel-mode, bootkits, firmware-level. The defining property is that **it subverts the very tools you'd use to find it** — the OS's own directory listing lies to you. Detection usually requires booting from separate trusted media, or offline disk analysis.
* **Logic bomb** - dormant code triggered by a condition. A date, or an employee record vanishing from payroll. Frequently an insider-threat tool.
* **Backdoor** - deliberate alternative access bypassing normal authentication.
* **Bloatware** - preinstalled unwanted software. Not malicious in intent, and it's unpatched attack surface you didn't choose.
* **PUP (Potentially Unwanted Program)** - adware and similar, in the grey zone.
* **Fileless malware** - lives in memory, using tools already on the system (PowerShell, WMI, `certutil`). **Leaves no file for a signature scanner to find**, which is precisely why detection had to move to behaviour.

### Indicators of infection

Unexpected outbound connections — especially **fixed-interval beaconing** (chapter 1.3). New persistence mechanisms (scheduled tasks, services, run keys, cron). Unusual process trees — **Word spawning PowerShell is almost never legitimate**. Disabled security tooling. High CPU (cryptomining) or disk/network (staging and exfiltration). Modified timestamps that don't fit. Missing logs.

### Why signature detection stopped being enough

A signature is a hash or byte pattern of known malware. Attackers defeat it by changing a byte:

* **Polymorphic** malware mutates its own code on each propagation.
* **Metamorphic** rewrites itself entirely.
* **Packers and crypters** wrap a known payload so the signature changes while the behaviour doesn't.

So detection moved to **behaviour** — what does this process actually *do*? Rapidly encrypting files, injecting into other processes, contacting known-bad infrastructure. Behaviour is expensive for an attacker to change, which is the IOC-versus-TTP argument from chapter 1.3 showing up in endpoint security. **This is exactly what EDR does that traditional antivirus doesn't.**

## 6.2 Weak Configurations

A large share of real breaches come from things being *configured* badly rather than *coded* badly. No exploit required.

* **Default credentials** - `admin/admin`. Searchable per device model, and still one of the most reliable ways into IoT, printers, network gear and management interfaces. Mirai built one of the largest botnets ever by trying roughly 60 default pairs.
* **Unnecessary open ports and services** - every listener is attack surface. The highest-value single hardening action is turning off what you don't use, because **a service that isn't running cannot be exploited by any vulnerability discovered in it in future**.
* **Unpatched systems** - most breaches use *known* vulnerabilities that simply weren't patched, not zero-days.
* **Open permissions** - world-readable secrets, world-writable directories, over-broad shares. This is the entire premise of credhound: a credential is only as safe as the permission bits guarding it.
* **Unsecured root/admin accounts** - direct root login, shared admin passwords, no MFA on privileged accounts.
* **Errors and verbose output** - stack traces exposing file paths, library versions, SQL structure. Free reconnaissance.
* **Weak or deprecated encryption** - SSLv3, TLS 1.0, RC4, DES, self-signed certs in production, unencrypted protocols.
* **Unsecured protocols** - Telnet, FTP, HTTP, SNMPv1/v2c. All send credentials in cleartext.
* **Missing logging and monitoring** - you can't respond to what you never saw.
* **Misconfigured cloud** - public storage buckets, over-permissive IAM roles, exposed management planes. Overwhelmingly the leading cause of cloud incidents, and it's a **customer** failure under the shared responsibility model, not a provider one.

### Secure baselines

The answer to all of the above is a documented, approved **secure baseline** per system type, derived from CIS Benchmarks or vendor guidance rather than invented. Three steps:

1. **Establish** the baseline.
2. **Deploy** it.
3. **Maintain** it — and this is the step that gets dropped. **Configuration drift** is inevitable; continuous checking (chapter 5.13) is what catches it.

## 6.3 Common Attacks

Attacks against a single system, as opposed to network attacks (chapter 7) and web attacks (chapter 11).

* **Privilege escalation** - **vertical** (user → admin/root) and **horizontal** (user A → user B's data). Vertical usually exploits a kernel or service vulnerability, a misconfigured SUID binary, a writable service path, or a weak sudo rule. Horizontal is frequently just an unchecked identifier.
* **Malicious USB / HID attacks** - a device that presents itself as a **keyboard** and types commands faster than a human can react. The OS trusts it because keyboards are trusted by definition. My own Rubber Ducky project is exactly this, and it's why USB port control exists on high-value machines.
* **DLL injection / process injection** - forcing code into another process's memory space so it runs under that process's identity and permissions. Common for both evasion and privilege escalation.
* **Shimming and refactoring** - abusing application compatibility infrastructure to intercept calls.
* **Driver manipulation** - malicious or vulnerable signed drivers. "Bring your own vulnerable driver" is a current technique: load a legitimately-signed but flawed driver to get kernel-level access.
* **Pass the hash** - authenticating with a stolen hash without cracking it (chapter 2.3).
* **Persistence mechanisms** - scheduled tasks, cron jobs, systemd units/timers, registry run keys, new services, startup folders, `.bashrc`, SSH `authorized_keys`. Worth knowing the list because **checking these locations is standard incident response**, and adding to them is standard post-exploitation.

## 6.4 Overflow Attacks

Memory corruption. Historically the root cause of an enormous share of critical CVEs.

### Buffer overflow

A program allocates a fixed-size buffer and then writes more data into it than it holds. The excess overwrites adjacent memory.

```c
char buffer[64];
strcpy(buffer, user_input);   // no bounds check — writes as much as you give it
```

**Stack overflow** is the classic: local variables live on the stack alongside the saved **return address**. Overflow far enough and you overwrite that return address, so when the function returns, execution jumps wherever you pointed it — typically to shellcode you also supplied. That's arbitrary code execution.

**Heap overflow** corrupts dynamically allocated memory and its metadata instead.

### Related memory bugs

* **Integer overflow** - a value exceeds its type's maximum and wraps around. A length check of `if (len < 100)` passes when `len` wrapped to a small number, and then a huge allocation or copy happens anyway.
* **Use-after-free** - memory is freed and then used again. If an attacker controls what gets allocated into that space in between, they control what the program operates on.
* **Format string vulnerability** - `printf(user_input)` instead of `printf("%s", user_input)`, letting an attacker read and write memory via format specifiers.
* **Race condition / TOCTOU** - Time Of Check to Time Of Use. State is validated and then changes before it's used. Check the balance, then withdraw — if two requests interleave, both succeed. Fixed with locking and atomic operations.

### Mitigations

* **ASLR (Address Space Layout Randomisation)** - randomises memory layout each run, so the attacker can't reliably predict where to jump.
* **DEP / NX (Data Execution Prevention / No-eXecute)** - marks data regions non-executable, so shellcode written to the stack can't run. Countered by **ROP (Return-Oriented Programming)**, which chains together fragments of existing legitimate code rather than injecting new code.
* **Stack canaries** - a known value placed before the return address; if it's changed at function exit, the overflow is detected and the program aborts.
* **Bounds checking** and safe functions (`strncpy` over `strcpy`, and better still, don't use C strings).
* **Memory-safe languages.** Rust, Go, Java, Python make whole categories of this structurally impossible. This was a real factor in my choosing Rust for the hound tools — they parse untrusted binary data (git packfiles, delta chains), which is exactly the situation where a C parser bug becomes remote code execution.

## 6.5 Password Attacks

Covered in depth in chapter 2.4. The short version:

* **Brute force** (every combination), **dictionary** (wordlists of what people actually pick), **hybrid** (dictionary + mangling rules), **spraying** (one password, many accounts, to dodge lockout), **credential stuffing** (pairs from other breaches, exploiting reuse), **rainbow tables** (precomputed, killed by salting).
* **Online vs offline** is the distinction that matters most: once the hash database is offline, rate limiting and lockout are irrelevant and only the **hashing algorithm choice** protects you.
* Defences: length over complexity, breach-list checking, MFA, slow salted hashes (bcrypt/scrypt/Argon2id), rate limiting, and no forced rotation without cause.

## 6.6 Bots and Botnets

* **Bot** - a compromised machine under remote control.
* **Botnet** - a network of them, coordinated by an attacker (the **bot herder**) through **C2 (command and control)** infrastructure.

### What they're used for

DDoS (the main one — scale is the whole point), spam distribution, credential stuffing at volume, cryptomining, proxying attacker traffic through residential IPs, and click fraud.

### C2 architectures

* **Centralised** - all bots talk to one server. Simple, and a single takedown point.
* **P2P / decentralised** - bots relay commands between themselves. Far more resilient, harder to disrupt.
* **Domain Generation Algorithms (DGA)** - the malware algorithmically generates hundreds of candidate domains daily and tries each; the operator only needs to register one. Defeats static domain blocklists, and it's detectable as a distinctive pattern of mass failed DNS lookups.
* **Fast flux** - rapidly rotating the IPs behind a domain.

### Detection

**Beaconing** is the signature behaviour: periodic outbound connections at regular intervals to the same destination. Regularity is the tell, because genuine human-driven traffic is irregular. Attackers add **jitter** (random variance) specifically to break that pattern, so detection looks for "regular-ish within a tolerance" rather than exact intervals.

Also: DNS anomalies (DGA patterns), traffic to newly-registered domains, connections at times the user isn't active, and unexplained bandwidth.

**Egress filtering** is the underrated control here. Most networks are far stricter on inbound than outbound, and **C2 is an outbound problem**. Restricting what internal hosts may talk to, and forcing traffic through an inspecting proxy, breaks a large fraction of malware regardless of how it got in.

## 6.7 Disk RAID Levels

RAID = Redundant Array of Independent Disks. Combines several physical disks into one logical volume for redundancy, performance, or both.

| Level | Method | Min disks | Fault tolerance | Capacity | Notes |
|---|---|---|---|---|---|
| **RAID 0** | Striping | 2 | **None** | 100% | Pure performance. One disk dies, everything is gone. |
| **RAID 1** | Mirroring | 2 | 1 disk | 50% | Simple, fast reads, expensive in space. |
| **RAID 5** | Striping + distributed parity | 3 | 1 disk | (n−1)/n | Good balance. Slow writes (parity calculation). |
| **RAID 6** | Striping + double parity | 4 | 2 disks | (n−2)/n | Survives a second failure during rebuild. |
| **RAID 10 (1+0)** | Mirror then stripe | 4 | 1 per mirror | 50% | Fast and resilient. The usual choice for databases. |

### The exam trap

**RAID 0 has no redundancy at all** despite the "R" standing for redundant. It's striping for speed and it *increases* your failure probability, since losing any one disk loses the whole array.

### Why RAID 6 exists

Rebuilding a large RAID 5 array after a failure takes hours to days, and hammers every remaining disk with sustained reads during that window. Disks bought together, from the same batch, that have aged identically, tend to fail at similar times — so a second failure *during rebuild* is a realistic scenario, not a paranoid one. RAID 6 survives it. This is why large arrays use RAID 6 or RAID 10 rather than 5.

### The point that matters most

**RAID is not backup.**

It protects against **disk failure** and nothing else. It will faithfully and instantly replicate:

* Ransomware encryption
* Accidental deletion
* A `DROP TABLE`
* Filesystem corruption
* A fire in the building holding the array

RAID is an **availability** control (chapter 1). Backup is a **recovery** control. They solve different problems and neither substitutes for the other. The 3-2-1 rule still applies on top: three copies, two media types, one off-site — and one offline/immutable, because modern ransomware deliberately hunts network-reachable backups.

## 6.8 Securing Hardware

Defence starts below the operating system.

* **BIOS/UEFI passwords** - prevent boot order changes and firmware modification. Without one, an attacker boots from USB into a live OS and reads your disk — unless it's encrypted.
* **Secure Boot** - firmware verifies the signature of the bootloader and kernel before execution, blocking bootkits and unsigned rootkits.
* **Measured boot / TPM attestation** - each boot stage is hashed and recorded in the TPM, producing a verifiable record of the boot state. Enables detecting that something changed even if it still boots.
* **TPM (Trusted Platform Module)** - a chip on the motherboard that generates and stores keys **inside a hardware boundary they never leave**. It performs crypto operations on request, so software can *use* a key without ever *possessing* it. Backs BitLocker and Secure Boot. Binds the disk key to *this machine in a known-good state*, so pulling the drive and mounting it elsewhere yields nothing.
* **HSM (Hardware Security Module)** - the enterprise version. Dedicated tamper-resistant appliance for high-value keys — CA private keys, payment processing, code signing. Often tamper-*responsive*: it destroys its keys if physically opened.
* **Secure enclave** - isolated execution environment inside the main processor. Apple Secure Enclave, Intel SGX, ARM TrustZone. Same principle at chip level.
* **Self-encrypting drives (SED / OPAL)** - encryption in the drive controller, transparent to the OS. Makes disposal trivial via crypto-erase.
* **Cable locks, port blockers, locked chassis** - the physical layer (chapter 3).
* **Firmware updates** - firmware vulnerabilities sit below the OS, survive reinstalls, and are frequently unpatched because nobody thinks about them.
* **Supply chain / hardware root of trust** - counterfeit components and implants. You inherit the trustworthiness of your manufacturer.

The recurring pattern across TPM, HSM and secure enclaves: **generate the key inside a hardware boundary and never let it out.** Even total OS compromise doesn't yield the key material.

## 6.9 Securing Endpoints

* **Antivirus** - signature-based, catches known malware. Necessary, not sufficient.
* **EDR (Endpoint Detection and Response)** - behavioural monitoring, telemetry collection, investigation and remote response (isolate the host, kill the process, roll back changes). This is what catches fileless and polymorphic malware that signatures miss.
* **XDR** - EDR correlated with network, cloud, identity and email telemetry.
* **HIDS/HIPS** - host intrusion detection/prevention.
* **Host-based firewall** - per-machine filtering. Matters more in a zero-trust world where you can't rely on a network perimeter.
* **Application allow listing** - only explicitly approved software may execute. **Far stronger than deny listing**, because a deny list can only block what you already know is bad, while an allow list blocks everything you never approved — including malware nobody has ever seen. Harder to operate, which is why fewer places do it.
* **DLP agent** - watches for sensitive data leaving.
* **Patch management** - the highest-value routine activity there is.
* **Disable unnecessary services and ports**.
* **Least privilege** - users don't run as local admin. Removes a large fraction of malware's ability to install persistently.
* **FIM (File Integrity Monitoring)** - alerts on changes to critical system files.
* **Sandboxing** - running untrusted code in isolation.
* **Hardened browsers**, since the browser is the primary attack surface on most endpoints.

### Hardening checklist

Encryption at rest · endpoint protection · **disable unused ports/services/protocols** · remove bloatware · change default passwords · least privilege · patch · Secure Boot · logging enabled and shipped off-host.

## 6.10 Securing Data with Encryption

### The three states

* **At rest** - stored. Full-disk encryption, file/folder encryption, database encryption.
* **In transit** - moving. TLS, IPsec, VPN, SSH.
* **In use** - loaded in memory, being processed. **The hardest**, because data generally has to be plaintext to be operated on. Addressed by secure enclaves, and increasingly by homomorphic encryption and confidential computing, both still niche.

### Full-disk encryption

* **BitLocker** (Windows) - TPM-backed, optionally with a PIN or USB key.
* **LUKS/dm-crypt** (Linux) - the standard.
* **FileVault** (macOS) - Secure Enclave-backed.
* **VeraCrypt** - cross-platform, supports containers and hidden volumes.

**What FDE actually protects against:** a stolen or improperly disposed **powered-off** device. That's it, and it's genuinely valuable — most laptop-loss incidents become non-events.

**What it does not protect against:** a running system, because the key is in RAM and the filesystem is mounted and readable. Malware running as you sees plaintext. This is why "my disk is encrypted" is not an answer to "you have malware".

**The suspend problem** worth internalising: a suspended laptop has the key in RAM. Cold-boot attacks and DMA attacks against a suspended machine are real. **Locking the screen is not the same as powering off**, and for genuinely sensitive work, hibernate or shut down.

### Other layers

* **File/folder encryption** - protects individual files even on a running system, with per-file keys.
* **Database encryption** - **TDE (Transparent Data Encryption)** encrypts the files at rest, transparent to the application; **column-level** encryption protects specific sensitive fields even from a DBA reading the table.
* **Key management** is where this actually fails in practice — excellent algorithms, keys stored next to the data or in a git repo. Which is, again, why leakhound exists.
* **Crypto-erase** - destroy the key, and the data becomes permanently unrecoverable ciphertext. The clean answer to SSD disposal (chapter 1.10).

## 6.11 Lab — Linux Software RAID

```bash
# inspect available disks
lsblk

# create a RAID 1 mirror across two disks
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# watch it build
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# filesystem and mount
sudo mkfs.ext4 /dev/md0
sudo mkdir /mnt/raid && sudo mount /dev/md0 /mnt/raid

# persist the array config so it reassembles at boot
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u

# simulate a failure and recover
sudo mdadm --manage /dev/md0 --fail /dev/sdc
sudo mdadm --manage /dev/md0 --remove /dev/sdc
sudo mdadm --manage /dev/md0 --add /dev/sdd
```

### What the lab actually teaches

Failing a disk deliberately and watching the array keep serving data is the part that makes RAID click. But two things stood out beyond the mechanics:

1. **Monitoring is the real requirement.** A degraded array keeps working silently. If nobody is alerted, you run degraded for months and then the second disk fails. `mdadm --monitor` with email, or a check feeding your central logging (chapter 5.12), is what makes RAID useful rather than decorative.
2. **The rebuild window is the dangerous period.** During rebuild, every remaining disk is under sustained load — which is exactly when the next one dies. That's RAID 6's entire justification.

And the point from 6.7 again, because it's the one people get wrong: this array survives a disk dying. It does not survive `rm -rf`, ransomware, or the building burning down.

## 6.12 Lab — Secure Enclave (macOS)

The Secure Enclave is a dedicated coprocessor with its own boot ROM and encrypted memory, isolated from the main CPU.

```bash
# check FileVault status
fdesetup status

# system integrity protection
csrutil status

# view keychain items (metadata, not secrets)
security list-keychains
security dump-keychain            # prompts per item
```

### What it does

* Generates and stores keys **that never leave the enclave**. The main CPU can request operations, not extraction.
* Handles **Touch ID / Face ID** — the biometric template is processed and stored in the enclave and **never leaves the device**, never goes to Apple, never syncs. Which is the right design given that biometrics can't be revoked (chapter 4.2).
* Manages FileVault keys and hardware-backed keychain items.
* Enforces rate limiting on unlock attempts *in hardware*, so brute forcing the passcode can't be accelerated by attacking the software.

### The generalisable idea

Same as TPM and HSM in 6.8: **the security boundary is the hardware, not the operating system.** Even a full OS compromise doesn't yield the key material, because the key was never in OS-accessible memory.

That's a genuinely different security model from "protect the key with file permissions", which is what credhound spends its time finding people doing.

---

## Chapter 6 — what I'd take away

* Worms self-propagate; viruses need a host and a user. That difference sets the speed of the incident.
* Signatures lose to polymorphism and fileless techniques — behaviour-based detection is the answer, and it's the IOC/TTP argument again.
* Misconfiguration causes more breaches than exploits. Defaults, open ports, open permissions, missing patches.
* Overflow attacks are why memory-safe languages matter, and why ASLR/DEP/canaries exist.
* Beaconing is the botnet tell, and **egress filtering** is the underrated control against C2.
* RAID 0 has no redundancy, and **RAID is not backup** under any level.
* FDE protects a powered-off device. A running or suspended machine still has the key in RAM.
* TPM, HSM and secure enclaves all encode the same idea: keep the key inside hardware and let software borrow the operation, never the key.
