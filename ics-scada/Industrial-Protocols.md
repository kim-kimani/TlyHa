# Industrial Protocols

The protocols that run industrial control were designed in an era of trusted serial links, so most carry commands with no authentication, no encryption, and no integrity checking. Knowing how each one works, and how each one fails, is how you recognize exposure and design compensating controls. This file covers the common ones you will meet on an OT network.

## The mental model

An industrial protocol is a simple request/response (or publish) format that lets a master (HMI/SCADA) read values from and write values to a controller (PLC/RTU), or lets controllers exchange data. The security weakness is almost universal: the device executes any well-formed message it receives, because it was built to trust whoever is on the wire. So for most of these, the security question collapses to "who can reach the device," and the defense is segmentation and monitoring, not protocol hardening (which often is not possible).

## The protocols and how they fail

- **Modbus (TCP 502, also serial RTU/ASCII)**: the simplest and most widespread. Function codes read/write coils (bits) and registers (16-bit words). No authentication, no encryption, no sessions. Anyone reaching port 502 can read all process values and write setpoints and actuator states. Tells: open 502, trivial `read holding registers` responses. There is no secure variant in common use; isolation is the only control.
- **DNP3 (TCP/UDP 20000)**: common in electric and water utilities, richer than Modbus (events, timestamps, unsolicited reporting). Classic DNP3 has no security; messages can be read, replayed, spoofed, and injected. DNP3 Secure Authentication (SAv5) adds message authentication, but it is frequently not deployed. Tells: 20000 open, outstations accepting commands without SA.
- **S7comm / S7comm-plus (TCP 102, Siemens)**: the protocol Siemens PLCs speak to engineering software and HMIs. Classic S7comm has no real authentication; it allows reading/writing memory and starting/stopping the CPU, and uploading/downloading program logic. S7comm-plus added anti-replay and integrity in newer families, but older ones and misconfigurations remain exposed. Stuxnet abused this layer. Tells: 102 open, CPU functions reachable.
- **EtherNet/IP and CIP (TCP/UDP 44818, Allen-Bradley/Rockwell and others)**: object-based protocol for reading/writing tags and controlling the PLC. Largely unauthenticated in common deployments; tag read/write and even firmware/logic operations can be reachable. Tells: 44818 open, CIP identity responses.
- **BACnet (UDP 47808, building automation)**: HVAC, lighting, access control in buildings. Minimal security; devices discoverable and controllable by anyone on the network. Relevant to "smart building" and the AC/HVAC side of facilities. Tells: 47808 traffic, who-is/i-am discovery.
- **OPC (OPC-DA legacy over DCOM; OPC UA modern)**: integration layer between systems. Legacy OPC-DA is a DCOM nightmare (wide ports, weak auth). OPC UA is the modern, security-capable successor (it supports authentication, signing, and encryption), but it is often deployed in "no security" mode for convenience. Tells: OPC UA endpoints advertising `SecurityPolicy#None`.
- **PROFINET, EtherCAT, and fieldbuses**: real-time device-level protocols, generally unauthenticated and assumed to be on a physically protected segment.

## Real material

Identifying protocols on an OT segment (gently, read-only, authorized):

```
# passive first: capture traffic and let a dissector (Wireshark has ICS dissectors)
# identify Modbus, DNP3, S7comm, ENIP, BACnet by port and message structure.
# cautious active identification:
nmap -p 502,102,20000,44818,47808 --script modbus-discover,s7-info,enip-info <host>
# device-identity reads reveal vendor, model, firmware -> map to known issues.
```

The uncomfortable truth for Modbus-class protocols: a read is one request, and a write is the same request with a write function code. Reachability is the vulnerability. For example, reading holding registers (function 03) shows process values; writing a single register (function 06) changes a setpoint. No credential is involved anywhere in the exchange.

Spotting insecure OPC UA (the one that can be secure but often is not):

```
# an OPC UA server advertises its endpoints and their SecurityPolicy.
# an endpoint offering  SecurityPolicy=None  with  MessageSecurityMode=None
# accepts unauthenticated, cleartext sessions -> read/write without credentials.
```

## Beyond the basics

- **Reachability is the whole game for the legacy protocols**: Modbus, classic DNP3, classic S7comm, EtherNet/IP, and BACnet cannot authenticate the caller. There is no payload cleverness required; the finding and the fix both live at the network layer. When you assess these, you are really assessing segmentation, firewall rules, and monitoring, not the device's (nonexistent) access control.
- **Where secure variants exist, check if they are actually on**: DNP3 Secure Authentication, S7comm-plus integrity, and OPC UA security modes exist but are routinely disabled for ease of integration. The high-value finding is "a security-capable protocol deployed with security turned off." Verify the mode, do not assume.
- **Logic and firmware operations, not just register writes**: the deepest impact is reprogramming the controller. S7comm and EtherNet/IP allow uploading/downloading PLC programs and stopping/starting the CPU. An attacker who can do this changes the process behavior itself and can hide it from the HMI (show normal telemetry while the logic misbehaves). This is categorically more serious than toggling a coil.
- **Man-in-the-middle on the wire**: on a flat, unauthenticated segment, an attacker positioned between HMI and PLC (ARP spoofing on the LAN) can rewrite commands going down and telemetry going up, blinding operators while manipulating the process. Integrity-free protocols make this undetectable at the protocol layer.
- **Replay and spoofing**: captured DNP3/Modbus/S7 exchanges replay successfully because there is no nonce or session binding; forged unsolicited DNP3 responses can feed false data to the master.
- **Fuzzing crashes real devices**: malformed-packet fuzzing of ICS protocol stacks frequently crashes or hangs the controller (a denial of the physical process). This is both a class of findings (robustness bugs, some are public CVEs) and a serious hazard; it is done on bench/offline equipment, never a live plant.
- **The field layer (Level 0/1) is assumed-trusted**: PROFINET/EtherCAT/fieldbus and analog signaling usually have no security at all, relying entirely on physical protection of the segment. Assess physical and network access to these, since there is nothing else protecting them.

## What to watch for (defensive framing)

- Active probing and especially fuzzing can crash fragile controllers with physical consequences; prefer passive capture and test active techniques only on offline/replica equipment or in authorized maintenance windows.
- For legacy protocols, do not report "no authentication" as if it were a device bug to patch; it is a protocol property, and the actionable finding is the missing network isolation and monitoring.
- A secure-capable protocol (DNP3 SA, OPC UA, S7comm-plus) is only as good as its configuration; confirm the security mode rather than assuming it is enabled.
- To defend: isolate OT segments and control inter-level traffic with firewalls or data diodes, allow only known master/outstation pairs, enable the secure variant/mode where the protocol supports it, deploy OT-aware intrusion detection that flags write/program/CPU-control commands and anomalous sources, physically protect field-level segments, and validate device robustness on the bench before deployment.

## Study and assessment checklist

- Identify which protocols are present (passively first) and their ports/vendors/firmware.
- For each, determine whether a secure variant/mode exists and whether it is actually enabled.
- Assess who can reach each device (segmentation, firewall rules, flat-network exposure, internet exposure).
- Understand the command set: what a read, a write, a CPU stop/start, and a program download look like for that protocol.
- Treat fuzzing and writes as hazardous; confine them to offline equipment.
- Map compensating controls (isolation, allow-listing, OT IDS) to each unauthenticated protocol.
