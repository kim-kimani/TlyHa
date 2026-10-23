# Firmware Analysis and Extraction

Firmware is the device's whole software stack in one blob. Once you have it, a hardware target becomes a software target: you read the filesystem, pull hardcoded secrets, find the vulnerable services, and emulate the thing to attack it without the physical device. Getting the firmware and taking it apart is the center of gravity for embedded security.

## The mental model

Firmware is usually a packed image containing a bootloader, a kernel, and a root filesystem (and sometimes a separate app partition), often compressed and occasionally encrypted. Analysis is a pipeline: obtain the image, identify and carve its parts, extract the filesystem, then audit it like any Linux/RTOS system for secrets, services, and vulnerable binaries. The best bugs are credentials and backdoors sitting in plaintext inside the root filesystem.

## Where it hides / tell-tales

- **Obtaining the image**: vendor firmware downloads and update servers (the easiest source, no hardware needed), OTA update traffic (intercept the download), reading the flash chip directly (see `Hardware-Hacking-Fundamentals.md`), or dumping via UART/JTAG. Update packages on the vendor site are a goldmine and require no soldering.
- **Format tells**: `binwalk` signatures reveal embedded filesystems (SquashFS, JFFS2, UBIFS, CramFS, ext), compression (gzip, LZMA, XZ), bootloaders (U-Boot), and kernels. High-entropy regions suggest compression or encryption.
- **Secrets in the rootfs**: `/etc/passwd`/`/etc/shadow` with crackable or hardcoded hashes, hardcoded credentials in web UIs and binaries, private keys and certificates (`/etc/`, `/usr/`), API keys, Wi-Fi/cloud provisioning secrets, and backdoor accounts.
- **The attack surface inside**: the web server (`lighttpd`/`boa`/`goahead` + CGI), `telnetd`/`dropbear`, UPnP, custom daemons, and the scripts that glue them. These are where the remote bugs are.

## Real material

The standard extraction pipeline:

```
binwalk firmware.bin                 # identify sections
binwalk -e firmware.bin              # auto-extract (carves + decompresses known types)
# if auto-extract is incomplete, carve manually at the offsets binwalk reports:
dd if=firmware.bin of=rootfs.squashfs bs=1 skip=<offset> count=<len>
unsquashfs rootfs.squashfs           # for SquashFS; jefferson for JFFS2; ubi_reader for UBIFS
```

Auditing the extracted filesystem:

```
cd squashfs-root
# credentials and keys:
cat etc/passwd etc/shadow
grep -rinE 'password|passwd|admin|root|secret|api[_-]?key|BEGIN (RSA|EC) PRIVATE' . 2>/dev/null
find . -name '*.pem' -o -name '*.key' -o -name '*.crt'
# the web/CGI surface:
ls www/ cgi-bin/ 2>/dev/null
# interesting binaries to reverse:
file bin/* sbin/* usr/sbin/* usr/bin/*  | grep ELF
# startup and backdoor hunting:
cat etc/init.d/* etc/inittab etc/rc* 2>/dev/null
```

Cracking recovered hashes offline:

```
# pull the hash from etc/shadow, then:
hashcat -m 1800 shadow.hash wordlist.txt      # sha512crypt
john --format=md5crypt shadow.hash            # older md5crypt ($1$)
```

## Beyond the basics

- **Vendor downloads beat soldering**: before touching hardware, grab the firmware from the vendor's support site or capture an OTA update. Diffing successive firmware versions also reveals exactly what a security patch changed, which hands you the vulnerability (patch diffing on embedded devices).
- **Handle encrypted firmware**: if the image is high-entropy and `binwalk` finds nothing, it may be encrypted or use a proprietary container. The decryption key often lives in the bootloader, a prior unencrypted firmware version, the SoC, or an update client you can reverse. Sometimes only the app partition is encrypted and the bootloader/kernel are not, giving you a foothold to recover the key. An earlier firmware version is frequently unencrypted and reveals the scheme.
- **Emulate instead of using the device**: tools like QEMU, `firmadyne`, and `FirmAE` boot the extracted filesystem so you can run and attack the web interface and daemons on your workstation, with gdb and a proxy attached. This lets you fuzz and debug services at scale without a physical unit, and run many firmware versions in parallel.
- **Audit the binaries for memory-corruption and command-injection**: embedded web servers and CGI handlers are notorious for command injection (user input into `system()`) and stack overflows (no ASLR/NX on many MIPS/ARM targets, so exploitation is often straightforward). Decompile with Ghidra, trace input from the network to dangerous sinks. See `Assembly-and-Reverse-Engineering.md`.
- **Hardcoded/backdoor credentials are endemic**: many devices ship a vendor support account, a debug password, or a hidden URL/command. Grep aggressively and read the auth code paths; a universal backdoor affecting every unit of a model is a top-tier finding.
- **Keys enable fleet-wide attacks**: a private TLS key or firmware-signing key extracted from one device compromises the whole product line (impersonation, malicious firmware signing). Signing keys in firmware are catastrophic and more common than they should be.
- **Config and NVRAM**: beyond the rootfs, device config (NVRAM, a config partition) holds per-unit secrets, Wi-Fi keys, and cloud tokens; extract and parse it too.

## What to watch for (defensive framing)

- A carved filesystem may be incomplete if offsets or compression were misdetected; verify you extracted the real rootfs (look for a sane Linux directory tree) before concluding "no secrets."
- High entropy is not proof of strong encryption; it may be compression binwalk did not recognize. Try known decompressors and look for an earlier plaintext version.
- Analyze firmware for devices you own or are authorized to test; redistribution of vendor firmware may have legal constraints.
- To defend: encrypt firmware with keys held in a secure element (not in the image), sign updates and verify signatures in a locked bootloader, remove debug accounts and hardcoded credentials, store per-device secrets in hardware not the shared image, and strip build artifacts/keys before shipping.

## Hunting checklist

- Obtain firmware from vendor downloads or OTA capture before resorting to hardware dumps.
- Run `binwalk` to identify and extract; carve manually where auto-extract fails.
- Mount/extract the rootfs; audit `passwd`/`shadow`, keys, certs, and grep for secrets.
- Map the network-facing services (web/CGI, telnet/ssh, UPnP, custom daemons).
- Emulate with QEMU/FirmAE to run and attack services without the device.
- Decompile network-facing binaries; trace input to command-injection and overflow sinks.
- Diff firmware versions to locate patched vulnerabilities.
