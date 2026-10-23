# Smart Appliances (Fridge, Washer, AC)

Smart white goods (refrigerators, washing machines, air conditioners, dishwashers, ovens) share one architecture: the appliance's own control board plus a bolt-on Wi-Fi module that bridges an internal serial bus to a vendor cloud. The aim is to learn and abuse that bridge, extract shared cloud secrets, and test cross-account control, with the appliance's internal command protocol as the deep prize.

## What it is and why it is a target (the aim)

- **Cloud fleet control**: appliances phone home to a vendor cloud (often one shared MQTT/HTTP backend). Aim: control or read another customer's appliance via device-id authorization bugs, potentially at scale.
- **Shared-secret extraction**: the Wi-Fi module's firmware often holds cloud credentials shared across units. Aim: one key that unlocks the fleet.
- **Internal protocol control**: the Wi-Fi module talks to the appliance board over UART. Aim: learn that command set and issue commands (change AC setpoint, start/stop a cycle, alter fridge temperature).
- **Network foothold and privacy**: the module is a network node; usage telemetry leaks behavior.

## Architecture (the four sides)

- **Device**: two boards. The appliance control board (a dedicated MCU running the machine) and a separate Wi-Fi module (often an ESP32 or a vendor Wi-Fi SoC) connected to it by a UART bus.
- **Radio**: Wi-Fi from the module to the cloud.
- **Cloud**: vendor cloud, frequently a shared MQTT broker keyed by device id, plus the app account.
- **App**: control and monitoring.

## Step-by-step how-to

### 1. Find and sniff the internal UART bus (aim: the command protocol)

The highest-value hardware step is tapping the UART between the Wi-Fi module and the appliance board:

```
# locate the 3-4 wire link between the Wi-Fi module and the main board; put a logic
# analyzer on it (see ../Bus-Protocols-Deep.md) and capture while you operate the
# appliance from the app. Decode the UART to learn the command frames:
#  e.g. "set AC temp = 16C", "start wash cycle", "read fridge temp".
```
Aim: a decoded command set you can replay/forge from the module side (or, if the board trusts the bus, from your own injected frames).

### 2. Extract the Wi-Fi module firmware (aim: shared cloud secrets)

```
# if the module is an ESP: esptool.py read_flash ... then strings for secrets.
# otherwise dump its flash via SPI/UART (see ../Firmware-Analysis-and-Extraction.md).
strings module.bin | grep -iE 'mqtt|broker|token|cert|key|ssid|pass|api'
```
Aim: cloud MQTT/HTTP credentials and certs; check whether they are per-device or shared across all appliances of the model.

### 3. Cloud/app (aim: cross-account control, the fleet lever)

Proxy the app, map the cloud API/MQTT, and test device-id authorization (see `../../web/IDOR-and-BOLA.md` and the MQTT topic pattern in `../IoT-and-Smart-Home-Devices.md`). Aim: subscribe to or command another appliance by id. If topics are keyed only by a guessable device id and authorization is weak, that is fleet-wide control.

### 4. Network recon (aim: local services)

```
nmap -sS -sV -p- APPLIANCE_IP     # some modules expose a local API or debug service
```

## Beyond the basics

- **Shared MQTT credentials are the classic white-goods bug**: many vendors embed one broker credential across the fleet; extracting it from one unit lets you connect to the broker and, if topic authorization is weak, read/command many appliances. This has been demonstrated across major appliance brands.
- **Internal-bus trust**: the appliance board usually trusts anything on its UART (no authentication), so whoever controls the Wi-Fi module (or injects on the bus) controls the appliance functions the protocol exposes. This is why the UART sniff matters: it is the real control plane.
- **Physical-effect framing**: beyond novelty (blinking a display), appliance control has real stakes: an AC or oven setpoint, a water-valve cycle on a washer/dishwasher (leak/flood potential), or a fridge/freezer temperature (food safety). Frame impact by these physical effects, and demonstrate only on your own appliance.
- **Telemetry/privacy**: usage data (when you run the washer, open the fridge) leaks occupancy and habits, often over weakly protected channels.
- **OTA**: appliance Wi-Fi modules take OTA updates; unsigned OTA is persistent compromise of the module and thus the control bridge.

## Defensive framing / what to watch for

- The dominant findings are shared cloud secrets and cloud device-id authorization; defenses are per-device credentials, ownership-based authz, and authenticated internal buses.
- Confirm the internal command actually affects the appliance on your own unit before claiming physical impact.
- Frame by physical effects (temperature, water cycles) and privacy.
- To defend: unique per-device cloud credentials in a secure element, ownership-based cloud authorization (not device-id knowledge), authenticate the module-to-board bus, sign OTA, and segment appliances on a separate VLAN.

## Checklist

- Identify the two boards and the UART bus between the Wi-Fi module and the appliance.
- Sniff the internal UART while operating the appliance; decode the command set.
- Extract the Wi-Fi module firmware; pull cloud/MQTT secrets; check shared vs per-device.
- Proxy the app/cloud; test device-id/MQTT cross-account control (fleet lever).
- Check OTA signing on the module.
- Frame impact by physical effect and privacy; test only your own appliance.
