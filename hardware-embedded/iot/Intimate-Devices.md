# Intimate Devices

Bluetooth-connected intimate (teledildonic) devices are IoT products with an unusually sensitive context: they have mics/cameras in some models, BLE control, apps with messaging and video, and cloud accounts, and the data and control involved are extremely private. They have had well-documented security and privacy failures. This guide treats them as what they are: BLE-plus-cloud IoT devices where the privacy stakes are exceptionally high, so the framing is strongly defensive.

## What it is and why it matters (the aim, defensively)

- **Unauthorized control**: BLE control is often unauthenticated, so a nearby device can connect and control. Aim (defensively): identify the lack of authentication so it is fixed, because unauthorized control of such a device is a form of assault.
- **Privacy of intensely sensitive data**: apps include usage data, messaging, and sometimes camera/video; cloud accounts store it. Aim: identify data exposure (several products leaked user data, images, and chats).
- **Cross-account exposure**: weak cloud backends. Aim: cross-account data/control.
- **Tracking**: BLE identifiers that reveal presence/use. Aim: identify trackability.

## Architecture (the four sides)

- **Device**: an MCU with BLE (often a Nordic nRF), a motor, sometimes a camera.
- **Radio**: BLE to the app; the control protocol frequently lacks authentication.
- **Cloud**: account, messaging, usage data, media (for camera/"long-distance" features).
- **App**: control, remote-control sharing, chat, and video.

## How this is studied (your own device / authorized, defensively)

### BLE control analysis (aim: authentication state)

```
# enumerate the device's GATT and the control characteristic:
bluetoothctl; gatttool -b DEV_MAC --characteristics
# research (and the open-source "Buttplug"/ intimate-device libraries) has mapped many
# devices' control protocols; the recurring finding is that the control characteristic
# accepts commands with no pairing/authentication, so any nearby device can control it.
```
Aim: determine whether control requires authentication; its absence (common) is the core finding, because it permits unauthorized control.

### Cloud/app (aim: data exposure)

Proxy the app and test the backend (see `../../web/IDOR-and-BOLA.md`, `../../web/REST-API-Security.md`): cross-account access to usage data, messages, and media. Products in this space have exposed user emails, chat logs, and intimate images through weak backends and IDORs. For camera-equipped models, treat the stream like `IP-Cameras-and-Baby-Monitors.md`, with heightened sensitivity.

### Firmware (aim: keys, protocol)

Extract firmware where possible (nRF-based, see `../Microcontrollers-and-Arduino.md`) for the control protocol, keys, and any cloud credentials.

## Beyond the basics (what the research teaches)

- **Unauthenticated BLE control is the defining issue**: because pairing friction is minimized, control often has no authentication, enabling a nearby party to take control. In this context that is not a novelty bug; it is a consent/assault concern, which is why the defensive priority is authenticated, consented control.
- **Extreme data sensitivity**: breaches here expose uniquely harmful data (identity tied to intimate usage, chats, and images). Several vendors have had major incidents; data minimization, encryption, and strict access control are essential, and the impact framing must reflect the severity.
- **Camera/"long-distance" features**: models with cameras/video add a camera attack surface with the highest privacy stakes; apply the camera guidance with maximum care.
- **BLE tracking and presence**: static BLE identifiers can reveal presence/use; BLE privacy matters acutely here.
- **Consent as a security property**: the research community frames control authentication explicitly around consent; a design that lets anyone connect fails a consent requirement, not just a technical one.

## What to watch for (critical)

- The data and control here are exceptionally sensitive; test only your own device and account, never connect to or access anyone else's, and disclose responsibly with heightened care for the people affected.
- Frame every finding around consent, privacy, and potential for harm; unauthorized control is a serious personal-safety and consent violation, and data exposure is acutely harmful.
- To defend (the objective): require authenticated, consented BLE pairing and command authorization; use BLE privacy (rotating addresses); enforce strict ownership-based cloud authorization; minimize, encrypt, and tightly access-control all usage data, messages, and media; and secure any camera behind strong authentication.

## Checklist (defensive/learning)

- Enumerate BLE GATT; determine whether control is authenticated (the core finding) on your own device.
- Proxy the app/cloud; test cross-account access to usage data, messages, and media (your own accounts).
- For camera models, apply camera-grade scrutiny with heightened sensitivity.
- Extract firmware for the control protocol and keys where accessible.
- Frame strictly around consent, privacy, and harm; own device/account only; disclose with care.
