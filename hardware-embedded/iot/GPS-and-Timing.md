# GPS and Timing

GPS (and the other GNSS systems) is an unauthenticated radio signal from space that an enormous range of systems trust blindly for position and time: phones, vehicles, drones, ships, aircraft, cell networks, financial systems, and power grids. The aim is to understand spoofing and jamming, how a faint, unauthenticated signal can be overridden, because so much critical infrastructure depends on it, and the defenses are what you are really learning.

## What it is and why it matters (the aim, defensively)

- **Position spoofing**: feed false location to anything that trusts GPS. Aim (defensively): understand how drones, vehicles, ships, and apps can be fed wrong positions, so systems can detect it.
- **Time spoofing**: GPS is a primary time source for telecom, finance, and the power grid. Aim: understand timing attacks, which can be more consequential than position.
- **Jamming/denial**: overpower GPS so receivers lose lock. Aim: understand availability impact and detection.
- **Geofence and tracking bypass**: defeat GPS-based geofences (drone no-fly, asset tracking, ankle monitors). Aim: understand the limits of GPS-based enforcement.

## Why GPS is weak (the core fact)

GPS satellites broadcast a known, unauthenticated signal that is extremely weak by the time it reaches the ground. A local transmitter can therefore overpower the real signal (spoofing) or drown it (jamming) with modest power. The civilian signal has no cryptographic authentication, so a receiver cannot tell a genuine satellite from a simulated one. That single design fact, an unauthenticated, faint, publicly-specified signal, is the root of every GPS attack.

## How this is studied (defensively, in a controlled/shielded setup)

### Simulated signals (shielded, no transmission)

```
# GPS signal generation is done with an SDR (HackRF/USRP) and open simulators
# (e.g. gps-sdr-sim) that create a baseband signal for a chosen location/time.
# This MUST be done into a shielded enclosure or via cable to a test receiver,
# NEVER transmitted over the air (illegal and dangerous: it affects aviation,
# emergency services, and more).
```
Aim: in a shielded lab, feed a generated signal to a test receiver to see it report a false position/time, demonstrating the trust problem for defensive study only.

### Receiver behavior and detection

Study how receivers and systems detect spoofing: sudden position/time jumps, signal-strength anomalies, inconsistency between GNSS constellations, and dead-reckoning/inertial cross-checks. The defensive goal is building systems that notice when GPS lies.

## Beyond the basics (what the research teaches)

- **Time is the quiet, critical target**: cellular base stations, financial trading timestamps, and power-grid synchrophasors rely on GPS time. Subtly shifting time (rather than position) can disrupt these without an obvious "teleport," which is why GPS time attacks are a serious infrastructure concern and why systems add holdover clocks and multi-source time.
- **Drones and geofencing**: drones trust GPS for position and no-fly geofences; spoofing can move a drone's perceived location (research has demonstrated misdirecting drones). Counter-drone systems also use spoofing/jamming. This cuts both ways.
- **Vehicles, ships, and aviation**: spoofing has been observed affecting ship navigation (reported "circle spoofing" incidents) and is a studied risk for autonomous vehicles and aviation, which is why aviation uses many redundant navigation sources and integrity monitoring (RAIM).
- **Tracking and monitoring bypass**: GPS-based asset trackers and ankle monitors can be jammed/spoofed; understanding this bounds how much to trust GPS-only enforcement.
- **Defenses**: multi-constellation and multi-frequency receivers, cryptographic authentication (Galileo's OSNMA adds signal authentication), antenna/direction and signal-strength checks, inertial/odometry cross-checks, and anomaly detection on sudden jumps. Military signals use encryption/authentication; civilian systems are catching up.
- **Jamming is simpler than spoofing**: cheap illegal "personal privacy device" jammers cause real disruption; detection and resilient (multi-source) timing/position are the answers.

## What to watch for (critical)

- Never transmit GPS (or jamming) signals over the air. GPS spoofing/jamming affects aviation, maritime, emergency services, telecom, and the grid; it is illegal in essentially every jurisdiction and genuinely dangerous. All experimentation is done in a shielded enclosure or over cable to a test receiver only.
- Study is defensive: understand the trust model and build detection; do not interfere with any real system.
- Frame findings around which systems over-trust GPS and how they can detect/resist spoofing.
- To defend (the objective): use multi-constellation/multi-frequency receivers, enable signal authentication (OSNMA) where available, cross-check GPS against inertial/odometry/network time, monitor for position/time anomalies, use holdover clocks for timing, and never rely on GPS alone for safety or security decisions.

## Checklist (defensive/learning)

- Understand why civilian GNSS is unauthenticated and overpowerable (the root cause).
- In a shielded/cabled lab only, observe a receiver accepting a simulated position/time.
- Identify which systems in scope over-trust GPS position and, especially, GPS time.
- Study detection: multi-constellation checks, signal-strength/direction, inertial cross-checks, OSNMA.
- Never transmit over the air; keep all work shielded; frame everything defensively.
- Map resilience measures for each dependent system.
