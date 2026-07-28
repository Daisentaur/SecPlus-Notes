# Chapter 4 — Identity and Access Management

## 4.1 Identification, Authentication, and Authorization

Three words that get slurred together in normal speech and are genuinely distinct steps. They happen in order and you cannot skip one.

* **Identification** - *who do you claim to be?* Just the claim. Typing a username. No proof involved yet.
* **Authentication** - *prove it.* Password, fingerprint, certificate, token.
* **Authorization** - *what are you allowed to do?* Happens strictly after authentication succeeds.
* **Accounting** - *what did you actually do?* Logging and audit trail.

Together the last three are **AAA**.

The ordering is the whole point. **You cannot authorize an identity you haven't authenticated, and you cannot account for actions you can't attribute to an identity.** A system with authentication and authorization but no accounting can tell you an attacker got in with admin rights, and nothing whatsoever about what they did with them.

### The distinction that matters in practice

Identification is a claim, authentication is proof. Confusing them is how you end up with systems that trust a user-supplied ID. Every "change the ID in the URL and see someone else's data" vulnerability is a system that identified but never authorized.

## 4.2 Authentication Methods

### The factors

Three real ones:

* **Something you know** - password, PIN, security question.
* **Something you have** - phone with an authenticator app, hardware token, smart card.
* **Something you are** - biometrics.

And two treated as attributes rather than true factors:

* **Somewhere you are** - location, geofencing, IP.
* **Something you do** - behavioural. Typing rhythm, gait, mouse movement.

### Multifactor authentication

**MFA means factors from *different* categories.** A password plus a security question is not multi-factor — both are something you know. That's just two passwords. This is the single most commonly misunderstood point in the whole topic.

Not all MFA is equal, and the ranking matters:

| Method | Strength | Weakness |
|---|---|---|
| **SMS OTP** | Weakest | SIM swapping, SS7 interception, phishable |
| **Email OTP** | Weak | If email is compromised, so is this |
| **TOTP app** (Google Authenticator, Aegis) | Good | Phishable in real time — user can be tricked into typing the code into a fake site |
| **Push notification** | Good | **MFA fatigue** — spam the user with prompts until they tap approve to make it stop |
| **Hardware key (FIDO2/WebAuthn)** | Strongest | Cost, and losing it |

**Why FIDO2 hardware keys are categorically better:** the key cryptographically checks the **origin domain** before it responds. A phishing site at `g00gle.com` gets nothing, because the key simply won't sign for the wrong origin. It's **phishing-resistant by design**, not by user vigilance. Every other method on that list ultimately depends on the human noticing something is wrong.

**Number matching** is the mitigation for push fatigue — the login screen shows a number the user must type into the app, so blind approval stops working.

### Biometrics

* **Types** - fingerprint, facial recognition, iris, retina, voice, vein pattern, gait.
* **FAR (False Acceptance Rate)** - wrong person accepted. **The security failure.**
* **FRR (False Rejection Rate)** - right person rejected. The usability failure.
* **CER (Crossover Error Rate)** - the sensitivity setting where FAR and FRR are equal. Lower CER means a better sensor overall, and it's how you compare two biometric systems fairly.

Tuning sensitivity trades one error for the other. High security → accept more false rejections. Convenience → accept more false acceptances.

**The fundamental problem with biometrics: you cannot revoke them.** A leaked password gets changed in thirty seconds. A leaked fingerprint template is leaked permanently — you have ten fingers and then you're out. This is why biometrics work best as *one* factor, usually unlocking a local key store, rather than as a credential sent to a server.

### Other methods

* **Certificate-based** - the client holds a private key and a CA-signed certificate. Strong, and it's how machines authenticate to each other.
* **Smart cards / PIV / CAC** - a certificate on a physical card, usually with a PIN, so it's inherently two-factor.
* **Token-based** - hardware or software tokens producing codes.
* **Passwordless / passkeys** - FIDO2 credentials replacing passwords entirely. Removes the thing attackers most want to steal.
* **SSH keys** - covered in chapter 2.5.

### Machine and service authentication

Not obvious at first and it matters a lot: **AAA isn't only for humans.** Machines authenticate constantly and can't type a password. They use certificates, API keys, or mutual TLS.

In most organisations machine identities vastly outnumber human ones, they're long-lived, and nobody rotates them. That gap is the entire reason my roundhound project exists.

## 4.3 Authorization

Once authenticated, what can you touch?

### Core principles

* **Least privilege** - the minimum access needed to do the job, and no more. Applies to service accounts as much as people, and that's where it's most often violated because "just give it admin, we'll tighten it later" is fast and nobody comes back.
* **Separation of duties** - no single person can complete a sensitive process alone. The person who requests a payment can't also approve it. Defends against both fraud and honest mistakes.
* **Job rotation** - moving people between roles. Surfaces fraud that depends on one person permanently controlling a process, and has the side benefit of exposing single points of knowledge.
* **Mandatory vacation** - the same idea. Schemes that need daily maintenance break when the person is forced to be away for two weeks.
* **Need to know** - access is scoped to what the specific task requires, even within a clearance level.

### Implementation mechanisms

* **ACL (Access Control List)** - attached to the *resource*, listing who can do what to it.
* **Permissions / capabilities** - attached to the *identity*, listing what it can reach.
* **Implicit deny** - anything not explicitly permitted is denied. This is the correct default and it's what makes firewall and IAM policies safe by construction. The opposite (implicit allow) means every new resource is exposed until someone remembers to lock it.

### Privilege creep

Someone joins as a developer, moves to ops, then to a team lead role — and at each move they *gain* the new permissions and *keep* the old ones, because removal is nobody's job. After four years they can reach everything.

The fix is **access reviews / recertification**: periodically making managers re-confirm that each person still needs what they hold. Tedious, and it's the only thing that catches this.

## 4.4 Accounting

The A everyone forgets. Logging what identities actually did.

### What it gives you

* **Audit trail** - reconstructing what happened during an incident. Without it you know you were breached and nothing else.
* **Non-repudiation** - the user can't credibly deny the action (see chapter 2 and 7 — logs alone are weak non-repudiation because the admin controls the logs; signatures are strong).
* **Compliance evidence** - regulators ask for this directly.
* **Detection** - unusual patterns in accounting data are exactly what the SIEM correlates on (chapter 5.11).

### What to log

Authentication attempts (success **and** failure — failures are the ones that show attacks), authorization decisions especially denials, privileged actions, data access on sensitive resources, configuration changes, and account lifecycle events.

### Doing it properly

* **Ship logs off-host in real time.** An attacker with admin on a box can delete its local logs. Missing logs are an indicator of compromise, but they're a poor substitute for the actual evidence.
* **Time synchronisation (NTP)** across all systems. Correlating events across ten servers is impossible if their clocks disagree by minutes, and a defence lawyer will happily point that out.
* **Protect the logs themselves** — write-once storage or a separate system with its own credentials, so compromising the monitored host doesn't compromise the record.

## 4.5 Access Control Schemes

The models for *how* authorization decisions get made.

* **DAC (Discretionary Access Control)** - the resource **owner** decides who gets access, at their discretion. Standard Unix/Windows file permissions. Flexible, and it scales badly and leaks, because owners are inconsistent and nobody audits their choices.
* **MAC (Mandatory Access Control)** - the **system** enforces access from labels and clearances, and users cannot override it even for their own files. Military and high-security. SELinux and AppArmor are practical implementations. Rigid on purpose.
* **RBAC (Role-Based Access Control)** - permissions attach to **roles**, users get assigned roles. The enterprise standard because it scales — you manage 40 roles instead of 4,000 individual grants, and onboarding becomes "assign the role" instead of a bespoke permission archaeology exercise.
* **ABAC (Attribute-Based Access Control)** - decisions computed from **attributes** of the user, the resource, the action, and the context. "Finance staff, on a managed device, during business hours, from a corporate IP." Most granular and most flexible, and it's what zero trust actually requires.
* **Rule-based** - system-wide rules independent of identity. "No logins outside 07:00–20:00."

### Choosing between them

RBAC for most enterprise situations. ABAC when context genuinely matters and you can afford the complexity. MAC when the consequences justify the rigidity. DAC is what you get by default and rarely what you'd choose deliberately at scale.

## 4.6 Account Management

### Account types

* **User account** - a normal human.
* **Privileged / administrator account** - elevated rights. Should be **separate from the holder's daily account**, so routine web browsing and email don't happen in an admin context.
* **Service account** - runs a process, not a person. No interactive login, long-lived credential, frequently over-privileged and never rotated. A recurring weak point.
* **Guest account** - minimal, usually should be disabled.
* **Shared / generic account** - **destroys accountability** completely, because you can never attribute an action to a person. Sometimes operationally unavoidable on legacy kit, in which case it needs compensating controls like a privileged access vault that checks the credential out to a named individual.

### Lifecycle

* **Provisioning** - creating the identity and granting appropriate access on joining. Ideally automated from the HR system so it's consistent.
* **Identity proofing** - verifying the human really is who they claim *before* issuing anything. If this is weak, everything downstream is theatre.
* **Modification** - role changes. The step where privilege creep happens if old access isn't removed.
* **De-provisioning** - **removal on leaving.** The step that fails most often, and orphaned accounts belonging to people who left years ago are a standard pentest finding and a standard breach cause. Must be triggered by HR, not by someone remembering.
* **Disable vs delete** - disable first, preserving the account for audit and for recovering the person's data; delete later per retention policy.

### Policies

* **Password policy** - length over complexity, breach-list checking, no forced rotation without cause (chapter 2.4).
* **Account lockout** - after N failed attempts. Balance: too aggressive and it becomes a DoS vector where an attacker locks out every account deliberately.
* **Time-of-day and location restrictions**.
* **Session management** - timeouts, concurrent session limits, re-authentication for sensitive actions.
* **Access reviews** - periodic recertification, the only real defence against creep.

### Privileged Access Management (PAM)

Specific controls for admin accounts, because standing admin rights are the biggest single prize on any network:

* **Password vaulting** - credentials checked out from a vault, never known by the user.
* **Just-in-time access** - elevated only for a specific window, then automatically revoked. Nobody holds standing admin.
* **Ephemeral credentials** - expire immediately after use.
* **Session recording** - full capture of what was done during elevated access.

## 4.7 Network Authentication

How authentication happens across a network rather than on a single box.

* **Kerberos** - ticket-based, the core of Active Directory. The user authenticates once to the **KDC (Key Distribution Center)**, receives a **TGT (Ticket Granting Ticket)**, and then exchanges that for service tickets. **Passwords don't cross the network**, and mutual authentication means the server proves itself to the client too. Depends heavily on time sync — more than about five minutes of clock skew and tickets are rejected outright.
  Attacks worth knowing: **pass-the-ticket**, **golden ticket** (forging TGTs with the KRBTGT account hash — total domain compromise), and **Kerberoasting** (requesting service tickets and cracking them offline to recover service account passwords, which works because service account passwords are often weak and never rotated).
* **RADIUS** - centralised AAA for network access. Used for VPNs, Wi-Fi enterprise auth, and switch/router admin. UDP, and **only encrypts the password field**, leaving the rest in the clear.
* **TACACS+** - Cisco's alternative. TCP, **encrypts the entire payload**, and separates authentication, authorization and accounting into independent functions — which is why it's preferred for device administration where you want granular command-level authorization.
* **802.1X** - port-based network access control. The device must authenticate **before it gets a usable network connection at all**. Three roles: the **supplicant** (the device), the **authenticator** (the switch or AP), and the **authentication server** (usually RADIUS). This is NAC at the physical port, and it's what stops someone plugging a laptop into a meeting-room ethernet socket and being on the corporate LAN.
* **EAP** - the authentication *framework* 802.1X carries. Variants: **EAP-TLS** (certificates on both sides — strongest, most work to deploy), **PEAP** (server certificate plus password inside a TLS tunnel), **EAP-TTLS**, and **EAP-FAST**.
* **LDAP** - directory protocol for querying identity stores. **Use LDAPS** (636); plain LDAP (389) carries credentials in the clear.

## 4.8 Identity Management Systems

* **SSO (Single Sign-On)** - authenticate once, reach many systems. Genuinely better security *and* usability, which is rare — fewer passwords means less reuse, fewer written down, and one place to enforce strong MFA. The trade is concentration of risk: compromise the SSO identity and you get everything, which is exactly why SSO accounts need the strongest available factor.
* **Federation** - trusting identities issued by *another organisation's* identity provider. Partners and contractors authenticate with their own credentials, so you never issue accounts you'll forget to de-provision. The trust relationship becomes the thing you have to manage.
* **SAML** - XML-based, the enterprise SSO workhorse. The **IdP (Identity Provider)** sends a digitally signed **assertion** to the **SP (Service Provider)** vouching for the user. Signature validation is the security-critical part; implementations that don't validate properly have had serious auth-bypass bugs.
* **OAuth 2.0** - an **authorization** framework, not authentication. It issues **access tokens** granting scoped access to resources on a user's behalf. Using OAuth alone for authentication is a classic mistake — a token proves someone granted access, not who is presenting it.
* **OIDC (OpenID Connect)** - the authentication layer built **on top of** OAuth 2.0, adding an **ID token** (a JWT) that actually identifies the user. This is what "Log in with Google" really is.
* **SCIM** - automates **provisioning and de-provisioning** across cloud apps. Employee joins → SCIM creates accounts everywhere. Employee leaves → SCIM disables them everywhere. This is the technical answer to the de-provisioning failure in 4.6.
* **IdP (Identity Provider)** - the system that authenticates users and issues assertions/tokens. Okta, Entra ID, Keycloak.

### JWT, briefly

JSON Web Tokens turn up everywhere in these flows. Structure is `header.payload.signature`, base64url-encoded and dot-separated.

The critical point: **the payload is encoded, not encrypted.** Anyone can read it. Never put secrets in a JWT. And the classic vulnerability is a server that fails to verify the signature, or accepts `alg: none`, which lets an attacker rewrite the claims (`"role": "admin"`) and be believed.

## 4.9 Lab — Creating Linux Users and Groups

```bash
# create a user with a home directory and a specific shell
sudo useradd -m -s /bin/bash darsh
sudo passwd darsh

# create a group and add the user to it
sudo groupadd developers
sudo usermod -aG developers darsh      # -a is essential: append, don't replace

# check what a user has
id darsh
groups darsh

# lock / unlock an account (disable, don't delete — 4.6)
sudo usermod -L darsh
sudo usermod -U darsh

# password aging policy
sudo chage -M 90 -W 7 darsh            # max age 90 days, warn 7 days before
sudo chage -l darsh                     # view current settings
```

**The `-a` flag on `usermod -aG` matters more than it looks.** Without it, `-G` *replaces* the user's entire supplementary group list instead of appending — a very easy way to accidentally remove someone from `sudo` and lock yourself out.

### The files behind it

* `/etc/passwd` - user records. **World-readable**, and despite the name it contains no passwords anymore (the `x` in the password field means "look in shadow"). Contains username, UID, GID, home directory, shell.
* `/etc/shadow` - the actual password hashes plus aging data. **Mode 640, root-owned.** This separation exists precisely because `/etc/passwd` has to be readable by everything, and hashes must not be.
* `/etc/group` - group membership.
* `/etc/sudoers` - who can elevate, and to what. **Edit with `visudo`**, never directly — it syntax-checks before saving, and a broken sudoers file means nobody can become root.

### Security-relevant details

* **UID 0 is root.** *Any* account with UID 0 is root, regardless of its name. A second UID-0 account is a classic persistence backdoor, and it's one of the things worth checking for: `awk -F: '$3==0 {print $1}' /etc/passwd`.
* **Shell of `/usr/sbin/nologin` or `/bin/false`** for service accounts — they need to own files and run processes, not log in interactively.
* **System accounts** conventionally hold UID < 1000.
* This is exactly the surface my credhound project walks — home directories, `.ssh` keys, permission bits — and the triage rule there was **severity = what the credential is × who else can read it**, which comes straight out of understanding these files and their modes.

---

## Chapter 4 — what I'd take away

* Identification is a claim; authentication is proof; authorization is scope; accounting is the record. In that order, no skipping.
* MFA means factors from *different* categories. Password + security question is not MFA.
* Hardware keys are phishing-resistant *by design*; everything else relies on the user noticing.
* You can't revoke a biometric.
* De-provisioning is the lifecycle step that fails, and SCIM is the technical answer to it.
* RBAC scales, DAC is what you get by accident, MAC is for when rigidity is worth it.
* OAuth is authorization. OIDC is the authentication layer on top. Mixing them up is a real vulnerability class.
