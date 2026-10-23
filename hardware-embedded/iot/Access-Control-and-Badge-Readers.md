# Access Control and Badge Readers

Physical access control (RFID/NFC badges, the readers, door controllers, and the management server) decides who gets through a door. The aim is to get through: clone a badge, exploit the reader-to-controller wiring, or compromise the controller/management system. Badge cloning is one of the most accessible and eye-opening hardware skills, and the lessons are directly defensive.

## What it is and why it is a target (the aim)

- **Badge cloning**: copy a credential to enter as someone else. Aim: clone a card/fob (the most common, often trivial, path).
- **Reader/wiring abuse**: the reader-to-controller link (Wiegand) is usually unauthenticated and sniffable/injectable. Aim: capture or inject credentials at the wire.
- **Controller/server compromise**: the door controller and the access-management software. Aim: unlock doors, add credentials, or read logs via the network/cloud.
- **Credential/key recovery**: the card's crypto keys. Aim: keys that let you clone or forge many cards.

## Architecture (the sides)

- **Credential**: an RFID/NFC card or fob (125 kHz LF like HID Prox/EM410x, or 13.56 MHz HF like MIFARE Classic/DESFire, iCLASS).
- **Reader**: reads the card and sends the credential to the controller, classically over Wiegand (an old, unauthenticated protocol).
- **Door controller/panel**: makes the unlock decision and drives the door strike; networked to the management server.
- **Management software/cloud**: credential database, logs, and administration.

## Step-by-step how-to (your own badges / authorized)

### 1. Identify the credential technology (aim: pick the clone method)

```
# with a Proxmark3 (the standard tool) or a Flipper Zero:
# LF (125 kHz): HID Prox, EM410x, Indala, mostly just a static ID, no crypto.
# HF (13.56 MHz): MIFARE Classic (broken crypto), DESFire/iCLASS SE (stronger).
proxmark3> lf search          # identify LF cards
proxmark3> hf search          # identify HF cards
```

### 2. Clone (aim: a working copy)

```
# LF static-ID cards (HID Prox / EM410x): read the ID and write to a blank/T5577:
proxmark3> lf hid read
proxmark3> lf hid clone --raw <id>
# MIFARE Classic: crack the keys (nested/darkside attacks) then dump and clone:
proxmark3> hf mf autopwn          # recovers keys, dumps, and can clone to a magic card
```
Aim: demonstrate that LF prox and MIFARE Classic badges are clonable (they usually are), so the organization moves to secure credentials.

### 3. Long-range read and reader attacks (aim: capture without touching the card)

```
# long-range readers (e.g. a concealed "Tastic RFID Thief" built from a commercial
# reader) can read LF prox badges from a distance in a pocket/bag.
# Wiegand wiring: the reader-to-controller link is unauthenticated; devices like the
# "ESPKey" implant tap the Wiegand lines to capture every badge presented and to
# replay/inject credentials (open the door).
```
Aim: show that credentials can be captured at a distance or at the wire, independent of card crypto.

### 4. Controller/management/cloud (aim: unlock and admin)

Network-scan the controller/management server; test default credentials and known CVEs; for cloud-managed access control, proxy the app and test the API for cross-tenant door control and credential management (see `../../web/REST-API-Security.md`, `../../web/IDOR-and-BOLA.md`). Aim: unlock doors or add a credential via the software, often the cleanest path.

## Beyond the basics

- **125 kHz prox and MIFARE Classic are broken**: HID Prox/EM410x carry just a static number (clone by copying it), and MIFARE Classic's Crypto-1 is cryptographically broken (keys recoverable in seconds with a Proxmark). The vast installed base still uses these; the defensive finding is to migrate to DESFire EV2/EV3 or Seos/iCLASS SE with proper key management.
- **Wiegand is the weak wire**: the decades-old reader-to-controller protocol has no authentication, so an implant (ESPKey) on the wires harvests and injects credentials. Reader tamper detection and OSDP Secure Channel (the authenticated replacement for Wiegand) are the defenses.
- **Downgrade and card-type confusion**: multi-tech readers that still accept legacy low-security cards undermine a secure-card rollout; the system is as weak as the weakest accepted credential.
- **Long-range and covert capture**: reading badges from a bag/pocket or across a desk defeats "the card never left my wallet"; distance is not protection.
- **Controller/management as the clean path**: often the easiest entry is the network/cloud side (default creds, CVEs, authorization bugs), which unlocks doors and issues credentials without any RFID work.
- **Mobile credentials/NFC**: phone-based access (BLE/NFC) adds the mobile and cloud attack surface (see `../../mobile/`).

## Defensive framing / what to watch for

- This is physical security; clone and test only your own badges and authorized systems, and do not use cloned credentials to enter spaces you are not permitted to.
- The findings are legacy clonable credentials, unauthenticated Wiegand, multi-tech downgrade, and weak controller/management security; defenses are secure credentials with key management, OSDP Secure Channel, reader tamper detection, and hardened/segmented controllers and management systems.
- Frame impact as physical entry.
- To defend: migrate to DESFire EV2/EV3 or Seos with managed keys, replace Wiegand with OSDP Secure Channel, disable legacy card acceptance, add reader tamper detection, and secure the controller/management network and cloud.

## Checklist

- Identify the credential tech (LF prox/EM vs HF MIFARE/DESFire/iCLASS).
- Clone legacy credentials (LF static ID, MIFARE Classic via key cracking) with a Proxmark, on your own badges.
- Assess long-range read and Wiegand-wire capture/injection risk.
- Test the controller/management server and cloud for default creds, CVEs, and cross-tenant door control.
- Check for multi-tech downgrade (legacy cards still accepted).
- Own/authorized only; map defenses (secure credentials, OSDP, tamper detection, segmentation).
