# Fault Injection and Glitching

Glitching is making a chip briefly misbehave on purpose so it skips an instruction that was protecting something: a signature check, a readout-protection fuse, a password comparison, a secure-boot verification. It feels like magic the first time a "locked" chip dumps its firmware, but it is a disciplined, repeatable process. This is advanced, so it comes after the cheap extraction paths fail (see `01-Device-Assessment-Methodology.md`, Phase 6).

## The mental model

A CPU executes instructions assuming stable power and clock. If you disturb that stability at exactly the right microsecond, the chip can corrupt a single operation: a comparison returns the wrong result, a loop counter skips, a branch is not taken, a memory read returns the wrong value. You are not breaking the crypto; you are making the chip skip the `if` that enforces it. The entire craft is timing: find the moment the protective instruction runs, and inject a precise fault there.

## What to aim for

Concrete, high-value targets where skipping one check wins everything:

- **Readout-protection bypass**: glitch the instruction that checks the RDP/lock fuse during early boot, so the chip proceeds as if unprotected and lets you read internal flash over JTAG/UART. Prize: the firmware you could not otherwise extract.
- **Secure-boot bypass**: glitch the signature-verification result so a modified/unsigned firmware is accepted as valid. Prize: run your own code on a locked device.
- **Password/PIN check bypass**: glitch the comparison in a bootloader or secure element so any input is accepted. Prize: access without the secret.
- **Bootloader command unlock**: glitch a state check so a disabled debug/dump command becomes available.

## The techniques

- **Voltage glitching**: briefly drop (or spike) the chip's supply voltage for nanoseconds-to-microseconds. The most common and accessible method. A MOSFET crowbar pulls VCC low at the target moment.
- **Clock glitching**: insert an extra or malformed clock edge so an instruction is fetched/executed wrong. Works on chips with an external clock you control; useless on internal-oscillator parts.
- **Electromagnetic (EM) glitching**: a fast EM pulse from a small coil over the die induces faults without electrical contact, and can be spatially targeted to a region of the chip. More flexible, needs more gear.
- **Optical/body-biasing**: decapsulate the chip and use a laser; lab-grade, for the hardest targets.

## The workflow (voltage glitch with a ChipWhisperer, the standard rig)

```
1. Identify the target moment. Watch the power trace (the ChipWhisperer scope, or an
   oscilloscope on a shunt resistor in the VCC line) while the device boots or checks
   the secret. The protective operation has a characteristic power signature; the
   window is often right after reset or right after you submit a password.
2. Build a reliable trigger. Glitches must land at a repeatable offset from a known
   event: the reset line releasing, a GPIO toggling, a UART byte you send, or a
   pattern in the power trace. The trigger starts the countdown.
3. Scan the parameter space. A glitch has a few knobs:
     - offset/delay: how long after the trigger to fire,
     - width: how long the glitch lasts,
     - (voltage) depth or (clock) which edge.
   Sweep offset x width across the plausible window, many attempts per point.
4. Classify each attempt: normal, crash/reset (too strong), or GLITCH SUCCESS
   (the protected behavior changed: flash now readable, bad signature accepted,
   wrong PIN passed). Log everything.
5. Narrow to the success region and increase reliability; some targets succeed 1 in
   1000, which is fine when you can retry automatically.
```

```python
# conceptual ChipWhisperer loop (voltage glitch sweep)
for offset in range(off_lo, off_hi):
    for width in range(w_lo, w_hi):
        scope.glitch.ext_offset = offset
        scope.glitch.width = width
        reset_target()
        arm_and_fire()                 # trigger + glitch
        r = read_target_response()     # did flash dump? did it accept the PIN?
        classify(offset, width, r)     # normal / reset / SUCCESS
```

## Beyond the basics

- **The power trace tells you where to aim**: do not glitch blindly across all of boot. Capture the power/EM trace and locate the protective routine (crypto has a distinctive signature, flash access another). Triggering off a feature in the trace itself (not just a GPIO) is how the hard targets get done.
- **Multi-glitch and double-fault**: some protections are checked twice (a verify-then-verify-again). You may need two precisely spaced glitches, or to combine a glitch with a logic condition. Modern secure elements add redundant checks specifically to raise the number of faults needed.
- **Characterize on a twin first**: buy two of the device. Practice and map the parameter space on a sacrificial one (which you can brick), then apply the found parameters to the real target. This is standard and saves the target.
- **Crowbar vs precise injection**: a MOSFET crowbar (short VCC to GND briefly) is crude but effective and cheap; a programmable glitch generator gives finer control for stubborn targets. Start crude.
- **Combine with other extraction**: a glitch that unlocks readout protection then feeds a normal JTAG/UART dump; a glitch that accepts unsigned firmware then lets you flash a shell. Glitching is the key that opens the door; the extraction behind it is the ordinary Phase 3 work.
- **Reliability engineering**: automate reset, trigger, glitch, and classification so you can run thousands of attempts unattended overnight. Success is often probabilistic; throughput matters more than a single perfect shot.
- **EM for no-contact and spatial targeting**: when you cannot cleanly interrupt VCC (lots of decoupling capacitance) or want to target a specific die region, EM injection is the escalation. It needs an XY stage for mapping on serious work.

## What to watch for (defensive framing)

- Glitching is destructive-risk: expect to brick sacrificial units; use twins and never your only copy of a target.
- A "success" can be a fluke; require repeatability before trusting that you bypassed a control.
- This is research-grade work on hardware you own; it assumes full physical possession and the right to destroy the device.
- To defend (this is the point for a defender): add redundant and randomized checks (verify a result multiple times, in random order), detect abnormal voltage/clock/temperature and reset, insert random delays so the trigger window is unstable, use glitch-hardened secure elements for key storage and secure boot, store keys where a single skipped instruction does not expose them, and do not gate security on one comparison. Every mitigation here maps directly to an attacker knob above.

## Checklist

- Only after cheap extraction fails: pick the target instruction (RDP check, signature verify, PIN compare).
- Set up a reliable trigger tied to reset/GPIO/UART/power-trace feature.
- Sweep offset x width (voltage) or clock-edge parameters; classify normal/reset/success.
- Practice on a twin; log the success region; automate for throughput.
- Chain the glitch into a normal flash/memory dump or an unsigned-firmware flash.
- Escalate to EM or multi-glitch for redundant/decoupled protections.
