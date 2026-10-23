# Assembly and Reverse Engineering

Reverse engineering is reading a program that was never meant to be read, to understand what it does and where it breaks. Assembly is the language you end up in when there is no source: firmware binaries, mobile native libs, malware, DRM, and closed-source services. This file is the working mental model plus the specifics that matter when you are staring at disassembly.

## The mental model

A compiled binary is machine code for a specific CPU architecture, organized into functions that move data between registers, the stack, and memory, and branch on comparisons. Reversing is recovering the intent: which inputs reach which operations, what the checks are (license, password, signature, bounds), and where an assumption can be violated (an unchecked length, a signed/unsigned confusion, a format string). You work top-down (what does this program do) and bottom-up (what does this function do) until they meet.

## Where it hides / what you are looking for

- **The check you want to defeat**: password/license/signature/root-detection comparisons. Find the branch, understand the condition, then either satisfy it, patch it, or glitch past it (on hardware).
- **Memory-corruption bugs**: `strcpy`/`sprintf`/`memcpy` with attacker-controlled length, missing bounds checks on array indexing, integer overflow feeding an allocation or copy, use-after-free, and format-string bugs (`printf(user_input)`).
- **Input-to-sink paths**: where network/file/IPC input enters and what dangerous function it reaches. The same taint-tracking mindset as web injection, in assembly.
- **Crypto and key handling**: constants that identify algorithms (AES S-box, SHA init values), keys loaded from memory, and weak/custom crypto you can break.

## Real material

Core x86-64 you must read fluently (System V calling convention: args in `rdi, rsi, rdx, rcx, r8, r9`, return in `rax`):

```asm
push   rbp                ; function prologue
mov    rbp, rsp
sub    rsp, 0x20          ; allocate locals
mov    edi, 0x...         ; first argument
call   strcmp             ; compare
test   eax, eax           ; eax == 0 ?  (strcmp returns 0 on match)
jne    .fail              ; jump if not equal
; ... success path (e.g. "access granted")
.fail:
leave
ret
```
The password check above is "patch `jne` to `je` (or `nop` it) and every password works." That is the essence of a crack and of understanding a check.

ARM (common in firmware/mobile; AArch64 args in `x0-x7`, return in `x0`):

```asm
stp   x29, x30, [sp, #-16]!   ; prologue, save frame + link register
mov   x0, x19                 ; arg
bl    verify_signature        ; branch-with-link (call)
cbz   w0, .reject             ; compare-and-branch-if-zero
```

A classic stack overflow shape to recognize in decompilation:

```c
void handle(char *net) {
    char buf[64];
    strcpy(buf, net);     // no bounds check: >64 bytes smashes the saved return address
}
```
On a target without stack canaries/NX/ASLR (common on embedded MIPS/ARM), this is a direct control-flow hijack.

Tools and a typical session:

```
# static:
ghidra            # free decompiler; best for reading pseudo-C from assembly
# or: IDA, Binary Ninja, radare2/cutter
objdump -d binary           # quick disassembly
# dynamic:
gdb -q ./binary ; (gdb) break *0x..., run, info registers, x/32x $sp
# with gef/pwndbg extensions for exploit dev
strings -n 8 binary | less  # fast first pass for secrets, format strings, URLs
```

## Beyond the basics

- **Decompiler first, assembly second**: Ghidra's/IDA's pseudo-C gets you 80% of the way; drop to assembly only where the decompiler is wrong or where the bug is in instruction-level detail (a missed sign extension, a flag check). Reading both in parallel builds the fluency that lets you trust neither blindly.
- **Patch, hook, or glitch the check**: once you locate a check, three options. Patch the binary (flip the branch, `nop` the call) for a permanent bypass and to prove the control is client-side. Hook it at runtime (Frida on mobile, a debugger elsewhere) to change inputs/outputs without modifying the file. Or on hardware, glitch past the instruction (see `Hardware-Hacking-Fundamentals.md`). The right tool depends on access.
- **Mitigations change the exploitation, not the bug**: recognize NX (no executable stack, so you need ROP, chaining existing code gadgets), ASLR (randomized addresses, so you need an info leak first), stack canaries (a guard value before the return address, so you need to leak or avoid it), and PIE. Embedded targets frequently lack all of these, which is why firmware RCE is often simpler than desktop RCE.
- **ROP and gadget chains**: when you cannot inject executable code, you reuse snippets ending in `ret` to perform operations (set up a `system()`/`execve` call). `ropgadget`/`ropper` find gadgets; the chain is the modern exploit payload.
- **Architecture breadth**: x86-64 for desktop/server, ARM/AArch64 and MIPS for embedded and mobile, Xtensa for ESP32, AVR for classic Arduino, RISC-V increasingly. The concepts transfer; the registers, calling conventions, and endianness do not. Know which you are in before reading.
- **Anti-reversing and packing**: obfuscation, packers (UPX and custom), anti-debug checks, and virtualized code slow you down. Unpack (dump from memory after the unpacker runs), patch anti-debug, and focus on the small part that matters rather than the whole binary.
- **Taint from input to bug**: the productive workflow for finding vulnerabilities (not just understanding) is to start at input functions (`recv`, `read`, `getenv`, CGI parameter parsing) and follow the data to sinks (`strcpy`, `system`, indexing, `malloc(size)`), exactly like tracing sources to sinks in web DOM analysis.

## What to watch for (defensive framing)

- The decompiler can be subtly wrong (types, calling conventions on stripped binaries); verify a suspected bug against the actual assembly before trusting it.
- A patched/hooked bypass proves the check is not robust, but device-side cracks only matter where the trust should have been server-side; frame impact accordingly.
- Reverse engineering may be restricted by license or law depending on jurisdiction and target; do it on binaries you own or are authorized to analyze.
- To defend: enforce security decisions server-side, compile with all mitigations (NX, ASLR/PIE, canaries, RELRO, CFI), avoid unsafe C string/memory functions, validate all lengths, and use hardware-backed attestation rather than client-side checks that a debugger can flip.

## Study and practice checklist

- Get fluent reading one architecture's assembly (x86-64 or ARM) end to end: prologue, args, branches, calls, returns.
- Learn to pivot between decompiler pseudo-C and raw assembly and to spot where each misleads.
- Practice locating and patching a check (crackmes), then hooking it with a debugger/Frida.
- Recognize the memory-corruption primitives in source and in disassembly.
- Understand NX/ASLR/canaries and the techniques that answer each (ROP, info leak, canary leak).
- Trace input to sink to find bugs, not just to read programs.
