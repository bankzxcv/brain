---
title: "RE Practice: Disassembly and Static Analysis"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - disassembly
  - static-analysis
  - radare2
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Disassembly and Static Analysis

13 progressive problems using objdump, radare2/rizin, and Ghidra for static reverse engineering.

---

## P1: Disassemble with Objdump

Compile a simple C program and disassemble it:

```bash
cat > /tmp/simple.c << 'EOF'
int square(int x) {
    return x * x;
}

int main() {
    return square(5);
}
EOF
gcc -O0 -o /tmp/simple /tmp/simple.c
objdump -M intel -d /tmp/simple | grep -A 30 '<square>:'
```

Trace through the disassembly. How many local variables are allocated?

**Expected output:**
```
Shows push rbp, mov rbp,rsp, then square function code
Local variables: 0 (just uses registers)
```

> [!hint]- Hint
> With `-O0`, the compiler is verbose. But if no local variables are needed, the `sub rsp, imm` amount will be 0 or minimal.

> [!success]- Solution
> With no local variables needed, the compiler may just do `push rbp; mov rbp,rsp` without adjusting the stack. The square function: loads `edi` (first int arg), `imul edi, edi`, result in `eax`, `pop rbp`, `ret`.

---

## P2: Linear Sweep vs Recursive Descent

Compare how `objdump` (linear sweep) and `radare2` (recursive descent) disassemble the same code:

```bash
# Linear sweep (objdump disassembles every byte)
objdump -d /tmp/simple | head -60

# Recursive descent (radare2 follows control flow)
# Install radare2: brew install radare2 (macOS) or sudo apt install rizin
rizin -q -c 'aaa; pdf @ main' /tmp/simple
```

What differences do you notice in how they handle the same code?

**Expected output:**
```
objdump: shows all bytes as instructions, may disassemble data as code
rizin: follows jumps/calls, skips data sections
```

> [!hint]- Hint
> Linear sweep disassembles sequentially. Recursive descent follows control flow (jumps, calls) and only disassembles reachable code.

> [!success]- Solution
> - `objdump` shows everything including bytes that might be data (like string literals or padding)
> - `rizin` with `aaa` (auto-analysis) identifies functions, follows calls, skips over data. It produces a more accurate control flow graph.

---

## P3: Find main in a Stripped Binary

Create and strip a binary, then find `main`:

```bash
cat > /tmp/stripped.c << 'EOF'
int compute(int x) { return x + 10; }
int main() { return compute(42); }
EOF
gcc -o /tmp/stripped /tmp/stripped.c
strip /tmp/stripped
nm /tmp/stripped 2>&1   # should show "no symbols"
```

Now try to find `main` using these methods:

```bash
# Method 1: use the entry point and trace
objdump -M intel -d /tmp/stripped | head -80

# Method 2: strings + xrefs
strings /tmp/stripped | grep -i main
```

**Expected output:**
```
With -O0, main is identifiable as it calls compute()
The binary with symbols would show main at e.g. 0x401126
```

> [!hint]- Hint
> In a stripped binary compiled with -O0, `main` is often identifiable because it's called by the C runtime startup code (`__libc_start_main`). Look for the pattern: `call main` from a known startup function.

> [!success]- Solution
> - `__libc_start_main` in `.plt` is called from `_start`
> - Its argument includes a pointer to `main`
> - Search: `objdump -d | grep -B5 'call.*main'`
> - Or use `rizin` with `afl` to list all functions (it will auto-identify `main` as `entry.fcn.main`)

---

## P4: Cross-References (XREFs) in Radare2

Open a binary in radare2 and find all references to a known function:

```bash
rizin -q -c 'aaa; axt sym.imp.strlen' /tmp/simple
# Or find xrefs to a specific string:
rizin -q -c 'aaa; / RAX' /tmp/simple | head -20
```

What is an XREF? How does radare2 track them?

**Expected output:**
```
CODE XREF: sub_401126+5D
DATA XREF: [0x404000]
```

> [!hint]- Hint
> XREFs are connections between code/data locations. CODE XREFs are from code to code (calls/jumps). DATA XREFs are from code to data (pointers, constants).

> [!success]- Solution
> - CODE XREF: shows where a function is called from. Format: `address ↑ context`
> - DATA XREF: shows where a data address is referenced from
> - `axt @ addr` = find all xrefs TO addr
> - `axf @ addr` = find all xrefs FROM addr

---

## P5: Identify Function Signatures

You find this function signature in a binary:

```
Function: main
  Parameters: int64_t arg1, int64_t arg2
  Returns: int64_t
  Locals: int64_t var_8h, int64_t var_10h, int64_t var_18h
```

What does this tell you about how the function was called? How much local stack space is allocated?

**Expected output:**
```
Total stack allocation: 0x18 bytes (24 bytes)
Parameters at positive offsets (rbp+X) = passed via registers (x86-64 SysV)
Locals at negative offsets (rbp-X) = stack-allocated
```

> [!hint]- Hint
> In x86-64 with frame pointers, parameters are at `[rbp+X]`, locals at `[rbp-X]`. Stack grows downward, so local variables are below rbp (negative offsets from rbp).

> [!success]- Solution
> - `var_8h` at rbp-0x8, `var_10h` at rbp-0x10, `var_18h` at rbp-0x18
> - Total = 0x18 bytes = 24 bytes of local variables
> - Parameters at positive rbp offsets = SysV calling convention (passed in registers)

---

## P6: Analyze a Function's Control Flow

Create a binary with branching logic and analyze its control flow:

```bash
cat > /tmp/branch.c << 'EOF'
int classify(int x) {
    if (x < 0) return -1;
    if (x == 0) return 0;
    if (x < 10) return 1;
    return 2;
}
EOF
gcc -O0 -o /tmp/branch /tmp/branch.c
objdump -M intel -d /tmp/branch | grep -A 50 '<classify>:'
```

Draw the control flow graph (CFG) based on the disassembly.

**Expected output:**
```
classify:
  cmp edi, 0
  jl neg        ; if x < 0, goto -1
  je zero       ; if x == 0, goto 0
  cmp edi, 0xa
  jl small      ; if x < 10, goto 1
  jmp default   ; goto 2
```

> [!hint]- Hint
> Each `cmp` sets flags, each `je/jl` creates a branch. The control flow graph shows all possible paths through the function.

> [!success]- Solution
> The CFG shows four paths: x<0 → -1, x==0 → 0, 0<x<10 → 1, x>=10 → 2. The disassembly confirms this with three conditional jumps plus one unconditional.

---

## P7: Identify Function Prologue and Epilogue

In the following x86-64 function prologue, how much stack space is allocated?

```
0:  push   rbp
1:  mov    rbp, rsp
4:  sub    rsp, 0x30
```

And what does the epilogue look like?

**Expected output:**
```
Stack allocated: 0x30 = 48 bytes
Epilogue: leave (or: mov rsp, rbp; pop rbp)
```

> [!hint]- Hint
> `push rbp` saves the old base pointer. `mov rbp, rsp` sets up the new frame. `sub rsp, N` allocates N bytes of local variable space.

> [!success]- Solution
> - `sub rsp, 0x30` allocates 48 bytes of local variable space
> - Epilogue is typically: `leave` (equivalent to `mov rsp, rbp; pop rbp`) then `ret`
> - The `leave` instruction automatically restores the stack frame

---

## P8: Radare2 Visual Mode Exploration

```bash
rizin -c 'aaa' /tmp/simple
# In visual mode:
# - Arrow keys to navigate
# - : to enter command
# - V to toggle between hex/asm/decompiler
# - q to quit
```

Use radare2 to navigate to the `main` function and find:
1. All cross-references TO `main`
2. All functions called FROM `main`

**Expected output:**
```
main calls: compute (or libc start functions)
main is called by: __libc_start_main (in the startup code)
```

> [!hint]- Hint
> In radare2: `afl` = list functions, `axt @ sym.main` = who calls main, `aft @ sym.main` = trace function calls from main.

> [!success]- Solution
> Commands:
> - `afl` → list all identified functions
> - `axt @ main` → xrefs to main (who calls it)
> - `aft @ main` → trace/analyze function call tree from main
> - `VV @ main` → visual graph of main's control flow

---

## P9: Identify Library Functions by Signature

You see these instructions in a stripped binary. Which libc function is being called?

```asm
lea rdi, [rbp-0x20]    ; address of buffer
call qword [rel 0x201018]  ; GOT entry
mov eax, 0
```

**Expected output:**
```
Most likely: scanf (reads input into buffer at rbp-0x20)
```

> [!hint]- Hint
> `lea rdi` loads a buffer address (first argument to the function). The `call [GOT entry]` calls a shared library function via the PLT.

> [!success]- Solution
> Pattern: address-loaded-into-rdi + call-via-GOT → first arg is buffer address → likely `scanf`, `gets`, `fgets`, `read`, or similar input function. `mov eax, 0` after the call suggests the function returns int (common for scanf). If the buffer is on the stack, it's likely `scanf("%s", buffer)` style input.

---

## P10: Identify String References

Find all strings referenced from a function:

```bash
# In Ghidra: Search -> References -> To "string name"
# In radare2:
rizin -q -c 'aaa; axt @ str.' /tmp/simple

# With objdump:
objdump -M intel -d /tmp/simple | grep -E 'call|mov.*\[rip'
```

What strings does the binary reference? What functions use them?

**Expected output:**
```
Strings like "Hello, %s!\n" referenced by printf
Binary uses libc functions via PLT/GOT
```

> [!hint]- Hint
> Strings in `.rodata` are referenced via `rip`-relative addressing: `lea rdi, [rip+offset]` or `mov edi, offset`.

> [!success]- Solution
> String references typically use `lea rdi, [rip+str_offset]` or a `mov` with the address in `rdi`/`edi` before a `call`. Use `objdump -d -M intel | grep rip` to find RIP-relative accesses, which often reference strings or global data.

---

## P11: Ghidra Decompiler — Rename and Annotate

Open `/tmp/simple` in Ghidra. Try these:
1. Import the binary
2. Let it auto-analyze
3. Find `main` in the Function Graph
4. Rename the decompiled variable `param_1` to `input_value`
5. Add a comment explaining what the function does
6. Export the modified database

```bash
# Headless Ghidra analysis:
analyzeHeadless /tmp/ghidra_proj -import /tmp/simple -scriptPath . -postScript PrintFunctions.java
```

**Expected output:**
```
Main identified, decompilation shows C-like pseudocode
```

> [!hint]- Hint
> In Ghidra, press `Tab` to switch between disassembly and decompiler view. Press `L` to rename symbols. Press `;` to add comments.

> [!success]- Solution
> Ghidra decompiles to C pseudocode. The `square` function becomes:
> ```c
> int square(int x) {
>   return x * x;
> }
> ```
> You can rename variables (`L` key), add comments (`;` key), and define structures (`Shift+F`) to improve the decompilation.

---

## P12: Analyze a CTF Binary's Main Function

You are given a CTF binary with the following Ghidra decompilation of `main`:

```c
undefined8 main(void) {
    long in_FS_OFFSET;
    long canary;
    
    canary = *(long *)(in_FS_OFFSET + 0x28);
    setvbuf(stdin, (char *)0x0, 2, 0x0);
    setvbuf(stdout, (char *)0x0, 2, 0x0);
    printf("Enter the password: ");
    __isoc99_scanf("%99s", local_50);
    if (local_10 == 0x1337) {
        puts("Correct!");
        return 0;
    }
    puts("Wrong!");
    return 1;
}
```

What are the security issues in this code?

**Expected output:**
```
Issues: 
1. Buffer overflow: local_50 is 99 bytes but input goes to local_10 (adjacent!)
2. Stack canary present (in_FS_OFFSET)
3. No null check on scanf return
```

> [!hint]- Hint
> Look at the stack layout. `local_50` (80 bytes?) and `local_10` (16 bytes?) are adjacent. `scanf("%99s")` reads 99 bytes into `local_50`, which can overflow into `local_10`.

> [!success]- Solution
> **Critical**: The buffer `local_50` overflows into `local_10` because `%99s` reads 99 bytes into a stack buffer adjacent to the check variable. The check value `0x1337` at `local_10` can be overwritten. This is a classic buffer overflow + stack canary bypass CTF challenge.

---

## P13: Use LLM to Analyze Decompiled Code

Take the decompiled output from P12 and ask an LLM:

```
PROMPT: "Here is the Ghidra decompilation of a CTF binary's main function.
Analyze it for:
1. The intended behavior (what does the program do?)
2. Security vulnerabilities
3. How to exploit it to get the 'Correct!' branch
4. Suggested patches to fix the vulnerabilities

[decompiled code above]"
```

**Expected output:**
```
(LLM identifies: classic stack buffer overflow, the magic check value 0x1337 at
local_10, suggests payload of 80 bytes + 0x1337 to overwrite and win)
```

> [!hint]- Hint
> The program reads a password with scanf into `local_50`. `local_10` is checked against `0x1337`. If we overflow 80 bytes (from local_50 start to local_10), we can overwrite local_10 with the magic value.

> [!success]- Solution
> The LLM correctly identifies the overflow distance: from `local_50` to `local_10`. The padding needed: 80 - 16 = 64 bytes of junk + `\x37\x13\x00\x00\x00\x00\x00\x00` (the little-endian 0x1337).

---

## AI Power-Up: Static Analysis

```
PROMPT: "I'm reversing a stripped binary. I found a function that does this
decompiled output shows. What is the function's purpose? Suggest a name:

[decompiled function]"

PROMPT: "Look at this objdump disassembly of an unknown function. I need to
understand: what inputs does it take, what does it return, and what does it do?
Work through the instructions step by step:

[disassembly]"
```
