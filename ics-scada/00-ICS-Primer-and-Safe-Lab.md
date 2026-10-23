# ICS Primer and Safe Lab

If you are new to industrial control systems, start here. This file explains what ICS and SCADA actually are in plain terms, why they are different from normal IT, and, most importantly, how to build a safe practice lab out of free simulators so you can learn hands-on without ever touching real equipment. In this world a mistake is not a crashed web app; it is a physical process, so the first skill is learning safely.

## What ICS and SCADA are (plain language)

ICS (Industrial Control System) is the umbrella term for the computers that monitor and control physical processes: a water treatment plant, a power substation, a factory line, a building's heating. SCADA (Supervisory Control and Data Acquisition) is the part that lets humans watch and command those processes from a control room. The physical work is done by field devices (valves, pumps, motors, sensors), which are driven by small rugged controllers called PLCs (Programmable Logic Controllers) or RTUs. Operators see and control everything through an HMI (Human-Machine Interface), a screen showing the plant as a diagram with live values and buttons.

So the stack, bottom to top: sensors and actuators (the physical process), PLCs/RTUs (the real-time controllers), HMI/SCADA (the human view and control), and a historian (a database logging everything). They talk over industrial protocols like Modbus and S7 (see `ICS-SCADA-Fundamentals.md` for the deeper model and the "Purdue" layering).

## Why ICS is different from normal IT (this shapes everything)

- **Availability and safety come first, not confidentiality**: you cannot reboot a running turbine to patch it. Systems run for decades unpatched. "Just update it" does not apply.
- **The protocols have no security**: Modbus, DNP3, classic S7, and friends were built for trusted, isolated networks, so they have no authentication or encryption. Any device that can reach a PLC can usually read and command it. The security lives in the network isolation, not the protocol.
- **Fragile devices**: old PLCs can crash from a simple port scan. Aggressive testing that is routine in IT can take down a physical process. This is why you learn on simulators first.
- **Physical consequences**: a bug here can open a valve, stop a pump, or disable a safety shutdown. The stakes are real-world, sometimes life-safety.

## Build a safe lab (do this before anything else)

You can learn the entire protocol and attack side on your own computer with free, legal simulators. No real hardware, no risk. This is the correct way to start.

- **A software PLC: OpenPLC**: a free, open-source PLC you run on your PC or a Raspberry Pi. It speaks Modbus like a real PLC and runs real ladder-logic programs, so you can practice reading and writing registers against something that behaves like the genuine article.
- **A Modbus simulator: ModbusPal / pymodbus**: simulate Modbus slaves (devices) and masters so you can send and inspect Modbus traffic.
- **A full virtual plant: GRFICS**: the "Graphical Realism Framework for Industrial Control Simulation" gives you a simulated chemical process, a real soft-PLC, an HMI, and a visual of the process reacting to your commands. You can attack the PLC and watch tanks overflow in the simulation. The best all-in-one learning environment.
- **A process simulator: Factory I/O**: a 3D factory you can wire to a (soft or real) PLC to see logic drive machines.
- **An ICS honeypot to study traffic: Conpot**: emulates ICS devices and protocols; useful to see what scanners and attackers do against ICS services and to practice recognizing the protocols.

A minimal first lab: run OpenPLC as the controller, load a sample ladder program, and talk to it with `pymodbus`/ModbusPal from another terminal. Then graduate to GRFICS to see a whole process respond.

## Real material (your first safe hands-on steps)

Reading and writing a soft-PLC over Modbus in the lab (OpenPLC + pymodbus):

```python
# talk to your OpenPLC instance (localhost in the lab) as a Modbus master
from pymodbus.client import ModbusTcpClient
c = ModbusTcpClient('127.0.0.1', port=502)
c.connect()
print(c.read_holding_registers(0, 10).registers)   # read 10 registers (sensor/logic values)
c.write_register(0, 1234)                           # write a register (changes a value/setpoint)
c.write_coil(0, True)                               # flip a coil (an on/off output, e.g. a "pump")
c.close()
```
In GRFICS you can watch the simulated process react to that write: this is the safe way to feel why "reachability equals control" for these protocols, with zero real-world risk.

Identifying ICS protocols in a capture (learn to recognize them):

```
# capture lab traffic and open in Wireshark; it dissects Modbus, S7comm, DNP3, ENIP.
# learn the shape: Modbus function codes 01/03 (read), 05/06/16 (write), unit id, registers.
```

## What to aim for (as a learner)

The goal at this stage is fluency and safety, not exploitation of anything real. You should be able to: explain the ICS stack and why its protocols are unauthenticated; stand up a soft-PLC and a simulated process; read and write a PLC over Modbus in your lab; and recognize the common protocols in a packet capture. With that foundation, `02-ICS-Assessment-Methodology.md` shows how real assessments are done (carefully, mostly passively), and `PLC-and-HMI-Attacks.md` goes deeper on the techniques, all practiced in your simulator.

## Safety and legality (read this twice)

- Practice only on simulators and lab equipment you own. Never scan, connect to, or send traffic to a real ICS/SCADA system without explicit written authorization; even "just looking" can disrupt fragile devices and may be a serious crime.
- Internet-exposed ICS devices you find (e.g. via Shodan) are real plants belonging to someone; do not touch them. Looking at search-engine metadata is passive; connecting is not.
- In real environments, ICS testing is done with operator involvement, mostly passively, on offline/duplicate equipment, and in maintenance windows. "Do no harm" is literal because harm is physical.
- The reason to learn this is to defend it: segmentation, monitoring, and safe configuration. Keep that framing.

## Checklist (beginner path)

- Understand the ICS stack (field devices, PLC/RTU, HMI/SCADA, historian) and why protocols are unauthenticated.
- Stand up a lab: OpenPLC + pymodbus/ModbusPal, then GRFICS for a full simulated plant.
- Practice reading/writing a soft-PLC over Modbus and watch a simulated process react.
- Learn to recognize Modbus/S7/DNP3/ENIP in Wireshark.
- Internalize the safety rules before going near anything real; proceed to the methodology and PLC/HMI files in the lab.
