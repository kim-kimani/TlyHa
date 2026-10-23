# Robot Vacuums

A robot vacuum is a Linux computer on wheels with LIDAR or a camera, often a microphone, Wi-Fi, and a cloud account that stores a map of the inside of your home. People think of it as an appliance; it is a mobile sensor platform. The aim is the map and sensors (a serious privacy and physical-security leak) and, as always, fleet-wide cloud control.

## What it is and why it is a target (the aim)

- **Home mapping data**: the vacuum builds and uploads a floor plan with room layout and object positions. Aim: read that map (reveals your home's interior, when rooms are occupied, high-value-item locations).
- **Sensors**: camera (navigation/obstacle), microphone (some models), and LIDAR. Aim: access the camera/mic, and, as research showed ("LidarPhone"), use the LIDAR as a crude microphone via reflected-light vibration sensing.
- **Cloud fleet control**: the vendor cloud controls and stores data for every unit. Aim: cross-account map/camera access and remote control.
- **Mobility**: it moves, so a compromised unit can reposition its camera.

## Architecture (the four sides)

- **Device**: a fairly powerful ARM Linux SoC (navigation and mapping need compute), running embedded Linux with Wi-Fi, often with a debug UART and an eMMC.
- **Radio**: Wi-Fi to the cloud; sometimes BLE for setup.
- **Cloud**: vendor account storing maps, schedules, and (for camera models) images; an MQTT or HTTP control channel.
- **App**: shows the map and controls the robot.

## Step-by-step how-to

### 1. Network recon (aim: local services)

```
nmap -sS -sV -p- VACUUM_IP
# look for: an MQTT client (to the cloud), a local HTTP/API, SSH/telnet (sometimes
# present for debug), and the setup/BLE interface.
```

### 2. Hardware foothold (aim: root and the map data)

Robot vacuums are a classic "root it over UART" target because they are roomy inside and run full Linux:

```
# open the robot; find the UART header on the mainboard (TX idles high, bursts at boot):
screen /dev/ttyUSB0 115200        # watch the boot log; try for a shell / U-Boot
# dump the eMMC/flash for firmware and the stored map/credentials:
#  via a shell (dd), via eMMC test points, or chip-off (see ../Firmware-Analysis-and-Extraction.md)
```
With a shell you can read the stored map, the Wi-Fi creds, the cloud token, and access the camera/mic directly. This is the local privacy jackpot on your own unit.

### 3. Cloud and app (aim: cross-account maps/cameras)

Proxy the app, map the cloud API, and test device-id authorization (see `../../web/IDOR-and-BOLA.md`). The high-impact bug is reading another account's uploaded map or camera stream by changing a device/user id. Camera-equipped models that relay video through the cloud have had BOLA-style exposure.

### 4. Firmware analysis (aim: secrets and the update path)

Extract and analyze firmware (see `../Firmware-Analysis-and-Extraction.md`): hunt cloud endpoints, shared keys, default credentials, and the OTA mechanism. Check whether OTA is signed; an unsigned OTA is persistent remote compromise.

## Beyond the basics

- **LIDAR-as-microphone (LidarPhone)**: research showed the navigation LIDAR can sense sound-induced vibrations of nearby objects, recovering audio. A reminder that sensors leak beyond their intended function; study what raw sensor access enables.
- **Camera access and repositioning**: with root or a cloud bug, access the camera and, because the device moves, drive it to a vantage point. A severe privacy/physical-surveillance capability; demonstrate only on your own unit.
- **Map exfiltration as recon**: the floor plan plus room labels (users name rooms) is reconnaissance for a physical burglary; frame this impact clearly.
- **Custom firmware community**: many vacuums have community projects to root and run local-only firmware (removing the cloud). These document the UART/OTA/root process per model and are a great learning resource for the hardware path.
- **OTA and persistence**: unsigned or weakly-authenticated OTA lets you push persistent firmware; check the update server and signature.

## Defensive framing / what to watch for

- The core issues are stored maps/credentials in cleartext, unsigned OTA, and cloud authorization by device id; the defenses are encryption-at-rest, signed updates, and ownership-based cloud authz.
- Camera/mic/map access is the real severity; frame by privacy and physical safety.
- Root and cloud testing must be on your own unit and account; reading a stranger's map or camera is a serious intrusion.
- To defend: segment the vacuum on a separate VLAN, prefer models allowing local-only/cloudless operation, keep firmware updated, and understand that a reset may not wipe stored maps/credentials from flash.

## Checklist

- Isolate and scan the vacuum; identify MQTT/cloud, local API, and debug services.
- Open it; get a UART shell / U-Boot; read the stored map, Wi-Fi creds, and cloud token.
- Dump and analyze firmware; find endpoints, shared keys, and the OTA path (signed?).
- Proxy the app; test cloud device-id authorization for cross-account map/camera access.
- Assess camera/mic/LIDAR access as a privacy and physical-security impact, on your own unit only.
