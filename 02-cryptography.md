# Chapter 2 — Cryptography

## 2.1 Cryptography Basics

Cryptography is the toolkit that gives us four properties. Every algorithm and protocol in this chapter and chapter 7 exists to deliver one or more of them.

* **Confidentiality** - only the intended party can read it.
* **Integrity** - it hasn't been altered.
* **Authenticity** - it came from who it claims to.
* **Non-repudiation** - the sender can't later deny sending it.

### The vocabulary

* **Plaintext** - the readable original.
* **Ciphertext** - the encrypted output.
* **Cipher** - the algorithm.
* **Key** - the secret input that makes the algorithm produce a specific result.

### Kerckhoffs's principle

The single most important idea to absorb early: **the security must depend entirely on the key, not on the algorithm being secret.**

Assume your attacker knows exactly which algorithm you're using. That's the default assumption because it's true — algorithms get published, reverse engineered, or leak with the source code. If your security relies on nobody knowing *how* it works, you have **security through obscurity**, and it fails permanently the moment someone looks.

This is why the good algorithms are all public and heavily analysed. AES was chosen through an open international competition where everyone attacked every candidate for years. That public beating is *why* we trust it. An algorithm that hasn't been publicly attacked isn't safe, it's just untested.

The practical version of this: **don't roll your own crypto.** Not because you're not smart enough, but because correctness here is only demonstrated by surviving years of expert attack, and your version hasn't had that.

### Symmetric encryption

**One key, used for both encryption and decryption.** Both sides share the same secret.

* Fast. With hardware acceleration (AES-NI) it's essentially free.
* Used for bulk data — disk encryption, TLS sessions once established, my password vault.
* **AES** is the standard, at 128/192/256-bit key lengths.

The problem is **key distribution**. We both need the same key, so how do I get it to you securely? If I already had a secure channel to send it over, I wouldn't have needed the encryption. Chicken and egg.

It also scales badly: n people all needing pairwise secrecy requires n(n−1)/2 keys. A thousand people is roughly half a million keys to manage.

### Asymmetric encryption

**Two mathematically linked keys.** What one encrypts, only the other can decrypt.

* **Public key** - handed to anyone, no secrecy needed.
* **Private key** - never leaves the owner.

This solves distribution — nothing secret is ever transmitted.

Two directions, and keeping them straight is the whole thing:

* **Encrypt with public, decrypt with private** → confidentiality. Anyone can write to me, only I can read it.
* **Encrypt (sign) with private, verify with public** → authenticity and non-repudiation. Only I could have made it, anyone can check.

**RSA** rests on the difficulty of factoring large numbers. **ECC (elliptic curve)** rests on the discrete logarithm problem over curves, and gets equivalent security from far smaller keys — a 256-bit ECC key ≈ a 3072-bit RSA key — which is why it dominates mobile and embedded.

Asymmetric is **slow**. Orders of magnitude slower than symmetric.

### The hybrid model

Neither is sufficient alone, so real systems use both. This is what TLS does on every HTTPS connection:

1. Use **asymmetric** to authenticate the server and agree a shared secret.
2. Derive a **symmetric** session key from it.
3. Encrypt the actual traffic symmetrically.

Slow expensive tool for the small hard problem, fast cheap tool for the large easy one. Once you see TLS as that trade, the whole handshake stops feeling arbitrary.

### Key exchange and forward secrecy

**Diffie-Hellman** lets two parties derive a shared secret over a public channel without ever transmitting it. **ECDHE** is the elliptic-curve ephemeral version, and it's what's used now.

The **ephemeral** part gives **perfect forward secrecy** — a fresh key per session, thrown away after. The payoff is worth being precise about: if my server's long-term private key is stolen next year, an attacker who recorded this year's traffic still can't decrypt it, because those session keys never touched disk and no longer exist anywhere. Without forward secrecy, one key compromise retroactively decrypts every conversation ever recorded.

### Entropy and randomness

Everything above depends on keys being unpredictable. **Entropy** is the measure of that unpredictability, and it's a genuine failure point — weak randomness at generation produces a guessable key no matter how many bits long it is. The Debian OpenSSL bug in 2008 crippled the entropy source and made every key generated on affected systems for two years predictable.

Use `/dev/urandom` or the OS CSPRNG. Never `rand()`, never anything seeded from the clock.

## 2.2 Hashing

A hash function takes input of any size and produces a fixed-size output, and it is **one-way** — you cannot reverse it. There is no key and no decryption. Anyone saying "encrypt the password" when they mean hash it has told you they don't know the difference.

### Required properties

* **Deterministic** - same input, same output, every time. Otherwise it's useless for verification.
* **Fixed-length output** - regardless of input size.
* **Fast to compute** - for general use. Deliberately *not* for passwords, see 2.4.
* **Avalanche effect** - change one input bit and roughly half the output bits flip. Similar inputs must not produce similar hashes.
* **Preimage resistant** - given a hash, you can't find an input that produces it.
* **Collision resistant** - you can't find two different inputs with the same hash.

### The algorithms

| Algorithm | Output | Status |
|---|---|---|
| MD5 | 128-bit | **Broken.** Practical collisions since 2004. Checksums only. |
| SHA-1 | 160-bit | **Broken.** Practical collision demonstrated (SHAttered, 2017). Deprecated. |
| SHA-256 / SHA-512 | 256/512-bit | Current standard (SHA-2 family). |
| SHA-3 | variable | Different internal construction, held in reserve if SHA-2 falls. |
| RIPEMD, Whirlpool | varies | Alternatives, less common. |

"Broken" here means collision-broken specifically. MD5 is still perfectly fine as a non-security checksum against accidental corruption — which is why my VAPT pipeline stores both MD5 and SHA-256 on every uploaded report. The SHA-256 is the security-relevant one; the MD5 is there for compatibility with tooling that expects it.

### What it's used for

* **File integrity** - download a file, hash it, compare to the published hash.
* **Password storage** - see 2.4.
* **Digital signatures** - hash first, then sign the hash (chapter 7).
* **Deduplication** - identical content produces identical hashes.
* **Forensics** - hash evidence at acquisition and re-verify later to prove it wasn't altered. This is what makes chain of custody technically enforceable rather than just a paper trail.

### HMAC

A hash combined with a shared secret key. Gives **integrity and authenticity** — you know the message wasn't altered *and* that it came from someone holding the key.

It does **not** give non-repudiation, because both parties know the same key, so either could have produced it. That's the line between HMAC and a digital signature.

## 2.3 Cryptographic Attacks

The recurring lesson across this whole section: **nobody breaks the algorithm.** Attacks land on implementation, configuration, negotiation and key handling. AES has never been meaningfully broken; keys get found in config files.

* **Brute force** - try every key. Defeated by key length alone. AES-256 is not brute-forceable with any conceivable classical computing.
* **Birthday attack** - exploits the birthday paradox to find **collisions** far faster than intuition says. For an n-bit hash you need roughly 2^(n/2) attempts, not 2^n. This is *why* hash outputs need to be twice as long as you'd naively think, and why 128-bit hashes aren't considered collision resistant.
* **Collision attack** - finding two inputs with the same hash. Once feasible, signatures break: a signature over a benign document is equally valid on a malicious one with the same hash. This killed MD5 and SHA-1 in practice.
* **Downgrade attack** - forcing the connection to negotiate an older, weaker protocol or cipher you know how to break. POODLE, FREAK, Logjam. **The defence is disabling legacy protocols entirely**, not merely preferring the modern one — "supported but not preferred" is still negotiable, and negotiation is exactly what the attacker manipulates.
* **Side channel attacks** - attacking the implementation rather than the maths, by measuring something the computation leaks:
  * **Timing** - how long an operation takes. The practical one for software people: if your password or HMAC comparison returns early on the first mismatched byte, the *time taken* reveals how much of the guess was correct. An attacker recovers the secret byte by byte. Fix is **constant-time comparison**, and it's an easy mistake to make without realising.
  * **Power analysis** - measuring power draw during operations.
  * **Electromagnetic and acoustic** - yes, people have recovered keys from the sound a CPU makes.
* **Pass the hash** - authenticating with a stolen hash directly, never cracking it to plaintext. Windows/NTLM. Worth internalising because it breaks the assumption that a strong uncracked password keeps you safe.
* **Replay attack** - capturing a valid encrypted exchange and resending it. Defeated with nonces, timestamps and sequence numbers.
* **Harvest now, decrypt later** - recording encrypted traffic today to decrypt once quantum computers can run Shor's algorithm against RSA and ECC. A genuine present-tense concern for anything with a long secrecy lifetime, and the reason post-quantum cryptography is being standardised now.

## 2.4 Password Cracking

### Why passwords need special treatment

Two separate problems with storing a plain hash of a password.

**Problem one — identical passwords produce identical hashes.** Two users both choosing `password123` get identical database rows, which leaks information immediately. Worse, an attacker can precompute the hash of every common password once — a **rainbow table** — and then look up your entire stolen database instantly.

**Salt** fixes this. A unique random value per user, stored alongside the hash, mixed in before hashing. It doesn't need to be secret. The effect is that the same password produces a different hash for every user, which destroys rainbow tables — the attacker now has to attack each password individually.

**Pepper** is the related idea: a secret value added to *all* passwords before hashing, stored **separately from the database** (app config, or an HSM). The reasoning is that a database dump alone becomes useless, because the attacker got the salts but not the pepper.

**Problem two — normal hashes are too fast.** SHA-256 is designed for speed, which is precisely wrong here. A GPU can compute billions per second.

So password hashing uses deliberately slow, resource-hungry functions — this is **key stretching**:

* **PBKDF2** - iterated hashing. Configurable iteration count. Old but still acceptable.
* **bcrypt** - based on Blowfish, deliberately slow, has a work factor.
* **scrypt** - memory-hard as well as slow.
* **Argon2** (Argon2id is the current recommendation) - memory-hard by design, specifically to defeat GPUs, which have huge parallelism but limited memory per core.

In my own password manager I used Argon2id with `time_cost=3`, `memory_cost=64 MiB`, `parallelism=4`. The memory parameter is the one that matters and the one people leave at default — it's what makes GPU brute-force expensive rather than merely annoying.

### The attack types

* **Brute force** - every possible combination. Guaranteed eventually, infeasible against a long password behind a slow hash, instant against a short one.
* **Dictionary attack** - a wordlist of likely passwords. Vastly more efficient than brute force because human password choice is nowhere near random. Modern wordlists (rockyou and descendants) come from real breaches, so they're a list of things people *actually chose*.
* **Hybrid attack** - dictionary plus mangling rules. `password` → `Password1` → `P@ssw0rd2024!`. This is what defeats naive complexity requirements, because the rules encode exactly what humans do when told to add a capital, a number and a symbol.
* **Password spraying** - one common password against many accounts. Deliberately inverted to dodge lockout, since each account only sees one or two failures. Hard to spot per-account; you have to correlate across accounts to see it at all.
* **Credential stuffing** - username/password pairs from *other* breaches, replayed against your site. Works purely on password reuse, and it's the single strongest argument for a password manager.
* **Rainbow table** - precomputed hash lookups. Killed by salting.
* **Offline vs online** - **online** attacks guess against the live login, so rate limiting and lockout work. **Offline** means the attacker has the hash database and cracks at their own pace with no rate limit and no lockout. Once it's offline, every login-side protection you built is irrelevant and only the *hashing algorithm choice* is standing between the attacker and the plaintext. That's the whole reason 2.4's first half matters.

### Defences

* **Length beats complexity.** A long passphrase has more entropy than a short mangled one and is easier to remember.
* **Ban known-breached passwords** rather than enforcing arbitrary character classes. Check against a breach corpus.
* **MFA defeats nearly all of the above**, because the password stops being sufficient on its own.
* **Slow hash + unique salt** for storage.
* **Rate limiting and lockout** for online attacks — with the caveat that aggressive lockout is itself a DoS vector, since an attacker can lock out every account deliberately.

Modern NIST guidance actually reversed the old advice: **no forced periodic rotation** unless there's evidence of compromise, because forced rotation makes people pick weaker, more predictable passwords and write them down.

### Tooling (demo)

`hashcat` (GPU-accelerated) and `John the Ripper` are the standard crackers. The instructive thing about watching a cracking demo isn't the tool, it's the speed differential — an unsalted MD5 database falls in seconds, the same passwords behind bcrypt take years. Same passwords. The entire difference is the storage decision made by a developer years earlier.

## 2.5 Lab — SSH Public Key Authentication

Asymmetric crypto applied to something I use constantly, which made it the section that made the concept concrete.

### Generating a key pair

```bash
# Ed25519 — modern, small, fast. Prefer this.
ssh-keygen -t ed25519 -C "darsh@laptop"

# RSA if you must support something old — 4096 bits minimum
ssh-keygen -t rsa -b 4096 -C "darsh@laptop"
```

This produces two files in `~/.ssh/`:
* `id_ed25519` — the **private key**. Never leaves this machine. Mode `600`.
* `id_ed25519.pub` — the **public key**. Safe to hand out freely.

The passphrase prompt matters: it encrypts the private key file at rest, so a stolen laptop doesn't hand over usable keys. `ssh-agent` holds the decrypted key in memory so you're not typing it constantly.

### Installing the public key on a server

```bash
ssh-copy-id user@server
# or manually: append the .pub contents to ~/.ssh/authorized_keys on the server
```

Permissions are enforced strictly by sshd and this is where it usually breaks:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
```
sshd refuses keys with loose permissions, silently falling back to password auth, which is confusing until you know to look for it.

### What actually happens on connect

1. Client says which public key it wants to use.
2. Server checks `authorized_keys` for it.
3. Server sends a challenge.
4. Client **signs the challenge with the private key**.
5. Server verifies the signature with the public key.

**The private key never crosses the network.** Neither does anything reusable — the challenge is fresh each time, so capturing the exchange gains an attacker nothing. That's the same signature mechanism from 2.1, doing authentication instead of document signing.

### Hardening the server

```
# /etc/ssh/sshd_config
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

Turning off password auth is the point of the whole exercise. It removes brute force, spraying and credential stuffing from the attack surface entirely — there is no password to guess. Any internet-facing SSH server sees constant automated password guessing; with keys only, all of it fails at step zero.

**The trade** is availability: lose the private key and you've locked yourself out. Which is exactly why my roundhound rotation tool **archives old keys rather than deleting them** — a rotation tool that can destroy your only access is worse than no rotation tool.

---

## Chapter 2 — what I'd take away

* Security lives in the key, never in the secrecy of the algorithm.
* Symmetric is fast but can't solve key distribution; asymmetric solves distribution but is slow. Real systems use both, and that's what TLS is.
* Hashing is not encryption. One-way, no key, no decryption.
* Salt kills rainbow tables; slow hashes kill GPU cracking. You need both.
* Once a hash database is offline, only the hashing algorithm choice protects it — nothing you did at the login page matters anymore.
* Attacks land on implementation and negotiation, not on the maths.
