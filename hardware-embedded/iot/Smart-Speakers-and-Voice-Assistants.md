# Smart Speakers and Voice Assistants

A smart speaker (Echo, Google Nest, HomePod, and the countless third-party Alexa/Assistant devices) is an always-listening microphone array with a cloud brain and control over the rest of the smart home. The aim is the microphone and the voice-command surface: covert listening, injecting commands the user never spoke, and pivoting into everything the assistant controls.

## What it is and why it is a target (the aim)

- **Covert listening**: the device has far-field mics and streams audio to the cloud on wake. Aim: access the mic or the audio pipeline, and understand accidental-activation leakage.
- **Command injection**: get the assistant to execute commands the owner did not speak (unlock a door, place an order, disable alarms, read calendars). Aim: inject voice commands via audio, radio, or the device itself.
- **Home control pivot**: the assistant controls lights, locks, thermostats, and more. Aim: use it as the hub to reach other devices.
- **Account and skills**: the linked cloud account and third-party "skills"/"actions." Aim: account data and malicious-skill abuse.

## Architecture (the four sides)

- **Device**: an ARM Linux SoC with a mic array, speaker, Wi-Fi/BLE, and usually a debug UART and eMMC.
- **Radio**: Wi-Fi to the cloud; BLE/Zigbee (some hubs, e.g. Echo with a built-in Zigbee hub) for device control.
- **Cloud**: the voice backend (speech-to-text, intent, skills) and the account.
- **App**: setup, skills/actions, and smart-home linking.

## Step-by-step how-to

### 1. Voice/acoustic command injection (aim: unspoken commands)

```
# the assistant acts on audio; attacks deliver commands the human does not register:
#  - inaudible/ultrasonic commands (DolphinAttack): modulate voice commands onto
#    ultrasound the mic demodulates but people cannot hear.
#  - hidden/obfuscated commands embedded in music/other audio, or played by a nearby
#    TV/radio/ad (a broadcast can trigger many devices at once).
#  - laser-injected commands (Light Commands): a modulated laser aimed at the mic
#    diaphragm from a distance injects "voice" with no sound at all.
```
Aim: demonstrate (on your own device) that commands can be delivered without the owner speaking, and assess what sensitive actions the assistant will perform without confirmation (unlock, purchase, disarm).

### 2. Network and hardware (aim: mic pipeline, firmware, pivot)

```
nmap -sS -sV -p- SPEAKER_IP        # local services
# open the device; UART for a boot log/shell, dump eMMC/flash for firmware:
#  see ../Bus-Protocols-Deep.md and ../Firmware-Analysis-and-Extraction.md
```
Aim: with a root/firmware foothold on your own unit, understand the mic/audio pipeline, stored tokens, and the device-control bridge.

### 3. Cloud, account, and skills (aim: data and malicious automation)

Proxy the app and test the voice/account cloud API (see `../../web/REST-API-Security.md`). Examine third-party skills/actions: a malicious or squatted skill name ("skill squatting," where a skill with a similar-sounding invocation intercepts requests) can phish or capture input. Aim: account data exposure and skill-based abuse.

### 4. Smart-home pivot (aim: control linked devices)

If the speaker is a hub (e.g. built-in Zigbee), or controls linked devices, a voice-injection or device compromise reaches locks, lights, and alarms. Aim: chain the assistant into physical-device control (see the lock/alarm guides).

## Beyond the basics

- **Light Commands and DolphinAttack are the signature research**: MEMS microphones respond to laser light and to ultrasonic carriers, so "voice" commands can be injected silently and, with a laser, from across a street through a window. Study these to understand that the mic itself is an attack surface, not just the software.
- **Accidental activation and audio retention**: devices misfire on wake-word-like sounds and stream/retain audio; this is a real, documented privacy leak (snippets sent to the cloud and to human reviewers). The retained-audio and review pipeline is part of the surface.
- **Confirmation gaps**: the severity of command injection depends on what the assistant does without confirmation. Unlocking doors, placing orders, and disarming alarms by voice without a second factor are the dangerous defaults; map them.
- **Skill/action squatting and phishing**: third-party voice apps with confusable invocation names can intercept or phish; the voice "app store" is a trust surface like any other.
- **BLE/Wi-Fi setup**: provisioning over BLE/soft-AP may leak Wi-Fi creds or allow hijacking an unclaimed device.
- **Hub pivot**: an Echo/Nest controlling the home is a single point whose compromise cascades to every linked device.

## Defensive framing / what to watch for

- Headline issues are silent command injection (acoustic/laser), sensitive actions without confirmation, and audio-privacy leakage; defenses are voice-match/confirmation for sensitive actions, mic hardening, and minimizing retained audio.
- Test acoustic/laser injection and mic access only on your own device; frame impact by covert surveillance and unauthorized physical actions.
- Command injection matters most where it triggers locks/purchases/alarms without confirmation; demonstrate that chain.
- To defend: require voice recognition and/or a spoken PIN or app confirmation for sensitive actions (unlock, purchase, disarm), use the physical mic-mute, limit third-party skills, keep firmware updated, and segment the device; understand retained-audio settings.

## Checklist

- Map what sensitive actions the assistant performs by voice without confirmation.
- Test silent command injection (ultrasonic, hidden-audio, laser) on your own device.
- Scan the device; UART/firmware for the audio pipeline, tokens, and control bridge.
- Proxy the app/cloud; test account data and malicious/squatted skills.
- Assess the smart-home pivot to locks/lights/alarms.
- Frame by covert listening and unauthorized actions; stay on your own device.
