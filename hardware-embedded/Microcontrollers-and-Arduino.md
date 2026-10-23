# Microcontrollers and Arduino

Arduino, ESP32/ESP8266, STM32, and similar microcontrollers are the brains of countless DIY and commercial devices. They have little or no operating system, minimal security features, and firmware that is often readable straight off the chip. Understanding their weaknesses tells you how to build devices that resist tampering and how to assess the ones already in the field.

## The mental model

A microcontroller runs a single firmware image directly on bare metal (or a thin RTOS like FreeRTOS). There is usually no user/kernel separation, no memory protection to speak of, and no concept of file permissions. Whatever is in the flash is the entire device, and whatever secrets it needs (Wi-Fi passwords, cloud tokens, crypto keys) are typically stored in that same flash in plaintext. The security model is "nobody will read the chip," which physical access breaks immediately.

## Where it hides / tell-tales

- **Readable flash**: most MCUs expose a programming/debug interface (SWD on STM32/nRF, the ESP serial bootloader, ISP on AVR) that can read the flash back unless a protection fuse is set. The fuse is frequently left unset.
- **Secrets in flash/NVS**: Wi-Fi SSID/password, cloud API tokens, MQTT credentials, and keys stored in plaintext in the firmware or in a key-value store (ESP32 NVS partition). Extract and read them.
- **No secure boot / unsigned firmware**: you can flash your own firmware, or an attacker can push malicious firmware via an unauthenticated OTA. The absence of signature verification is the tell.
- **Debug interfaces left enabled**: SWD/JTAG unlocked, the ESP serial bootloader accessible, and verbose serial logging that prints secrets.
- **Weak OTA**: ESP OTA and Arduino OTA without authentication or signature checks, so anyone on the network can reflash the device.

## Real material

Reading firmware off common MCUs:

```
# ESP32 / ESP8266 (serial bootloader over USB-UART):
esptool.py --port /dev/ttyUSB0 read_flash 0x0 0x400000 full_dump.bin
# dump just the NVS (key-value store with Wi-Fi creds, tokens) then parse it:
#   partition table tells you the NVS offset; nvs_read tools or manual parsing follow.

# STM32 (SWD via ST-Link + OpenOCD), if readout protection (RDP) is not set:
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg \
  -c "init; halt; dump_image fw.bin 0x08000000 0x20000; exit"

# AVR / classic Arduino (ISP via avrdude), if lock bits allow:
avrdude -c usbasp -p m328p -U flash:r:fw.hex:i
```

Pulling secrets from the dump:

```
strings full_dump.bin | grep -iE 'ssid|pass|token|mqtt|http|key|BEGIN'
# ESP32 NVS often holds Wi-Fi creds and provisioning tokens in a readable structure.
```

Checking/abusing weak OTA (ESP example): if the device runs ArduinoOTA or an HTTP OTA endpoint without auth, anyone on the LAN can push firmware:

```
# unauthenticated ArduinoOTA upload:
espota.py -i 192.168.1.60 -f malicious.bin
# or an HTTP OTA endpoint that accepts an unsigned image:
curl -F "firmware=@malicious.bin" http://192.168.1.60/update
```

## Beyond the basics

- **Readout protection is the one setting that matters most**: STM32 RDP level, AVR lock bits, ESP flash encryption + secure boot. Unset, the firmware and all its secrets are trivially extractable. Even when set, some chips' protections have been bypassed via voltage/clock glitching (see `Hardware-Hacking-Fundamentals.md`) or known silicon bugs. Assessing whether protection is enabled is the first question for any MCU product.
- **ESP32 flash encryption and secure boot**: ESP32 supports encrypting the flash and verifying signed firmware, with keys burned into eFuses. Many products do not enable these, so the flash (and its Wi-Fi/cloud secrets) is readable and replaceable. When enabled correctly, it raises the bar substantially; check the eFuse state.
- **Shared keys and derivable secrets**: as with larger IoT, MCUs often use one key across all units, or derive "secret" keys from the chip id/MAC (which is not secret). Extracting one device's key or understanding the derivation compromises the fleet. Look for key-from-MAC patterns in the firmware.
- **Serial and OTA as remote-ish vectors**: an unauthenticated OTA (very common in hobby and cheap commercial ESP devices) is remote code execution over the network, persistent and total. A verbose serial console leaks secrets and may accept commands. These are the easiest wins and the easiest to fix (add auth and signing).
- **Reverse the firmware for logic and bugs**: MCU firmware is a single binary for ARM Cortex-M (Thumb), Xtensa (ESP), or AVR. Load it in Ghidra with the right processor and base address (from the vendor memory map), and analyze protocol handlers for buffer overflows and injection, exactly as in `Assembly-and-Reverse-Engineering.md`. No ASLR/NX means overflows are directly exploitable.
- **Physical glitching to bypass protections**: on protected MCUs, a precisely timed power glitch during the readout-protection check or secure-boot verification can skip it, dumping "protected" flash. This is why serious products add glitch detection and store keys in a separate secure element.
- **Peripheral and bus attacks**: secrets read from an external EEPROM or secure element over SPI/I2C at boot can be sniffed with a logic analyzer; a spoofed sensor/peripheral can feed the MCU malicious data.

## What to watch for (defensive framing)

- Match programming voltage and interface to the exact chip; wrong connections brick or damage the MCU. Work on your own hardware.
- An extractable key or unauthenticated OTA is a fleet-wide issue; prove minimally on your own device and report.
- "Secret" values derived from a MAC/chip id are not secret; treat them as public when assessing.
- To defend: enable readout protection / flash encryption / secure boot and verify it, sign and authenticate OTA updates, use per-device keys stored in a secure element (not derived from the MAC), disable or authenticate serial/debug in production, strip secrets from logs, and add glitch/tamper resistance for high-value devices.

## Hunting checklist

- Identify the MCU and its debug/bootloader interface; check whether readout protection / flash encryption is enabled.
- Dump the flash (and NVS/EEPROM); extract Wi-Fi creds, tokens, and keys with `strings` and format-aware parsing.
- Test OTA for authentication and signature verification; try pushing an unsigned image on your own unit.
- Look for shared or MAC-derived keys in the firmware.
- Load the firmware in Ghidra (correct architecture/base) and audit network/protocol handlers for overflows and injection.
- On authorized research hardware only, assess glitch resistance of readout/secure-boot checks.
