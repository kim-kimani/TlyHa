# Bus Protocols Deep (UART, SPI, I2C, JTAG, SWD)

Buses are how chips talk, and every one of them is a place to listen, inject, or extract. This file goes past "UART gives a shell" into how to actually find the pins, wire them correctly, decode the traffic, and interact, for each of the buses you will meet. Mastering these four or five protocols is most of practical hardware hacking.

## The mental model

Each bus is a small contract: a number of wires, a clocking scheme, and a framing format. If you know the contract, a logic analyzer turns wiggling voltages back into bytes, and a USB interface lets you become a participant (talk to the flash, halt the CPU, open a console). The recurring workflow is: identify the bus and its pins, confirm voltage, capture and decode, then interact. The "find the pins" step is where beginners stall, so each protocol below leads with how to locate it.

## UART (asynchronous serial): the console bus

Goal: a shell or boot log. UART is two data wires plus ground, no clock (both ends agree on a baud rate).

- **Pins**: `TX` (device transmits), `RX` (device receives), `GND`, sometimes `VCC`. Usually a row of 3 to 4 pads/holes.
- **Finding them**: `TX` idles high (at the logic voltage, e.g. 3.3V) and bursts activity right after power-on. `GND` reads continuity to ground. `RX` often idles high but shows no traffic. `VCC` is a steady logic voltage with continuity to the chip's power. Use a multimeter to spot GND and steady VCC, then a logic analyzer/scope on the remaining pads during boot to see which one bursts (that is `TX`).
- **Wiring**: adapter `RX` to device `TX`, adapter `TX` to device `RX`, `GND` to `GND`. Do not connect `VCC`. Match voltage.
- **Baud**: try 115200 first, then 57600, 38400, 19200, 9600. If you see garbage that changes density when you change baud, you are close; `sigrok`/PulseView can auto-measure the bit width from a capture to compute the baud.
- **Interact**: `screen /dev/ttyUSB0 115200` or `picocom`. Enter for a shell; spam a key at boot to catch U-Boot.

```
# measure baud from a logic-analyzer capture (narrowest pulse = one bit):
# baud ~= 1 / (shortest-pulse-width-in-seconds)
# common result: 8.68us -> 115200
```

## SPI (synchronous, 4-wire): the flash bus

Goal: read/write the external flash where firmware lives. SPI is fast, clocked, full-duplex.

- **Pins**: `CLK` (clock), `MOSI` (master out), `MISO` (master in), `CS` (chip select), plus `VCC`/`GND`. On a standard 8-pin SOIC NOR flash (25-series), the pinout is fixed by the datasheet: `CS`, `DO(MISO)`, `WP`, `GND`, `DI(MOSI)`, `CLK`, `HOLD`, `VCC`.
- **Finding them**: identify the flash chip by its markings, pull the datasheet, and you have the exact pinout. This is why chip identification matters.
- **Reading in-circuit**: clip a SOIC-8 test clip on the chip, connect to a CH341A or FT2232 programmer, run `flashrom -p ch341a_spi -r dump.bin`. Sometimes the SoC fights you on the bus; hold the SoC in reset (tie its reset pin), or power only the flash. If in-circuit fails, chip-off (desolder) and read in a socket.
- **Sniffing**: a logic analyzer on `CLK/MOSI/MISO/CS` with the SPI decoder shows exactly what the SoC reads at boot, including keys read from the flash or a secure element.

```
flashrom -p ch341a_spi -r dump.bin          # read
flashrom -p ch341a_spi -w modified.bin       # write (to patch firmware)
binwalk dump.bin                             # then analyze
```

## I2C (synchronous, 2-wire): the EEPROM/sensor/config bus

Goal: read config, secrets, and keys from small EEPROMs; sniff or spoof sensor/peripheral traffic.

- **Pins**: `SDA` (data), `SCL` (clock), both idle high (pulled up), shared by multiple devices each with an address.
- **Finding them**: two lines idling high with periodic bursts; the pull-up resistors near a small 8-pin EEPROM (24-series) are a tell. Datasheet of the EEPROM confirms.
- **Interact**: a Bus Pirate, FT2232, or a Raspberry Pi can scan the bus for device addresses and dump an EEPROM.

```
# on a Linux host with an I2C adapter:
i2cdetect -y 1                 # list device addresses on the bus
i2cdump -y 1 0x50              # dump the EEPROM at address 0x50 (often config/secrets)
```
EEPROMs frequently hold provisioning data, serial numbers, calibration, and sometimes credentials or keys in plaintext.

## JTAG and SWD: the CPU debug buses

Goal: total control: halt the CPU, read/write all memory and registers, single-step, and dump internal flash. JTAG is the full standard; SWD is ARM's 2-wire version.

- **JTAG pins**: `TCK` (clock), `TMS` (mode select), `TDI` (data in), `TDO` (data out), optional `TRST` (reset), plus `GND`/`VCC`. Often a 2x5 or 2x7 header, but frequently just test pads.
- **SWD pins**: `SWDIO` (data), `SWCLK` (clock), plus `GND` and usually `RESET`. Only two signal wires, common on ARM Cortex-M.
- **Finding them (the hard part)**: unknown pinouts are brute-forced with a **JTAGulator**, which drives candidate pins and detects valid JTAG/SWD responses, or a logic analyzer during a debugger connection. `TCK` is a steady clock when active; `TMS/TDI/TDO` move in step with it.
- **Interact**: OpenOCD with an adapter (ST-Link for SWD, FT2232 for JTAG) and the right target config:

```
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg
# then via telnet localhost 4444:
halt
dump_image flash.bin 0x08000000 0x20000     # read internal flash
mdw 0x20000000 16                            # read RAM
reg                                          # dump registers
```
- **If locked**: a readout-protection fuse (STM32 RDP, etc.) blocks flash dumping over JTAG/SWD. Check its state; if set, your options are glitching the protection (see `Fault-Injection-and-Glitching.md`) or another extraction path.

## Beyond the basics

- **Pin-finding is the real skill**: most time is spent identifying which pad is what. Invest in a logic analyzer workflow (capture during boot and during a debugger connection) and a JTAGulator for unknown debug ports. Datasheet-driven identification (find the chip, read its pinout) beats guessing every time.
- **Sniff to learn secrets in transit**: the highest-value passive attack is capturing a bus at boot. SoCs read keys, credentials, and config from external flash, EEPROM, or a secure element over SPI/I2C in the clear. A logic-analyzer capture with the right decoder hands you those secrets without ever getting code execution.
- **Become a participant, not just a listener**: once you can speak the bus, you can feed the SoC forged data (spoof a sensor on I2C, present a modified image on SPI), read/modify flash to patch out checks, or use JTAG to patch running memory and skip authentication. This bridges into fault-free logic bypass.
- **Boot-mode straps**: many SoCs sample certain pins at reset to decide the boot source (internal flash vs UART vs USB download mode). Pulling a strap pin can force the chip into a serial/USB bootloader that accepts firmware or memory reads, bypassing the normal boot. The SoC datasheet lists the strap pins; this is a frequent extraction path.
- **eMMC/NAND over test points**: larger storage uses eMMC (an SD-card-like protocol). You can often read it in-system by wiring an SD reader to eMMC test points (`CLK/CMD/DAT0/VCC/GND`) and putting the SoC in reset, avoiding chip-off.

## What to watch for (defensive framing)

- Confirm voltage on every bus before connecting; SPI/JTAG tools can drive lines and damage a mismatched chip.
- In-circuit reads can be corrupted by the SoC also driving the bus; verify the dump (sane filesystem, repeatable read) before trusting it.
- A locked JTAG/RDP is a real control; do not assume debug access until a `halt`/read actually succeeds.
- To defend: disable or fuse-lock JTAG/SWD and boot-download modes in production, encrypt data stored in external flash/EEPROM and keys in a secure element, authenticate the serial console and U-Boot, and detect/limit bus-level tampering for high-value devices.

## Checklist

- Identify each bus by chip datasheets plus logic-analyzer behavior (idle states, clocks).
- UART: find TX/RX/GND, match voltage, sweep baud, get console/U-Boot.
- SPI: clip the flash, read with flashrom (in-circuit, then chip-off); sniff boot reads.
- I2C: scan addresses, dump EEPROMs for config/secrets.
- JTAG/SWD: find pins (JTAGulator/logic analyzer), connect OpenOCD, check readout-protection, dump flash/memory.
- Check boot-strap pins for a forced serial/USB download mode.
