# EV Chargers

An electric-vehicle charger (home wallbox or public charge point) is a networked power device: it switches high current, authenticates users and billing, talks to a backend over OCPP, and talks to the car over its own protocols. The aim spans billing fraud, charger control, user-data and RFID abuse, and, at scale, grid impact. Chargers bridge payments, the grid, and vehicles, so they are a rich and sensitive target.

## What it is and why it is a target (the aim)

- **Billing and free charging**: chargers authenticate sessions (RFID card, app, plug-and-charge). Aim: charge for free or bill someone else (RFID cloning, session/auth abuse).
- **Charger control**: start/stop charging, change power, or take the unit offline. Aim: unauthorized control, and at scale, load manipulation (grid concern).
- **User data**: accounts, payment, and charging history in the backend. Aim: cross-account data via the cloud/OCPP backend.
- **Foothold**: the charger is a Linux device on a network (home or operator). Aim: pivot.

## Architecture (the four sides)

- **Device**: a Linux/MCU controller switching the contactor, with RFID, a display, and connectivity.
- **Protocols**: OCPP (Open Charge Point Protocol, charger-to-backend, over WebSocket) is the key one; ISO 15118 (charger-to-vehicle, "Plug and Charge"); Modbus/local APIs; RFID (often MIFARE) for user auth.
- **Cloud/backend**: the Charge Point Operator's management system (billing, remote control) via OCPP.
- **App**: start/stop, payment, history.

## Step-by-step how-to (your own charger / authorized)

### 1. Network and local interfaces (aim: local control and config)

```
nmap -sS -sV -p- CHARGER_IP
# look for: a local web/config UI (default creds common), Modbus (502) on some home
# units, SSH/UART debug, and the OCPP WebSocket client connecting out.
```

### 2. OCPP backend channel (aim: control and backend trust)

```
# OCPP runs over WebSocket (ws:// or wss://). Capture/inspect it (see ../../web/WebSockets.md):
#  - is it ws:// (cleartext) so session tokens and commands are exposed?
#  - is the charger identity authenticated, or can you impersonate a charge point by
#    its id to the backend, or impersonate the backend to the charger?
# OCPP messages include RemoteStart/StopTransaction, ChangeConfiguration, Reset,
# firmware update, and meter values.
```
Aim: understand whether OCPP is encrypted and both ends authenticated; cleartext OCPP and weak charge-point authentication allow session hijacking, spoofed meter values (billing fraud), and remote-control message abuse.

### 3. RFID authentication (aim: free/fraudulent charging)

```
# many chargers authenticate users with MIFARE Classic cards, which are clonable:
#  read a card with a Proxmark/ACR122; MIFARE Classic's crypto is broken, so UID-only
#  or weak-key cards can be cloned to charge as another user (see Access-Control-and-Badge-Readers.md).
```
Aim: demonstrate (with your own card) that RFID auth is clonable, enabling billing to another user or free charging.

### 4. Cloud/app and firmware (aim: cross-account and persistence)

Proxy the app; test the operator/cloud API for cross-account billing/history and remote control (see `../../web/IDOR-and-BOLA.md`). Extract firmware (see `../Firmware-Analysis-and-Extraction.md`) for backend credentials, OCPP endpoints, and the update path.

## Beyond the basics

- **OCPP security is the crux**: historically OCPP was often deployed over unencrypted WebSocket with weak or no charge-point authentication, enabling meter-value spoofing (bill less/more), session hijacking, and remote-control abuse. OCPP 2.0.1 and secure profiles add TLS and authentication; the finding is deployments that do not use them. This overlaps with the WebSocket and API material.
- **Billing fraud paths**: clone an RFID card to charge on someone's account; spoof meter values to under/over-report; or hijack a session. Payment and RFID are the fraud surface.
- **Grid/load at scale**: a Charge Point Operator's backend controls many chargers; compromising it (or spoofing OCPP broadly) could toggle large aggregate load, a grid-stability concern (ties to `../../ics-scada/`). The backend is the fleet lever.
- **ISO 15118 / Plug and Charge**: the car-charger protocol uses certificates for automatic billing; its PKI and implementation are an emerging surface (certificate handling, contract credentials).
- **Public-charger physical access**: public units expose ports, SIMs, and sometimes debug interfaces to anyone standing there; physical tamper is part of the model.
- **Firmware and shared keys**: unsigned firmware and shared backend credentials across a fleet are the usual IoT pitfalls.

## Defensive framing / what to watch for

- Chargers switch high current and tie to billing and the grid; test only your own home charger or authorized units, and do not manipulate public chargers or the grid.
- The headline issues are insecure OCPP (cleartext/weak auth), clonable RFID, and backend authorization; defenses are OCPP over TLS with mutual authentication, strong card/app auth, ownership-based backend authorization, and signed firmware.
- Frame impact by billing fraud, charger control, and grid-scale load manipulation.
- To defend: deploy OCPP securely (TLS, authenticated charge points), use secure card technologies (not UID-only MIFARE Classic), enforce ownership on the operator API, sign firmware, and segment/monitor charger networks.

## Checklist

- Scan the charger; find local UI/Modbus, debug, and the OCPP WebSocket.
- Inspect OCPP: encrypted? both ends authenticated? meter-value and remote-control message integrity?
- Test RFID auth for clonability (your own card).
- Proxy the app/operator cloud; test cross-account billing/history and remote control.
- Extract firmware; find backend creds, OCPP endpoints, and update signing.
- Own/authorized chargers only; frame by fraud, control, and grid risk.
