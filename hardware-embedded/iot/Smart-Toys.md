# Smart Toys

Connected toys (talking dolls, kid smartwatches, app-controlled robots, learning tablets) are IoT devices pointed at children, with microphones, cameras, location, and cloud accounts full of childrens' data. They have produced some of the largest and most sensitive IoT breaches. The aim is understanding the acute privacy risk (childrens' voice, images, location, and personal data) and the usual IoT weaknesses behind it.

## What it is and why it is a target (the aim)

- **Childrens' personal data**: cloud accounts hold names, birthdays, photos, voice recordings, chat logs, and sometimes location. Aim: identify cross-account/fleet data exposure (several real breaches leaked millions of childrens' records and voice clips).
- **Live mic/camera/location**: many toys and kid smartwatches have mics, cameras, and GPS. Aim: understand unauthorized monitoring and (for watches) cross-account tracking and even two-way calling to a child.
- **Unauthenticated BLE control**: app-connected toys often use BLE with no real authentication. Aim: control the toy (talk through it) from nearby.
- **Account/cloud flaws**: the vendor backends are frequently weak. Aim: cross-account access.

## Architecture (the four sides)

- **Device**: an MCU/Linux toy with BLE and/or Wi-Fi, a mic/speaker, sometimes a camera/GPS.
- **Radio**: BLE (app control) and/or Wi-Fi/cellular (watches) to a cloud.
- **Cloud**: account with childrens' profiles, media, and location.
- **App**: parent control, messaging, and location.

## Step-by-step how-to (your own device / authorized)

### 1. BLE control toys (aim: unauthenticated control)

```
# many BLE toys expose command characteristics with no authentication:
bluetoothctl; gatttool -b TOY_MAC --characteristics      # enumerate GATT
# find a writable "speak"/"move"/"play audio" characteristic and trigger it; a famous
# case let anyone nearby make a connected doll speak and use its mic over unauthenticated BLE.
```
Aim: show that a nearby device can control the toy (including speaking through it or activating the mic) with no pairing/authentication.

### 2. Cloud/app (aim: childrens' data and cross-account access)

Proxy the app and test the backend (see `../../web/REST-API-Security.md`, `../../web/IDOR-and-BOLA.md`): can you read another child's profile, photos, voice recordings, messages, or location by changing an id. Kid-smartwatch platforms have repeatedly had BOLA letting strangers track and message any child on the platform, the most serious class here.

### 3. Hardware/firmware (aim: secrets and local access)

UART/firmware extraction (see `../Firmware-Analysis-and-Extraction.md`) for cloud credentials, shared keys, default credentials, and the command protocol. Check whether stored media/credentials are encrypted.

## Beyond the basics

- **Kid smartwatches are the acute case**: they combine GPS, a mic, and two-way calling with weak cloud authorization. Documented flaws let attackers locate a child in real time, spoof the location, call/message the child, and enable the mic remotely. The fleet-wide cloud BOLA is the mechanism; it is a web/cloud bug with extreme real-world stakes.
- **Massive data breaches**: connected-toy vendors have leaked millions of childrens' and parents' records, including photos and voice recordings, through exposed databases and weak backends. The data at rest and its exposure are often the biggest issue.
- **Unauthenticated BLE is endemic**: toys optimize for easy pairing, so action characteristics are frequently open to any nearby device; the fix is authenticated pairing and command auth.
- **Voice/image retention**: recordings of children are stored (sometimes insecurely); understanding the retention and protection is part of the assessment.
- **Regulatory weight**: childrens' data carries specific legal protection (COPPA in the US, GDPR-K in the EU); findings here have strong compliance implications and should be disclosed responsibly.

## Defensive framing / what to watch for

- This involves childrens' data and safety; test only your own device and account, never access another child's data or device, and disclose responsibly given the sensitivity and legal weight.
- Headline issues are unauthenticated BLE control, cross-account cloud access (tracking/messaging children), and insecure storage of childrens' media/data; defenses are authenticated pairing/commands, ownership-based cloud authorization, encrypted storage, and data minimization.
- Frame impact squarely as child safety and privacy.
- To defend: require authenticated BLE pairing and command authorization, enforce strict ownership-based cloud authz, encrypt and minimize stored childrens' data, secure the mic/camera/location behind strong auth, and comply with childrens'-data regulation.

## Checklist

- Enumerate BLE GATT; test unauthenticated control (speak/mic/move) on your own toy.
- Proxy the app/cloud; test cross-account access to profiles, media, location, and messaging (your own accounts).
- For kid watches, specifically test location read/spoof, remote mic, and messaging authorization.
- Extract firmware; find shared keys, default creds, and check media/credential encryption.
- Treat as child-safety/privacy critical; own device/account only; disclose responsibly.
