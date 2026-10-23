# IoT Device Methodology (and Index)

This folder is a detailed, per-device how-to for each kind of equipment the IoT files name. Every guide follows the same shape so you can jump in anywhere, but they all rest on one method and one lab. Read this file first; it is the generic approach you specialize per device, plus the index and the rules that keep you legal and safe. You are new to hardware, so this is written to build a repeatable process, not to assume prior knowledge.

## The one method behind every guide

Every smart device is the same four-sided system, and every guide walks these four sides in roughly this order of effort-versus-reward. The "aim" at each step is the prize you are chasing.

1. **The network/local side (cheapest first)**: put the device on a lab network you control, scan it, and probe its local services and radios. Aim: an open debug/control service, default creds, or an unauthenticated local API.
2. **The companion app side**: proxy the phone app (bypass pinning) and read its traffic and its code. Aim: the cloud API endpoints, embedded secrets, and the command protocol.
3. **The cloud side (usually highest impact)**: test the vendor cloud API the app and device use. Aim: authorization flaws (addressing other people's devices by id), which compromise the whole fleet, not one unit.
4. **The hardware side**: open the device, get a serial console, dump the firmware, and sniff internal buses/radios. Aim: firmware secrets, shared keys, and the raw command protocol.

Most real IoT wins are on the cloud side (side 3), because one authorization bug affects every customer. The hardware side (4) is where you learn the protocol and extract the keys that make the other sides exploitable. Each device guide tells you which side is softest for that device.

## The lab you need (build once, reuse for every device)

- **An isolated test network**: a dedicated router/AP (or a VLAN) so the device is not on your real LAN, with your laptop able to sniff and intercept. A Raspberry Pi as an AP works. This is non-negotiable: you will run scans and attacks that you do not want on your home network, and you want to capture all the device's traffic.
- **Traffic interception**: Wireshark for passive capture, and `mitmproxy`/Burp for the app's HTTPS (with pinning bypass via Frida/objection, see `../../mobile/Mobile-API-and-Transport-Security.md`).
- **The hardware kit** from `../00-Lab-Setup-and-Electronics-Basics.md`: multimeter, USB-UART, logic analyzer, SPI flash programmer, and, for radio devices, an RTL-SDR (receive) and optionally a HackRF/Flipper (transmit), plus BLE/Zigbee sniffer dongles (see `../Radio-and-Wireless.md`).
- **The app and account**: install the vendor app, create your own account, and register your own device so you have a legitimate baseline and a second account for cross-account tests.

## How each device guide is structured

- **What it is and why it is a target (the aim)**: the realistic prize, framed by impact (privacy, physical safety, fleet-wide control).
- **Architecture**: the four sides for this specific device (chips, radios, cloud, app).
- **Step-by-step how-to**: the concrete recon-to-exploitation steps, device-specific, with the aim of each step.
- **Beyond the basics**: the advanced, device-specific techniques.
- **Defensive framing / what to watch for**: how to recognize and close the weakness (the real goal), and the limits and legal/safety notes.
- **Checklist**: a fast pass for that device.

## Index

- `Smart-TVs.md`
- `Smart-Light-Bulbs.md`
- `Robot-Vacuums.md`
- `IP-Cameras-and-Baby-Monitors.md`
- `Smart-Locks-and-Garage-Doors.md`
- `Smart-Plugs-and-Switches.md`
- `Smart-Appliances-Fridge-Washer-AC.md`
- `Network-Printers.md`
- `VoIP-Phones.md`
- `Alarm-Sensors-and-Home-Security.md`
- `Smart-Speakers-and-Voice-Assistants.md`
- `Medical-Devices.md`
- `Smart-Meters.md`
- `Solar-Inverters.md`
- `EV-Chargers.md`
- `HVAC-and-Building-Automation.md`
- `Access-Control-and-Badge-Readers.md`
- `GPS-and-Timing.md`
- `Smart-Toys.md`
- `Wearables.md`
- `Intimate-Devices.md`
- `Air-Gap-Covert-Channels.md`

Cars are covered in `../Automotive-and-CAN-Bus.md`. The conceptual overviews are `../IoT-and-Smart-Home-Devices.md` and `../IoT-What-Is-Actually-Hackable.md`.

## The rules (read before every device)

- Test only devices you own and your own accounts/device ids. Viewing or controlling someone else's camera, lock, vehicle, meter, or medical device is a real intrusion with real victims, even as research.
- The industrial and medical guides (meters, inverters, EV chargers, building automation, medical devices) carry physical-safety and legal weight; they are written for recognition and defense, and real testing there is done under authorization and responsible disclosure, often with the vendor and regulators.
- Transmitting on radio bands is regulated; receive freely, transmit only where legally permitted and on your own devices.
- The objective throughout is defensive: find the weakness so it can be fixed. Keep that framing, disclose responsibly, and keep proofs minimal.

## Checklist (applies to every device)

- Put the device on an isolated lab network; capture all its traffic from first power-on and pairing.
- Walk the four sides: local/network, companion app, cloud API, hardware.
- Identify which side is softest for this device (the guide tells you) and aim there first.
- For cloud, always test cross-account/device-id authorization (fleet-wide impact).
- For hardware, extract firmware and hunt shared secrets and the command protocol.
- Frame impact by privacy and physical safety; stay within devices and accounts you own.
