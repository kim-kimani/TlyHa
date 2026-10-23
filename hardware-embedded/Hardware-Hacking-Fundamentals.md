# Hardware Hacking Fundamentals

Physical access changes the rules. A device you can hold exposes debug ports, flash chips, and buses that the designers assumed nobody would probe. The goal of hardware analysis is almost always the same: get the firmware and get a shell, because once you have the code and a console, the device becomes a normal software target. This file is written so you can recognize these weaknesses and design them out.

## The mental model

A typical embedded device is a circuit board with a main processor (SoC/MCU), a flash chip holding firmware, RAM, and a set of debug and communication interfaces the manufacturer used during development and often left enabled. Your job is to identify those interfaces, speak their protocol, and use them to read firmware, get a console, or halt and inspect the CPU. Every one of these is a trust assumption ("the enclosure is closed, so these pads are safe") that physical access breaks.

## Where it hides / tell-tales

- **UART (serial console)**: a row of 3 to 4 pads/pins (often labeled or just a header footprint) carrying `TX`, `RX`, `GND`, and sometimes `VCC`. It frequently drops you to a boot log and, far too often, a root shell or U-Boot prompt with no password. The single highest-yield hardware foothold.
- **JTAG / SWD (debug)**: full CPU debug access: halt, read/write registers and memory, single-step, read out flash. JTAG is a multi-pin standard (`TCK`, `TMS`, `TDI`, `TDO`, `TRST`); SWD is the 2-wire ARM variant (`SWDIO`, `SWCLK`). If present and unlocked, it is total control.
- **SPI / I2C flash chips**: the firmware lives in an external flash chip (often an 8-pin SOIC, e.g. a 25-series SPI NOR flash, or an I2C EEPROM). You can read it in-circuit or desolder it and read it with a programmer. No auth, it is just a memory chip.
- **eMMC / NAND**: larger storage, read via test points or a chip-off approach.
- **Test points and header footprints**: unpopulated headers, labeled pads (`TP*`, `J*`), and vias are the manufacturer's own debug access. Silkscreen labels (`RX`, `TX`, `CLK`, `DIO`) are gifts.

## Real material

Identifying and using UART:

```
# find candidate pads with a multimeter/logic analyzer: GND is continuous to ground,
# TX idles high (~3.3V) and pulses when the device boots.
# connect a USB-TTL adapter (match voltage! usually 3.3V): adapter RX -> device TX,
# adapter TX -> device RX, GND -> GND.  Do NOT connect VCC.
# common baud rates: 115200, 57600, 38400, 9600
screen /dev/ttyUSB0 115200
# or
picocom -b 115200 /dev/ttyUSB0
# watch the boot log; try Enter for a shell; interrupt U-Boot with a keypress to get its prompt.
```

Dumping an SPI flash chip (in-circuit with a clip, or desoldered):

```
# flashrom with a CH341A or FT2232-based programmer and a SOIC-8 test clip:
flashrom -p ch341a_spi -r firmware.bin
# then analyze the dump (see Firmware-Analysis-and-Extraction.md):
binwalk -e firmware.bin
```

JTAG discovery and halt (when pinout is unknown):

```
# JTAGulator or a logic analyzer to find TCK/TMS/TDI/TDO among candidate pins.
# then with OpenOCD + an adapter (FT2232, ST-Link for SWD):
openocd -f interface/ftdi/ft2232.cfg -f target/<soc>.cfg
# in a telnet session to OpenOCD:
halt
dump_image flash_dump.bin 0x08000000 0x40000   # read internal flash to a file
```

## Beyond the basics

- **UART first, always**: it is the cheapest and most productive. A root shell on the console, a U-Boot prompt (where you can often change the boot args to `init=/bin/sh` and boot straight to root, or `tftp` a modified image), or at minimum a boot log naming the SoC, bootloader, and partition layout. Many consumer devices ship with an open serial root shell.
- **U-Boot is a backdoor when accessible**: interrupting the bootloader lets you inspect/modify environment variables, change `bootargs` to spawn a root shell, load firmware over TFTP/serial, read/write memory and flash, and bypass whatever the OS would have enforced. Secure boot and a locked/authenticated bootloader are the defenses; their absence is the finding.
- **Chip-off and in-circuit flash reads defeat software protections**: reading the flash directly bypasses any OS-level access control. Extract the firmware, then pull out secrets, keys, hardcoded credentials, and the filesystem (see `Firmware-Analysis-and-Extraction.md`). For encrypted flash, the decryption key often lives in the SoC or in an accessible region; look for it.
- **JTAG/SWD read-out protection**: MCUs have a "readout protection" fuse (STM32 RDP, etc.) meant to block flash dumping. Check whether it is set; it frequently is not, and even when set it has historically been bypassable on some chips via glitching (see below). An unlocked debug port is complete compromise.
- **Fault injection (glitching)**: briefly disturbing the power (voltage glitch) or clock at a precise moment can make the CPU skip an instruction, such as the one checking a password, verifying a signature, or enforcing readout protection. Tools like the ChipWhisperer automate this. It is how locked bootloaders and secure-boot checks get bypassed on hardware that is "protected."
- **Side-channel analysis**: power consumption and electromagnetic emissions leak information about the operations a chip performs; differential power analysis can recover cryptographic keys from a device that never exposes them over any interface. Advanced, but it is why secure elements and constant-time, masked implementations exist.
- **Bus sniffing**: a logic analyzer on SPI/I2C/UART traffic between chips reveals secrets in transit: keys read from an EEPROM at boot, commands to a secure element, and inter-chip protocols you can later spoof.

## What to watch for (defensive framing)

- Match logic levels (3.3V vs 1.8V vs 5V) before connecting anything, and never drive `VCC` from your adapter into a powered board; you will damage hardware. This is a safety note, not just etiquette.
- A found serial shell or JTAG is only meaningful on hardware you own or are authorized to test; physical attacks assume possession of the device.
- Not every header is UART/JTAG; confirm with a logic analyzer before assuming, and label pins by observed behavior.
- To defend: disable or lock debug interfaces in production (blow JTAG/RDP fuses), require authentication on the serial console and U-Boot, enable secure boot with signature verification, encrypt flash and store keys in a secure element, and add glitch/tamper detection. The attacks above are the checklist of what to close.

## Hunting checklist

- Open the device; photograph the board; identify the SoC, flash chip, and RAM by their markings.
- Find and test UART pads; get a console or boot log; try for a shell and a U-Boot prompt.
- Locate JTAG/SWD; test whether debug is unlocked (halt, read memory).
- Read the external flash in-circuit or via chip-off; extract and analyze firmware.
- Check readout-protection/secure-boot status; consider glitching only on authorized research hardware.
- Sniff inter-chip buses for secrets in transit.
