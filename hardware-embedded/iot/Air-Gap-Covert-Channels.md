# Air-Gap Covert Channels

"Air-gapped" means a system is physically disconnected from networks, and people treat that as absolute safety. It is not. A computer or device still emits light, sound, heat, vibration, electromagnetic fields, and power fluctuations, and any of those can carry data to a nearby sensor. This guide covers how data crosses an air gap, so you understand why "not networked" is not "no output," and how to defend against it.

## The mental model

Exfiltration needs two things: something on the isolated system that can modulate a physical emission (a status LED, the CPU, a fan, the speaker, the screen, the power draw), and a receiver in the environment that can sense it (a camera, microphone, radio, magnetometer, light sensor, or power meter). Malware already on the air-gapped system (introduced via USB, supply chain, or an insider) encodes data into the emission; a receiver with line of sight or proximity decodes it. The data rates are low, but keys and credentials are small. Bridging the gap inward (sending commands in) is also possible via some of the same channels.

## The channels (how data leaves an isolated system)

- **Optical**: blink a status/keyboard/router LED, modulate screen brightness (including changes too subtle or too fast for humans), or use an IR LED (invisible). A camera with line of sight (even a security camera, or a drone at a window) receives it. The smart-bulb covert channel (see `Smart-Light-Bulbs.md`) is a variant: modulate a networked bulb's brightness to a camera outside.
- **Acoustic**: emit data as sound from the speaker, or ultrasonic (inaudible) tones, received by a nearby microphone (a phone, a smart speaker). Even fan speed and hard-drive/coil noise can be modulated ("fansmitter," "DiskFiltration").
- **Thermal**: modulate CPU/GPU heat; a nearby system's temperature sensors read it (very slow, but real, "BitWhisper").
- **Electromagnetic**: modulate EM emissions from the CPU, memory bus, or video cable (the classic TEMPEST/"AirHopper" using the monitor cable as a transmitter received by a nearby radio/phone). GSM-band emissions from the memory bus ("GSMem") reach a basic phone.
- **Magnetic**: modulate CPU-generated magnetic fields that pass through Faraday shielding, read by a magnetometer in a nearby phone ("MAGNETO").
- **Power**: modulate the system's power draw, read via the power line or a nearby sensor ("PowerHammer").
- **Vibration**: modulate vibrations (fans) read by an accelerometer ("AiR-ViBeR").

## How this is studied (research framing)

```
# the pattern: a sender program on the isolated machine modulates an emission
# (e.g. toggles CPU load to create an EM/thermal/acoustic pattern), encoding bits.
# a receiver (radio/phone/camera/mic/magnetometer) samples the environment and decodes.
# the Ben-Gurion University air-gap research group catalogued dozens of these
# (AirHopper, GSMem, Fansmitter, DiskFiltration, BitWhisper, MAGNETO, PowerHammer, etc.).
```
These are demonstrated in controlled research to prove the channel exists and to measure range and bitrate; the point is to inform defenses for high-security environments.

## Beyond the basics

- **Getting the malware there is the hard part**: the exfiltration channel assumes code is already running on the air-gapped system. That initial compromise comes via USB (see `../USB-and-Physical-Interface-Attacks.md`), supply chain, or an insider, Stuxnet reached air-gapped centrifuge controllers via USB. The air-gap covert channel is the exfil half of a chain whose first half is a USB/supply-chain/insider vector.
- **Inbound bridging**: some channels work inward, delivering commands or triggering actions on the isolated system via light (a scanner receiving light signals), sound, or by a compromised peripheral. The gap is not one-way.
- **Range and bitrate trade-offs**: optical and EM channels can reach meters to tens of meters; thermal and magnetic are short-range and slow. Attackers choose the channel by the environment (is there a camera with line of sight, a phone nearby, a window). Small secrets (keys, passwords) fit even slow channels.
- **The networked-neighbor trick**: an attacker does not need their own receiver inside; they can compromise a nearby networked device (a smart bulb, a camera, a phone, a smart speaker) and use its sensors/emitters as the receiver or relay, which is where this intersects the rest of the IoT folder.
- **Shielding is imperfect**: Faraday cages block radio but magnetic (MAGNETO) and power-line channels can bypass them; acoustic/optical need physical/line-of-sight isolation. No single countermeasure covers all channels.

## Defensive framing / what to watch for

- This is high-end research relevant to genuinely high-security (classified, critical-infrastructure) environments; the practical lesson for most is that "air-gapped" is a strong control but not a magic one, and the first defense is preventing code from reaching the isolated system at all.
- The study is for building defenses; the sender requires prior compromise, which is the link to actually break.
- To defend: control the initial-access vectors (strict USB/removable-media policy and port control, supply-chain and insider controls); enforce zoning (distance, no cameras/phones/recording devices with line of sight or proximity to sensitive systems); use Faraday shielding plus magnetic/acoustic isolation for the highest tiers; disable or cover LEDs, speakers, and unneeded emitters; monitor for anomalous CPU/fan/power patterns; and treat nearby networked IoT devices as potential receivers to keep away from sensitive systems.

## Checklist (defensive/learning)

- Understand that isolated systems still emit optical, acoustic, thermal, EM, magnetic, power, and vibration signals that can carry data.
- Recognize that the exfil channel requires prior compromise (USB/supply-chain/insider), the real link to defend.
- Note that nearby networked IoT devices (bulbs, cameras, phones, speakers) can serve as receivers/relays.
- For high-security environments, map which channels the physical setup allows and apply layered isolation.
- Prioritize initial-access prevention, emitter control, zoning/shielding, and emission monitoring.
