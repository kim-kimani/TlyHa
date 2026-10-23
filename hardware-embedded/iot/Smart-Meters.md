# Smart Meters

A smart meter (electricity, gas, water) is a utility-owned computer on the side of a building that measures consumption, reports it over a wide-area or mesh radio, and in many cases can remotely disconnect service. Because meters connect homes to the grid and number in the millions, the aim and the stakes are fleet-scale: billing integrity, privacy, and grid stability. This is utility infrastructure, so it is studied defensively and under authorization.

## What it is and why it matters (the aim, defensively)

- **Billing/consumption integrity**: the meter reports usage for billing. Aim (defensively): understand how consumption data could be forged or tampered, so it can be protected.
- **Remote disconnect abuse**: many meters can be remotely disconnected/reconnected by the utility. Aim: understand the command path, because mass or unauthorized disconnect is a grid and safety concern.
- **Privacy**: fine-grained consumption reveals occupancy and appliance usage. Aim: identify data exposure.
- **Grid stability at scale**: coordinated manipulation of many meters/endpoints is a systemic risk. Aim: understand the fleet command/key model.

## Architecture (the sides)

- **Device**: a hardened metrology MCU plus a communications module; tamper-resistant by design, with a secure element in better units.
- **Radio/comms**: an AMI (Advanced Metering Infrastructure) network: RF mesh (e.g. 900 MHz neighborhood mesh), cellular, power-line communication (PLC), or a Home Area Network radio (Zigbee/ Wi-SUN) to in-home displays.
- **Head-end / utility backend**: collects data and sends commands (including disconnect) across the fleet.
- **HAN/consumer side**: an in-home display or energy-management link.

## How this is studied (defensively, on authorized units)

### Radio/AMI analysis

```
# with an SDR, the meter's RF (often ~900 MHz mesh or cellular) can be observed:
rtl_433 -f 915M          # some consumption-broadcast meters are decodable off the shelf
# research tools decode specific AMI stacks; the questions are: is the traffic
# encrypted and authenticated, and are consumption reports and commands protected.
```
Some older meters broadcast consumption essentially in the clear (enabling passive consumption monitoring from nearby); the defensive finding is to encrypt and authenticate.

### Hardware/firmware (authorized test meters)

Open a test meter; the metrology section is tamper-protected, but the comms module can be examined via UART/SPI/JTAG (see `../Bus-Protocols-Deep.md`) for firmware, keys, and the command protocol. The critical question: are keys unique per device and held in a secure element, or shared/extractable. Shared keys are the fleet risk.

### Head-end/HAN interfaces

The utility backend and the HAN interface are software surfaces; assessment (authorized) looks at authentication of commands (especially disconnect), key management, and the HAN radio's security.

## Beyond the basics (what the research teaches)

- **Shared keys are the systemic danger**: if disconnect/command authentication relies on keys shared across many meters, extracting one could enable fleet-scale abuse (mass disconnect), a grid-stability and public-safety concern. The defense is per-device keys in secure elements and authenticated commands, which modern AMI deployments emphasize.
- **Consumption privacy**: high-resolution interval data reveals when people are home, sleep/wake patterns, and even which appliances run (load disaggregation). Unencrypted reporting or weakly protected backend data is a privacy exposure.
- **Remote disconnect as a weapon**: the remote-disconnect feature is the highest-consequence function; its command path must be strongly authenticated. Research attention focuses here because of the mass-outage potential.
- **Tamper and theft**: energy-theft via meter tampering is the traditional threat the metrology side is hardened against; the comms side is the newer attack surface.
- **PLC and mesh specifics**: power-line and RF-mesh AMI have their own protocol stacks (e.g. Wi-SUN, proprietary mesh) with varying security; the questions remain encryption, authentication, and key management.
- **Standards**: AMI security maps to utility frameworks and standards (and overlaps with ICS/SCADA, see `../../ics-scada/`); findings land through utility and regulator channels.

## What to watch for (critical)

- Smart meters are utility-owned critical infrastructure; do not tamper with, transmit to, or interfere with a meter in service. Passive RF observation of your own meter's region may be possible, but connecting to or manipulating utility infrastructure is illegal and dangerous without explicit authorization.
- Study is done on vendor/utility-provided test units under formal authorization, with coordinated disclosure.
- Frame findings around billing integrity, privacy, and grid stability, and route them through utility/regulator channels.
- To defend (the objective): encrypt and authenticate AMI traffic and especially disconnect commands, use per-device keys in secure elements, protect backend/head-end systems, minimize and protect high-resolution consumption data, and segment/monitor the AMI and utility networks.

## Checklist (defensive/learning)

- Understand the AMI architecture (meter, RF-mesh/cellular/PLC, head-end, HAN).
- Passively characterize the radio (encrypted/authenticated?) only within legal limits.
- On authorized test units, examine the comms module firmware for keys (unique vs shared) and the command/disconnect protocol.
- Assess consumption-data privacy and the remote-disconnect command authentication.
- Treat this as critical infrastructure: authorization and coordinated disclosure only.
- Map defenses: per-device keys, authenticated commands, encrypted telemetry, segmentation.
