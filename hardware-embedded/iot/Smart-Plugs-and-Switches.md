# Smart Plugs and Switches

A smart plug is the simplest smart-home device and therefore the best beginner hardware project: a small MCU (very often an ESP8266/ESP32) with a relay, Wi-Fi, and a cloud account. The aim is modest on its own (toggle power, read the Wi-Fi key) but plugs teach the whole IoT method cleanly, and they are a common pivot and credential-leak source.

## What it is and why it is a target (the aim)

- **Wi-Fi credential leak**: the plug stores the home Wi-Fi password in flash. Aim: extract it (a discarded or borrowed plug leaks the network).
- **Local/cloud control**: toggle the relay, and via cloud bugs, toggle other people's plugs. Aim: unauthorized control, and cross-account control via the cloud.
- **Pivot and recon**: the plug is a Linux-less MCU but still a network node and a source of cloud tokens.
- **Learning platform**: plugs are the canonical ESP hacking target and often reflashable with open firmware.

## Architecture (the four sides)

- **Device**: usually an ESP8266/ESP32 (Tuya-based plugs are extremely common) plus a relay and sometimes a power-metering chip.
- **Radio**: Wi-Fi, direct to a vendor cloud (often Tuya's cloud).
- **Cloud**: vendor/Tuya cloud with an MQTT or HTTP control channel and the app account.
- **App**: provisioning and control.

## Step-by-step how-to

### 1. Identify the chip (aim: confirm ESP and the toolchain)

Open the plug; if it is an ESP8266/ESP32 (or a Tuya module wrapping one), the `../Microcontrollers-and-Arduino.md` toolchain applies directly. This is the friendliest possible first hardware target.

### 2. Dump flash and extract secrets (aim: Wi-Fi creds and cloud token)

```
esptool.py --port /dev/ttyUSB0 read_flash 0x0 0x200000 plug.bin
strings plug.bin | grep -iE 'ssid|pass|psk|token|key|mqtt|tuya'
# the NVS/user partition commonly holds the Wi-Fi SSID/PSK and the cloud device
# token/secret in the clear.
```
Aim: recover the Wi-Fi password and the cloud credentials from your own plug, demonstrating the disposal/credential-leak risk.

### 3. Local control and provisioning (aim: toggle and setup abuse)

```
# during setup, many plugs expose an open AP or a weakly-secured provisioning flow;
# some (Tuya) have a documented local control protocol on the LAN once the local key
# is known (extractable from the app/cloud or flash).
# toggle the relay via the local or cloud protocol as proof of control.
```

### 4. Cloud/app (aim: cross-account control)

Proxy the app, map the cloud/MQTT API, and test device-id authorization (see `../../web/IDOR-and-BOLA.md` and the MQTT pattern in `../IoT-and-Smart-Home-Devices.md`). Aim: control another account's plug by id, or subscribe to its state, on your own test devices.

## Beyond the basics

- **Tuya ecosystem specifics**: a huge fraction of white-label plugs/switches are Tuya-based. The local key (per-device, used for LAN control) can often be retrieved from the app/cloud or flash; with it, local control tools talk to the plug directly. Studying the Tuya local protocol covers a large swath of the market at once.
- **Open-firmware reflashing (Tasmota/ESPHome)**: the community reflashes ESP plugs to local-only firmware, which both demonstrates the lack of secure boot (you can replace the firmware) and is a great way to learn the hardware/OTA path. Unsigned OTA on these plugs is remote firmware replacement.
- **Power metering as a side channel**: metering plugs report power draw; this leaks appliance usage patterns (occupancy, what is running), a minor privacy channel worth noting.
- **Shared keys**: check whether cloud/MQTT credentials are unique per device or shared; shared secrets are fleet-wide.
- **Relay/foothold**: while an MCU plug is not a full Linux pivot, its extracted Wi-Fi key and cloud token can bridge to the network and the fleet.

## Defensive framing / what to watch for

- The main real issues are cleartext stored Wi-Fi/cloud credentials, unsigned OTA, and cloud control by device id; defenses are encrypted storage, unique keys, signed updates, and ownership-based authorization.
- Toggling a plug is low impact alone; the credential leak and cross-account control are the findings. Frame accordingly.
- Reflashing and local-key extraction are for your own devices.
- To defend: encrypt stored secrets, enable ESP flash encryption/secure boot, sign OTA, use per-device cloud keys with ownership checks, and advise that factory reset may not wipe flash.

## Checklist

- Open the plug; confirm ESP/Tuya; apply the ESP toolchain.
- Dump flash; extract Wi-Fi creds and cloud token; check unique vs shared.
- Explore local control/provisioning and the Tuya local key.
- Proxy the app/cloud; test device-id/MQTT cross-account control.
- Check OTA signing (reflash risk) on your own unit.
- Frame impact by credential leak and fleet control, not just toggling power.
