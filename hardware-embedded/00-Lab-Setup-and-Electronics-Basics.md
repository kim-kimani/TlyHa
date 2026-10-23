# Lab Setup and Electronics Basics

Before any attack makes sense, you need to read a board, speak its voltages, and own the handful of tools that turn a sealed plastic box into a target. This file is the genuine starting point: the electronics you must understand, the gear to buy in order, and the safety rules that keep you from frying the device or yourself. Everything else in this track assumes this.

## The mental model

A hardware target is a collection of chips talking to each other over wires (buses) at specific voltages, with a few of those wires brought out to debug points the manufacturer used and forgot to remove. Your whole job starts with: what are the chips, what voltage do they run at, which wires carry what, and where are the debug points. You cannot attack a bus you cannot identify or a signal you might destroy by applying the wrong voltage. So the basics are not optional; they are the difference between a dumped firmware and a dead board.

## Electronics you actually need (the minimum, no more)

- **Voltage, current, ground**: voltage is the pressure that pushes current; ground (GND) is the common reference everything is measured against. Every signal is "this many volts above GND." Find GND first on any board, because every measurement and every connection needs it.
- **Logic levels are the thing that will bite you**: digital chips represent 1 and 0 as a high and low voltage. Common levels are 3.3V (most modern embedded), 1.8V (newer/low-power SoCs), and 5V (older/Arduino). Connecting a 5V tool to a 3.3V chip can destroy the chip. Measure the level before you wire anything, and match it.
- **Digital vs analog, and buses**: the interesting signals are digital buses (UART, SPI, I2C, JTAG) carrying framed data. A bus is just one or more wires toggling between high and low in a pattern a protocol defines. A logic analyzer turns those toggles back into bytes.
- **Pull-ups and idle states**: many lines idle high (held up by a resistor) and pulse low to signal. UART TX idles high; I2C lines idle high. This idle behavior is how you tell lines apart with a multimeter/scope before you know the pinout.
- **Datasheets are the map**: every chip has a part number printed on it and a datasheet online that gives its pinout, voltage, and bus type. Reading the datasheet of the flash chip and the SoC tells you most of what you need before you probe anything. This is the single highest-leverage habit.

## The toolkit, in buying order

Start cheap; add as targets demand. You can do a huge amount with the first tier.

- **Tier 1 (get these first, ~the cost of a night out):**
  - **Multimeter**: find GND, measure voltages, check continuity (which pads connect to what). Your first instrument on every board.
  - **USB-to-UART (TTL) adapter** (FTDI/CP2102/CH340), ideally switchable 3.3V/1.8V/5V: the tool that gets you a serial console. The highest-yield single purchase.
  - **Logic analyzer** (a cheap 8-channel clone works with Saleae's Logic or `sigrok`/PulseView): decodes UART/SPI/I2C so you can read buses and find pinouts.
  - **Jumper wires, SOIC-8 test clip, breadboard, soldering iron, flux**: to physically connect.
- **Tier 2 (for firmware and deeper buses):**
  - **SPI flash programmer** (CH341A is dirt cheap; a FT2232H-based one is better) plus `flashrom`: read/write external flash chips in-circuit or chip-off.
  - **Bus Pirate or FT2232H breakout**: a Swiss-army interface for SPI/I2C/UART/JTAG from your PC.
  - **JTAG/SWD adapter** (ST-Link clone for SWD, FT2232 for JTAG) plus OpenOCD: CPU debug and flash dumping.
  - **Hot air rework station + tweezers**: for chip-off (desoldering flash/eMMC).
- **Tier 3 (specialist):**
  - **JTAGulator**: brute-forces unknown JTAG/UART pinouts automatically.
  - **ChipWhisperer**: fault injection (glitching) and side-channel/power analysis.
  - **Oscilloscope**: for timing-critical work and glitch tuning.
  - **SDR (RTL-SDR to receive, HackRF to transmit)**: for the radio side (see `Radio-and-Wireless.md`).

## Reading a PCB (the first 15 minutes on any device)

A repeatable teardown-to-orientation flow:

```
1. Open the enclosure; photograph both sides of the board at high resolution.
2. Identify the big chips by their printed markings and look up each datasheet:
   - the SoC/MCU (the brain),
   - the flash/EEPROM/eMMC (the storage, often an 8-pin SOIC or BGA),
   - the RAM, the Wi-Fi/radio module, any secure element.
3. Find GND with the multimeter (continuity to a known ground like a shield can or
   a USB/power connector shell).
4. Find the main logic voltage: measure VCC on the flash chip or SoC (datasheet
   pin) relative to GND -> tells you 3.3V vs 1.8V, which sets your tool voltage.
5. Look for debug access: unpopulated headers, rows of 3-4 or 2x5 pads, and
   silkscreen labels (TX, RX, GND, CLK, DIO, TCK, TMS, J1, TP*). Photograph them.
6. Note test points (TP*) near the SoC; many are UART/JTAG/boot-strap pins.
```

## What to aim for at this stage

The goal of "basics" is not an exploit yet; it is orientation that makes the next steps safe and fast. Concretely, by the end of setup you should be able to say: this device uses an `<SoC>` at `<voltage>`, stores firmware on a `<flash part>`, and has candidate debug pads at `<locations>`. With that, you move to `01-Device-Assessment-Methodology.md`, which turns orientation into a plan.

## Safety and not-bricking-it rules

- Measure logic level before connecting any powered tool; mismatched voltage kills chips. Use a level shifter when levels differ.
- Never drive VCC from your UART/programmer into an already-powered board; connect only TX/RX/GND for serial, and power the board from its own supply unless a chip-off read requires external power.
- Find and share a common GND between your tool and the target, or signals are meaningless and you can cause damage.
- Static matters: ground yourself before handling bare boards.
- Chip-off (hot air) can destroy the chip and board if overheated; practice on scrap first.
- Work only on hardware you own or are authorized to test; physical attacks assume possession.

## Checklist

- Acquire Tier 1 tools; learn your multimeter and logic analyzer on a known board (an Arduino or a cheap router).
- For every target: teardown photos, identify chips, pull datasheets, find GND and VCC, locate debug pads.
- Confirm logic voltage before wiring anything.
- Record the orientation summary (SoC, flash, voltage, candidate interfaces) to feed the methodology file.
