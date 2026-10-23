# Side-Channel Analysis

A chip can keep a key secret from every interface and still leak it through how much power it draws, how long it takes, or what it radiates while using that key. Side-channel analysis recovers secrets by measuring these physical byproducts of computation. It is the attack that works when the firmware is perfect and no bus carries the key in the clear.

## The mental model

Computation has physical cost. Flipping a bit from 0 to 1 draws slightly different power than leaving it at 0; a conditional branch takes a different amount of time; the electromagnetic field around the die mirrors the switching activity. When the data being processed is a secret key, these physical measurements are correlated with the key's bits. By collecting many measurements and correlating them against a model of what each key guess would produce, you recover the key one piece at a time, without ever reading it directly.

## What to aim for

- **Cryptographic key recovery**: the headline prize. Extract an AES, DES, or RSA key from a device (a secure element, a smart card, an MCU doing crypto) that never exposes it over any port. This defeats disk/firmware encryption, secure boot signing (if the private key is on-device), and authentication.
- **PIN/password recovery via timing**: a comparison that returns early on the first wrong byte leaks the correct prefix through timing; recover a secret byte by byte.
- **Operation/branch leakage**: infer which code path ran (which key bits were set) from the power trace, undermining constant-time assumptions.

## The techniques, simplest to strongest

- **Timing analysis**: measure how long an operation takes as a function of input. Non-constant-time comparisons and key-dependent branches leak. Works remotely sometimes (network timing) and locally always.
- **Simple Power Analysis (SPA)**: read the key almost directly from a single (or few) power traces when operations are visibly key-dependent. Classic on naive RSA square-and-multiply: a multiply (for a 1 bit) looks different from a square-only step (for a 0 bit), so the exponent (the private key) is readable off the trace.
- **Differential/Correlation Power Analysis (DPA/CPA)**: the workhorse for symmetric keys. Collect thousands of traces with known/varying inputs, hypothesize a small part of the key (e.g. one byte), model the power for each hypothesis (usually the Hamming weight of an intermediate value), and correlate the model against the measured traces. The correct key byte produces the highest correlation. Repeat for each byte.
- **EM analysis**: the same as power analysis but measured with a small near-field probe over the die, often cleaner and spatially targetable, and it needs no modification to the power line.

## The workflow (CPA on AES with a ChipWhisperer, the standard teaching rig)

```
1. Instrument: measure the target's power (shunt resistor in VCC, or the
   ChipWhisperer's capture) synchronized to the crypto operation via a trigger
   (a GPIO the target toggles, or a pattern in the trace).
2. Collect traces: have the device encrypt many known plaintexts (thousands),
   recording one power trace per encryption, aligned on the trigger.
3. Pick a leakage model and an attack point: for AES, the output of the first-round
   S-box (SubBytes) is the classic target because it mixes one key byte with one
   known plaintext byte.
4. For each key-byte guess (0..255), compute the modeled intermediate (e.g. Hamming
   weight of the S-box output) for every trace, then correlate that model series
   against the measured power at each sample point.
5. The guess with the highest correlation peak is the correct key byte. Recover all
   16 bytes independently; the key falls out in minutes to hours.
```

```python
# conceptual CPA inner idea (per key byte)
best = None
for kguess in range(256):
    model = [hamming_weight(sbox[pt[i] ^ kguess]) for i, pt in enumerate(plaintexts)]
    corr  = max_abs_correlation(model, traces)   # across sample points
    if best is None or corr > best.corr:
        best = (kguess, corr)
# best.kguess is the recovered byte
```

## Beyond the basics

- **Trace alignment is half the battle**: jitter (misaligned operations across traces) destroys correlation. Align traces on a sharp feature, or defeat randomized delays with static alignment/resampling. Clean triggering and alignment often matter more than the number of traces.
- **Template attacks**: profile an identical device you control (build a statistical "template" of its leakage for each key value), then recover the key from very few traces of the target. The most powerful class when you have a clone to profile, which for commodity devices you usually do.
- **Breaking RSA/ECC**: SPA on unprotected modular exponentiation/scalar multiplication reads the private key off a single trace; even protected implementations leak through more advanced analysis (e.g. horizontal attacks on a single trace, or recovering nonces in ECDSA that then yield the private key via lattice math). A single leaked ECDSA nonce bit across many signatures can recover the key.
- **EM for targeting and no-contact**: when you cannot cleanly measure VCC (decoupling, package), an EM probe over the die gets a localized signal, and an XY scan finds the hottest leakage spot. This is how modern, physically protected parts still fall.
- **Fault + side channel together**: combine with glitching (see `Fault-Injection-and-Glitching.md`): inject a fault to skip a side-channel countermeasure, or use faulted outputs (differential fault analysis) to recover a key from just a couple of faulty ciphertexts, which can be far faster than DPA.
- **Countermeasures tell you what the vendor feared**: masking (randomizing intermediate values), hiding (constant-time, balanced logic, noise injection), and shuffling (randomizing operation order) are what good secure elements add. Recognizing their presence (and their imperfections) is part of the assessment; many implementations mask incompletely and still leak.

## What to watch for (defensive framing)

- Side-channel work is measurement-heavy and finicky; expect to spend time on probes, triggering, alignment, and noise before any key appears.
- A correlation peak must be validated (does the recovered key actually decrypt/verify?) before claiming success; noise produces false peaks.
- This assumes physical access and usually a clone to profile; it is research-grade and device-specific.
- To defend: use vetted secure elements with masking/hiding/shuffling for key operations, constant-time comparisons and crypto (no secret-dependent branches or table lookups), fresh random nonces with hardware RNG, key diversification so one recovered key does not unlock the fleet, and tamper/EM shielding for high-value secrets. Each maps to an attacker lever above.

## Checklist

- Decide the target secret (symmetric key, RSA/ECC private key, PIN) and the operation that uses it.
- Instrument power or EM with a clean trigger; collect aligned traces.
- Start with timing/SPA for obvious leaks; escalate to CPA/DPA for symmetric keys.
- Use a profiling clone for template attacks when few target traces are available.
- Consider combined fault + side-channel (DFA) for faster key recovery.
- Validate the recovered secret end to end.
