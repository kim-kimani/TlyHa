# IoT and Smart Home Devices

Smart TVs, fridges, washing machines, air conditioners, cameras, door locks, and plugs are small Linux (or RTOS) computers with a cloud account, a phone app, and a radio, shipped fast and patched rarely. The attack surface is the sum of all four: the device, the app, the cloud API, and the radio. The weaknesses are depressingly consistent across vendors, which is what makes them learnable and defendable.

## The mental model

A smart-home product is a four-sided system: the device (embedded Linux/RTOS, local services, firmware), the mobile app (control plane, see `../mobile/`), the cloud backend (the API that both talk to, see `../web/REST-API-Security.md`), and the local/radio interface (Wi-Fi, BLE, Zigbee, Z-Wave, matter, IR, see `Radio-and-Wireless.md`). An attacker picks the weakest side. The recurring failure is that the cloud treats device-to-cloud and app-to-cloud messages as trusted, and the device trusts the local network, so crossing one boundary often grants control of the whole account's devices.

## Where it hides / tell-tales (by device)

- **Smart TVs**: full Android TV/Tizen/webOS systems with browsers, app stores, ADB (often enabled over the network), debug ports, and microphones/cameras. Tells: open ADB on port `5555`, exposed developer menus, unauthenticated local HTTP control APIs (DIAL, companion-screen APIs), and apps loading content over HTTP. The remote-control companion protocols are frequently unauthenticated on the LAN.
- **Fridges / washing machines / AC (white goods)**: a Wi-Fi module (often an ESP32 or a vendor module) bridging the appliance's internal bus (UART) to a cloud MQTT/HTTP service. Tells: UART between the main board and the Wi-Fi module (sniff it to learn the command protocol), hardcoded MQTT credentials shared across all units, cloud topics keyed only by device id (so you can subscribe/publish to other people's appliances), and provisioning that trusts the local network.
- **Cameras and doorbells**: RTSP/ONVIF streams (often weakly or not authenticated), cloud relay for remote viewing, and firmware with the classic backdoor accounts. Tells: default credentials, unauthenticated RTSP, and cloud APIs that let you enumerate other users' camera ids.
- **Locks and plugs**: BLE or Zigbee control; tells: replayable or unauthenticated BLE commands, keys derived from the device MAC, and a cloud API that authorizes by device id rather than ownership.

## Real material

LAN reconnaissance of a smart device:

```
nmap -sS -sV -p- 192.168.1.50            # full port scan of the device
# common finds: 5555 (ADB on Android TV), 554 (RTSP), 1883/8883 (MQTT),
#               80/443/8080 (local admin/control API), 23 (telnet), 22 (ssh), 49152+ (UPnP)
adb connect 192.168.1.50:5555            # if ADB is open: shell on a TV
curl http://192.168.1.50/api/... -v      # probe the local control API, often unauthenticated
```

MQTT abuse (the backbone of most white-goods and many smart devices). If credentials are shared or topics are guessable:

```
# subscribe to everything the broker lets you see:
mosquitto_sub -h broker.vendor-cloud.com -p 8883 -u <leaked-user> -P <leaked-pass> -t '#' -v
# device state and commands often flow on topics like:
#   vendor/appliance/<deviceId>/status    vendor/appliance/<deviceId>/cmd
# if authorization is only by topic-name knowledge, publishing a command controls the appliance:
mosquitto_pub -h ... -t 'vendor/appliance/<victim-deviceId>/cmd' -m '{"power":"on","temp":16}'
```

Sniffing the internal UART bus between an appliance board and its Wi-Fi module reveals the plaintext command set (see `Hardware-Hacking-Fundamentals.md`), which you then replay from the cloud or local side.

## Beyond the basics

- **Cross the cloud boundary to control the fleet**: the highest-impact IoT bugs are cloud-side authorization failures (BOLA on `deviceId`, see `../web/IDOR-and-BOLA.md`). If the cloud lets your account send commands to a device id you do not own, or subscribe to its telemetry, you control or surveil strangers' homes at scale. This beats any single-device hardware attack. Always test the cloud API for device-ownership enforcement.
- **Shared secrets are a fleet-wide key**: many products embed one MQTT credential, one API key, or one firmware-signing key across every unit (extract it per `Firmware-Analysis-and-Extraction.md`). One extracted secret compromises the entire product line. This is the defining IoT anti-pattern.
- **The local network is treated as trusted**: devices expose unauthenticated control APIs, debug services, and provisioning flows on the LAN, assuming "if you are on the Wi-Fi you are allowed." A single compromised device, a guest on the network, or a cross-protocol attack (a malicious web page making requests to `192.168.x.x` via DNS rebinding) then controls everything. Test what the device does for a request from the LAN with no auth.
- **Provisioning and pairing**: the setup phase (often an open AP or BLE with a default password, or a cloud claim that is not tied to physical possession) is where devices get hijacked. A device-claim API that lets you claim a device by id, or re-claim one still owned by someone else, is account/device takeover.
- **Privacy surface**: TVs, cameras, and voice devices have mics and cameras; telemetry often exfiltrates viewing/usage data, sometimes over HTTP. The ability to enable a mic/camera or read telemetry cross-account is a severe privacy finding.
- **Update mechanism as both risk and defense**: OTA updates delivered over HTTP without signature verification let an on-path attacker push malicious firmware (persistent compromise). Conversely, a device that never updates accumulates known CVEs. Check how updates are fetched and verified.
- **Chaining across the four sides**: extract a shared secret from firmware, use it against the cloud MQTT to control other units, and confirm via the app. Or find a cloud BOLA, then use it to push a state that triggers a device-side bug. The product is only as strong as its weakest side, and the sides trust each other.

## What to watch for (defensive framing)

- Test only devices you own and your own cloud account/device ids; controlling or viewing someone else's appliance or camera is a real intrusion, so prove cross-account issues with a second account/device you control.
- A reachable local API is expected on your own LAN; the finding is missing authentication or cross-account control, not mere reachability.
- Shared-secret and BOLA findings affect every customer; demonstrate minimally and report for coordinated fixing.
- To defend: per-device unique credentials and keys in a secure element, cloud authorization by ownership not by id knowledge, authenticated local APIs (no implicit LAN trust), signed and verified OTA updates, disabled debug services (ADB/telnet) in production, and encrypted telemetry. The attacks above map one-to-one to these controls.

## Hunting checklist

- Scan the device on the LAN; identify open control/debug services (ADB, RTSP, MQTT, telnet, local HTTP).
- Test local control APIs for missing authentication and LAN-trust assumptions (including DNS-rebinding reachability).
- Extract firmware; pull shared MQTT/API/signing secrets and default credentials.
- Test the cloud API for device-ownership (BOLA on `deviceId`), claim/pairing abuse, and cross-account telemetry.
- Sniff internal UART and the radio to learn and replay the command protocol.
- Check OTA update fetching and signature verification.
