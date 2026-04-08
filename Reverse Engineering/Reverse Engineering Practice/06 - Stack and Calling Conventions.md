---
title: "RE Practice: Stack and Calling Conventions"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - stack
  - calling-conventions
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Stack and Calling Conventions

12 progressive problems tracing stack frames, argument passing, and return addresses.

---

## P1: Trace Stack Frame Setup

Compile and trace the stack frame of a simple function:

```bash
cat > /tmp/frame.c << 'EOF'
int add(int a, int b) {
    int result = a + b;
    return result;
}

int main() {
    return add(3, 4);
}
EOF
gcc -O0 -o /tmp/frame /tmp/frame.c
gdb -q /tmp/frame << 'GDB'
break add
disassemble add
info registers
x/8gx $rsp
GDB
```

Draw the stack layout at the entry of `add`.

**Expected output:**
```
Stack at add entry:
[rsp]   = return address
[rsp+8] = saved RBP (if frame pointer used)
[rsp+16] = local result variable
[rsp+24] = padding (if needed)
```

> [!hint]- Hint
> With -O0, `add` pushes RBP, sets up frame, allocates space for local variables. Arguments arrive in registers (rdi, rsi in x86-64 SysV), not on the stack.

> [!success]- Solution
> At `add` entry:
> - `$rdi = 3` (arg1), `$rsi = 4` (arg2)
> - Stack: return address (from `call`), then saved RBP
> - No local variables needed (result can stay in `eax`)
> - Prologue: `push rbp; mov rbp,rsp`

---

## P2: Identify Calling Convention from Disassembly

You see this disassembly. Is it x86-64 SysV or Windows x64?

```asm
mov     edi, 1
mov     esi, 2
mov     edx, 3
mov     ecx, 4
call    func
```

**Expected output:**
```
Windows x64 (first 4 args in rcx, rdx, r8, r9)
Wait, this is AT&T syntax! Let me check...
Actually: this is using edi, esi, edx, ecx = x86-64 SysV convention
```

> [!hint]- Hint
> x86-64 SysV: `rdi, rsi, rdx, rcx, r8, r9` for arguments 1-6
> Windows x64: `rcx, rdx, r8, r9` for arguments 1-4

> [!success]- Solution
> This is **x86-64 SysV** (Linux/macOS). The arguments are in `edi, esi, edx, ecx` — exactly matching the SysV register order. On Windows x64, the first 4 args would be in `rcx, rdx, r8, r9`.

---

## P3: Stack After a Function Call

What is on the stack immediately after `call func` pushes the return address?

```asm
; main function
push   rbp
mov    rbp, rsp
sub    rsp, 0x10
mov    edi, 10       ; arg1 = 10
call   func
add    rsp, 0x10
; func:
func:
push   rbp
mov    rbp, rsp
sub    rsp, 0x20
mov    [rbp-0x8], edi ; store arg1 on stack
```

**Expected output:**
```
Stack (high to low):
rbp (main's) <- rbp points here
local vars
...
return address <- pushed by CALL
rbp (func's saved)
local vars (func)
```

> [!hint]- Hint
> The CALL instruction automatically pushes the return address before jumping. The PUSH in func's prologue then saves the caller's RBP.

> [!success]- Solution
> Stack after `call func` but before func's prologue:
> - `[rsp]` = return address (address of the `add rsp, 0x10` instruction)
> - Then func's `push rbp` saves main's RBP
> - Then func's `sub rsp, 0x20` allocates space

---

## P4: Analyze Frame Pointer Omission

Compare a normal frame vs an optimized frame:

```bash
# Normal (with frame pointer)
gcc -O0 -o /tmp/nofp /tmp/frame.c
objdump -M intel -d /tmp/nofp | grep -A 15 '<add>:'

# Optimized (no frame pointer)
gcc -O2 -fomit-frame-pointer -o /tmp/yesfp /tmp/frame.c
objdump -M intel -d /tmp/yesfp | grep -A 15 '<add>:'
```

What difference do you see in the prologues?

**Expected output:**
```
-O0: push rbp; mov rbp, rsp; [sub rsp, N]
-O2: just sub rsp, N (no push rbp, no mov rbp, rsp)
```

> [!hint]- Hint
> `-fomit-frame-pointer` is enabled at -O2. RBP becomes a general-purpose register, saving instructions but making stack traces harder.

> [!success]- Solution
> - `-O0`: `push rbp; mov rbp, rsp; sub rsp, N` — 3 instructions, RBP saved
> - `-O2`: `sub rsp, N` only — 1 instruction, RBP is free for other use
> - Stack frame is tracked via RSP offsets in optimized code

---

## P5: Callee-Saved vs Caller-Saved Registers

You are analyzing a function and see:
- `rbx` is saved at the start and restored at the end
- `rax` changes throughout without being saved

What does this tell you about which registers are callee-saved vs caller-saved?

**Expected output:**
```
rbx = callee-saved (preserved across calls)
rax = caller-saved (can be clobbered by called functions)
```

> [!hint]- Hint
> Callee-saved registers must be preserved by the callee (function being called). Caller-saved registers can be freely used by the callee without saving.

> [!success]- Solution
> - If a function saves/restores `rbx`, it's using it and must preserve the caller's value → `rbx` is callee-saved
> - `rax` changing freely → it's caller-saved (the caller expects it to be modified by called functions)
> - x86-64 SysV callee-saved: `rbx, rbp, r12, r13, r14, r15`

---

## P6: Variadic Functions (x86 cdecl)

The `printf` function is variadic. How do variadic functions know how many arguments were passed?

```c
// C declaration
int printf(const char *fmt, ...);
```

In assembly (cdecl, 32-bit):

```asm
push [ebp+16]   ; 3rd arg (variadic)
push [ebp+12]   ; 2nd arg
push [ebp+8]    ; 1st arg (format string)
call printf
add esp, 24     ; caller cleans 3 args
```

How does `printf` know there are 3 variadic arguments?

**Expected output:**
```
printf doesn't know — it uses the format string (%d, %s, etc.) to parse the stack!
```

> [!hint]- Hint
> Variadic functions have no type information at compile time. They rely on the format string (or other conventions) to know how to read the stack.

> [!success]- Solution
> `printf` reads the format string at `[ebp+8]`. For each `%` specifier, it reads the next argument from the stack at progressively higher offsets. The caller is responsible for cleaning up the stack (not the callee, which is why cdecl is used for varargs).

---

## P7: Windows x64 Shadow Space

On Windows x64, the caller must allocate "shadow space." Demonstrate:

```asm
; Caller preparing to call Windows API
sub rsp, 28h       ; allocate shadow space (4*8 = 32 bytes) + possible register saves
mov rcx, arg1      ; shadow: rcx = 1st arg
mov rdx, arg2      ; shadow: rdx = 2nd arg
mov r8, arg3       ; shadow: r8 = 3rd arg
mov r9, arg4       ; shadow: r9 = 4th arg
call func
add rsp, 28h       ; caller cleans up
```

Why is shadow space needed?

**Expected output:**
```
Shadow space gives the callee space to spill the first 4 arguments.
It also ensures the stack is 16-byte aligned before the call (required by Windows x64 ABI).
```

> [!hint]- Hint
> Windows x64 ABI requires 16-byte stack alignment before `call`. The shadow space + return address pushed by `call` ensures this.

> [!success]- Solution
> - The 32-byte shadow space (4 registers × 8 bytes) allows the called function to save its incoming arguments if needed
> - Combined with the 8-byte return address pushed by `call`, the stack is 16-byte aligned before the callee executes its first instruction
> - This is a Windows x64 requirement, different from SysV

---

## P8: ARM64 Stack Frame

Trace the ARM64 stack frame:

```asm
; Function prologue
stp x29, x30, [sp, #-0x10]!  ; save FP and LR (pre-index decrement)
mov x29, sp                     ; set frame pointer
sub sp, sp, #0x20              ; allocate 32 bytes local

; Function epilogue
add sp, sp, #0x20              ; deallocate locals
ldp x29, x30, [sp], #0x10      ; restore FP and LR (post-index)
ret                             ; branch to LR
```

What is the stack layout?

**Expected output:**
```
High:    [saved FP] [saved LR] <- X29 and X30
         [local vars]
SP ->    [new frame...]
```

> [!hint]- Hint
> ARM64 uses `STP` (store pair) to save both FP (x29) and LR (x30) atomically. The `!` suffix means pre-index writeback.

> [!success]- Solution
> - `stp x29, x30, [sp, #-0x10]!` — first decrements sp by 16, then stores x29 and x30 (at new sp)
> - This saves both the frame pointer and link register (return address) in one instruction
> - The stack grows downward, just like x86
> - `ldp` (load pair) restores in reverse order with post-index

---

## P9: Return Address Location

At what exact stack offset is the return address in x86-64 SysV?

```bash
gdb -q /tmp/frame << 'GDB'
break add
run
info registers rsp
x/2gx $rsp
GDB
```

**Expected output:**
```
Return address is at [rsp] (the very top of the stack, 8 bytes from current RSP)
```

> [!hint]- Hint
> `call` pushes the return address, decrementing rsp by 8. At the first instruction of the called function, the return address is at `[rsp]` (top of stack).

> [!success]- Solution
> The return address sits at `$rsp` (0 bytes offset from stack pointer). It's the first thing on the stack after a `call`. If you `pop` it, you'll get the return address and increment rsp.

---

## P10: Security Implication — Stack Smash

What would happen if a function's buffer overflow overwrites the return address?

```c
void vulnerable(char *input) {
    char buf[64];
    strcpy(buf, input);  // no bounds check!
    // return address is at buf[72] through buf[79]
}
```

An attacker passes 80 'A' characters followed by `0x0000000000401234`.

What happens on `ret`?

**Expected output:**
```
On ret: pop return address into RIP -> jumps to 0x00401234
If that's attacker-controlled code, code execution is achieved
```

> [!hint]- Hint
> The return address sits 8 bytes above the saved RBP, which is 8 bytes above the buffer. Total offset from buffer start to return address = 64 (buffer) + 8 (saved RBP) = 72 bytes. So byte 73-80 (8 bytes) overwrite the return address.

> [!success]- Solution
> - Buffer at `rbp-0x40` (64 bytes)
> - Saved RBP at `rbp-0x38` (8 bytes above buffer)
> - Return address at `rbp-0x30` (8 bytes above saved RBP)
> - Overflow: bytes 64-71 overwrite saved RBP, bytes 72-79 overwrite return address
> - On `ret`, `pop rsp` → jump to overwritten address → control flow hijacked

---

## P11: Function Pointer Call via Stack

What does this assembly do?

```asm
mov rax, [rbp-0x8]    ; load function pointer from local variable
mov rdi, 42            ; first argument
call rax               ; call through function pointer
mov [rbp-0x10], eax    ; store return value
```

What C code might this correspond to?

**Expected output:**
```c
int (*func_ptr)(int);  // function pointer variable
func_ptr = /* something at [rbp-0x8] */;
int result = func_ptr(42);
```

> [!hint]- Hint
> Loading a value from a local variable and calling it is the signature of a function pointer invocation.

> [!success]- Solution
> This is a classic function pointer call pattern:
> ```c
> int (*callback)(int);  // stored at rbp-0x8
> callback = get_callback();
> int result = callback(42);  // arg goes in rdi, return in eax
> // result stored at rbp-0x10
> ```

---

## P12: Use LLM to Analyze Stack State

```
PROMPT: "I'm debugging a binary and hit a breakpoint. Here is the GDB state:

(gdb) info registers
rax = 0x0, rbx = 0x0, rcx = 0x0
rdx = 0x7ffff7dd5a58, rsi = 0x7ffff7dd5a60
rdi = 0x1
rbp = 0x7fffffffd8c0, rsp = 0x7fffffffd890
rip = 0x0000000000401187

(gdb) x/8gx $rsp
0x7fffffffd890: 0x00000000  0x00007fff  0x41414141  0x41414141
0x7fffffffd8a0: 0x41414141  0x00000000  0x00000000  0x00000000

What is the stack layout? Which value is the return address? Is there an overflow?"
```

**Expected output:**
```
(LLM identifies: return address is at rsp (0x7fffffffd890) = 0x00000000
Buffer overflow from A's filling the saved RBP and potentially the return address.
The first A starts at rsp+0x10 in this case.)
```

> [!hint]- Hint
> The return address is at `$rsp` (offset 0). 0x00000000 there is the actual return address. The 0x41414141 values are in adjacent stack locations, indicating overflow.

> [!success]- Solution
> The LLM correctly identifies the layout. Return address at rsp is still 0 (null or libc), but the adjacent 0x41414141 values indicate an overflow that could have corrupted it if the buffer were larger. This is a stack canary failure scenario.

---

## AI Power-Up: Calling Conventions

```
PROMPT: "I see this function prologue in a Windows x64 binary:
sub rsp, 38h
mov qword ptr [rsp+20h], rcx
mov qword ptr [rsp+28h], rdx

What is happening here? Which arguments are being saved and why?"

PROMPT: "I have two binaries, one compiled for Linux (SysV) and one for Windows.
They call the same C function with the same arguments. How will the assembly
differ at the call site?"
```
