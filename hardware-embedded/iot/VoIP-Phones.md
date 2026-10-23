# VoIP Phones

A desk VoIP phone is a networked computer with a microphone, a speaker, a web admin panel, and SIP credentials. In offices they sit on trusted network segments and nobody patches them. The aim is call interception and toll fraud, credential theft, and turning the phone into a room microphone and a network foothold.

## What it is and why it is a target (the aim)

- **Call interception and fraud**: capture or reroute calls, or place unauthorized calls (toll fraud). Aim: intercept RTP audio and abuse SIP registration.
- **Room bug**: the phone has a mic; a compromised phone can be made to listen. Aim: covert audio surveillance.
- **Credential theft**: SIP, provisioning, and admin credentials stored on the device or fetched insecurely. Aim: extract or capture them.
- **Foothold**: a persistent, unmonitored host on a trusted LAN. Aim: pivot.

## Architecture (the four sides)

- **Device**: an embedded Linux/RTOS phone with a web admin UI, a SIP stack, and RTP media.
- **Network**: SIP (signaling) and RTP (media), a web/SSH admin interface, and often TFTP/HTTP provisioning.
- **Provisioning server**: phones auto-configure by fetching a config file (with credentials) from a provisioning server at boot, frequently over cleartext.
- **PBX/cloud**: the call server (Asterisk/FreePBX or a cloud PBX) the phone registers to.

## Step-by-step how-to

### 1. Network recon (aim: services and provisioning)

```
nmap -sS -sV -p- PHONE_IP
# look for: 5060/5061 SIP, 80/443 web admin, 22 SSH, 69 TFTP (provisioning),
#           RTP media ports (high UDP), and vendor management ports.
```

### 2. Provisioning capture (aim: config files with credentials)

```
# phones fetch config from a provisioning server at boot, often over TFTP/HTTP cleartext:
#  sniff the boot traffic (Wireshark) to capture the config file, which commonly
#  contains SIP username/password, admin credentials, and server addresses.
# or request the config directly if the filename (often MAC-based) is guessable:
tftp PROV_SERVER -c get cfg<PHONE_MAC>.cfg
```
Aim: SIP and admin credentials straight from the (unencrypted) provisioning file, a very common finding.

### 3. Web admin (aim: control and stored creds)

Test default/blank admin credentials and known model CVEs; the admin UI exposes SIP accounts, call logs, and settings, and often has command-injection or auth-bypass bugs on specific firmwares. Aim: full device control and credential disclosure.

### 4. SIP and RTP (aim: interception and fraud)

```
# SIP enumeration/registration testing (your own lab PBX):
#  tools like sipvicious/svmap/svwar enumerate extensions; test weak registration auth.
# RTP interception: if media is unencrypted (no SRTP) and you are on-path (ARP spoof on
#  the LAN), capture RTP and reconstruct the call audio in Wireshark (RTP -> play).
```
Aim: intercept call audio (no SRTP) and test unauthorized registration/call placement (toll fraud) against your own PBX.

### 5. Hardware and firmware (aim: root, keys, persistence)

UART/SSH for a shell; dump firmware (see `../Firmware-Analysis-and-Extraction.md`) for stored SIP/admin credentials, provisioning secrets, and the update path.

## Beyond the basics

- **Insecure provisioning is the classic VoIP bug**: cleartext TFTP/HTTP config with MAC-based filenames lets anyone on the segment (or who can guess the URL) pull credentials. Autoprovisioning that does not authenticate the server also enables a rogue provisioning server feeding malicious config.
- **The phone as a room microphone**: with admin/root or a firmware bug, the mic can be activated remotely; some phones auto-answer or have an intercom/paging feature abusable to open the mic. A serious covert-surveillance capability; demonstrate only on your own phone.
- **Toll fraud**: weak SIP registration or an open PBX extension lets an attacker place calls (premium-rate fraud). The phone's stored SIP creds enable registering as that extension from elsewhere.
- **Unencrypted media (no SRTP) and signaling (no TLS)**: on-path capture reconstructs calls; this is why SRTP/SIP-TLS exist. Check whether they are enforced.
- **Firmware CVEs and command injection**: many desk-phone web UIs have public auth-bypass/RCE bugs; fingerprint and map.
- **Foothold and pivot**: a rooted phone is a persistent LAN node, often on a voice VLAN that can reach sensitive segments.

## Defensive framing / what to watch for

- Headline issues are cleartext provisioning, no SRTP/TLS, default admin creds, and firmware CVEs; defenses are encrypted/authenticated provisioning, SRTP + SIP-TLS, strong admin auth, and patching.
- Confirm captured credentials/audio on your own lab before claiming; frame mic access as covert surveillance.
- Toll fraud and call interception have direct financial and privacy impact.
- To defend: use HTTPS-authenticated provisioning (no TFTP), enforce SRTP and SIP over TLS, set admin passwords, disable auto-answer/unused features, segment voice on its own VLAN, and keep phone firmware updated.

## Checklist

- Scan the phone; find SIP/RTP, web admin, SSH, and TFTP/HTTP provisioning.
- Capture or fetch the provisioning config; extract SIP/admin credentials.
- Test web admin defaults and known firmware CVEs.
- Test SIP registration auth and RTP interception (no SRTP) on your own PBX.
- Assess mic/auto-answer abuse as a room-bug capability.
- Extract firmware; find stored creds and the update path; test only your own phone.
