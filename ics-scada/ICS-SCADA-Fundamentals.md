# ICS and SCADA Fundamentals

Industrial control systems run the physical world: power, water, manufacturing, building management, oil and gas. They were built for reliability and safety over decades, not for a hostile network, and most of their protocols have no authentication at all. Understanding how they are attacked is how you learn to segment, monitor, and protect them, because here a bug is not data loss, it is a physical process going wrong.

## The mental model

An ICS is a hierarchy. At the bottom are sensors and actuators (the physical process). Above them, PLCs (Programmable Logic Controllers) and RTUs read sensors and drive actuators in real time. Above those, SCADA/HMI systems (Human-Machine Interfaces) and historians let operators monitor and command the process, and engineering workstations program the PLCs. The protocols connecting these layers (Modbus, DNP3, EtherNet/IP, S7, OPC) were designed for trusted serial links and carry commands with no authentication. The attacker's goal is to reach the control network and then speak these protocols to read process state and manipulate the process, or to reprogram a PLC.

This is the Purdue model in shorthand: Level 0 (field devices), Level 1 (controllers/PLCs), Level 2 (HMI/SCADA), Level 3 (operations/historian), and above that the IT network and the internet. Security depends on keeping the lower levels isolated; most incidents are a failure of that isolation.

## Where it hides / tell-tales

- **IT-to-OT bridges**: the single most important weakness. A jump host, a dual-homed historian, a vendor remote-access box, a misconfigured firewall rule, or an engineering laptop that moves between networks. Once an attacker crosses from IT into the control network, the unauthenticated protocols do the rest.
- **Internet-exposed ICS**: PLCs, HMIs, and SCADA directly on the internet (Shodan/Censys index them by protocol banners and device fingerprints). Tells: open Modbus (502), S7 (102), EtherNet/IP (44818), BACnet (47808), DNP3 (20000), and HMI web interfaces with default or no credentials.
- **Flat control networks**: no segmentation between HMI, PLCs, and field devices, so any foothold reaches everything.
- **Default and shared credentials**: HMIs, historians, and engineering software with vendor defaults; PLCs with no password or a well-known one.
- **Unpatched, long-lived systems**: Windows systems a decade out of support running the HMI, because "you cannot patch the plant." Known IT vulnerabilities persist for years.
- **No protocol authentication**: Modbus, DNP3 (without Secure Authentication), and S7 (classic) accept commands from anyone who can reach them. The device cannot tell an operator from an attacker.

## Real material

Reconnaissance (read-only, carefully) of a control network. Even scanning can disrupt fragile devices, so this is done gently and only when authorized:

```
# Shodan-style external exposure (passive, no packets to the target):
#   search by protocol/port and product banners; many ICS are indexed.
# on-network identification with ICS-aware tooling (use with extreme caution):
nmap -sn 10.0.0.0/24                      # host discovery only, slowly
nmap -p 502,102,44818,20000,47808 --script-timeout 60s <host>   # ICS ports
# ICS-specific enumeration scripts (e.g. Nmap's modbus/s7/enip scripts, or PLCScan)
# can read device identity, but aggressive scanning can crash old PLCs.
```

Reading process state over Modbus (unauthenticated by design):

```
# Modbus function codes: read coils (01), read holding registers (03), write (05/06/16).
# with a client library (pymodbus) or a tool:
#   read register 40001..40010 to see sensor values the PLC exposes.
# a WRITE (function 06/16) to a register or coil changes a setpoint or actuator state.
# this is why reachability == control for Modbus.
```

The point is not the payload; it is that any host able to open TCP 502 to the PLC can both read and command it. The same is true, with protocol differences, for S7comm, EtherNet/IP, and DNP3 without Secure Authentication.

## Beyond the basics

- **The real attack is crossing the boundary, not speaking Modbus**: landing malicious commands is trivial once you can reach the device; the defended perimeter is the IT/OT boundary. Famous ICS incidents (Stuxnet, the Ukraine grid attacks, Triton/Trisis) were primarily about getting from IT into OT (spear-phishing, stolen VPN/remote-access, supply-chain) and then abusing the trusted control protocols and engineering software. Study the boundary.
- **PLC logic manipulation is the deep capability**: beyond toggling a register, an attacker who reaches the engineering interface can download modified ladder/structured-text logic to the PLC, changing how the process behaves while the HMI still shows normal values (Stuxnet's approach, and the "false data to operators" pattern). This requires the vendor's engineering protocol and tooling, and it is the difference between a nuisance and a process-safety event.
- **Safety instrumented systems (SIS) are the red line**: SIS are the independent controllers that trip a process to a safe state (shut it down) when something dangerous happens. Triton/Trisis targeted a Triconex SIS specifically to disable safety, which could allow a physical catastrophe. Attacks that touch safety systems are in a different severity class; in authorized testing they are approached with extreme caution or not at all on live plants.
- **Protocol-level attacks**: spoofing sensor values to the HMI (operator sees normal while the process drifts), replaying or forging commands, and man-in-the-middle on unauthenticated links. DNP3 and Modbus/TCP MITM let an attacker rewrite both the commands going down and the telemetry coming up.
- **HMI and historian as pivots**: the HMI is usually a Windows box with normal IT vulnerabilities and broad access to the control network; compromising it gives both a view and a command path. Historians sit between OT and IT and are a common bridge.
- **Legacy and air-gap myths**: "air-gapped" networks are routinely bridged by USB media, vendor laptops, remote-maintenance modems, and wireless links. The air gap is a design intent, rarely a reality; assess the actual connectivity.
- **Testing discipline is different here**: active scanning and writes can crash decades-old PLCs or disrupt a live process with physical consequences. ICS assessment favors passive monitoring, network captures, testing on offline/duplicate equipment and vendor test benches, and maintenance windows. "Do no harm" is literal.

## What to watch for (defensive framing)

- Scanning and even benign reads can disrupt fragile ICS devices; in real environments, prefer passive traffic analysis and test active techniques only on offline replicas or during authorized windows with operators present.
- Reachability equals control for unauthenticated protocols; the defensive priority is isolation and monitoring, not hardening the protocol (which usually cannot be made to authenticate).
- Impacts are physical and can be safety-critical; never experiment on a live process, and treat SIS as off-limits except in tightly controlled, authorized tests.
- To defend: segment rigorously (Purdue levels, firewalls/data diodes at boundaries), lock down and monitor IT/OT bridges and remote access, enforce allow-listing and change control on engineering workstations, use authenticated protocol variants (DNP3 Secure Authentication, OPC UA with security) where possible, monitor OT traffic for anomalous commands, and keep SIS independent and protected.

## Study and assessment checklist

- Map the Purdue-model layout and every connection between IT and OT, including remote access and removable media paths.
- Inventory devices and protocols (Modbus, DNP3, S7, EtherNet/IP, BACnet, OPC) and their exposure (internal, internet via Shodan).
- Assess the IT/OT boundary: firewall rules, jump hosts, dual-homed systems, vendor access.
- Check for internet-exposed ICS, default/shared credentials, and unpatched HMI/historian hosts.
- Understand (do not execute on live systems) how unauthenticated protocols allow read/write and PLC logic download.
- Treat SIS and any safety function as a hard boundary; plan testing around offline equipment and windows.
