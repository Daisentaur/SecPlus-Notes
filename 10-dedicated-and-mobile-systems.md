# Chapter 10 — Dedicated and Mobile Systems

## 10.1 Embedded Systems

A purpose-built computer inside a larger device. Not a general-purpose machine — it does one job, usually for a very long time.

Everywhere: cars, medical devices, routers, printers, cameras, appliances, industrial equipment, aircraft.

### Components

* **Microcontroller (MCU)** - CPU, memory and I/O on one chip. Arduino, ESP32, STM32.
* **SoC (System on Chip)** - a full system including CPU, GPU, memory controller and peripherals. Raspberry Pi, phones.
* **FPGA (Field Programmable Gate Array)** - reconfigurable hardware. Logic defined after manufacture, so it can be updated — and that reprogrammability is itself an attack surface.
* **ASIC** - fixed-function custom silicon. Fast, efficient, unchangeable.
* **RTOS (Real-Time Operating System)** - guarantees response within a hard time bound. FreeRTOS, VxWorks, QNX. Used where **timing is a safety property** — airbag deployment, pacemakers, flight controls.

### Why RTOS matters for security

The timing guarantee *is* the safety guarantee. A security control that adds unpredictable latency is unacceptable, because a late response in a real-time system is a failed response. That constrains what you can bolt on — you can't just add a scanner that occasionally takes 200ms.

### Security characteristics

Generally poor, for structural reasons rather than negligence:

* **Constrained resources.** Limited CPU, memory and power means limited room for crypto, logging or an agent.
* **No secure boot** on many devices, so modified firmware runs happily.
* **No update mechanism at all**, or one that requires physical access, or one nobody ever uses. A vulnerability found in 2019 is still live in 2026.
* **Hardcoded credentials** baked into firmware — and since every unit ships identical firmware, extracting it from one device compromises all of them.
* **Exposed debug interfaces** — UART, JTAG, SWD headers left populated on production boards. Physical access to these frequently yields a root shell or a firmware dump.
* **Unsigned firmware** so anything can be flashed.
* **Long lifecycles** — 10 to 30 years, far outlasting vendor support.

### Mitigations

Secure boot and signed firmware where supported, disable or physically remove debug interfaces, unique per-device credentials, hardware root of trust, and — because you often can't fix the device — **network segmentation and tight egress control as the compensating control.** That's the recurring answer for everything in this chapter.

## 10.2 Industrial Control Systems

**ICS** is the family; **SCADA (Supervisory Control and Data Acquisition)** is the class that monitors and controls physical processes across distributed sites — power grids, water treatment, pipelines, manufacturing, building management.

### Components

* **PLC (Programmable Logic Controller)** - the device actually controlling machinery. Reads sensors, drives actuators.
* **RTU (Remote Terminal Unit)** - similar, built for remote/distributed sites.
* **HMI (Human Machine Interface)** - the operator's screen.
* **Historian** - records process data over time.
* **DCS (Distributed Control System)** - process control within a single plant.

### Why OT security is a genuinely different discipline

This is the part worth understanding properly rather than as a list of facts.

1. **The CIA priority inverts.** In IT, confidentiality usually leads. In OT, **safety and availability lead by a wide margin**. A leaked temperature reading is nothing; a turbine that stops responding is a physical safety event. The IT instinct — "isolate it and shut it down while we investigate" — can be the actively *dangerous* choice here.

2. **You cannot patch on a normal schedule.** Downtime windows may be annual, or require shutting a production line at enormous cost. Some systems have run without reboot for years. Known vulnerabilities therefore stay live far longer than anyone is comfortable with, and **compensating controls** (chapter 1.5) do the work that patching would.

3. **The protocols have no security whatsoever.** Modbus, DNP3, Profinet, BACnet were designed for isolated, trusted, serial networks. **No authentication, no encryption, no integrity checking.** A validly-formatted command is obeyed, because the protocol has no concept of an invalid sender. You don't need an exploit — you just need to reach the device and speak the protocol.

4. **Lifecycles are 20–30 years.** Equipment installed in 2001 still runs, on operating systems that reached end of life a decade ago, and can't be upgraded without replacing physical plant.

5. **The air gap mostly isn't real anymore.** IT/OT convergence, remote vendor maintenance access, and the business wanting production data in dashboards have all put holes in the isolation these systems were designed to assume.

6. **Safety systems are separate for a reason.** SIS (Safety Instrumented Systems) exist to bring a process to a safe state independently. TRITON/TRISIS malware specifically targeted a safety controller, which is a meaningful escalation — attacking the thing that prevents people dying.

### The Purdue model

The reference architecture for segmenting OT, levels 0–5:

* **Level 0** - physical process (sensors, actuators)
* **Level 1** - basic control (PLCs, RTUs)
* **Level 2** - area supervisory control (HMIs)
* **Level 3** - site operations (historians, scheduling)
* **DMZ** - the boundary between OT and IT
* **Level 4/5** - enterprise IT

**The DMZ between levels 3 and 4 is the critical control.** Data flows up to the business; commands should not flow down. **Data diodes** enforce this in hardware — physically unidirectional, so information can leave and nothing can enter.

### Case studies

* **Stuxnet (2010)** - crossed an air gap via USB, targeted specific Siemens PLCs, manipulated centrifuge speeds while **replaying recorded normal readings to the operators**, so the HMI showed everything was fine. The canonical demonstration that OT attacks cause physical destruction and that the operator's view can be lied to.
* **Ukraine grid (2015, 2016)** - remote attackers opened breakers and cut power to hundreds of thousands, then wiped systems to slow recovery and DDoSed the call centres so customers couldn't report outages.
* **Colonial Pipeline (2021)** - ransomware hit the **IT billing** systems, and the pipeline was shut down as a precaution because the operator couldn't be confident the OT side was unaffected. The lesson is about IT/OT convergence: the OT network didn't have to be breached for OT to stop.
* **Oldsmar water treatment (2021)** - remote access software used to alter sodium hydroxide levels. Caught by an operator watching the screen.

### Defences

Segmentation per the Purdue model, data diodes, OT-aware monitoring (passive, because active scanning can crash fragile PLCs — a genuine constraint IT people get wrong), strict control of vendor remote access, allow-listing rather than antivirus, and physical security.

## 10.3 Internet of Things Devices

Consumer and commercial connected devices — cameras, thermostats, doorbells, TVs, speakers, wearables, medical devices, sensors.

### Why security is consistently bad

Structural incentives, not incompetence:

* **Cost pressure.** Margins on a ₹1,500 camera don't fund a security programme.
* **Time to market** beats security review.
* **No update path**, or one requiring an app nobody opens.
* **Default credentials**, often identical across the entire product line and documented publicly.
* **Unnecessary services** enabled — Telnet, UPnP, open debug ports.
* **Cleartext protocols** and weak or absent cloud API authentication.
* **Vendor abandonment.** The company folds or moves on and the device keeps running forever.
* **No visibility.** The owner cannot see what it does, cannot install anything on it, cannot inspect it.

### Mirai

The case study. It scanned the internet for devices accepting Telnet and tried a list of roughly **60 default credential pairs**. That's the entire technique — no exploit, no vulnerability, just defaults nobody changed. It built a botnet large enough to take down Dyn and with it a large slice of the western internet (chapter 9.3).

The lesson: **the aggregate risk of millions of trivially-compromised devices is a systemic problem**, even though each individual device is worthless as a target.

### Mitigations

Since you cannot harden the device itself, the answer is architectural:

* **Segment IoT onto its own VLAN** with **tight egress control**. Monitor what it talks to, and treat deviation as an incident.
* **Change default credentials** immediately.
* **Disable UPnP** on the router — it lets devices open firewall ports without asking you.
* **Update firmware** where possible.
* **Consider whether it needs internet access at all.** A great many "smart" devices work fine with outbound blocked, and blocking it removes the whole remote attack surface.
* **Buy from vendors with a stated support lifetime**, and treat end-of-support as end-of-life.
* **Inventory them** — you can't protect what you don't know is there, and IoT is where shadow devices accumulate.

## 10.4 Connecting to Dedicated and Mobile Systems

The connectivity methods, each with its own exposure:

* **Cellular (4G/5G)** - wide area, carrier-managed. Bypasses corporate network controls entirely, which is the security concern — a device on cellular isn't behind your filtering. **5G** adds network slicing and better subscriber identity protection over 4G.
* **Wi-Fi** - chapter 8.
* **Bluetooth / BLE** - short range, ubiquitous in IoT. Attacks in 8.2.
* **NFC / RFID** - very short range. 8.2.
* **Zigbee / Z-Wave / Thread** - low-power mesh protocols for home and building automation. Mesh means one compromised node reaches the rest, and pairing/key exchange is the usual weak point.
* **LoRaWAN** - long range, very low power, for sensor networks.
* **Infrared** - line of sight, largely legacy.
* **USB** - wired, and a malicious HID device is a keyboard as far as the OS is concerned (chapters 3.3, 6.3).
* **GPS** - receive-only positioning. **Spoofable**, because civilian GPS signals are unauthenticated — a stronger local transmitter overrides the satellites. Relevant for anything using location as a security signal.
* **SCADA/serial** - RS-232/485, Modbus over serial. No security by design.

### The general point

Every additional radio is an additional attack surface that **does not pass through your network perimeter**. A device with cellular, Wi-Fi and Bluetooth has three independent paths in, and only one of them is one you can filter. Disabling unused radios is the same "reduce attack surface" principle from chapter 6.2, applied at the physical layer.

## 10.5 Security Constraints for Dedicated Systems

Why you can't just apply normal IT security to these devices. Worth knowing as a set, because it explains why the answer is always compensating controls.

* **Power** - battery-operated devices can't afford continuous crypto or radio use.
* **Compute and memory** - not enough headroom for an EDR agent, or sometimes even for modern TLS.
* **Cryptographic capability** - some MCUs have no hardware crypto acceleration, making strong encryption prohibitively slow.
* **Network capability** - low-bandwidth links (LoRaWAN, cellular IoT) can't carry telemetry or large updates.
* **Inability to patch** - no mechanism, or patching requires downtime that isn't available, or the vendor no longer exists.
* **Authentication limits** - no screen, no keyboard, so no interactive login and no MFA.
* **Range and physical exposure** - devices deployed in publicly accessible locations, where physical attack is realistic.
* **Cost** - security features that would double the unit price won't ship.
* **Implied trust** - the system was designed assuming a trusted network, and that assumption is now false.

**The consequence:** security has to be applied *around* the device rather than *on* it. Segmentation, monitoring, egress filtering, physical protection and network-level authentication — because the device itself cannot participate in its own defence.

## 10.6 Mobile Device Deployment and Hardening

### Deployment models

| Model | Who owns it | Control | Privacy | Notes |
|---|---|---|---|---|
| **BYOD** (Bring Your Own Device) | Employee | Lowest | Best for user | Cheapest; **you have security obligations over hardware you don't own** |
| **COPE** (Corporate Owned, Personally Enabled) | Company | High | Some personal use permitted | Common enterprise compromise |
| **CYOD** (Choose Your Own Device) | Company | High | Employee picks from a list | Standardised support |
| **COBO** (Corporate Owned, Business Only) | Company | Highest | None | High-security environments |
| **VDI** | Company | High | Data never lands on the device | Device becomes a viewing terminal |

**BYOD is the difficult one.** You need to protect corporate data on a device you don't own, can't fully control, and can't legally wipe entirely. Remote-wiping an employee's personal phone including their family photos is a lawsuit. Hence **containerization** — a managed, encrypted work profile that can be wiped independently of personal data. Android Work Profile and iOS managed apps do this.

### Management tooling

* **MDM (Mobile Device Management)** - device-level: enrollment, policy, remote lock and wipe, compliance checks.
* **MAM (Mobile Application Management)** - application-level, more appropriate for BYOD since it controls corporate apps without controlling the whole device.
* **UEM (Unified Endpoint Management)** - one console for mobile, desktop and IoT.
* **MDM policies worth knowing:** passcode requirements, encryption enforcement, screen lock timeout, jailbreak/root detection, app allow/deny lists, certificate deployment, per-app VPN, disabling camera or removable storage in sensitive areas, and geofencing.

### Mobile-specific threats

* **Jailbreaking / rooting** - removing the OS security model wholesale. Breaks app sandboxing, disables signature verification, and invalidates every assumption MDM makes. Detection is a cat-and-mouse game since jailbreaks actively hide.
* **Sideloading** - installing outside the official store, bypassing review. The single largest source of mobile malware on Android.
* **Malicious apps** in official stores — review catches a lot, not everything.
* **Excessive app permissions** - a torch app requesting contacts and location.
* **Insecure storage** - apps writing sensitive data unencrypted to shared storage.
* **Unpatched OS** - Android fragmentation means many devices stop receiving updates within 2–3 years while remaining in use.
* **Public Wi-Fi and evil twins** (chapter 8.4) - mitigated by always-on VPN.
* **Juice jacking** - malicious USB charging ports delivering data or power-only attacks. Mitigated by a USB data blocker or carrying your own charger.
* **SIM swapping** - social engineering the carrier to port a number, defeating SMS-based MFA. This is the concrete reason SMS is the weakest MFA factor (chapter 4.2).
* **Loss and theft** - by far the most likely real incident. Encryption plus remote wipe plus screen lock covers it.

### Hardening checklist

Full-disk encryption on (default on modern iOS/Android) · strong passcode, not a 4-digit PIN · biometric unlock backed by a strong passcode · automatic screen lock · OS and app updates current · install only from official stores · review and revoke app permissions · disable unused radios · **always-on VPN on untrusted networks** · remote wipe configured and tested · no jailbreaking · backup encrypted · disable lock-screen notification previews for sensitive apps.

## 10.7 Lab — Smartphone Hardening

Walking my own device, and the useful part was noticing how much is off by default.

### Android

```
Settings → Security & privacy
  - Screen lock: strong passcode (not pattern — patterns are shoulder-surfable and smudge-recoverable)
  - Biometrics as convenience, passcode as the real credential
  - Auto-lock: 30 seconds
  - Encryption: verify enabled
  - Find My Device: on (remote lock and wipe)

Settings → Privacy
  - Permission manager: review every app's location/camera/mic/contacts
  - Remove permissions from unused apps (Android does this automatically now)
  - Disable ad personalisation, reset advertising ID

Settings → Apps
  - "Install unknown apps": OFF for every app (blocks sideloading)
  - Uninstall or disable bloatware

Settings → System → Developer options
  - OFF unless actively needed (USB debugging is a real exposure)

Settings → Notifications
  - Hide sensitive content on lock screen
```

### iOS

```
Settings → Face ID & Passcode
  - Alphanumeric passcode, not 6 digits
  - "Erase Data" after 10 failed attempts (consider carefully — it's a self-DoS if you have kids)
  - Disable lock-screen access to Control Centre, USB accessories, wallet

Settings → Privacy & Security
  - App privacy report: see what apps actually access
  - Location: "While Using" rather than "Always" wherever possible
  - Lockdown Mode for high-risk individuals

Settings → General → Software Update → Automatic Updates: on
```

### What stood out

1. **Permissions are the biggest real exposure.** Going through the permission manager, several apps had location and microphone access they had no functional need for, granted at install and never revisited. That's not a hypothetical — it's active data collection I had consented to without noticing.

2. **USB debugging / USB accessories on the lock screen is a genuine gap.** iOS's "USB Accessories" toggle specifically exists to block forensic-extraction tooling that connects over Lightning while locked. Leaving it enabled means a locked phone is still talking to whatever you plug in.

3. **Biometric versus passcode is a legal distinction as much as a technical one** in some jurisdictions — compelling a fingerprint has been treated differently from compelling a passcode. Worth knowing that the "strong passcode behind the convenient biometric" setup matters for reasons beyond entropy.

4. **The lock screen leaks a lot** by default — message previews, notification content, assistant access. All of that is readable without unlocking, which is a shoulder-surfing and stolen-device exposure that costs nothing to close.

---

## Chapter 10 — what I'd take away

* Embedded and OT devices generally can't defend themselves, so security is applied *around* them — segmentation, egress control, monitoring.
* In OT the CIA priority inverts: safety and availability outrank confidentiality, and the IT instinct to shut things down can be the dangerous option.
* ICS protocols (Modbus, DNP3) have no authentication at all — reaching the device is the whole attack.
* Stuxnet showed the operator's view can be lied to; Colonial showed OT can stop without OT being breached.
* Mirai needed no exploit — 60 default credential pairs and internet-facing Telnet.
* Every extra radio is an extra path in that doesn't cross your perimeter.
* BYOD means security obligations over hardware you don't own — containerization is the answer, not full-device wipe.
* SIM swapping is why SMS is the weakest MFA factor.
