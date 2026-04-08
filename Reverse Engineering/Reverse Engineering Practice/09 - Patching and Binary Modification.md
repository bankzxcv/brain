---
title: "RE Practice: Patching and Binary Modification"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - patching
  - binary-modification
  - hexedit
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Patching and Binary Modification

12 progressive problems covering NOPing, byte patching, control flow redirection, and string modification.

---

## P1: NOP a Comparison

You have this disassembly and want to always take the true branch:

```asm
401000: cmp    eax, 0
401003: je     401010    ; jump if equal (ZF=1)
401005: mov    edx, 99
40100a: jmp    401015
401010: mov    edx, 42
401015: ...
```

Replace the `je` with NOPs so it always falls through to `mov edx, 99`.

**Expected output:**
```
Original: 74 0B (JE short +0B)
NOPped:   90 90 (two NOP bytes)
```

> [!hint]- Hint
> `JE` is opcode `74` (short jump, 1-byte offset). Two NOPs (`0x90 0x90`) replace the 2-byte `74 0B` instruction.

> [!success]- Solution
> - Original: `74 0B` (JE, takes 2 bytes including offset)
> - NOP'd: `90 90` (two NOP instructions)
> - After patching, `cmp eax, 0` still sets flags, but the `je` is replaced by NOPs → execution falls through to `mov edx, 99` unconditionally.

---

## P2: Patch a Conditional Jump to Unconditional

Change the `JE` (opcode `74`) to an unconditional `JMP` (opcode `EB`):

```asm
401000: cmp    eax, 0
401003: je     401010    ; currently: 74 0B
```

What do you replace the bytes with?

**Expected output:**
```
Original: 74 0B (JE +0x0B from next instruction = 401005+0B=401010)
Replace with: EB 09 (JMP +0x09 from next instruction = 401005+9=40100E)
Wait, that goes to wrong place... Let me recalculate
```

> [!hint]- Hint
> `JE` offset is relative to the next instruction (401005), not 401003. `74 0B` = jump +0x0B from 401005 = 401010. `JMP` needs the same offset.

> [!success]- Solution
> - Original: `74 0B` → JE takes branch if equal, offset = 0x0B from 401005 = destination 401010
> - Replacement: `EB 0B` → unconditional JMP, same offset 0x0B from 401005 = destination 401010
> - Byte patch: change `74` to `EB`, keep offset `0B`
> - This makes the jump always taken, going to 401010 instead of falling through

---

## P3: Patch with radare2

```bash
# Open binary in radare2 (read-write)
rizin -w /tmp/branch

# Disassemble at address
[0x00401000]> pd 5
[0x00401000]> wa jmp 0x00401010
[0x00401000]> pd 5  # verify
```

What does `wa` do? How do you verify the patch?

**Expected output:**
```
wa = write assembly
The instruction at the address changes to JMP 0x401010
```

> [!hint]- Hint
> `wa` (write assembly) assembles the instruction and writes bytes directly into the binary file.

> [!success]- Solution
> `wa jmp 0x00401010` assembles `EB 0B` (or appropriate offset for short jump) and writes it to the current address. `pd` (print disassembly) verifies the new bytes.

---

## P4: Patch a String

Find and modify a string in a binary:

```bash
cat > /tmp/strtest.c << 'EOF'
#include <stdio.h>
int main() {
    printf("LICENSED\n");
    return 0;
}
EOF
gcc -o /tmp/strtest /tmp/strtest.c

# Find the string
strings /tmp/strtest | grep LICENSED
xxd /tmp/strtest | grep -A1 LICENSED

# The string "LICENSED" is 8 bytes including null terminator
# To patch "UNLICENSED" (10 bytes) you'd need same length or shorter
```

**Expected output:**
```
Found string at offset 0x2008 (example)
Original: LICENSED (8 bytes)
Can only replace with same-length or shorter string
```

> [!hint]- Hint
> Strings in the binary are null-terminated. You can overwrite them in place if the new string is the same length or shorter. Longer strings require moving to a new location.

> [!success]- Solution
> - `strings /tmp/strtest | grep LICENSED` → finds the offset
> - `xxd` shows the hex at that offset
> - To patch: `printf "UNLICENSED" | dd of=/tmp/strtest bs=1 seek=$((offset)) count=8 conv=notrunc`
> - "LICENSED" (8 bytes) can be replaced with "UNLICENSED" if padded: "UNLIC\0\0" (8 bytes)

---

## P5: Calculate Jump Offset for Long Jump

The current code:

```asm
0x401000: je     0x401050    ; currently 74 4E (short jump)
0x401002: nop
```

You want to change it to a near jump (`E9 xx xx xx xx`). Calculate the 32-bit relative offset.

**Expected output:**
```
Near jump opcode: E9 [offset]
Offset = target - (next_instruction_addr + 5)
      = 0x401050 - (0x401002 + 5)
      = 0x401050 - 0x401007
      = 0x49

New bytes: E9 49 00 00 00
```

> [!hint]- Hint
> Near jump (E9) is followed by a 32-bit little-endian signed offset. The offset is relative to the instruction AFTER the jump instruction (next instruction = address + opcode_size + 4 = address + 5).

> [!success]- Solution
> - Target: 0x401050
> - Next instruction: 0x401002 (short jump at 0x401000, 2 bytes, so next is 0x401002)
> - Offset = 0x401050 - (0x401002 + 5) = 0x401050 - 0x401007 = 0x49
> - New bytes: `E9 49 00 00 00` (little-endian 32-bit = `49 00 00 00`)

---

## P6: Redirect Control Flow via Register

Sometimes a function pointer is called via a register:

```asm
401000: mov    rax, [rbp-0x8]   ; load function pointer
401004: call   rax              ; call the function (FF D0)
```

Change it to always call `0x401100` instead of the dynamic address.

**Expected output:**
```
Original: FF D0 (CALL RAX)
Replace with: FF D0 (CALL RAX) but before it: mov rax, 0x401100
Or use: FF D2 (CALL RDX) with rdx set to 0x401100
```

> [!hint]- Hint
> `FF D0` = `CALL RAX`. To redirect, you need to either load a fixed address into RAX before the call, or replace the `CALL RAX` with a `CALL rel32` if possible.

> [!success]- Solution
> Option 1: Add a `mov rax, 0x401100` before the call:
> ```
> 48 B8 00 11 40 00 00 00 00 00  ; MOV RAX, 0x401100 (10 bytes)
> FF D0                          ; CALL RAX
> ```
> Option 2: If RAX is already set elsewhere and you want to preserve that, a simpler approach is to NOP the load and replace the call target via GOT or direct patching.

---

## P7: Patch in Ghidra

In Ghidra:
1. Open the binary in Ghidra
2. Navigate to the instruction to patch
3. Right-click → "Patch Instruction"
4. Change `JE` to `JMP`
5. Right-click on bytes → "Patch Data" → change bytes
6. File → Export → Modified Program

**Expected output:**
```
Modified binary saved with patched instructions
Can verify with objdump -d on the exported binary
```

> [!hint]- Hint
> Ghidra's "Patch Instruction" lets you overwrite bytes with different assembly. "Patch Data" lets you edit raw bytes. Export as the same format to get a patched binary.

> [!success]- Solution
> Ghidra's patching is intuitive — click on the bytes or instruction, type the new instruction, and the underlying bytes change in the database. Export the modified program to get a patched binary file.

---

## P8: Patch Using printf and dd

Patch a single byte without any special tool:

```bash
# Original binary at 0x1234 has byte 0x74 (JE)
# Replace with 0xEB (JMP)

# Method: printf + dd
printf '\xEB' | dd of=/tmp/branch bs=1 seek=$((0x1234)) count=1 conv=notrunc

# Verify
xxd /tmp/branch | grep -A1 "1234"
```

**Expected output:**
```
Byte at offset 0x1234 changed from 74 to EB
```

> [!hint]- Hint
> `printf '\xEB'` outputs one byte (0xEB). `dd` writes it to the offset. `conv=notrunc` prevents truncation of the output file.

> [!success]- Solution
> This is the most basic byte-level patching:
> ```bash
> printf '\xEB' | dd of=binary bs=1 seek=$((0x1234)) count=1 conv=notrunc
> ```
> Works on any Unix system with no special tools needed. Verify with `xxd` or `objdump -D`.

---

## P9: BinDiff — Compare Patched vs Original

```bash
# Create original and patched versions
cp /tmp/branch /tmp/branch_orig
# Patch the copy
# (apply patch)

# Use diff on objdump output
diff <(objdump -d /tmp/branch_orig) <(objdump -d /tmp/branch)
```

What differences does the diff show?

**Expected output:**
```
Only the patched location shows difference
All other instructions unchanged
```

> [!hint]- Hint
> A good patch changes only the intended bytes. `diff` of objdump output shows exactly what changed.

> [!success]- Solution
> The diff output shows only the single instruction that was changed. Everything else is identical, confirming a clean targeted patch with no collateral damage.

---

## P10: Patch License Check — Full Workflow

You analyze a binary and find this license check:

```c
// Decompiled from Ghidra
int check_license(char *input) {
    char expected[32];
    strcpy(expected, "L1C3N5E");  // hidden in binary
    if (strcmp(input, expected) == 0) {
        return 1;  // valid
    }
    return 0;  // invalid
}
```

You want to make every license valid. Identify where to patch:

**Expected output:**
```
Option 1: NOP the strcmp call and set return to 1
Option 2: Change the JE after strcmp to JMP unconditionally
Option 3: Patch the string "L1C3N5E" to match any input (not possible with strcmp)
```

> [!hint]- Hint
> The check is `if (strcmp(...) == 0)`. If we NOP the `je` (or change it to `jmp`), we always take the valid path regardless of input.

> [!success]- Solution
> Best approach: patch the conditional jump after `strcmp`:
> 1. Find the `JE` (or `JNE`) instruction that follows `strcmp`
> 2. Change `74 xx` (JE) to `EB xx` (JMP) to always take the valid branch
> Or: patch `strcmp` to return 0 always → change `call strcmp` + next instruction to `xor eax, eax; nop`

---

## P11: Verify Patch with GDB

After patching, verify the change works:

```bash
# Patch the binary (JE -> JMP at address 0x401003)
printf '\xEB' | dd of=/tmp/branch bs=1 seek=$((0x401003)) count=1 conv=notrunc

# Verify with GDB
gdb -q /tmp/branch << 'GDB'
disassemble 0x401000, 0x401020
break *0x401005
run
continue  # should now skip to valid path
GDB
```

**Expected output:**
```
Disassembly shows JMP instead of JE
Breakpoint at 0x401005 now hit (previously might have been skipped)
```

> [!hint]- Hint
> GDB confirms the bytes changed. Running the patched binary confirms the behavior change.

> [!success]- Solution
> GDB's `disassemble` shows the patched opcode (`EB` instead of `74`). Running with breakpoints confirms the program now takes the patched path.

---

## P12: Use LLM to Guide a Patch

```
PROMPT: "I need to patch a binary to bypass a license check. Here is the
disassembly around the check:

0x401000: call   strcmp
0x401005: test   eax, eax
0x401007: je     0x401020    ; 74 17
0x401009: mov    edi, 1
0x40100e: call   validate
0x401013: mov    [rbp-0x4], eax
0x401016: jmp    0x401035
0x401020: mov    edi, 0
0x401025: call   exit

Which bytes do I need to change to make the program always call validate()?
Show me the before/after bytes."
```

**Expected output:**
```
(LLM calculates: change 74 17 (JE +0x17) to EB 00 (JMP +0 = next instruction, falling through)
or EB 11 (JMP to 0x40101A to skip to the mov edi,1 path)
Actual patch depends on desired outcome: always valid or always invalid)
```

> [!hint]- Hint
> If you want to always validate: change the `JE` to `JMP` with an offset that skips to the valid path (0x401009). If you want to always reject: change to `JMP` to 0x401020.

> [!success]- Solution
> To always call `validate()`:
> - Change `74 17` (JE to 0x401020) to `EB 09` (JMP forward +9 bytes to 0x401009, which is `mov edi, 1` / `call validate`)
> - Byte patch: `74 17` → `EB 09`

---

## AI Power-Up: Patching

```
PROMPT: "I'm patching a binary to bypass a password check. The relevant
disassembly is:

0x401000: cmp    byte [rbp-0x10], 0
0x401004: je     0x401050
0x401006: jmp    0x401080    ; fail path

I want to always take the success path. Give me the exact bytes to write and
at what offset, with the calculation explained."
```
