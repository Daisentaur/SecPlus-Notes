# Chapter 8 — Wireless Security

Wireless is a genuinely different security problem from wired, for one reason: **the transmission medium is shared and public.** You cannot control who receives your radio waves. Everything in this chapter follows from that — on a wired network an attacker needs physical access to a port, and on wireless they need to be in the car park.

## 8.1 Wi-Fi Encryption Standards

### The progression

| Standard | Year | Encryption | Auth | Status |
|---|---|---|---|---|
| **WEP** | 1999 | RC4, 24-bit IV | Shared key | **Broken.** Crackable in minutes. |
| **WPA** | 2003 | TKIP (RC4) | PSK / 802.1X | Deprecated. Interim fix for WEP. |
| **WPA2** | 2004 | **CCMP (AES)** | PSK / 802.1X | Still widespread. PSK is offline-crackable. |
| **WPA3** | 2018 | **GCMP (AES)** | **SAE** / 802.1X | Current. Fixes the offline attack. |

### Why WEP failed

Worth knowing because it's a clean case study in implementation failure rather than algorithm failure.

WEP used RC4 with a **24-bit initialisation vector** sent in cleartext. 24 bits is only ~16.7 million values, so on a busy network **IVs repeat within hours**. RC4 is a stream cipher, and reusing a keystream is catastrophic (chapter 7.3) — collect enough IV collisions and the key falls out statistically. Tools automated this to the point where WEP cracking takes minutes and needs no skill.

The algorithm (RC4) had weaknesses, but the fatal flaw was the **too-small IV** — a design decision, not a maths problem.

### WPA and TKIP

A software-upgradeable stopgap for hardware that couldn't do AES. TKIP added per-packet key mixing and a longer IV, still built on RC4. Better, and eventually broken too. Deprecated — if you see TKIP enabled anywhere, it's dragging security down to its level.

### WPA2

**CCMP** (Counter Mode with CBC-MAC Protocol) using **AES** — real authenticated encryption. The cipher itself is sound and unbroken.

**The weakness is the authentication, not the encryption.** WPA2-Personal uses a **Pre-Shared Key**, and the 4-way handshake at association contains enough material to verify a password guess *offline*. Capture one handshake and you can guess forever, at GPU speed, with no interaction with the network. See 8.5.

### WPA3

The significant fix is **SAE (Simultaneous Authentication of Equals)**, also called Dragonfly — a password-authenticated key exchange replacing the PSK handshake.

**Why it matters:** SAE makes the handshake **useless for offline cracking**. Each guess requires a fresh interaction with the network, so an attacker can't capture once and grind offline. That single change removes the entire WPA2-Personal attack model. It also gives **forward secrecy**, so capturing traffic and later learning the password doesn't decrypt it.

Also in WPA3:
* **192-bit mode** for enterprise/government.
* **OWE (Opportunistic Wireless Encryption)** for open networks — encrypts traffic on a public hotspot without any password. Doesn't authenticate the AP (so evil twins still work), but it stops passive sniffing on open Wi-Fi, which is a meaningful improvement over nothing.
* **Protected Management Frames (PMF)** mandatory — this blocks deauthentication attacks (8.4).

Known weakness: the **Dragonblood** vulnerabilities (2019) found side-channel and downgrade issues in early implementations, largely patched. And **WPA3-Transition mode**, which allows WPA2 clients, reopens the WPA2 attacks — so transition mode is a compatibility choice with a security cost.

### Personal vs Enterprise

* **WPA2/WPA3-Personal (PSK)** - one shared password for everyone. Problems: it's shared, so an employee leaving means rotating it everywhere or accepting they still have access; and everyone who knows it can decrypt others' traffic on WPA2.
* **WPA2/WPA3-Enterprise (802.1X)** - each user authenticates individually against a RADIUS server with their own credentials or certificate. Per-user keys, per-user revocation, proper accounting. **This is what any organisation should use**, and the individual-credential property is the reason.

### WPS — turn it off

Wi-Fi Protected Setup was meant to simplify connection with an 8-digit PIN. The design validates the PIN in **two halves**, and the last digit is a checksum — so the search space collapses from 10^8 to about **11,000 attempts**. Brute-forceable in hours. **Disable WPS.** It's a permanent bypass of whatever strong passphrase you chose.

## 8.2 RFID, NFC, and Bluetooth

### RFID

Radio Frequency Identification. Tag plus reader, used for access badges, inventory, tolls.

* **Passive tags** - no battery, powered by the reader's field. Short range, cheap.
* **Active tags** - battery powered, longer range.

Security issues: **most low-frequency access badges have no authentication or encryption at all** — they simply broadcast a static ID. Cloning is trivial with a cheap reader (Proxmark, Flipper Zero), and badge cloning is a standard physical pentest technique (chapter 3.2).

Defences: cryptographic challenge-response cards (MIFARE DESFire, iCLASS SE) instead of static-ID cards, shielded wallets, and — importantly — **the badge should be one factor, not the only one**, for anything sensitive.

### NFC

Near Field Communication. Very short range (~4cm), used for contactless payment, transit, pairing.

The short range is the primary control, and it's not as strong as it sounds:

* **Eavesdropping** at greater distance than expected with a good antenna.
* **Relay attack** - two attackers relay the signal between a legitimate card and a legitimate reader across a distance. Nothing is "cracked" — the *proximity assumption* is broken. This is how keyless car theft works: one device near the house captures the key fob signal, another near the car relays it.
* **Skimming** - unauthorised reads from a card in a pocket.

Payment NFC is actually reasonably well protected by **tokenization** (chapter 7.2) — the terminal receives a one-time token, not your actual card number.

### Bluetooth

* **Bluetooth Classic** vs **BLE (Low Energy)**. BLE is everywhere in IoT.
* Pairing methods: Just Works (no authentication — convenient, MITM-able), Numeric Comparison, Passkey Entry, Out of Band.

Named attacks:
* **Bluejacking** - sending unsolicited messages. Nuisance.
* **Bluesnarfing** - unauthorised **data theft** from the device (contacts, messages).
* **Bluebugging** - taking **control** of the device.
* **BlueBorne** - a set of vulnerabilities allowing takeover without pairing.
* **KNOB (Key Negotiation of Bluetooth)** - forcing negotiation down to a 1-byte encryption key, which is then trivially brute-forced. A textbook **downgrade attack** (chapter 2.3).

Defences: turn Bluetooth off when unused, set devices non-discoverable, use current firmware, avoid Just Works pairing for anything sensitive, and unpair devices you no longer use.

## 8.3 Wi-Fi Coverage and Performance

Availability is a third of the triad, and wireless coverage is where availability meets physics.

### Bands

* **2.4 GHz** - longer range, better wall penetration, only 3 non-overlapping channels (1, 6, 11), heavily congested (microwaves, Bluetooth, cordless phones, baby monitors all sit here).
* **5 GHz** - shorter range, worse penetration, many more non-overlapping channels, much less interference.
* **6 GHz (Wi-Fi 6E)** - even more spectrum, shortest range.

### Site survey and heat maps

Walking the space measuring signal strength, then mapping it. Used to place APs, choose channels, and find dead spots.

**Security-relevant, and this is the part people miss:** a heat map shows **where your signal spills outside the building**. Coverage reaching the car park or the street means an attacker can sit there comfortably and attack your network without ever entering the premises. Tuning transmit power down so coverage stops at the walls is a genuine security control, not just a performance tweak.

### Other factors

* **Channel overlap and co-channel interference** — neighbouring APs on the same channel compete.
* **AP placement and density** — too few gives dead spots, too many creates interference and constant roaming.
* **Antenna types** — omnidirectional radiates in all directions; **directional/Yagi** focuses the signal, useful both to shape coverage deliberately and, from the attacker's side, to reach a network from much further away.
* **MIMO / MU-MIMO** and beamforming for capacity.

### An aside from my own project

My wifi-radar project reads RSSI from `/proc/net/wireless` and watches its variance to detect motion — because a body moving nearby perturbs the multipath environment and changes the received signal.

That's a **side channel worth noting in a security context**: signal characteristics leak information about the physical environment, with no sensor on the target and no access to any device. Research systems have gone considerably further with per-subcarrier CSI. It's a reminder that RF emissions carry more than the payload, which is the same underlying idea as TEMPEST shielding in chapter 3.4.

## 8.4 Wi-Fi Discovery and Attacks

### Discovery

* **Wardriving** - driving around mapping networks (and warwalking, warflying). Tools: Kismet, airodump-ng, WiGLE.
* **SSID broadcast** - APs advertise their name in beacon frames.
* **Hiding the SSID is not a security control.** The name still appears in probe requests and association frames from any connecting client, so anyone passively listening recovers it in seconds. Worse, hidden networks make **clients** noisier — they actively probe for the hidden SSID everywhere they go, broadcasting where you work to anyone listening.
* **MAC filtering is not a security control either.** MACs are transmitted in cleartext in every frame, so an attacker sniffs an allowed MAC and spoofs it.

Both of these are worth being firm about, because they're widely recommended and they're security theatre.

### Attacks

* **Rogue access point** - an unauthorised AP on your network. Frequently not malicious — an employee plugging in a consumer AP for convenience creates an unmanaged backdoor past all your perimeter controls.
* **Evil twin** - an AP impersonating a legitimate SSID, often with stronger signal so clients prefer it. Clients associate automatically because they remember the network name. The attacker is then MITM on everything.
* **Deauthentication attack** - 802.11 management frames were historically **unauthenticated**, so an attacker can forge deauth frames and forcibly disconnect clients. Used as a DoS, and used to force reconnection so the handshake can be captured (8.5). **PMF (Protected Management Frames), mandatory in WPA3, fixes this.**
* **Karma / probe response attack** - devices probe for remembered networks; the attacker answers "yes, I'm that network" to whatever is asked for. Defence: delete saved networks you don't need and disable auto-connect.
* **Captive portal attacks** - fake login portals harvesting credentials.
* **Jamming** - RF denial of service. Illegal essentially everywhere.
* **IV attacks** - the WEP break.
* **Bluetooth attacks** - 8.2.

### The evil twin problem specifically

This is the attack that best illustrates the wireless trust problem. **A client cannot easily tell a legitimate AP from an impostor**, because SSID and MAC are both trivially spoofable and there's no authentication of the AP in WPA2-Personal.

WPA2/WPA3-Enterprise with **certificate validation** does solve it — the client verifies the RADIUS server's certificate, and an evil twin can't produce a valid one. But it only works if clients are configured to *actually validate* the certificate and not just click through the warning, which is exactly the same failure mode as browser certificate warnings (chapter 7.12).

For users, a VPN on untrusted wireless is the practical mitigation — even if you associate with an evil twin, the tunnel is encrypted end to end.

## 8.5 Cracking WPA2

Understanding the attack is what makes the defence make sense.

### The 4-way handshake

When a client associates with a WPA2-PSK network:

1. The passphrase plus SSID are run through PBKDF2 (4096 iterations) to derive the **PMK (Pairwise Master Key)**. Note the SSID is the salt — which is why common SSIDs like `linksys` have precomputed tables.
2. AP and client exchange nonces (ANonce, SNonce).
3. Both derive the **PTK (Pairwise Transient Key)** from PMK + both nonces + both MAC addresses.
4. A **MIC (Message Integrity Check)** in the handshake proves each side derived the same key.

### The attack

Everything needed to *verify* a passphrase guess is present in the captured handshake, and none of it requires talking to the network:

1. **Capture the 4-way handshake** — either wait for a client to connect, or **deauthenticate** one to force a reconnect (8.4).
2. **Guess offline.** For each candidate passphrase: derive PMK, derive PTK, compute MIC, compare to the captured MIC. Match means the passphrase is correct.

The critical property: **this is entirely offline.** No rate limiting, no lockout, no interaction with the AP, no detection possible. GPU acceleration means billions of guesses.

This is precisely the online-versus-offline distinction from chapter 2.4 — once it's offline, every network-side control is irrelevant and only passphrase entropy protects you.

### PMKID attack

A more recent variant that's worse for defenders: on some APs the **PMKID** can be requested directly from the AP with **no client present at all**. No waiting, no deauthentication, no handshake capture — just ask the AP and start cracking.

### What actually determines success

**Passphrase entropy, and nothing else.**

* `password123` or anything in a wordlist → seconds.
* A dictionary word plus digits → minutes to hours (hybrid rules, chapter 2.4).
* 20+ random characters or a 5-word random passphrase → computationally infeasible.

The encryption is not broken. AES-CCMP is fine. **The password is the whole attack surface.**

### Why WPA3 kills this

SAE requires a **live interaction with the AP per guess**. There's no offline verification material, so an attacker gets rate-limited, detectable, network-speed guessing instead of offline GPU-speed guessing. That's the difference between "crackable overnight" and "not happening".

## 8.6 Wi-Fi Hardening

Practical checklist, roughly in order of value:

1. **Use WPA3** where supported; WPA2-AES/CCMP as the minimum. **Disable WEP, WPA, and TKIP entirely** — leaving them enabled for compatibility drags the whole network down to their level.
2. **Use Enterprise (802.1X) rather than PSK** in any organisation. Individual credentials, individual revocation, real accounting. With **EAP-TLS** (certificates both ways) if you can manage the PKI, PEAP with proper server certificate validation otherwise.
3. **If you must use PSK, make the passphrase long and random.** 20+ characters. This is the single variable that determines whether 8.5 succeeds.
4. **Disable WPS.** Non-negotiable — it bypasses everything above it.
5. **Enable PMF (Protected Management Frames)** to block deauthentication attacks.
6. **Change default admin credentials** on the AP itself, and keep its firmware updated.
7. **Segment wireless** — guest networks fully isolated with internet-only egress, IoT on its own VLAN, corporate wireless treated as untrusted and required to authenticate to resources anyway (zero trust, chapter 7.16).
8. **Enable client isolation** on guest networks so devices can't reach each other.
9. **Tune transmit power** so coverage stops at the building. Conduct a site survey and look at where the signal spills.
10. **Deploy a WIDS/WIPS** to detect rogue APs, evil twins and deauth floods.
11. **Rotate the PSK** when staff leave, if you're stuck with PSK.
12. **Disable unused radios and bands.**

### What not to bother with

* **Hiding the SSID** — recovered in seconds, and it makes your clients broadcast your network name everywhere they go.
* **MAC filtering** — MACs are cleartext in every frame and spoofing takes one command.

Both cost administrative effort, provide effectively no security, and create false confidence, which is worse than doing nothing.

## 8.7 Lab — WPA2 Cracking

Against **my own** access point, in my own lab. Doing this to a network you don't own is a criminal offence in essentially every jurisdiction — this is the same authorisation boundary as chapter 5.7, and it's not a grey area.

```bash
# 1. put the adapter into monitor mode
sudo airmon-ng check kill
sudo airmon-ng start wlan0            # becomes wlan0mon

# 2. survey the area — find BSSID, channel, connected clients
sudo airodump-ng wlan0mon

# 3. capture on the target's channel, writing to file
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon

# 4. force a reconnect to capture the handshake
#    (in a second terminal; -0 is deauth, 4 packets, -c targets one client)
sudo aireplay-ng -0 4 -a AA:BB:CC:DD:EE:FF -c 11:22:33:44:55:66 wlan0mon

#    airodump shows "WPA handshake: AA:BB:CC:DD:EE:FF" in the top right when captured

# 5. crack offline against a wordlist
aircrack-ng -w /usr/share/wordlists/rockyou.txt -b AA:BB:CC:DD:EE:FF capture-01.cap

# faster, with hashcat on a GPU
hcxpcapngtool -o hash.hc22000 capture-01.cap
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt
```

### What the lab actually taught me

Three things that reading about it didn't convey:

1. **The deauth step is trivially easy and completely undetectable to the user.** From the victim's side, the Wi-Fi blipped for two seconds and reconnected. That's it. There's no alert, no log on a consumer AP, nothing. The attacker got what they needed from those two seconds.

2. **Steps 1–4 take under two minutes.** The entire "attack" is the *waiting* in step 5, and that waiting is determined purely by passphrase strength. I set a deliberately weak passphrase from rockyou for the lab and it fell in seconds. Setting a 24-character random one and re-running made it obviously hopeless. **Same encryption, same tools, same handshake — the only variable was the passphrase.**

3. **Nothing here attacks AES.** The cipher is never touched. This is chapter 2.3's lesson made physical: the algorithm isn't the weak point, the key material is.

The defensive conclusions follow directly — long random passphrase, or better, WPA3/SAE so there's no offline verification material at all, plus PMF so step 4 doesn't work in the first place.

---

## Chapter 8 — what I'd take away

* The medium is public. Everything else follows from not being able to choose who receives your transmission.
* WEP died from a 24-bit IV, not from RC4 alone — an implementation decision, not a maths failure.
* WPA2's encryption is fine; its **PSK handshake allows offline cracking**, and offline means no rate limit and no detection.
* WPA3/SAE removes offline cracking entirely by requiring live interaction per guess. That's the real upgrade.
* Enterprise (802.1X) over Personal (PSK), always, in an organisation — individual credentials and individual revocation.
* Disable WPS. It bypasses whatever passphrase you chose.
* Hiding SSIDs and MAC filtering are security theatre that make clients noisier.
* Signal spilling into the car park is an attack surface decision, and transmit power is a security control.
