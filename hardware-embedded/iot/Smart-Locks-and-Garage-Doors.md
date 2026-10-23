# Smart Locks and Garage Doors

These control physical access to a home, so the aim is unambiguous: open the door. The attack surfaces are the radio (BLE, sub-GHz, Zigbee/Z-Wave), the cloud/app, and the hardware. Because the impact is physical entry, this is approached carefully and only on your own locks, and the defensive lessons are the point.

## What it is and why it is a target (the aim)

- **Unauthorized entry**: open the lock or garage door. Aim: replay, forge, or relay a command, or recover a key that lets you command the lock.
- **Credential/key recovery**: pull the lock's keys from firmware or sniff them over the radio. Aim: a key that unlocks the fleet or this unit.
- **Cloud bypass**: grant yourself access or open the lock via cloud authorization flaws. Aim: cross-account unlock or unlogged entry.
- **Audit/log evasion**: open without leaving a record. Aim: understand whether entry is detectable.

## Architecture (the four sides)

- **Device**: an MCU (often with a BLE or sub-GHz radio) driving the bolt motor; a secure element in better designs.
- **Radio**: BLE (phone-as-key), sub-GHz 315/433/868 MHz (garage remotes), or Zigbee/Z-Wave (hub-based locks).
- **Cloud**: for Wi-Fi/bridge locks, remote unlock and guest-access management.
- **App**: the phone key, guest sharing, and logs.

## Step-by-step how-to

### Garage doors and fixed-code remotes (aim: replay)

```
# many older garage/gate remotes use FIXED codes (same bits every press) on 315/433 MHz:
rtl_433 -f 433.92M            # identify/decode the remote protocol
# capture the signal with an SDR / Flipper, then retransmit to open:
#  a fixed-code remote replays perfectly -> the door opens. (see ../Radio-and-Wireless.md)
```
Aim: demonstrate that a fixed-code or weak-rolling-code opener is replayable. Rolling codes resist naive replay; RollJam-style jam-and-capture can still defeat weak implementations.

### BLE smart locks (aim: unauthenticated or replayable unlock)

```
# enumerate the lock's GATT services/characteristics:
bluetoothctl         # scan/connect; then gatttool or nRF Connect:
gatttool -b LOCK_MAC --characteristics
# look for a writable "command" characteristic; sniff a legitimate unlock with a BLE
# sniffer to see the unlock payload and whether it is static/replayable or uses a nonce.
```
Aim: find an unlock command that is unauthenticated, static, or replayable; or recover the session key if pairing is weak (Just Works). Keys derived from the lock's MAC are a known weakness.

### Cloud/app (aim: cross-account unlock and guest-access abuse)

Proxy the app; test the cloud API (see `../../web/IDOR-and-BOLA.md`): can you unlock a lock you do not own by id, escalate a revoked guest back to access, or replay an unlock command. Test whether guest/temporary access is properly revoked and whether unlocks are logged.

### Hardware (aim: keys and a direct actuation path)

Open the lock; UART/SWD for a shell/debug; dump firmware (see `../Firmware-Analysis-and-Extraction.md`) to find keys, the unlock protocol, and whether a secure element protects them. On some locks the motor driver can be actuated directly once open, which is a tamper finding.

## Beyond the basics

- **Relay attacks (the big one for proximity keys)**: for BLE/keyless systems that unlock when the phone/fob is "near," relay the radio signal between the real key (e.g. inside the house) and the lock so the lock believes the key is present. Defeats presence-based unlock without any crypto break; the same class as car keyless-entry relay (see `../Automotive-and-CAN-Bus.md`).
- **Jam-and-replay / RollJam**: jam the receiver so a pressed rolling code is not accepted, capture it, let the user press again (you capture a second), forward the first, keep the second for later. Defeats weak rolling codes.
- **Key-from-MAC and shared keys**: if the unlock key is derived from the public BLE MAC or shared across the product line, recover it once and unlock many; check the firmware's key derivation.
- **Z-Wave/Zigbee lock joins**: capturing the secure join or exploiting weak key exchange can yield the network key and command the lock; S0 Z-Wave had a known key-exchange weakness.
- **Denial and alarm suppression**: jamming the lock/sensor radio can prevent it reporting state; relevant where the lock is part of a monitored system.
- **Log/audit gaps**: entry methods that bypass the app may not be logged; demonstrate whether an unlock is recorded.

## Defensive framing / what to watch for

- This is physical access; test exclusively on your own locks, and treat any "it opened" result with care.
- A replayable code or unauthenticated BLE unlock is a serious finding; the defenses are rolling codes with anti-jam-replay, authenticated BLE (not Just Works), per-device keys in a secure element, presence checks resistant to relay (distance bounding), and logged/verified cloud unlocks with proper guest revocation.
- Proximity is weaker protection than it seems (antennas and relays extend range); do not dismiss a BLE/sub-GHz issue as low because it "needs to be near."
- Frame impact as physical entry and safety.

## Checklist

- Identify the radio (BLE, sub-GHz, Zigbee/Z-Wave) and whether unlock is presence-based.
- For garage/gate remotes, capture and test replay; assess fixed vs rolling code (and RollJam).
- For BLE locks, enumerate GATT, sniff the unlock, test for static/replayable/unauthenticated commands; assess pairing strength and key-from-MAC.
- Test relay attacks on presence-based unlock.
- Proxy the app/cloud; test cross-account unlock, guest-revocation, and unlock logging.
- Extract firmware; find keys and the unlock protocol; check for a secure element.
- Test only your own locks; frame by physical-access impact.
