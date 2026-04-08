---
title: "RE Practice: Anti-Reverse Engineering"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - anti-debug
  - anti-disassembly
  - obfuscation
  - packing
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Anti-Reverse Engineering

12 progressive problems covering anti-debug, anti-disassembly, packing detection, and obfuscation analysis.

---

## P1: Detect ptrace Anti-Debug

Create a binary that uses `ptrace` to detect debuggers:

```bash
cat > /tmp/antidebug.c << 'EOF'
#include <stdio.h>
#include <sys/ptrace.h>

int main() {
    if (ptrace(PTRACE_TRACEME, 0, 1, 0) == -1) {
        printf("Debugger detected! Exiting.\n");
        return 1;
    }
    printf("Normal execution.\n");
    return 0;
}
EOF
gcc -o /tmp/antidebug /tmp/antidebug.c

# Run normally (should work)
./tmp/antidebug

# Run with strace (tracer detected!)
strace /tmp/antidebug 2>&1 | head -5
```

**Expected output:**
```
Normal execution when run directly
ptrace fails when run under strace (strace itself acts as tracer)
```

> [!hint]- Hint
> `PTRACE_TRACEME` allows a process to be traced by its parent. If already being traced, it fails. `strace` attaches as a tracer, so `ptrace` returns -1.

> [!success]- Solution
> - Direct execution: `PTRACE_TRACEME` succeeds, program runs normally
> - Under `strace`: strace acts as a tracer, so `ptrace` fails with `EPERM` (operation not permitted because already traced)
> - The detection works because `strace` attaches before the program can `ptrace(PTRACE_TRACEME)`

---

## P2: Anti-Disassembly — Opaque Predicates

You encounter this code:

```asm
401000: mov    eax, 1
401005: and    eax, eax        ; sets ZF based on eax
401007: jne    401010         ; jmp if eax != 0
401009: jmp    401020          ; unreachable!
401010: ...
```

The jump at 401009 can never be reached because `and eax, eax` is always non-zero when `eax=1`. This is an **opaque predicate**.

What is the purpose of unreachable code in anti-disassembly tools?

**Expected output:**
```
Opaque predicates confuse linear sweep disassemblers.
The unreachable code might contain real code that only executes via indirect jumps.
```

> [!hint]- Hint
> Opaque predicates are used to insert dead code that looks real but is never executed via normal control flow. This can mislead disassemblers that use recursive descent.

> [!success]- Solution
> The unreachable code at 401009 is dead code. However:
> 1. It might contain bytes that look like instructions but are never reached via the visible flow
> 2. A jump-to-middle-of-instruction attack could land in the dead code and execute it
> 3. It makes the control flow graph look more complex than it really is

---

## P3: Detect Packed Binary with Entropy

```bash
cat > /tmp/pack_check.py << 'EOF'
import math

def entropy(data):
    freq = [0] * 256
    for b in data: freq[b] += 1
    ent = 0
    for f in freq:
        if f: ent -= (f/len(data)) * math.log2(f/len(data))
    return ent

# Test on normal binary
with open("/bin/ls", "rb") as f:
    data = f.read(10000)
    print(f"Normal binary entropy: {entropy(data):.2f}")

# Test on compressed data
import zlib
compressed = zlib.compress(data)
print(f"Compressed data entropy: {entropy(compressed[:10000]):.2f}")
EOF
python3 /tmp/pack_check.py
```

What entropy range indicates packing?

**Expected output:**
```
Normal binary: ~6.5-7.0
Compressed/packed: ~7.5-8.0
```

> [!hint]- Hint
> High entropy (closer to 8.0 bits/byte) indicates random or compressed data. Normal code has patterns (lower entropy). Packers compress/encrypt the original code.

> [!success]- Solution
> - Entropy near 8.0 = highly random/compressed (likely packed or encrypted)
> - Entropy around 6.5-7.0 = structured binary data (normal)
> - The transition point is about 7.3 — above this is suspicious for normal executables

---

## P4: Detect UPX Packing

```bash
cat > /tmp/upx_test.c << 'EOF'
#include <stdio.h>
int main() { printf("UPX test\n"); return 0; }
EOF
gcc -o /tmp/upx_test /tmp/upx_test.c

# Check strings
strings /tmp/upx_test | grep -i upx

# Try to pack with UPX (install first: apt install upx-tools or brew install upx)
# upx -9 /tmp/upx_test

# Check entry point
readelf -h /tmp/upx_test | grep Entry
objdump -d /tmp/upx_test | head -10
```

What changes after packing?

**Expected output:**
```
Before: strings include normal libc strings
After UPX: strings include "UPX0", "UPX1", compression markers
Entry point changes from .text to UPX stub
```

> [!hint]- Hint
> UPX adds its own loader/stub. The original entry point is stored in the packed binary and restored at runtime.

> [!success]- Solution
> After UPX packing:
> - New sections: `.upx0`, `.upx1` added
> - Entry point changes to the UPX stub (instead of original `.text`)
> - Original entry point stored in UPX metadata
> - Strings like "UPX!" appear in the packed binary

---

## P5: Unpack UPX Manually

```bash
# Install UPX
# Linux: sudo apt install upx-tools
# macOS: brew install upx

# Pack a binary
upx -9 /tmp/upx_test

# Unpack it
upx -d /tmp/upx_test

# Verify
diff <(objdump -d /tmp/upx_test.original) <(objdump -d /tmp/upx_test.unpacked)
```

**Expected output:**
```
After unpacking, entry point and sections restored
```

> [!hint]- Hint
> UPX has a built-in unpacker. For custom packers, you need to manually unpack by running the packed binary and dumping memory.

> [!success]- Solution
> `upx -d` restores the original binary. For custom packers, the manual process is:
> 1. Set breakpoint at original entry point (OEP)
> 2. Let the unpacker stub run
> 3. Dump memory at OEP
> 4. Fix imports if needed

---

## P6: Anti-Debug — Timing Attack

```bash
cat > /tmp/timing.c << 'EOF'
#include <stdio.h>
#include <time.h>
#include <sys/time.h>

unsigned long rdtsc() {
    unsigned int lo, hi;
    __asm__ __volatile__ ("rdtsc" : "=a" (lo), "=d" (hi));
    return ((unsigned long)hi << 32) | lo;
}

int main() {
    unsigned long start = rdtsc();
    volatile int x = 0;
    for (int i = 0; i < 100; i++) x++;
    unsigned long end = rdtsc();
    
    printf("Cycles elapsed: %lu\n", end - start);
    
    if ((end - start) > 100000) {
        printf("Debugger detected (slow execution)!\n");
        return 1;
    }
    return 0;
}
EOF
gcc -o /tmp/timing /tmp/timing.c
./tmp/timing
strace -c /tmp/timing
```

What does the timing attack detect?

**Expected output:**
```
Normal execution: few thousand cycles
Under strace/gdb: hundreds of thousands of cycles
```

> [!hint]- Hint
> Debugging slows execution dramatically. RDTSC counts CPU cycles. If the cycle count is unexpectedly high, a debugger may be present.

> [!success]- Solution
> The `rdtsc` timing check measures execution time. Under a debugger, each instruction takes much longer, so the cycle count spikes. This detects strace, GDB, and other debuggers that significantly slow execution.

---

## P7: Control Flow Flattening Detection

You encounter this pattern in a function:

```asm
; Flattened control flow — loop-based dispatch
401000: mov    eax, [rbp-0x4]    ; load state variable
401003: sub    eax, 1
401006: shl    eax, 2             ; eax = (state-1) * 4
401009: add    eax, offset jumptable
40100f: mov    eax, [eax]         ; load target address
401011: jmp    eax                ; dispatch

; Instead of natural if/else or switch
```

What does this indicate?

**Expected output:**
```
Control flow flattening — natural control flow replaced with a loop + switch.
This is a common obfuscation technique.
```

> [!hint]- Hint
> Natural control flow is a tree. Flattened control flow is a loop with a state variable determining which basic block executes next.

> [!success]- Solution
> Control flow flattening replaces the natural hierarchical control flow with:
> 1. A loop that iterates over basic blocks
> 2. A state variable (or dispatch table) that determines which block runs next
> 3. A jump table for indirect dispatch
> This makes the CFG look like a switch inside a loop, obscuring the original logic. It's commonly used by obfuscators like VMProtect.

---

## P8: Import Hashing Recognition

You see this in a binary's import table:

```asm
; Instead of:
call qword [rel 0x201018]  ; calls GetProcAddress by name

; The binary does:
push 0xA2D4E8F1   ; hash of "GetProcAddress"
call resolve_by_hash  ; returns function address
mov [rel 0x203000], rax  ; stores in IAT entry
```

How do you find the actual function being resolved?

**Expected output:**
```
The binary computes GetProcAddress by hash. To reverse:
1. Find the hash algorithm
2. Brute-force possible hashes
3. Or hook GetProcAddress and log calls
```

> [!hint]- Hint
> Import hashing makes static analysis harder. Dynamic analysis (hooking) reveals the resolved function names.

> [!success]- Solution
> Import hashing (IAT hashing) hides imports by computing hashes at runtime:
> 1. Write a script to hook `GetProcAddress` and `LoadLibrary`
> 2. Run the binary and log each resolved name
> 3. Or reverse the hash algorithm and try to find which strings produce the observed hashes
> Tools like `Frida` are ideal for this

---

## P9: Recognize Anti-Disassembly Tricks

You see this disassembly:

```asm
401000: jmp    401002        ; jumps to middle of next instruction
401002: 90                   ; NOP
401003: 48                   ; middle of: 48 89 C3 (mov rbx, rax)
401004: 89 C3
401006: ret
```

What is happening here?

**Expected output:**
```
Jump to middle of instruction (anti-disassembly trick).
A linear sweep disassembler would see:
40 90 48 89 C3  (inc eax; nop; mov rbx, rax)
But the actual execution is:
jmp 401002 -> NOP -> 48 (partial) -> ... which is garbage
The real code is only reached via the jump.
```

> [!hint]- Hint
> The `jmp 401002` lands on the `90` (NOP). The `48` byte at 401003 is never executed via normal flow. But a disassembler starting at 401000 would decode `40 90` as `inc eax; nop`, completely misleading analysis.

> [!success]- Solution
> This is a classic anti-disassembly trick:
> 1. `jmp 401002` skips the `48` byte
> 2. The `48` is the first byte of `mov rbx, rax` (which would be `48 89 C3`)
> 3. Without the jump, a linear disassembler decodes garbage
> 4. With the jump, the real code path executes correctly

---

## P10: Obfuscated String Decryption

You encounter this decryption function:

```asm
decrypt_loop:
401000: movzx  eax, byte [rsi]     ; load encrypted byte
401003: xor    al, 0x55             ; XOR with key
401006: mov    byte [rdi], al       ; store decrypted byte
401009: inc    rsi
40100c: inc    rdi
40100f: loop   decrypt_loop          ; repeat until RCX=0
```

How would you decrypt the strings offline?

**Expected output:**
```
The key is 0x55. Write a Python script:
for each encrypted_byte in data:
    decrypted = encrypted_byte ^ 0x55
```

> [!hint]- Hint
> Single-byte XOR with key `0x55`. Each byte is XORed with the same key.

> [!success]- Solution
> Python decryption script:
> ```python
> key = 0x55
> encrypted = bytes.fromhex("...")  # extracted from binary
> decrypted = bytes(b ^ key for b in encrypted)
> print(decrypted.decode('ascii', errors='replace'))
> ```
> For multi-byte keys or more complex algorithms, extend accordingly.

---

## P11: Virtualization Detection

You see an unusual section in a binary:

```asm
; .vmp0 section contains bytecode-like data
; The VM entry function:
vm_entry:
401000: mov    rbp, offset vm_context
401007: mov    rax, [rbp+0x10]     ; program counter
40100f: shl    rax, 3
401013: add    rax, offset handlers ; jump table
40101a: jmp    rax                  ; dispatch to handler
```

What does this indicate?

**Expected output:**
```
Custom virtual machine (VM) obfuscation.
The binary contains bytecode that is interpreted by an embedded VM.
```

> [!hint]- Hint
> A jump table + dispatch pattern that reads a "program counter" from a context structure is the hallmark of VM-based obfuscation (like VMProtect or Themida).

> [!success]- Solution
> VM-based obfuscation:
> 1. The binary contains `.vmp0` or similar sections with bytecode
> 2. A VM interpreter reads opcodes and dispatches to handler functions
> 3. The handlers implement the original program's logic in bytecode
> 4. To RE, you need to: reverse the VM architecture, decode the bytecode, write a disassembler for the bytecode

---

## P12: Use LLM to Deobfuscate

```
PROMPT: "I'm analyzing a malware sample with XOR obfuscation. The decryption
function is shown below. Write a Python script that:
1. Extracts the encrypted strings from the binary at the given offset
2. Decrypts them using the algorithm shown
3. Outputs all decrypted strings with their addresses

[decryption function disassembly]"
```

**Expected output:**
```
(LLM writes Python script to decrypt the strings.
Key identified as single-byte XOR.
For each encrypted byte, XOR with key to recover plaintext.)
```

> [!hint]- Hint
> The LLM can read the disassembly, identify the algorithm, and write the decryption script.

> [!success]- Solution
> The LLM correctly identifies:
> 1. The XOR key (e.g., `0x55`)
> 2. The encryption loop (XOR each byte with key)
> 3. Writes a Python script to extract and decrypt all strings from the binary

---

## AI Power-Up: Anti-RE

```
PROMPT: "I'm analyzing a packed binary. The unpacker stub runs and then jumps
to the original entry point. Here is the strace of the unpacked process.
Can you identify the OEP and suggest how to dump memory at that point?"

PROMPT: "This binary uses string encryption with a single-byte XOR. Here is the
decryption function. Write a Ghidra Python script that:
1. Finds all encrypted string locations
2. Decrypts them in-place
3. Replaces the encrypted bytes with decrypted strings"
```
