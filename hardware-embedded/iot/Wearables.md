# Wearables

Wearables (fitness trackers, smartwatches, continuous glucose monitors, smart rings) are sensor-dense devices strapped to a person, pairing over BLE to a phone and syncing to a cloud. They carry location, health, and biometric data, and some control or display safety-relevant information. The aim is the personal data (location, health, habits), BLE-level attacks, and cross-account cloud exposure.

## What it is and why it is a target (the aim)

- **Location and movement data**: trackers log GPS routes, steps, and timing. Aim: access location/activity data, which has real-world consequences (the Strava heatmap exposed military base locations and patrol patterns from aggregated fitness data).
- **Health/biometric data**: heart rate, sleep, glucose, and more. Aim: understand exposure of sensitive health data.
- **BLE attacks**: tracking by MAC, unauthenticated characteristics, pairing weaknesses, and notification sniffing. Aim: BLE-level data access and device tracking.
- **Cloud/account**: the sync backend with all the above. Aim: cross-account data.

## Architecture (the four sides)

- **Device**: a low-power MCU (often Nordic nRF BLE SoCs) with sensors, little storage, and a display.
- **Radio**: BLE to the phone (primary), some with Wi-Fi/LTE (standalone watches).
- **Cloud**: health/fitness account and social features.
- **App**: sync, display, and sharing.

## Step-by-step how-to (your own device / authorized)

### 1. BLE analysis (aim: data access and tracking)

```
# scan and enumerate the wearable's GATT:
bluetoothctl; gatttool -b DEV_MAC --characteristics
# look for readable health/notification characteristics without authentication, and
# sniff the pairing/sync with a BLE sniffer (nRF dongle) to assess encryption.
# BLE privacy: does the device rotate its MAC (random resolvable) or use a static MAC
# that lets you track the wearer across locations?
```
Aim: find unauthenticated readable data, sniff sync traffic, and assess MAC-based trackability of the wearer.

### 2. Cloud/app (aim: cross-account health/location data)

Proxy the app and test the fitness/health cloud API (see `../../web/IDOR-and-BOLA.md`, `../../web/REST-API-Security.md`): can you read another user's activity, routes, or health data by id; are "private" activities actually private; do social/friend features leak location. Fitness platforms have had IDORs exposing private routes and profiles.

### 3. Firmware/hardware (aim: secrets, raw data)

For devices you can access, extract firmware (see `../Microcontrollers-and-Arduino.md` for nRF-based ones) for keys, the sync protocol, and stored data. Many are nRF SoCs with the usual MCU readout-protection questions.

## Beyond the basics

- **Aggregate-data leaks are the surprising impact**: the Strava heatmap case showed that aggregated, "anonymous" fitness data revealed sensitive locations (military bases) and individual patterns. Privacy of location/activity data, individually and in aggregate, is the signature wearable risk. Frame this when assessing sharing/social features.
- **BLE tracking via static MAC**: if a wearable does not use BLE privacy (rotating random addresses), its static MAC lets anyone with sensors track the wearer's movements, a physical-surveillance privacy issue.
- **Health-data sensitivity and CGMs**: continuous glucose monitors and medical-adjacent wearables carry health data and sometimes drive decisions; some (like certain CGM/insulin ecosystems) have been studied for BLE command/data integrity, overlapping with `Medical-Devices.md`. Treat health-affecting wearables with that care.
- **Notification and message leakage**: smartwatches mirror phone notifications over BLE; unauthenticated or sniffable notification streams leak messages, OTPs, and more.
- **Pairing weaknesses**: "Just Works" BLE pairing (no authentication) enables man-in-the-middle and unauthorized connection; assess the pairing method.
- **Cloud social features**: friend/leaderboard/route-sharing features are a common source of unintended location/identity exposure; test their authorization and privacy defaults.

## Defensive framing / what to watch for

- Wearables carry location, health, and biometric data about a specific person; test only your own device/account and never access another person's data.
- Headline issues are location/health data exposure (individual and aggregate), BLE trackability and unauthenticated data/notifications, and cross-account cloud access; defenses are BLE privacy (rotating MACs), authenticated pairing and characteristics, strong cloud authorization, and privacy-protective sharing defaults.
- Frame impact by privacy and, for health-affecting devices, safety.
- To defend: use BLE random resolvable addresses, require authenticated pairing and gate sensitive characteristics, enforce ownership-based cloud authz, default sharing/social features to private, encrypt stored/synced health data, and minimize retained location data.

## Checklist

- Enumerate BLE GATT; test unauthenticated data/notification reads; sniff sync; assess MAC trackability.
- Proxy the app/cloud; test cross-account access to activity, routes, and health data (your own accounts).
- Assess sharing/social features for location/identity leakage and privacy defaults.
- Extract firmware (nRF etc.) for keys and the sync protocol where accessible.
- Treat health-affecting wearables like medical devices; own device/account only; frame by privacy/safety.
