# Smart TVs

A smart TV is a full computer (Android TV, Tizen, webOS, or Roku) with a microphone, often a camera, a browser, an app store, several radios, and a cloud account tracking everything you watch. It is one of the most capable and most exposed devices in a home. The aim ranges from local control and surveillance to using the TV as a foothold into the home network.

## What it is and why it is a target (the aim)

- **Surveillance**: microphone (always-on voice), camera (some models), and ACR (automatic content recognition) that fingerprints everything on screen and reports it. Aim: access the mic/camera or the viewing data.
- **Local control and pivot**: unauthenticated LAN control APIs and open debug services let you control the TV and pivot into the home network. Aim: command execution and a network foothold.
- **Account and tracking data**: the vendor cloud holds viewing history, account, and linked services. Aim: cross-account data via cloud bugs.

## Architecture (the four sides)

- **Device**: an ARM SoC running a full OS (Android TV is the most common and the most familiar to attack), with apps, a browser engine, and ADB.
- **Radios**: Wi-Fi, Bluetooth (remote, headphones), sometimes an IR blaster, and HDMI-CEC (control over the HDMI cable).
- **Cloud**: vendor account, app store, ACR/telemetry backend, and voice-assistant backend.
- **App**: a phone remote app that controls the TV locally (companion-screen protocols) and via cloud.

## Step-by-step how-to

### 1. Network recon (aim: open services and the control API)

```
# TV on your isolated lab network; find it and scan:
nmap -sS -sV -p- 192.168.50.0/24
# common finds:
#  5555  ADB (Android TV) - often open on the network
#  8008/8009 Google Cast / DIAL
#  7676/8001/8002 Samsung Tizen control (SmartView / WebSocket API)
#  3000/9080 webOS (LG) control
#  1900  UPnP/SSDP (DIAL discovery)
#  8060  Roku ECP (External Control Protocol, unauthenticated)
```

### 2. Local control APIs (aim: control the TV without auth)

Many TV control protocols are unauthenticated on the LAN or use a one-time pairing you can often replay:

```
# Roku ECP (completely unauthenticated): press keys, launch apps, type:
curl -d '' http://TV_IP:8060/keypress/Home
curl -d '' http://TV_IP:8060/launch/12          # launch a channel/app by id
curl http://TV_IP:8060/query/device-info        # full device info, no auth
# DIAL (launch apps):
curl http://TV_IP:8008/apps/YouTube -X POST
# Samsung/LG WebSocket control: connect and send key events; first connect may
# prompt on-screen to allow, which is the only gate.
```

### 3. ADB on Android TV (aim: a shell on the TV)

```
adb connect TV_IP:5555        # if ADB-over-network is enabled (often is)
adb shell                     # a shell as a privileged-ish user
adb install malicious.apk     # sideload; aim: persistence / deeper access
adb shell pm list packages    # enumerate installed apps
adb shell screencap /sdcard/s.png ; adb pull /sdcard/s.png   # read the screen
```
Open ADB is effectively remote code execution on the TV from anywhere on the LAN.

### 4. Companion app and cloud (aim: cross-account data and remote control)

Proxy the remote app (bypass pinning) and test the cloud API with the methodology in `../../web/REST-API-Security.md` and `../../web/IDOR-and-BOLA.md`. Aim: addressing another account's TV or reading another user's viewing/account data.

### 5. Hardware (aim: firmware, keys, and persistent access)

Open the TV; find the UART on the mainboard for a serial console and boot log (see `../Bus-Protocols-Deep.md`), and dump the eMMC/flash for firmware (see `../Firmware-Analysis-and-Extraction.md`). Aim: root over serial, firmware secrets, and the ACR/telemetry endpoints and keys.

## Beyond the basics

- **ACR and telemetry interception**: capture what the TV sends about your viewing. Many TVs send ACR fingerprints and app telemetry, sometimes over HTTP or with weak TLS. Intercepting it demonstrates the privacy exposure and may reveal tokens.
- **Browser and app-store surface**: the built-in browser and app runtime have normal web/engine bugs; a malicious app or a crafted web page (or a DIAL-launched URL) can attack the TV's browser. Casting/DIAL lets a LAN attacker force the TV to load a URL.
- **HDMI-CEC and side channels**: CEC commands travel over HDMI and can control/power devices; a malicious HDMI source (or a dropped device) can issue them. Also the IR and Bluetooth remote pairing can be spoofed.
- **Mic/camera access**: the serious privacy prize. Via a shell (ADB/serial) or a malicious app, enumerate and access the mic/camera. Demonstrate only on your own set.
- **Pivot into the LAN**: with a shell, the TV is a persistent Linux host on the victim network: scan, relay, and pivot. TVs are rarely monitored or patched, making them ideal footholds.
- **Vendor-specific debug modes**: hidden service menus (remote key sequences) unlock developer and debug options, sometimes enabling ADB or factory interfaces.

## Defensive framing / what to watch for

- Open network ADB and unauthenticated control APIs are the headline issues; confirm they genuinely allow control/shell on your own TV before claiming it.
- Casting/DIAL being reachable is by design; the finding is missing auth on sensitive control or the ability to force arbitrary URLs/apps.
- Privacy impact (mic/camera/ACR) is often the real severity; frame it as such.
- To defend: disable ADB-over-network and developer mode, require authentication/pairing on control APIs, isolate the TV on a separate VLAN, disable ACR and unnecessary telemetry, keep the TV updated, and treat it as an untrusted LAN device. Test only your own TV.

## Checklist

- Isolate and scan the TV; find ADB (5555), DIAL/Cast (8008/8009), vendor control ports, Roku ECP (8060).
- Test unauthenticated control APIs (Roku ECP, DIAL) and the pairing gate on Samsung/LG.
- Try `adb connect` for a shell; enumerate apps and screen access.
- Proxy the remote app; test the cloud API for cross-account/device-id bugs.
- Intercept ACR/telemetry; assess the privacy exposure.
- Open the set for UART/serial root and firmware extraction; hunt keys and endpoints.
