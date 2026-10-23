# Solar Inverters

A solar inverter converts panel DC to grid AC and, when "smart," connects to a vendor cloud and the grid with remote monitoring and control. Millions are internet-connected, and aggregated they represent significant generation capacity, so the aim and stakes are both individual (one system) and grid-scale. This is energy infrastructure, studied defensively and under authorization.

## What it is and why it matters (the aim, defensively)

- **Generation control**: inverters can be remotely adjusted or curtailed (output, grid parameters). Aim (defensively): understand the control path, because mass manipulation of many inverters affects grid frequency/stability.
- **Fleet exposure**: inverters and their cloud portals are widely internet-exposed with weak auth. Aim: find the authentication and authorization gaps that would allow remote control.
- **Data and account**: production data and owner accounts. Aim: identify cross-account exposure.
- **Local foothold**: the inverter and its monitoring dongle are network devices. Aim: understand the local interfaces.

## Architecture (the sides)

- **Device**: a power-electronics controller plus a communications unit; often a separate Wi-Fi/Ethernet "data logger"/dongle bridging the inverter's internal bus (Modbus/RS-485 or proprietary) to the cloud.
- **Comms**: Wi-Fi/Ethernet/cellular to the vendor cloud; internally Modbus over RS-485 is extremely common (overlaps with `../../ics-scada/Industrial-Protocols.md`).
- **Cloud**: monitoring portal and app; remote configuration/control.
- **Grid interface**: the inverter follows grid codes and may accept utility/aggregator control signals.

## How this is studied (defensively, on your own/authorized systems)

### Local interfaces and Modbus

```
# the data-logger often exposes a local web UI and/or Modbus:
nmap -sS -sV -p- INVERTER_IP
# Modbus/TCP (502) or Modbus-over-RS-485 internally carries registers for production
# data and settings; like all Modbus it is unauthenticated (see ../../ics-scada/).
# the local web UI frequently has default/weak credentials.
```
Aim: understand that the inverter's internal control (Modbus) is unauthenticated and what the local UI/dongle exposes, on your own system.

### Cloud/app

Proxy the app and test the monitoring-portal API (see `../../web/REST-API-Security.md` and `../../web/IDOR-and-BOLA.md`): can you read or control another owner's inverter by id, and what control does the portal allow (output limits, firmware, grid settings). Vendor portals for major inverter brands have had authorization flaws exposing/controlling other users' systems.

### Firmware/dongle

Extract the data-logger firmware (see `../Firmware-Analysis-and-Extraction.md`) for cloud credentials (shared?), the command protocol, and the update mechanism.

## Beyond the basics (what the research teaches)

- **Grid-scale aggregation is the systemic risk**: a single owner's inverter is minor, but a cloud authorization flaw or shared key that lets someone command many inverters at once (curtail or change grid parameters) is a grid-stability concern that regulators and researchers take seriously. The defensive priority is strong cloud authorization and authenticated control.
- **Internet exposure and weak portals**: large numbers of inverter dashboards are exposed with default credentials or authorization bugs; this is the most realistic mass-impact path, and it is a web/cloud problem more than a hardware one.
- **Unauthenticated Modbus internally**: the inverter's own control bus trusts the network/dongle; whoever controls the dongle controls the inverter's settings it exposes. Ties directly to the ICS/SCADA material.
- **Shared cloud keys in the dongle**: extracting a cloud credential shared across data loggers enables fleet access if the backend authorizes by device id.
- **Firmware/update integrity**: unsigned firmware on the inverter or dongle allows persistent tampering; signed updates are the defense.
- **Grid-code and control-signal security**: utility/aggregator control channels (for demand response) must be authenticated; their abuse is a grid concern.

## What to watch for (critical)

- Inverters tie into the electrical grid; do not manipulate grid parameters or output on a live, grid-connected system, and do not touch systems you do not own. Changing inverter/grid settings can be dangerous and illegal.
- Test only your own system, or vendor/owner-authorized units, and coordinate disclosure for cloud/portal findings (which are often the real issue and testable like any web app with authorization).
- Frame findings by individual exposure and grid-scale risk; route infrastructure findings through vendor/utility channels.
- To defend (the objective): strong authentication and ownership-based authorization on the cloud portal/API, authenticated control signals, per-device keys, authenticated internal buses, signed firmware, and not exposing dashboards/dongles to the internet without protection.

## Checklist (defensive/learning)

- Map the architecture (inverter, data-logger/dongle, cloud, grid interface).
- On your own system, examine the local web UI (default creds) and internal Modbus (unauthenticated).
- Proxy the app; test the monitoring portal for cross-account read/control (the key fleet risk).
- Extract dongle firmware; check for shared cloud keys and the update/signing path.
- Understand the grid-scale aggregation risk and the control-signal authentication.
- Own/authorized systems only; coordinate disclosure; map the defenses.
