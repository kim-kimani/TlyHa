# IoT: What Is Actually Hackable

People draw a mental line between "computers, which get hacked" and "appliances, which do not." That line is imaginary. If a thing has a processor and a way to receive input (a radio, a port, a sensor, a cloud link), it has an attack surface. This file is a tour of the things people assume are safe, what is genuinely possible, and why, so you recognize the surface and understand the real-world stakes. You are new to this, so it is written to build intuition, not to weaponize.

## The one principle behind all of it

Every "smart" or electronic device is the same four-part system from `IoT-and-Smart-Home-Devices.md`: a small computer (often Linux or an MCU), one or more radios or ports, a cloud account, and a phone app. Security fails when any of those four is weak and the others trust it. The surprising targets below are surprising only because people forget the device is a computer. Once you see the processor and its inputs, "can it be hacked" becomes "which of the four sides is weakest."

## Things people assume are unhackable, and what is actually possible

- **Smart light bulbs**: full radios (Zigbee/BLE/Wi-Fi) and a small CPU. Demonstrated: worming malware bulb-to-bulb over Zigbee across a building; extracting the Wi-Fi/network keys from a bulb's flash (so a discarded bulb leaks your network); and covert data exfiltration by modulating brightness (including near-infrared invisible to humans) to a camera outside. A bulb is a networked computer with a key store.
- **Robot vacuums**: Linux computers with LIDAR/cameras, microphones, and a cloud link. Possible: reading the house's floor map and room layout (a privacy/physical-security leak), turning on the mic, and using the LIDAR as a sound sensor (laser vibrometry, "LidarPhone") to eavesdrop. It literally maps and can listen to your home.
- **Smart TVs**: covered in the smart-home file; worth repeating that the mic and camera, ACR (automatic content recognition) tracking everything you watch, and network-reachable control APIs make a TV a surveillance device. Possible: remote channel/app control, mic access, and pivoting into the home network.
- **Baby monitors and IP cameras**: among the most-abused IoT, because default/weak credentials plus cloud BOLA (addressing other users' cameras by id) give strangers live audio/video. Possible and repeatedly real: watching and talking through someone else's camera.
- **Printers and VoIP phones (the forgotten office computers)**: full network stacks, web admin panels, stored credentials (scan-to-email/LDAP creds in the printer), and historically wormable (a printer worm spread across the internet to make a point). Possible: credential theft, a persistent network foothold that nobody patches, and using the printer as a pivot. Office phones can be turned into room microphones.
- **Smart locks, garage doors, alarm sensors**: covered in `Radio-and-Wireless.md`. Possible: replay/relay to open fixed-code or weak-rolling-code systems, BLE command injection on unauthenticated locks, and jamming an alarm sensor's "door open" signal so it never reports. Direct physical-security impact.
- **Medical devices (the serious end)**: implantable pacemakers and insulin pumps have had demonstrated wireless command vulnerabilities (deliver a shock, alter dosing); hospital infusion pumps and monitors run old, networked software. These are regulated and life-critical, which is exactly why the research exists: to force fixes. Possible, demonstrated in controlled research, and the reason device security standards tightened.
- **Cars**: a whole file (`Automotive-and-CAN-Bus.md`). Remote-to-physical control has been demonstrated. Keyless entry relay attacks steal cars today.
- **Building and industrial "facilities" gear**: HVAC/AC, elevators, access-control badge readers, smart meters, solar inverters, and EV chargers are networked controllers. Possible: smart-meter manipulation, solar-inverter/EV-charger control at scale (a grid-stability concern), badge-reader cloning, and HVAC manipulation (which was the pivot in at least one famous retail breach). Smart meters and inverters connect the home to the grid, so fleet-level control has outsized impact.
- **GPS and timing**: GPS is an unauthenticated radio signal; it can be spoofed with an SDR to feed false position/time to anything that trusts it (drones, vehicles, ships, and systems that rely on GPS time). "It comes from satellites" does not mean it is authenticated.
- **Toys, wearables, and intimate devices**: kids' toys with mics and cloud accounts (leaked millions of voice recordings in real breaches), fitness trackers leaking location, and Bluetooth intimate devices with unauthenticated control. All are small computers with radios and cloud backends, with real privacy stakes.
- **Air-gapped and "offline" systems**: covert channels jump the gap: a device's status LED, fan/coil whine (acoustic), heat, power-line modulation, or a nearby bulb/screen can exfiltrate data to a sensor in the room. "Not on the network" is not "no output."

## Beyond the basics (how these actually get compromised)

- **The cloud account is the fleet-wide lever**: the highest-impact IoT bugs are not on the device at all; they are authorization failures in the vendor cloud that let you address other people's devices by id (BOLA) or claim/control them. One cloud bug equals every customer's camera/lock/car. When assessing, test the cloud API first (see `../web/IDOR-and-BOLA.md` and `../web/REST-API-Security.md`). This is where "watch a stranger's baby monitor" actually comes from, not from hardware wizardry.
- **Shared secrets turn one device into all devices**: extract a key or credential from one unit's firmware (see `Firmware-Analysis-and-Extraction.md`) and, if it is shared across the product line, you compromise the fleet. Endemic in cheap IoT.
- **The radio is the local no-touch entry**: replay, relay, injection, and jamming (see `Radio-and-Wireless.md`) reach locks, cars, sensors, and bulbs without wires. Proximity is weaker protection than people think (directional antennas extend range).
- **Privacy impact is often the real severity**: for cameras, vacuums, TVs, toys, and wearables, the win is not code execution; it is audio/video/location/behavioral data about people. Frame that impact clearly; it is frequently more serious than a crash.
- **Physical-safety impact raises the stakes**: locks, cars, medical devices, and facilities gear can cause real-world harm. Research here is done carefully, responsibly disclosed, and often with vendors and regulators, precisely because the point is to get them fixed.
- **Longevity is the quiet problem**: appliances live 10 to 20 years and stop getting updates in 2 or 3. Today's "secure" fridge/TV/camera is a known-vulnerable internet host for most of its life. The defense is network segmentation and lifecycle planning, not hoping for patches.

## What to watch for (you are learning, so build good habits)

- Test only devices you own and your own accounts/device ids; viewing or controlling someone else's camera, lock, or vehicle is a real intrusion with real victims, even as "research."
- The surprising-target list is to train recognition of attack surface, not a to-do list against strangers' property; the serious categories (medical, automotive, facilities) are researched under authorization and responsible disclosure.
- Reachability or a weak default is not the same as a working exploit; confirm the specific issue on your own device before claiming it.
- To defend (the real goal here): isolate IoT on a separate network/VLAN, change defaults, keep it updated where possible and retire it when it is not, prefer devices with unique per-unit keys and ownership-based cloud authorization, and assume any mic/camera/radio device is a potential sensor for an attacker.

## Checklist (how to look at any "unhackable" thing)

- Identify the processor and every input: radios, ports, sensors, cloud link, app.
- Ask which of the four sides (device, radio, cloud, app) is weakest and whether the others blindly trust it.
- For anything with a cloud account, suspect BOLA/ownership bugs first (fleet-wide, no hardware needed).
- Extract firmware to check for shared secrets and default credentials.
- For local attacks, examine the radio for replay/relay/injection.
- Weigh impact by privacy and physical safety, not just code execution, and stay within devices and accounts you own.
