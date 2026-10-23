# PLC and HMI Attacks

This is the hands-on layer: what you can actually do to a PLC and to the HMI/SCADA systems once you are on the control network, and what each technique aims for. Everything here is for your simulator lab (see `00-ICS-Primer-and-Safe-Lab.md`) or authorized offline equipment, never a live process. The point is to understand the techniques so you can recognize and defend against them.

## The mental model

A PLC is a small computer running a control program (its "logic") in a loop: read inputs (sensors), compute, write outputs (actuators). An attacker on the control network can interact with it at three depths: the process-variable layer (read/write the registers and coils that hold sensor values and actuator states), the program layer (upload, download, or modify the control logic itself), and the operational layer (stop/start the CPU, change its mode). The HMI and historian sit above, and compromising them gives both visibility and a trusted command path. Deeper access means bigger, stealthier impact.

## The three depths of PLC attack (increasing impact)

### Depth 1: process variables (read and write registers/coils)

Goal: read the live process state and manipulate setpoints and outputs. This is the simplest and uses the plain protocol (Modbus function codes, S7 read/write, etc.). In your lab:

```python
# OpenPLC / soft-PLC over Modbus (lab only)
from pymodbus.client import ModbusTcpClient
c = ModbusTcpClient('127.0.0.1', 502); c.connect()
c.read_holding_registers(0, 20)     # read process values (e.g. tank level, temp)
c.write_register(4, 9000)           # change a setpoint
c.write_coil(2, True)               # force an output on (e.g. "pump = on")
```
Impact: in GRFICS you can watch a tank overflow or a process destabilize from register writes. On real systems the same writes change actuators directly, because there is no authentication. This is "reachability equals control."

### Depth 2: the control logic (upload/download/modify the program)

Goal: change how the PLC behaves, not just its current values, and optionally hide it from operators. Using the vendor's engineering protocol (S7comm for Siemens, the equivalent for Rockwell/others), an attacker can read the existing logic, download modified logic, and thereby redefine the process. This is what Stuxnet did: it altered PLC logic to damage centrifuges while replaying normal readings to the operators, so the HMI showed "all fine." Modifying logic is categorically more serious than toggling a register because it is persistent and can be made invisible.

### Depth 3: CPU control and firmware (stop/start, mode, firmware)

Goal: disrupt or persist. Many PLCs let a network peer stop or start the CPU (an instant process halt, a denial-of-service with physical effect), switch run/program mode, or in some cases update firmware. Malicious firmware is the deepest persistence (it survives logic reloads). These are high-impact and, on real systems, dangerous; demonstrate only in the lab.

## HMI, SCADA, and historian attacks

- **The HMI is usually a Windows PC**: with normal IT vulnerabilities, often old and unpatched, with broad access to the control network. Compromising it (via a standard IT exploit, a malicious project file, or credential theft) gives an attacker the operator's own view and controls, a fully trusted command path to the PLCs, plus the ability to show operators false information. The HMI is frequently the easiest and highest-leverage target because it is just a vulnerable computer that happens to control a plant.
- **SCADA/HMI software bugs**: HMI and SCADA products have a long history of memory-corruption and web vulnerabilities (many public CVEs), plus insecure project-file formats and default credentials. These are normal software bugs in software that controls physical processes.
- **The historian as a pivot**: the historian logs process data and usually sits between the OT and IT networks (sometimes dual-homed), making it a prime bridge for crossing from IT into OT. Its database and web front-ends are attackable with ordinary techniques.
- **False data to operators**: whether via logic manipulation (Depth 2) or HMI compromise, feeding operators normal-looking readings while the process misbehaves is the signature ICS attack pattern, because it delays or prevents human response.

## Beyond the basics

- **Man-in-the-middle on the control LAN**: on a flat, unauthenticated segment, ARP-spoofing between the HMI and a PLC lets an attacker rewrite commands going down and telemetry going up, undetectably at the protocol layer. This achieves the "false data" effect without even touching the PLC's logic.
- **Replay and spoofing**: captured Modbus/S7/DNP3 exchanges replay successfully (no nonce/session), and forged responses feed false values to the master. Practice recognizing this in your captures.
- **The attack chain that matters**: real incidents are IT-to-OT. The chain is usually: phish or exploit into corporate IT, pivot to a dual-homed system or through weak segmentation into OT, use the HMI/engineering workstation and the unauthenticated protocols to reach the PLCs, then manipulate the process while blinding the operators. Each link is a defensive control to test (see `02-ICS-Assessment-Methodology.md`).
- **Engineering workstation as the keys to the kingdom**: the machine running the vendor's programming software holds the project files, the logic, and the ability to reprogram every PLC. Compromising it is often the cleanest route to Depth 2/3, and it is a normal (often poorly secured) computer.
- **Safety systems are the red line**: the Triton/Trisis malware specifically targeted a safety controller to disable the last line of defense, which could permit a physical catastrophe. This is why SIS are isolated and, in assessments, never actively tested outside dedicated offline settings.
- **Why you cannot just "patch the protocol"**: Depth 1-3 all work because the protocols trust the network. The defense is not a protocol fix (usually impossible for legacy gear) but isolation, allow-listing which devices may command which PLCs, and monitoring for abnormal write/program/stop commands.

## What to watch for (safety and defensive framing, the whole purpose)

- Every technique here can disrupt or damage a physical process on real equipment; practice exclusively on simulators (OpenPLC, GRFICS) or authorized offline/bench devices.
- Logic download, CPU stop, and firmware operations are especially dangerous and are not performed on live systems in assessments; they are demonstrated in the lab or reviewed architecturally.
- Never touch safety instrumented systems actively; treat them as off-limits.
- To defend: segment and allow-list control traffic (only known HMI/engineering hosts may command a given PLC), lock down and patch HMIs/historians/engineering workstations as the real soft targets, deploy OT intrusion detection that alerts on write/program/stop commands and new talkers, use authenticated protocol variants where available, physically protect the control LAN against MITM, and keep safety systems isolated. Each maps to an attack depth above.

## Checklist (lab-practiced)

- Depth 1: read/write PLC registers and coils in the lab; watch a simulated process react.
- Depth 2: understand logic upload/download/modify and the "false data to operators" pattern (Stuxnet-style), in simulation.
- Depth 3: understand CPU stop/start and firmware as disruption/persistence; lab only.
- HMI/SCADA/historian: treat as vulnerable computers; practice standard IT techniques against lab copies.
- Study the IT-to-OT chain and the role of the engineering workstation.
- For every technique, state the defensive control that stops it; keep SIS off-limits.
