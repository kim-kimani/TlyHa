# Secure Boot and Chain of Trust

Secure boot is the control that is supposed to make all the firmware attacks impossible: if only signed code runs, you cannot flash a shell or patch out a check. In practice, the chain of trust has links, and attackers break the weakest one. Understanding how it is built shows you exactly where it fails and how to design it so it does not.

## The mental model

Secure boot is a chain: an immutable root (burned into the SoC, the "root of trust") verifies the next stage's signature before running it, that stage verifies the next, and so on up to the application. The security rests on three assumptions: the root really is immutable and trusted, every stage actually verifies the next before executing it, and the verification keys and logic cannot be bypassed or downgraded. Break any one assumption at any link and everything above it can be replaced with your code. The attacker's job is to find the link where an assumption does not hold.

## What to aim for

- **Run unsigned code**: get the device to execute firmware you modified (a shell, a patched check, a backdoor), persistently.
- **Dump protected firmware/keys**: many chains also gate readout; breaking boot often breaks that too.
- **Recover or abuse the signing key**: the ultimate prize, because it compromises every device of the model (you can sign malicious firmware that passes verification everywhere).

## Where the chain breaks (link by link)

- **The root is not actually locked**: secure boot is supported but not enabled (eFuses not blown), or the key-hash fuse is writable, or debug/JTAG is left open so you bypass boot entirely. Enormous numbers of devices ship with secure boot "available" but off. Checking the fuse state is the first question.
- **A stage does not verify before executing**: a bootloader that loads the next stage and only checks the signature afterward (TOCTOU), or verifies only part of the image (header but not the whole payload), or has a path (recovery mode, USB download, a debug command) that loads code without verification. Recovery/download modes are a classic hole.
- **Verification can be skipped physically**: glitch the signature-check instruction so an invalid signature is accepted (see `Fault-Injection-and-Glitching.md`). The single most common practical defeat of otherwise-correct secure boot.
- **Rollback/downgrade**: the chain verifies signatures but does not enforce a minimum version, so you flash an older, legitimately-signed firmware that contained a known vulnerability, then exploit that. Anti-rollback counters (monotonic version fuses) are often missing.
- **Key and crypto weaknesses**: a signing key leaked in firmware or a development build, a test/debug key left trusted in production, a weak algorithm, or a verification bug (accepting `e=1` RSA, not checking padding, confusing signature with hash). A trusted debug key is a backdoor by design.
- **Mutable storage of the "immutable" root**: the root public key or its hash stored in writable flash rather than fuses, so you replace it with your own key and sign your own firmware.

## Real material

Checking whether secure boot and readout are actually enforced (concept; exact commands are SoC-specific):

```
# ESP32: read the eFuses that enable secure boot and flash encryption
espefuse.py --port /dev/ttyUSB0 summary
#   look for: SECURE_BOOT_EN, ABS_DONE_x, FLASH_CRYPT_CNT, JTAG disable fuses.
#   not set -> you can flash/read freely.

# Many SoCs: if JTAG/SWD still halts the CPU, secure boot is moot for extraction.
openocd ... ; halt ; dump_image ...     # if this works, boot verification is bypassed
```

Downgrade attack shape:

```
1. Current firmware v4.2 fixes a bug; v3.1 is vulnerable and still vendor-signed.
2. The bootloader checks the signature (v3.1 passes) but not the version.
3. Flash v3.1, then exploit the known bug -> code execution despite "secure boot".
```

Recovery-mode bypass shape:

```
- Hold a button / strap a pin at power-on to enter recovery/download mode.
- That mode accepts a firmware image over USB/UART and, on many devices, skips or
  weakens signature verification -> load unsigned code.
```

## Beyond the basics

- **Enumerate the whole chain first**: identify every stage (boot ROM, first-stage bootloader, U-Boot/second-stage, kernel, rootfs, app) and ask, for each link, "does this verify the next, fully, before running it, with a key I cannot change?" The break is wherever the honest answer is no. The boot log and the SoC technical reference manual map the chain.
- **Glitching is the universal fallback**: when the chain is cryptographically sound, fault injection on the verification step bypasses it without touching the crypto. This is why serious designs add redundant/randomized verification and glitch detection; its absence is the finding. Pair this file with the glitching file as a combined attack.
- **Downgrade is underrated and easy**: anti-rollback is frequently omitted. Collect every previously-signed firmware version (vendor archives, CDN, other units) and look for one with a known, exploitable bug. No crypto break required.
- **Trusted debug/test keys and fused secrets**: production units that still trust a development key, or that store the signing key where firmware can read it (for OTA re-signing), hand you fleet-wide compromise. Hunt the firmware for keys (see `Firmware-Analysis-and-Extraction.md`); a private signing key in an image is catastrophic.
- **Encrypted-firmware chicken-and-egg**: flash encryption protects confidentiality, secure boot protects integrity; they are separate. A device may verify signatures but not encrypt (so you read it), or encrypt but with a key recoverable from the SoC. Breaking one often assists the other (read the plaintext bootloader to find how verification works, then glitch it).
- **TOCTOU and partial verification in the real world**: verifying a header/manifest but not every byte, verifying into a buffer then loading from a different location, or checking a hash you also supply. These logic bugs defeat secure boot without any physical attack; reverse the bootloader to find them.
- **Chain into persistence**: once you can run unsigned code at an early stage, you own everything above it and can install persistence that survives updates (infect the bootloader itself). This is the difference between a one-time shell and a durable implant.

## What to watch for (defensive framing)

- "Secure boot supported" is not "secure boot enabled"; verify the fuse/config state before assuming anything is enforced.
- A single bypass on one unit may be a fluke (especially glitching); confirm repeatability and that unsigned code truly runs.
- Downgrade and recovery-mode bypasses depend on the specific device's version/anti-rollback and recovery design; verify rather than assume.
- To defend: enable secure boot and lock the fuses in production, verify the entire image before execution (no TOCTOU/partial checks), enforce anti-rollback with monotonic version counters, remove all test/debug keys and never store the private signing key on the device, keep the root of trust in immutable fuses, add glitch-resistant redundant verification, and disable or authenticate recovery/download modes and debug ports. Every link above is a control to get right.

## Checklist

- Map every boot stage and whether each fully verifies the next before executing.
- Check whether secure boot / flash encryption / JTAG-lock fuses are actually set.
- Test recovery/download modes for unverified code loading.
- Try downgrade to an older signed-but-vulnerable firmware (anti-rollback present?).
- Hunt firmware for trusted test keys and any stored signing key.
- Reverse the bootloader for TOCTOU/partial-verification logic bugs.
- If the chain is sound, fall back to glitching the verification step.
