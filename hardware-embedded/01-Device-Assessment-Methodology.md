# Device Assessment Methodology

This is the spine of hardware hacking: a phased plan that takes you from a sealed device to extracted firmware, a shell, and real findings, with an explicit goal at every stage so you always know what you are aiming for and when to move on. Most people flail because they skip phases or chase the hardest attack first. The order below is the order of effort-versus-reward.

## The mental model

Hardware assessment is a funnel. Each phase has a cheap, high-probability win and a fallback. You always try the cheapest path to the same prize first, because the prize is usually the same regardless of method: get the firmware (to read secrets and find bugs) and get interactive access (a shell or debug) to the running device. You only escalate to expensive attacks (chip-off, glitching, side channel) when the cheap paths are closed. Knowing the goal of each phase stops you from soldering for three hours when the vendor's website has the firmware.

## The phases, each with its goal

### Phase 0: OSINT and paperwork (goal: free intel before touching the device)

What to aim for: the firmware, the schematics, and the chip identities without opening anything.

```
- Vendor support site: download firmware/update files directly (often the whole prize).
- FCC ID (printed on any radio device, US): fccid.io gives internal photos, block
  diagrams, test reports, and sometimes schematics and the exact chips.
- Datasheets for every chip you can see in listing photos.
- Prior research: CVEs, teardown blogs, and forum threads for this model/SoC.
- Existing firmware dumps and tools from others who did this device.
```
If Phase 0 hands you the firmware, you can start `Firmware-Analysis-and-Extraction.md` in parallel while you work the hardware.

### Phase 1: teardown and orientation (goal: know the board)

What to aim for: the orientation summary from `00-Lab-Setup-and-Electronics-Basics.md`: SoC, flash part, logic voltage, and candidate debug interfaces, with photos.

### Phase 2: the serial console (goal: a shell or boot log, the cheapest foothold)

What to aim for: an interactive root shell, or at minimum a boot log and a U-Boot prompt.

```
1. Identify UART among the candidate pads (logic analyzer or scope: TX idles high,
   bursts data at boot; see Bus-Protocols-Deep.md for pin-finding).
2. Connect USB-UART at the right voltage; try baud 115200 first, then 57600/9600.
3. Watch the boot log (it names the SoC, bootloader, kernel, partitions, and the
   root filesystem type: free recon).
4. Press Enter for a shell. If a login prompt appears, note it; try defaults.
5. Interrupt U-Boot (spam a key during the "hit any key" window) for a bootloader
   prompt, which is often unauthenticated.
```
Goal reached if: you get a shell (jackpot, treat the device as a Linux box now) or a U-Boot prompt (use it in Phase 4). If the console is silent/locked, continue to Phase 3.

### Phase 3: get the firmware off the chip (goal: the code, by any means)

What to aim for: a full, correctly carved firmware image to analyze.

```
Order of effort:
a. Did Phase 0/2 already give it? If yes, skip to analysis.
b. In-circuit SPI flash read: SOIC-8 clip + CH341A + flashrom (-r firmware.bin).
   Works often; sometimes you must hold the SoC in reset or power the chip alone.
c. JTAG/SWD flash dump: OpenOCD dump_image, if the debug port is unlocked.
d. eMMC/NAND read: via test points (in-system programming) or chip-off.
e. Chip-off: desolder the flash, read in a socket/programmer. Last resort, highest
   risk, always works if the flash is not encrypted.
```
Goal reached if: `binwalk` finds a real filesystem in the dump. If the dump is high-entropy (encrypted), note it and plan Phase 6.

### Phase 4: interactive control and privilege (goal: run your code on the device)

What to aim for: persistent, privileged execution and the ability to modify the system.

```
- From a root shell (Phase 2): you already have it; enumerate and pivot.
- From U-Boot: change bootargs to `init=/bin/sh` and boot straight to a root shell,
  or `tftp`/`loadb` a modified kernel/rootfs, or dump/modify flash via U-Boot.
- From JTAG/SWD: halt the CPU, read/write memory, patch out checks, or load code.
```

### Phase 5: software analysis (goal: secrets and remote bugs)

What to aim for: credentials, keys, backdoors, and vulnerable network services, from the extracted firmware. This is `Firmware-Analysis-and-Extraction.md` and `Assembly-and-Reverse-Engineering.md`: grep for secrets, audit the web/CGI and daemons for command injection and overflows, and emulate to attack without the device.

### Phase 6: defeating protections (goal: break what the cheap paths could not)

What to aim for: firmware/keys that are locked behind readout protection, secure boot, or encryption. This is where the expensive, advanced work lives, and only here, after the cheap paths failed:

```
- Readout-protection / secure-boot bypass via fault injection (glitching): see
  Fault-Injection-and-Glitching.md. Aim: dump "protected" flash or skip a signature
  check.
- Key recovery via side-channel (power/EM) analysis: see Side-Channel-Analysis.md.
  Aim: extract a crypto key the device never exposes.
- Encrypted-firmware key recovery: find the key in the bootloader/SoC/an older
  unencrypted firmware version; see Secure-Boot-and-Chain-of-Trust.md.
- Bus sniffing: capture secrets read from an EEPROM/secure element at boot.
```

## The decision tree, in one view

```
Firmware from vendor/FCC?  --yes--> analyze (Phase 5), hardware in parallel
        |no
Serial console -> shell?   --yes--> you own it; dump flash via the shell
        |no
Serial console -> U-Boot?  --yes--> boot to root / load modified image
        |no
In-circuit SPI/JTAG dump?  --yes--> analyze; use JTAG for live control
        |no
Chip-off read?             --yes--> analyze (if not encrypted)
        |no / encrypted
Glitch readout protection / recover key (Phase 6)
```

## What to aim for overall

Two prizes define success and you should always know which you are chasing:
1. **The firmware** (static prize): unlocks secrets, keys, and the whole remote attack surface, and lets you emulate and fuzz without the device.
2. **Interactive access** (dynamic prize): a shell, U-Boot, or JTAG, which lets you observe and modify the running system, confirm exploits, and extract runtime secrets.

Most engagements are won in Phases 2 and 3. Phase 6 is real hardware hacking but it is a minority of real-world needs; reach for it only when the funnel forces you to, and recognize that its difficulty is why secure boot and readout protection are the controls you are testing.

## Discipline

- Timebox each phase; if the cheap win is not landing, move down the funnel rather than forcing one method.
- Photograph and label every connection; it is easy to lose track of which pad is which after an hour.
- Keep the firmware analysis running in parallel with hardware work; they inform each other (the boot log names partitions; the firmware names the debug features).
- Only test devices you own or are authorized to; Phase 6 especially assumes research hardware you can destroy.

## Checklist

- Phase 0: pull firmware/schematics/FCC data and prior research before opening.
- Phase 1: teardown, identify chips, voltage, debug pads.
- Phase 2: get a serial console; aim for shell or U-Boot.
- Phase 3: extract firmware by the cheapest working method up to chip-off.
- Phase 4: achieve privileged interactive control.
- Phase 5: analyze firmware for secrets and remote bugs.
- Phase 6: only if needed, defeat readout/secure-boot/encryption via glitching, side channel, or key recovery.
