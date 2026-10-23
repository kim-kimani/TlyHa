# Smart Light Bulbs

A smart bulb is a tiny radio-connected computer that most people would never imagine as a target. It is, famously, the device that demonstrated Zigbee worms, network-key extraction from discarded units, and covert data exfiltration by flickering. For someone new to hardware, bulbs are a great teacher: cheap, radio-based, and full of lessons about keys and radios.

## What it is and why it is a target (the aim)

- **Network key extraction**: a Wi-Fi bulb stores the home Wi-Fi credentials; a Zigbee bulb holds the Zigbee network key. Aim: pull those keys from the flash, so a bulb (even one thrown in the trash) leaks the home network or lets you join the mesh.
- **Mesh worm / mass control**: over Zigbee, a malicious update or command can spread bulb to bulb across a building. Aim: understand and demonstrate mesh propagation and mass control in your lab.
- **Covert channel**: modulating brightness (including near-IR, invisible to humans) exfiltrates data to a camera with line of sight. Aim: demonstrate data crossing an air gap via light.
- **Cloud/app control**: Wi-Fi bulbs tie to a vendor cloud. Aim: cross-account control via cloud bugs.

## Architecture (the four sides)

- **Device**: a small MCU (often an ESP8266/ESP32 in Wi-Fi bulbs, or a Zigbee SoC like a Silicon Labs/NXP part in Zigbee bulbs) plus the LED driver.
- **Radio**: Wi-Fi (direct-to-cloud bulbs) or Zigbee/Z-Wave/BLE (hub-based bulbs). The radio is the primary interface.
- **Cloud**: for Wi-Fi bulbs, a vendor cloud and app. For Zigbee, a hub bridges to the cloud.
- **App**: provisioning and control.

## Step-by-step how-to

### 1. Identify the radio and chip (aim: know what you are attacking)

Open a bulb (they pry apart); identify the main chip. An ESP means Wi-Fi and the ESP toolchain applies (see `../Microcontrollers-and-Arduino.md`); a Zigbee SoC means Zigbee attacks and a sniffer.

### 2. Extract firmware and keys (aim: the network key / Wi-Fi creds)

For an ESP-based Wi-Fi bulb, dump the flash and read the stored secrets:

```
esptool.py --port /dev/ttyUSB0 read_flash 0x0 0x400000 bulb.bin
strings bulb.bin | grep -iE 'ssid|pass|psk|token|key|mqtt'
# the NVS partition commonly holds the Wi-Fi SSID/PSK and cloud token in the clear.
```
This is the "discarded bulb leaks your Wi-Fi" finding: the credentials sit in flash with no encryption on many units.

### 3. Zigbee bulbs: sniff and join the mesh (aim: network key, mesh control)

```
# with a Zigbee sniffer (e.g. a TI CC2531/CC1352 running a sniffer firmware) + Wireshark:
#  capture the join (commissioning) of a bulb. Legacy Zigbee sends the network key
#  encrypted with a well-known default "Trust Center link key" (ZigBeeAlliance09),
#  so capturing a join can reveal the network key.
# with the network key, you can decrypt mesh traffic and inject commands.
```
Aim: recover the Zigbee network key, then demonstrate decrypting traffic and sending on/off/color commands in your lab mesh.

### 4. Wi-Fi bulb cloud/app (aim: cross-account control)

Proxy the app, map the cloud API, and test device-id authorization (see `../../web/IDOR-and-BOLA.md`). Many bulbs use a shared MQTT broker keyed by device id; test whether you can subscribe/publish to another device's topics (see `../IoT-and-Smart-Home-Devices.md` for the MQTT pattern).

### 5. Covert channel (aim: data over light)

Demonstrate exfiltration by modulating brightness via the control API or firmware, and recovering the signal with a light sensor/camera. Near-IR LEDs in some bulbs allow an invisible channel. This is a research demonstration of air-gap bridging.

## Beyond the basics

- **The Zigbee worm concept**: research showed a malicious over-the-air update spreading bulb to bulb via the Zigbee radio across a city block, exploiting a weak OTA signing/proximity check. Study the OTA update path: is it signed, is it proximity-limited, can one compromised bulb push to neighbors.
- **Key reuse and derivation**: check whether the Zigbee/Wi-Fi/cloud keys are unique per device or shared across the product; shared keys make one extraction a fleet key.
- **BLE provisioning**: some bulbs provision over BLE with a weak or static passkey; sniff the pairing to recover Wi-Fi creds in transit (see `../Radio-and-Wireless.md`).
- **Hub attacks**: for Zigbee/Z-Wave bulbs, the hub is a richer target (a Linux device bridging to the cloud); pivot there.

## Defensive framing / what to watch for

- A bulb is a networked computer with a key store; the core defenses are encrypting stored credentials, unique per-device keys, signed and proximity/authenticated OTA, and secure Zigbee commissioning (not default link keys).
- Confirm you actually recovered a working key/credential before claiming it; strings in flash can be stale or placeholder.
- Transmitting on Zigbee/BLE and modulating devices is for your own lab; receiving/sniffing is passive.
- The discarded-device leak is a real disposal risk: advise factory-reset and that reset may not wipe flash secrets.

## Checklist

- Open the bulb; identify Wi-Fi (ESP) vs Zigbee/Z-Wave/BLE radio and the chip.
- Dump flash; extract Wi-Fi creds / cloud tokens / keys; check if unique or shared.
- For Zigbee, sniff a join; attempt network-key recovery; test mesh command injection in your lab.
- Proxy the app/cloud; test device-id and MQTT-topic authorization.
- Study the OTA update path for the worm/mass-control risk.
- Demonstrate the brightness covert channel as the "air gap" lesson.
