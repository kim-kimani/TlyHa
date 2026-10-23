# Alarm Sensors and Home Security

Home security systems (door/window contacts, motion sensors, the hub/panel, keypads, and sirens) are mostly cheap radios reporting to a panel. The aim is to defeat the alarm: suppress a sensor's report, spoof an "all clear," jam the system, or compromise the hub/cloud. Because this is physical security, test only your own gear, and the defensive lessons are the goal.

## What it is and why it is a target (the aim)

- **Sensor suppression**: prevent a door/window/motion sensor from reporting an intrusion. Aim: open a door without the panel registering it (jamming or signal manipulation).
- **Spoofing**: forge sensor messages to trigger false alarms or mask real ones. Aim: inject "closed/clear" or flood the panel.
- **Hub/panel compromise**: the panel is the brain and often cloud-connected. Aim: disarm via the hub, cloud, or app.
- **Credential/key recovery**: pull keys from the hub firmware or sniff the sensor protocol. Aim: forge authenticated messages.

## Architecture (the four sides)

- **Device**: battery sensors (a tiny MCU + sub-GHz or Zigbee/Z-Wave radio) and a hub/panel (a richer Linux/MCU device with a cellular/Wi-Fi/Ethernet uplink).
- **Radio**: sub-GHz (315/345/433/868 MHz) for many sensors, or Zigbee/Z-Wave for smart-home-integrated systems.
- **Cloud**: monitoring service and app (arm/disarm, notifications).
- **App**: control and alerts.

## Step-by-step how-to

### 1. Identify the sensor radio and protocol (aim: know the link)

```
# capture the sensor-to-panel traffic with an SDR:
rtl_433 -f 345M        # try 315/345/433/868; rtl_433 decodes many alarm-sensor protocols
# you will often see per-sensor IDs and state (open/closed, tamper, low-battery).
```
Aim: decode the sensor messages and learn whether they are authenticated or just an ID + state in the clear.

### 2. Replay and spoofing (aim: forge sensor state)

```
# many legacy sensors send unauthenticated, unencrypted state frames:
#  capture a "closed/clear" frame and replay it, or forge one with a known sensor ID,
#  to mask an open door. (transmit only on your own system, where legal)
```
Aim: demonstrate that a sensor's state can be spoofed because the panel trusts any frame with the right ID.

### 3. Jamming (aim: suppress reporting)

Legacy sub-GHz sensors are vulnerable to RF jamming: flood the frequency so the panel never hears the "door opened" message. Some panels detect jamming (supervision) and some do not. Aim: demonstrate (on your own system) whether jamming suppresses an alarm and whether jam-detection fires. This is a well-documented weakness of unencrypted, unsupervised sensor links.

### 4. Hub/panel, cloud, and app (aim: disarm)

Treat the hub as a normal device: network-scan it, UART/firmware for keys and the protocol (see `../Firmware-Analysis-and-Extraction.md`), and proxy the app to test the cloud API for remote arm/disarm authorization (see `../../web/IDOR-and-BOLA.md`). Aim: disarm or read state cross-account, or recover the keys that authenticate sensor/keypad messages.

## Beyond the basics

- **Unauthenticated sensor protocols are the core flaw**: many systems (including some big-brand ones) send sensor state with no encryption or authentication, enabling replay, spoofing, and (with the per-sensor ID) targeted suppression. Supervision (periodic check-ins and jam detection) is the mitigation; its absence is the finding.
- **Jam-and-the-panel-stays-silent**: the practical burglary technique is jamming the sensor band during entry; systems without jam detection simply miss the event. Demonstrate detection behavior on your own panel.
- **Keypad and code attacks**: wireless keypads may send codes with weak/no encryption (sniff the disarm code); wired keypads and panels may have default installer codes.
- **Z-Wave/Zigbee security systems**: capture the secure join / exploit weak key exchange to join the network and inject disarm/sensor messages (S0 Z-Wave key-exchange weakness is a known example).
- **Cellular uplink and cloud**: the panel's cellular/cloud reporting can be jammed (to stop it calling the monitoring center) or attacked at the cloud layer for remote disarm; test the app/cloud authorization.
- **Battery/tamper abuse**: forcing low-battery/tamper states or exploiting fail-open behaviors.

## Defensive framing / what to watch for

- This is physical security; test only your own system, and transmit/jam only where legal and on your own gear.
- The findings are unauthenticated/unencrypted sensor links, no jam detection/supervision, weak keypad codes, and weak cloud authorization; defenses are encrypted+authenticated sensor protocols, supervision and jam detection, rolling/authenticated messages, and ownership-based cloud arm/disarm.
- Frame impact as alarm defeat and physical entry.
- To defend: choose systems with encrypted, authenticated, supervised sensors and jam detection; use strong unique codes; keep the hub updated; and ensure the monitoring uplink has anti-jam/redundancy.

## Checklist

- Identify sensor radio/protocol; decode state frames and sensor IDs.
- Test replay/spoofing of sensor state (your own system).
- Test jamming and whether the panel detects it/supervises sensors.
- Attack the hub/panel: network, firmware/keys, and cloud/app arm-disarm authorization.
- For Zigbee/Z-Wave systems, test the join/key exchange.
- Frame by alarm defeat and physical entry; stay on your own gear and legal bands.
