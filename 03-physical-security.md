# Chapter 3 — Physical Security

## 3.1 Overview

Easy to skip as a software person, and a mistake to do so, because **physical access beats most logical controls you have**.

If I can get my hands on your server I can pull the disk. If I get five minutes alone with your unlocked laptop I can install anything. If I can plug a device into a USB port I can emulate a keyboard and type commands faster than you can read them (my own Rubber Ducky project is precisely this).

Encryption at rest is one of the very few controls that still helps once someone physically has the hardware — and even that depends on the machine being powered off rather than suspended, since the key is in RAM on a running machine.

The mental model: physical security is the **outermost layer of defence in depth**, and every logical control you built assumes it is holding.

## 3.2 Physical Security Controls

### Perimeter and barriers

* **Fencing** - keeps people out and marks the boundary. Height and gauge determine whether it's a deterrent or an actual barrier.
* **Bollards** - short posts that stop *vehicles*. Specifically there to prevent someone driving through a lobby or into a loading bay. You see them outside every data centre and government building.
* **Lighting** - genuinely one of the highest value-per-rupee controls in the whole list. Attackers dislike being visible, and cameras are useless in the dark.
* **Signage** - pure deterrent. "Restricted area — CCTV in operation" changes behaviour without physically stopping anyone. Also has a legal function, since it removes the "I didn't know I couldn't be here" defence.

### Doors and entry

* **Locks** - mechanical (keys, which get copied and never returned), electronic (badge/PIN, revocable instantly and they log), or biometric.
* **Access badges** - cheap, instantly revocable, and they generate **accounting** data — you know who went where and when. Downside: RFID badges are cloneable, and badge cloning is a standard physical pentest technique. A reader that only checks the card ID and not a cryptographic challenge is trivially defeated.
* **Access control vestibule (mantrap)** - two interlocking doors where the second won't open until the first has closed and latched. This is *the* specific defence against tailgating.
* **Turnstiles** - the lighter-weight version, one person per authentication.

**Tailgating vs piggybacking** — tailgating is following an authorised person through without their knowledge or consent; piggybacking is when they knowingly hold the door for you. The technical control is the same, but piggybacking is a *training* problem, because the authorised person is choosing politeness over policy and nobody wants to be the person who slams a door in a colleague's face.

### Guards and people

* **Security guards** - the only control that exercises judgement, which is both the strength and the weakness. They can spot a situation no sensor is configured for, and they can also be socially engineered, which is why guard training on social engineering is standard.
* **Two-person integrity / dual control** - no single person can access the critical thing alone. Standard for nuclear material, bank vaults, and root CA key ceremonies.
* **Visitor management** - sign-in, escort, visible badges, and the escort actually escorting rather than pointing down a corridor.

### Surveillance

* **Video surveillance / CCTV** - **detective, not preventive.** It tells you what happened; it does not stop it happening. Cameras get treated as prevention because the *deterrent* effect is real, but the control itself is detective.
* **Motion recognition and object detection** - pushes cameras toward real-time alerting rather than after-the-fact review, which matters because nobody is watching 200 feeds live.
* Retention of footage is a policy decision with both security and privacy implications.

### Sensors

Worth knowing what each physically detects, because they fail in different ways and get combined to cover each other:

| Sensor | Detects | Fails on |
|---|---|---|
| **Infrared** | Body heat | Cold objects, blocked line of sight |
| **Pressure** | Weight on a pad / floor | Light objects, being stepped over |
| **Microwave** | Motion via reflected radio waves | Passes through some walls → false positives |
| **Ultrasonic** | Motion via reflected sound | Air currents, ambient noise |

### Other

* **Faraday cage** - blocks electromagnetic signals. Used both to stop RF emissions leaking *out* (TEMPEST/emanation security) and to stop signals getting in.
* **Secure data destruction** - covered in chapter 1.10; the physical side is the shredder and the degausser.
* **Air gap** - complete physical isolation from other networks. The strongest control there is, and Stuxnet is the standing proof that it's defeatable by removable media and by humans who need to move data across.

## 3.3 Keyloggers

Worth its own section because it's the clearest example of "physical access defeats everything".

### Types

* **Hardware keylogger** - a small device inline between the keyboard and the machine, or hidden inside the keyboard housing. Looks like an adapter. **The OS cannot see it at all** — there's no process, no file, no registry key, no network connection. Antivirus and EDR are blind to it by construction. Some store to internal flash for later collection, others exfiltrate over Wi-Fi.
* **Software keylogger** - a program capturing keystrokes. Detectable by endpoint tooling, and much easier to deploy since it doesn't require physical access.
* **Firmware/BIOS level** - between hardware and software, survives OS reinstall.

### Why the hardware one is the instructive case

It captures **everything typed before any software runs** — including the full-disk encryption passphrase at boot, which is the one secret the disk encryption was supposed to protect. All your careful key derivation and AES-256 is bypassed by a ₹2,000 dongle, because the attacker isn't attacking the cryptography at all, they're reading the input.

This is the physical-world version of the side channel idea from 2.3: attack the implementation and its environment, not the maths.

### Detection and defence

* **Physically inspect ports.** That's genuinely the primary detection method for hardware keyloggers, and it's the reason data centres and secure facilities do port inspections.
* **Port blockers / epoxy** in the USB ports of high-value machines.
* **Locked cases** so internal devices can't be planted.
* **Full-disk encryption with TPM + Secure Boot** so a keylogged passphrase alone isn't enough — the TPM binds the key to *this machine in a known-good state*.
* **MFA**, again — a keylogged password is useless without the second factor.
* **On-screen keyboards** for high-value credential entry defeat hardware keyloggers specifically (though not screen-capturing malware).

## 3.4 Environmental Controls

The availability leg of the triad, at the physical layer. A server that overheats is as unavailable as one that's been ransomwared, and this is the category people neglect because it feels like facilities management rather than security.

### Temperature and airflow

* **HVAC** - heating, ventilation and air conditioning. Server hardware throttles and then fails at sustained high temperature. HVAC failure in a full rack room causes thermal shutdown within minutes, not hours.
* **Hot aisle / cold aisle** - the standard data centre layout. Racks are arranged so all equipment intakes face one aisle (cold, fed by the CRAC units) and all exhausts face the other (hot, returned to the coolers). Without this you get recirculation — hot exhaust drawn straight back into the intake next to it — and cooling efficiency collapses.
* **Containment** - physically separating the aisles with doors and roof panels so the two air masses don't mix at all.

**Security relevance:** HVAC systems are increasingly network-connected building management systems, which makes them an attack surface *into* the facility. The Target breach began through an HVAC vendor's credentials. So the air conditioning is both an availability control and a third-party risk.

### Humidity

Narrower band than people expect, roughly 40–60%.

* **Too low** → static electricity builds up, and an electrostatic discharge into a component destroys it silently.
* **Too high** → condensation and corrosion.

### Power

* **UPS (Uninterruptible Power Supply)** - battery, covers brief outages and — more importantly — gives enough runtime for a **clean shutdown** rather than an abrupt power cut mid-write.
* **Generators** - for long outages. They need fuel contracts and **regular test runs under load**, both of which get skipped, and a generator that hasn't been load-tested in two years is decoration.
* **PDUs (Power Distribution Units)**, dual power supplies, and dual feeds from separate substations for real redundancy.
* **Power problems worth naming:** *blackout* (total loss), *brownout* (sustained low voltage, which is harder on equipment than a clean outage), *sag* (brief dip), *spike/surge* (brief over-voltage).

### Fire suppression

The important part: **water destroys the thing you're protecting.** So server rooms use alternatives.

* **Clean agent systems** (FM-200, Novec 1230, inert gas blends) - suppress fire without residue and without damaging electronics. Modern agents are chosen to be safe for people in the room; older Halon was not, and is phased out for ozone reasons anyway.
* **Pre-action sprinkler systems** - pipes stay dry until a detector triggers, so a single burst pipe doesn't flood the racks. Requires two triggers (detection *and* sprinkler head activation) before water flows.
* **Fire classes** - electrical fires are Class C (US) / Class E (elsewhere), which is why you want CO₂ or clean agent extinguishers in a server room, not water or foam.
* **Detection** - smoke, heat, and flame detectors, ideally with very early warning aspirating systems in high-value rooms.

### Shielding and interference

* **EMI/RFI shielding** - protects cabling from electromagnetic interference that corrupts signals. Also stops emissions leaking outward, which is the TEMPEST concern — enough EM leakage from a monitor or cable can in principle be reconstructed remotely.
* **Faraday cages** for spaces where that matters.

### Monitoring

All of the above needs sensors reporting into something that alerts. Temperature, humidity, water detection under raised floors, door position, and power state. The failure mode here is a rack thermometer that nobody's dashboard is watching.

## 3.5 Lab — Physical Security Assessment

The course lab walks a facility. The useful framing is to walk your own space as an attacker and ask a fixed set of questions:

1. **Can I get in without authenticating?** Propped doors, tailgating opportunity, a loading bay that's always open, a smoking-area door that doesn't latch.
2. **Once in, what's unattended?** Unlocked workstations, unlocked server cabinets, network ports live in meeting rooms, printers holding queued documents.
3. **What's plugged in that shouldn't be?** Unknown USB devices, unfamiliar dongles between keyboard and tower, a device on a network port under a desk.
4. **What's visible?** Passwords on monitors and under keyboards, screens facing windows or corridors, whiteboards with architecture on them, badge photos that make a convincing fake easy.
5. **What's in the bin?** Paper that should have been shredded, discarded hardware, media.
6. **What would a camera actually have captured?** Coverage gaps, poor lighting at the point of entry, cameras that record but nobody reviews.

For my own setup the honest findings were: full-disk encryption is on (good), but the laptop suspends rather than shuts down, so the key sits in RAM — a cold-boot or DMA attack against a suspended machine is real, and "lock the screen" is not the same as "power off". And USB ports are wide open, which for a personal machine is a deliberate acceptance rather than an oversight — but per 1.4 it only counts as acceptance if I've actually decided it, which is the point of writing it down here.

---

## Chapter 3 — what I'd take away

* Physical access defeats most logical controls. Everything else assumes this layer holds.
* Cameras are detective, not preventive, however much they feel like prevention.
* Tailgating is a technical control problem; piggybacking is a training and culture problem.
* A hardware keylogger is invisible to every piece of software you own — inspection is the detection method.
* Environmental controls are availability controls, and HVAC being network-connected makes them an attack surface too.
* Water is the wrong fire suppressant for the room you're trying to protect.
