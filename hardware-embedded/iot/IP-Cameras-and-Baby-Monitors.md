# IP Cameras and Baby Monitors

IP cameras and baby monitors (a baby monitor is just an IP camera with a nursery theme) are among the most abused IoT devices, because a weak default and a cloud authorization bug give a stranger live audio and video of someone's home. The aim is almost always the stream, and the most common path is the cloud, not the hardware.

## What it is and why it is a target (the aim)

- **Live audio/video access**: the core prize and the real-world harm. Aim: view/listen to a camera, either locally or, far worse, across accounts via the cloud.
- **Two-way audio**: many have a speaker; aim includes talking through someone's camera (the widely-reported baby-monitor abuse).
- **Fleet-wide cloud exposure**: addressing cameras by id in the vendor cloud. Aim: enumerate and access other users' cameras.
- **Network foothold**: the camera is a Linux host to pivot from.

## Architecture (the four sides)

- **Device**: an ARM Linux SoC with a camera sensor, often running an embedded web server and an RTSP/ONVIF stack, plus a UART and SPI/NAND flash.
- **Radio/network**: Wi-Fi or Ethernet; the stream is served locally (RTSP/ONVIF) and/or relayed via the cloud for remote viewing.
- **Cloud**: a relay/P2P service so users can view remotely without port-forwarding; this is where mass exposure lives.
- **App**: live view, settings, and cloud pairing.

## Step-by-step how-to

### 1. Network recon (aim: the stream and the admin interface)

```
nmap -sS -sV -p- CAMERA_IP
# common finds:
#  554   RTSP (the video stream, often weakly/un-authenticated)
#  80/443 web admin (default creds are endemic)
#  8000/8899/34567 vendor-specific control ports
#  ONVIF (SOAP, often on 80/8080) for discovery/PTZ/stream URLs
#  UPnP, and a P2P/cloud client connecting out
```

### 2. Local stream and admin (aim: view without the app)

```
# try the RTSP stream (paths are vendor-specific, well-documented per brand):
ffplay rtsp://CAMERA_IP:554/live/ch0
ffplay rtsp://admin:admin@CAMERA_IP:554/Streaming/Channels/101
# ONVIF to enumerate stream URIs and PTZ (move the camera):
#  use an ONVIF client; GetStreamUri, GetDeviceInformation, PTZ moves.
# web admin: test default creds (admin/admin, admin/<blank>, vendor defaults) and
# known auth-bypass CVEs for the specific firmware.
```
Aim: an unauthenticated or default-credential stream is an immediate finding on your own device.

### 3. Cloud / P2P (aim: cross-account access, the big one)

Proxy the app and analyze the P2P/cloud protocol. Many cameras use a P2P cloud keyed by a device UID printed on the camera; if UIDs are guessable/enumerable and the cloud authorizes by UID knowledge rather than ownership, strangers reach other cameras. Test with the methodology in `../../web/IDOR-and-BOLA.md` and `../../web/REST-API-Security.md`, on your own devices/accounts. This cloud class is the source of most real baby-monitor abuse, not hardware wizardry.

### 4. Hardware (aim: firmware, backdoors, keys)

```
# UART for a root shell / boot log; dump SPI/NAND flash:
#  see ../Bus-Protocols-Deep.md and ../Firmware-Analysis-and-Extraction.md
binwalk -e firmware.bin
grep -rinE 'password|admin|telnet|rtsp|p2p|key|BEGIN' squashfs-root/
```
Camera firmware is notorious for hardcoded/backdoor credentials and a telnet root shell. A universal backdoor affecting a whole model line is a top-tier finding.

## Beyond the basics

- **Default and backdoor credentials are endemic**: many brands ship a fixed support/debug account or a web-UI backdoor; check the firmware's auth code and known model backdoors. Mirai-class botnets were built largely on default-credential cameras/DVRs.
- **RTSP/ONVIF without auth**: streams and PTZ frequently lack authentication on the LAN; ONVIF's `GetStreamUri` hands you the paths. The win is viewing/controlling without credentials.
- **P2P/UID enumeration**: the cloud UID scheme is the fleet-wide lever; if UIDs are sequential/derivable and ownership is not enforced, it is mass exposure. This is the highest-impact, no-hardware path.
- **Credential and key extraction**: cloud/P2P keys in firmware, shared across units, let you impersonate or decrypt the relay.
- **Botnet/foothold**: cameras are classic botnet nodes and LAN pivots; a shell makes the camera a persistent foothold.
- **Firmware CVEs**: specific camera firmwares have many public auth-bypass and RCE CVEs; fingerprint the firmware and map to known issues.

## Defensive framing / what to watch for

- The dominant real-world harm is cross-account stream access and default credentials; confirm on your own camera/account, and never access a stream that is not yours.
- Reachable RTSP/ONVIF on your LAN is expected; the finding is missing authentication or cross-account cloud access.
- Frame impact squarely as privacy (live audio/video of people) and, for two-way audio, direct harassment potential.
- To defend: force a strong unique password at setup, authenticate RTSP/ONVIF, enforce ownership-based cloud authorization (not UID knowledge), use per-device keys, disable telnet/backdoors, keep firmware updated, and segment cameras on their own VLAN with no internet if remote viewing is not needed.

## Checklist

- Isolate and scan; find RTSP (554), web admin, ONVIF, vendor ports, and the P2P/cloud client.
- Test local stream/admin for default creds and unauthenticated RTSP/ONVIF/PTZ.
- Analyze the cloud/P2P UID scheme; test cross-account access on your own devices.
- Extract firmware; hunt backdoor credentials, telnet root, and shared cloud keys.
- Map the firmware version to known camera CVEs.
- Frame findings by privacy/physical harm; stay within your own devices and accounts.
