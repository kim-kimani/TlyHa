# Radio and Wireless

Everything smart talks over the air, and radio is a boundary people forget to secure because it feels invisible. Wi-Fi, Bluetooth/BLE, Zigbee, Z-Wave, and sub-GHz links (key fobs, sensors, garage doors) carry commands and secrets that are often unauthenticated, replayable, or weakly encrypted. Software-defined radio turned this from specialist territory into something you can explore with a cheap dongle.

## The mental model

A radio protocol is just a message format modulated onto a frequency. If you can receive and transmit on that frequency and understand the format, you can read the traffic, replay it, or forge it, unless cryptographic authentication prevents it. The security questions mirror the network ones: is the channel encrypted, are messages authenticated, is there replay protection (a rolling counter or nonce), and is pairing/key-exchange secure. The answers for cheap IoT are frequently "no."

## Where it hides / tell-tales (by technology)

- **Wi-Fi**: the device's own AP during provisioning (open or default-password), WPA2 handshake capture for offline cracking, deauth to force re-provisioning, and KRACK-style protocol flaws on unpatched gear. Tells: an open setup SSID, a predictable default password, and management frames that are not protected (no 802.11w), enabling deauth.
- **Bluetooth Low Energy (BLE)**: GATT services/characteristics that control the device, often with no pairing or "Just Works" pairing (no authentication). Tells: writable characteristics that trigger actions (unlock, toggle) readable/writable by any nearby device, static passkeys, and no link encryption. Keys derived from the MAC.
- **Zigbee / Z-Wave (smart home mesh)**: network keys sometimes transmitted in the clear during joining (the classic Zigbee "Trust Center link key" default of `ZigBeeAlliance09`), replayable commands, and weak key management. Tells: devices that pair without a secure key exchange.
- **Sub-GHz (433/315/868/915 MHz)**: key fobs, garage doors, doorbells, alarm sensors, and some industrial links. Tells: fixed-code transmitters (literally the same bits every time, fully replayable) and weak rolling-code schemes.
- **NFC/RFID**: access cards and tags; cloning weak (static-UID or cracked-key) cards.

## Real material

Capturing and replaying a sub-GHz fixed-code signal (e.g. a cheap garage/doorbell/outlet remote) with an RTL-SDR or a tool like a Flipper/HackRF:

```
# receive and record with an SDR (rtl_433 decodes many common protocols automatically):
rtl_433 -f 433.92M                       # identify the device and see decoded packets
# capture raw I/Q for replay with a TX-capable radio (HackRF):
hackrf_transfer -r capture.iq -f 433920000 -s 2000000
# replay it back:
hackrf_transfer -t capture.iq -f 433920000 -s 2000000
# a fixed-code remote replays perfectly -> you reproduce the button press.
```

Enumerating and poking a BLE device:

```
# scan for devices and their services/characteristics:
bluetoothctl                     # scan on; then connect <MAC>
gatttool -b <MAC> --primary      # list services
gatttool -b <MAC> --characteristics
# read/write a characteristic (e.g. an unauthenticated "command" handle):
gatttool -b <MAC> --char-write-req -a 0x0025 -n 01    # try a command value
# or use nRF Connect / bleah for a friendlier interface; sniff pairing with a sniffer dongle.
```

Capturing a WPA2 handshake for offline cracking (your own network):

```
airmon-ng start wlan0
airodump-ng wlan0mon                     # find the AP + channel
airodump-ng -c <ch> --bssid <AP> -w cap wlan0mon
aireplay-ng --deauth 5 -a <AP> wlan0mon  # force a client to reconnect -> handshake
hashcat -m 22000 cap.hc22000 wordlist.txt
```

## Beyond the basics

- **Replay is the first test and often the whole bug**: for sub-GHz fobs, many BLE commands, and some Zigbee/Z-Wave frames, simply recording and retransmitting reproduces the action. Fixed-code remotes are trivially cloneable. The defense is a rolling code / counter / nonce that the receiver tracks; its absence is the finding. Even rolling codes have been broken (RollJam-style: jam the receiver, capture two codes, use the first, keep the second).
- **BLE "Just Works" and GATT-without-auth**: the recurring BLE bug is a device that exposes action characteristics with no pairing or with unauthenticated "Just Works" pairing, so any phone in range can send commands (unlock a lock, open a gate, read sensor data). Enumerate the GATT table and try writing to command-looking characteristics. Also sniff the pairing to recover keys when pairing is weak.
- **Key management is where mesh protocols fail**: Zigbee/Z-Wave security depends on the network/link keys. Default well-known keys, keys sent in the clear at join time, and no re-keying let an attacker join the mesh or decrypt traffic. Capturing a device joining can hand you the network key.
- **Jamming and denial as part of the attack**: deauth (Wi-Fi), BLE jamming, and sub-GHz jamming are used both as DoS and as enablers (force re-pairing to capture a handshake, jam-and-replay rolling codes, block an alarm sensor's "I'm open" signal). Note these as availability and bypass risks.
- **Protocol reverse engineering with SDR**: for undocumented links, capture the raw signal, identify the modulation (OOK/ASK, FSK, GFSK) and encoding (Manchester, PWM) in a tool like Universal Radio Hacker (URH), and reconstruct the packet format. Then you can forge arbitrary commands, not just replay captured ones.
- **Cross-technology chaining**: a weak radio link is often the local foothold that reaches the device, which then reaches the cloud account (see `IoT-and-Smart-Home-Devices.md`). A BLE command to a lock, or a sub-GHz replay to a gate, is a direct physical-security impact; a captured provisioning secret bridges to the cloud fleet.
- **Distance and antennas change the threat model**: "needs proximity" is weaker protection than it sounds; directional antennas and amplifiers extend BLE and sub-GHz range far beyond the advertised few meters.

## What to watch for (defensive framing)

- Transmitting on many frequencies is regulated; only transmit on bands and power levels you are legally permitted to use, and test your own devices. Receiving is generally passive, but replay/forge requires care and authorization.
- A successful replay proves missing anti-replay, but confirm the real-world effect (the device actually actuates) and that it is your device.
- Proximity-gated attacks are still real; do not dismiss a BLE/sub-GHz bug as low purely because it needs range.
- To defend: encrypt and authenticate radio links, use rolling codes/nonces with proper receiver-side tracking (and anti-jam-replay measures), require authenticated pairing (not "Just Works") for action characteristics, use secure key exchange and per-device keys for mesh protocols, enable 802.11w to resist deauth, and never ship fixed-code or static-UID security functions.

## Hunting checklist

- Identify the radio technologies the device uses (Wi-Fi, BLE, Zigbee/Z-Wave, sub-GHz, NFC).
- For sub-GHz remotes/sensors, capture and attempt replay; check for fixed vs rolling codes.
- Enumerate BLE GATT services/characteristics; test unauthenticated writes to command handles; assess pairing strength.
- For Zigbee/Z-Wave, capture a join and check key management (default/cleartext keys).
- Capture WPA2 handshakes on your own network and test deauth resistance (802.11w).
- For undocumented links, reverse the protocol with SDR + URH to forge commands.
