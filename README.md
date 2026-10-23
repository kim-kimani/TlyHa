# Think Like a Hacker (TlyHa)

A refresher course for people who already hunt. This is not a "what is XSS" primer. The goal is to reload the attacker mindset, the tell-tale signs that a target is weak, and the techniques that separate someone who reports `id=1` to `id=2` from someone who finds the account takeover chain underneath it.

## How this is organized

Every file is named after the topic it covers. Each topic follows the same spine so you can skim fast:

1. **The mental model**: what the vulnerability actually is at the protocol or logic layer, not the textbook definition.
2. **Where it hides / tell-tales**: the signals an experienced tester reads off a target, including the ones most people walk past.
3. **Real material**: actual payloads, JSON bodies, JWTs, headers, and requests you can adapt.
4. **Beyond the basics**: the chains, bypasses, and edge cases that turn a low into a critical.
5. **What to watch for**: false positives, rabbit holes, and things that look exploitable but are not.
6. **Hunting checklist**: a short pass to run against any target.

## Directory map

### Methodology and mindset
- `00-methodology/Recon-and-Attack-Surface-Mapping.md`
- `00-methodology/The-Attacker-Mental-Model.md`

### Web
- `web/IDOR-and-BOLA.md`
- `web/Broken-Access-Control.md`
- `web/JWT-Attacks.md`
- `web/OAuth-and-OIDC.md`
- `web/SAML-and-SSO.md`
- `web/Session-and-Cookie-Security.md`
- `web/Password-Reset-and-Account-Recovery.md`
- `web/MFA-Bypass.md`
- `web/SSRF.md`
- `web/XXE.md`
- `web/SQL-Injection.md`
- `web/NoSQL-Injection.md`
- `web/Server-Side-Template-Injection.md`
- `web/Insecure-Deserialization.md`
- `web/Cross-Site-Scripting.md`
- `web/CSRF.md`
- `web/CORS-Misconfiguration.md`
- `web/File-Upload.md`
- `web/Path-Traversal-and-LFI.md`
- `web/Business-Logic-and-Race-Conditions.md`
- `web/HTTP-Request-Smuggling.md`
- `web/Web-Cache-Poisoning-and-Deception.md`
- `web/GraphQL.md`
- `web/REST-API-Security.md`
- `web/Prototype-Pollution.md`
- `web/Open-Redirect-and-URL-Parsing.md`
- `web/Subdomain-Takeover.md`
- `web/WebSockets.md`
- `web/Command-Injection.md`
- `web/WAF-and-Filter-Bypass.md`
- `web/Secrets-and-Information-Disclosure.md`
- `web/Clickjacking-and-UI-Redressing.md`
- `web/CRLF-and-HTTP-Response-Splitting.md`
- `web/Host-Header-Attacks.md`
- `web/LDAP-XPath-and-Less-Common-Injections.md`
- `web/XS-Leaks-and-Client-Side-Leaks.md`
- `web/Rate-Limiting-and-Anti-Automation.md`
- `web/Account-Takeover-Methodology.md`
- `web/Web-LLM-and-Prompt-Injection.md`
- `web/Dependency-Confusion-and-Supply-Chain.md`
- `web/Overlooked-and-Low-Hanging-Bugs.md`

### Mobile
- `mobile/Android-App-Testing.md`
- `mobile/iOS-App-Testing.md`
- `mobile/Mobile-API-and-Transport-Security.md`
- `mobile/Deep-Links-and-IPC.md`

### Cloud and infrastructure
- `cloud-and-infra/Cloud-Metadata-and-SSRF-to-Cloud.md`

The AWS cloud exploitation track lives in the sibling `AWS/` directory and goes much deeper on post-access cloud attacks.

### Hardware and embedded
Start-from-the-basics ordering (read `00` and `01` first):
- `hardware-embedded/00-Lab-Setup-and-Electronics-Basics.md`
- `hardware-embedded/01-Device-Assessment-Methodology.md`
- `hardware-embedded/Hardware-Hacking-Fundamentals.md`
- `hardware-embedded/Bus-Protocols-Deep.md`
- `hardware-embedded/Firmware-Analysis-and-Extraction.md`
- `hardware-embedded/Assembly-and-Reverse-Engineering.md`
- `hardware-embedded/Microcontrollers-and-Arduino.md`
- `hardware-embedded/Fault-Injection-and-Glitching.md`
- `hardware-embedded/Side-Channel-Analysis.md`
- `hardware-embedded/Secure-Boot-and-Chain-of-Trust.md`
- `hardware-embedded/IoT-and-Smart-Home-Devices.md`
- `hardware-embedded/IoT-What-Is-Actually-Hackable.md`
- `hardware-embedded/Radio-and-Wireless.md`
- `hardware-embedded/Automotive-and-CAN-Bus.md`
- `hardware-embedded/USB-and-Physical-Interface-Attacks.md`

Written for someone new to hardware: lab setup and electronics basics, a phased device-assessment playbook with explicit goals, the bus protocols (UART/SPI/I2C/JTAG/SWD) with pin-finding, firmware extraction and emulation, reversing (x86-64, ARM, MIPS, Xtensa, AVR), then the advanced layer (glitching, side channels, secure-boot bypass). Plus smart-home devices (TVs, fridges, washing machines, AC, cameras, locks), what is really hackable (and why), radio (Wi-Fi/BLE/Zigbee/Z-Wave/sub-GHz/NFC), automotive/CAN, and USB/physical-interface attacks.

#### Per-device IoT how-to guides: `hardware-embedded/iot/`
A detailed, step-by-step guide (with explicit aims) for each kind of equipment, following one shared method (local, companion app, cloud, hardware). Start with `iot/00-IoT-Device-Methodology.md`, then:
- `iot/Smart-TVs.md`, `iot/Smart-Light-Bulbs.md`, `iot/Robot-Vacuums.md`, `iot/IP-Cameras-and-Baby-Monitors.md`
- `iot/Smart-Locks-and-Garage-Doors.md`, `iot/Smart-Plugs-and-Switches.md`, `iot/Smart-Appliances-Fridge-Washer-AC.md`
- `iot/Network-Printers.md`, `iot/VoIP-Phones.md`, `iot/Alarm-Sensors-and-Home-Security.md`, `iot/Smart-Speakers-and-Voice-Assistants.md`
- `iot/Medical-Devices.md`, `iot/Smart-Meters.md`, `iot/Solar-Inverters.md`, `iot/EV-Chargers.md`
- `iot/HVAC-and-Building-Automation.md`, `iot/Access-Control-and-Badge-Readers.md`, `iot/GPS-and-Timing.md`
- `iot/Smart-Toys.md`, `iot/Wearables.md`, `iot/Intimate-Devices.md`, `iot/Air-Gap-Covert-Channels.md`

The medical, smart-meter, solar-inverter, EV-charger, building-automation, and GPS guides carry heavy safety/legal weight and are written for recognition, defense, and authorized/coordinated-disclosure work, never for touching systems in use. Cars are covered in `hardware-embedded/Automotive-and-CAN-Bus.md`.

### ICS and SCADA
Start-from-the-basics ordering, with a safe simulator lab (read `00` first):
- `ics-scada/00-ICS-Primer-and-Safe-Lab.md`
- `ics-scada/ICS-SCADA-Fundamentals.md`
- `ics-scada/Industrial-Protocols.md`
- `ics-scada/02-ICS-Assessment-Methodology.md`
- `ics-scada/PLC-and-HMI-Attacks.md`

Written for a newcomer and built around a safe, legal practice lab (OpenPLC, GRFICS, ModbusPal, Conpot) so you learn hands-on without touching real equipment. Covers the Purdue model, the IT/OT boundary, PLC/RTU/HMI/SIS, the unauthenticated industrial protocols (Modbus, DNP3, S7comm, EtherNet/IP, BACnet, OPC), a passive-first assessment methodology, and PLC/HMI attack depths. Strong defensive framing throughout: these systems control physical processes, so the material is for recognizing and closing exposure, never for touching live plants.

## One rule that overrides everything

Only test what you are authorized to test. Everything here assumes a scoped engagement, a bug bounty program that permits the activity, or your own lab. Payloads that modify, delete, or exfiltrate are written so you can reason about impact, not so you fire them blind at production.
