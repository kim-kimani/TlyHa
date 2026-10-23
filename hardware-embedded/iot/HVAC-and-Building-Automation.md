# HVAC and Building Automation

Building automation systems (BAS/BMS) run the physical environment of offices, hospitals, factories, and data centers: heating/cooling (HVAC), lighting, elevators, access control, and energy management. They speak BACnet, Modbus, KNX, and LonWorks, protocols with little or no authentication, and they are frequently reachable from IT networks or the internet. The aim is control of building systems, and famously, a pivot: an HVAC vendor's access was the entry point in at least one massive retail breach.

## What it is and why it is a target (the aim)

- **Environmental control**: HVAC setpoints, ventilation, lighting, and energy. Aim: manipulate conditions (comfort, cost, and in sensitive facilities like data centers or hospitals, a real operational/safety impact).
- **Elevators and access**: BAS often ties to elevators and door/access control. Aim: understand control and safety implications.
- **The pivot**: BAS connects to IT and to vendors; it is a soft bridge into the corporate network (the retail-breach lesson). Aim: pivot from BAS to IT or vice versa.
- **Fleet/portfolio control**: cloud-managed BAS across many buildings. Aim: cross-building control via the cloud.

## Architecture (the sides)

- **Field devices**: controllers (like PLCs) for HVAC units, VAV boxes, sensors, actuators.
- **Supervisory layer**: a BAS front-end/HMI (often a Niagara/Tridium, Johnson, Honeywell, or Siemens system) with a web interface.
- **Protocols**: BACnet/IP (UDP 47808), Modbus (502), KNX (3671), LonWorks; mostly unauthenticated (see `../../ics-scada/Industrial-Protocols.md`).
- **Network/cloud**: integration with IT, remote/vendor access, and increasingly cloud portals.

## Step-by-step how-to (authorized)

### 1. Discovery (aim: find the BAS and its protocols)

```
nmap -sU -p 47808 --script bacnet-info <range>     # BACnet devices
nmap -sS -p 502,1911,4911,3671,80,443 <range>      # Modbus, Niagara Fox (1911/4911), KNX, web
# BACnet discovery (who-is / i-am) enumerates devices and their objects:
#  tools like the BACnet stack utilities, or Nmap's bacnet-info, reveal device ids,
#  vendor, and object lists (setpoints, sensors, actuators).
# shodan/censys index internet-exposed BACnet/Niagara/Modbus BAS.
```

### 2. BAS front-end (aim: control the building)

The supervisory web interface (e.g. Niagara) often has default/weak credentials and known CVEs. Aim: log in and reach the control graphics, which expose setpoints and schedules for the whole building. On your own/authorized system, demonstrate reading and (carefully) changing a setpoint.

### 3. Protocol-level control (aim: direct device manipulation)

```
# BACnet and Modbus are unauthenticated: a reachable controller accepts reads/writes.
#  BACnet ReadProperty/WriteProperty to a setpoint object changes HVAC behavior;
#  Modbus register writes change actuator states (see ../../ics-scada/).
# KNX likewise is typically unauthenticated on the bus/IP gateway.
```
Aim: understand that reachability equals control for these protocols; the fix is network isolation, not the protocol.

### 4. The pivot and cloud (aim: cross between BAS and IT, or across buildings)

Map the connections between the BAS and the IT network and any vendor/remote access; these bridges are the real risk (the retail-breach lesson). For cloud-managed BAS, test the portal/API for cross-building authorization (see `../../web/REST-API-Security.md`).

## Beyond the basics

- **BAS as the IT pivot (and vice versa)**: the famous lesson is that a building/HVAC system, or a vendor's access to it, bridged into a corporate network and led to a massive breach. BAS networks are often poorly segmented from IT; the highest-value finding is the bridge between them and weak vendor/remote access. This is the same IT/OT-boundary thinking as `../../ics-scada/02-ICS-Assessment-Methodology.md`.
- **Unauthenticated protocols, internet-exposed**: huge numbers of BACnet/Modbus/Niagara systems are directly internet-reachable; combined with no protocol auth and default front-end credentials, that is direct building control. Exposure discovery (passively) is a major finding category.
- **Niagara/Tridium and vendor front-ends**: these supervisory platforms have had serious CVEs (credential, path traversal, RCE); fingerprint and map. They are the control hub for the whole building.
- **Elevators, access control, and safety**: BAS integration with elevators and door control raises safety and physical-security stakes; data centers and hospitals add critical-environment stakes (cooling failure = outage/safety).
- **Energy and comfort manipulation**: beyond dramatic scenarios, manipulating setpoints wastes energy, damages equipment, or disrupts operations; a realistic, lower-drama impact.
- **KNX/LonWorks specifics**: KNX has optional (often unused) security; LonWorks is legacy and unauthenticated. Know the protocol's security options and whether they are enabled.

## Defensive framing / what to watch for

- BAS controls physical environments (including critical ones); test only authorized systems, go passive-first, and avoid changes that affect occupants/equipment (treat like ICS, see `../../ics-scada/`).
- The headline issues are internet exposure, unauthenticated protocols, default front-end credentials, and poor IT/OT/vendor segmentation; defenses are isolation, strong authentication on front-ends, protocol security where available, and locked-down vendor access.
- Frame impact by operational disruption, the IT pivot, and (for critical facilities) safety.
- To defend: segment BAS from IT and the internet, secure and patch supervisory front-ends, control and monitor vendor/remote access, enable protocol security (KNX Secure, BACnet/SC) where possible, and monitor for anomalous writes.

## Checklist

- Discover BAS devices and protocols (BACnet/Modbus/Niagara/KNX); check internet exposure passively.
- Test the supervisory front-end for default creds and known CVEs (authorized).
- Understand protocol-level control (unauthenticated BACnet/Modbus/KNX) on authorized systems.
- Map and assess the IT/OT and vendor/remote-access bridges (the pivot, the key finding).
- For cloud BAS, test cross-building authorization.
- Authorized systems only, passive-first; map defenses (segmentation, auth, protocol security).
