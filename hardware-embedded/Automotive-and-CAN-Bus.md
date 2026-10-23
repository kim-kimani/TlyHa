# Automotive and CAN Bus

Cars are computers on wheels: dozens of small controllers talking over a shared bus with no authentication, because the network was designed in the 1980s for a closed, trusted system. Learning the CAN bus is one of the most approachable ways into hardware hacking, because the tools are cheap, the protocol is simple, and the effects (lights, dashboard, locks) are immediate and visible. Start here if you are new; it builds intuition fast.

## The mental model (start from zero)

A modern vehicle has many ECUs (Electronic Control Units): engine, brakes, doors, dashboard, infotainment, each a small computer. They coordinate over the CAN bus, a two-wire shared network where every ECU hears every message. A CAN message is tiny: an arbitration ID (which says what the message is about, not who sent it) and up to 8 bytes of data. Crucially, there is no sender authentication and no encryption: any device on the bus can send any message, and every ECU trusts messages addressed to its function. That single fact is the root of most car-hacking: if you can put messages on the bus, you can often make things happen.

## Where you plug in and what you need

- **The OBD-II port**: every car since the mid-1990s has a diagnostic port (usually under the dashboard) that exposes CAN. This is the beginner's entry point, no soldering required.
- **Cheap gear**: a USB-to-CAN adapter (a CANable/CANtact, or an ELM327 for basic work) plus `can-utils` on Linux. A few tens of dollars gets you reading and writing.
- **Safer practice**: an "instrument cluster on a bench" (a junkyard dashboard powered on a desk) or a simulator like ICSim (a software CAN car simulator) lets you learn without a real vehicle. Strongly recommended as the first step.

## Real material (first hands-on steps)

Set up a virtual CAN and the simulator (no car needed) to learn:

```
# Linux, can-utils + the ICSim simulator:
sudo modprobe can vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
./icsim vcan0            # the simulated car
./controls vcan0         # a virtual dashboard you control (doors, speed, signals)
```

Read and understand traffic:

```
candump vcan0                 # watch every message (ID + data bytes) live
candump -l vcan0              # log to a file for later replay/analysis
cansniffer vcan0             # groups by ID and highlights changing bytes (great for finding "what does what")
```

Find the message for an action (the core beginner skill), then replay it:

```
# 1. cansniffer while you trigger an action (e.g. turn signal) and watch which
#    ID's bytes change.
# 2. replay a captured frame to reproduce the action:
cansend vcan0 244#0000000810   # example: ID 0x244, 5 data bytes
# 3. fuzz a byte to map its effect:
cangen vcan0 -I 244 -D i -L 8  # generate frames on ID 0x244 to see what moves
```

On a real vehicle, the same steps through the OBD-II port let you read live data, reproduce dashboard/lighting/lock behavior, and interact with diagnostic services.

## Beyond the basics (what to aim for as you grow)

- **Reverse the message map**: the real work is building a dictionary of ID-to-meaning for a given car. `cansniffer` plus triggering actions and diffing is how you learn that "ID 0x3D0 byte 2 = speedometer." Public DBC files and community databases jump-start this for known models.
- **Diagnostic services (UDS/OBD)**: beyond raw CAN, the Unified Diagnostic Services protocol (ISO-TP over CAN) exposes powerful functions: read/clear fault codes, read/write ECU memory, flash firmware, and run "routines." Many are gated by a "security access" challenge-response (a seed/key algorithm) that is frequently weak, reused across models, or recoverable from the diagnostic software. Unlocking security access can let you reflash an ECU.
- **ECU firmware extraction and reflashing**: UDS routines or direct chip access let you pull and modify ECU firmware (tuning, and from a security view, persistence and logic changes). This connects back to the rest of this track (firmware analysis, glitching to defeat ECU readout protection).
- **Injection and spoofing impact**: because there is no authentication, spoofed frames can control real functions. Research has demonstrated remote control of braking, steering assistance, and acceleration on specific vehicles by reaching the CAN bus. On the bus, a flood or targeted frames can also deny service (disable the dashboard, trigger errors).
- **The remote attack surface is the prize**: physical CAN access is powerful but needs the car. The high-impact research chains a remote entry point (the infotainment/telematics unit over cellular, Wi-Fi, Bluetooth, a malicious media file, or a companion app and its cloud API) to the CAN bus, turning a remote bug into physical vehicle control. The telematics/infotainment ECU is the bridge; it is a normal Linux/QNX computer with normal software bugs (see the web and firmware files).
- **Key fobs and immobilizers**: the RF side (see `Radio-and-Wireless.md`): rolling-code capture/replay, relay attacks on keyless entry (amplify the fob signal from inside the house to the car), and weak immobilizer crypto. A major real-world car-theft surface.
- **Newer buses**: CAN-FD (faster, bigger frames), Automotive Ethernet, and FlexRay appear on newer vehicles; the no-authentication legacy persists in many, though newer designs add message authentication (SecOC) and gateways that segment the infotainment from the drivetrain.

## What to watch for (safety and legality)

- Never experiment on a vehicle in motion or one you will then drive on public roads; bus interference can disable safety systems. Use a bench cluster or simulator to learn, and a car only on a lift/off-road with care and authorization.
- Work on vehicles you own or are explicitly authorized to test; automotive security research has legal sensitivities.
- Writing to the bus or to ECUs can brick modules; practice on junkyard parts first.
- To defend: segment the infotainment/telematics from the drivetrain with a gateway, add message authentication (SecOC), harden the remote entry points (cellular/BT/Wi-Fi/app/cloud) as the real perimeter, use strong unique security-access algorithms for diagnostics, and protect key-fob crypto against replay/relay.

## Checklist (beginner path)

- Set up `can-utils` + `vcan` + ICSim; learn `candump`, `cansniffer`, `cansend`, `cangen` with no hardware.
- Practice finding the frame for an action by diffing, then replaying it.
- Move to a bench instrument cluster, then (authorized, stationary) to the OBD-II port.
- Learn UDS/OBD diagnostic services and security-access challenge weaknesses.
- Study how remote entry points (infotainment/telematics/app) bridge to CAN.
- Explore the key-fob RF surface with the radio file.
