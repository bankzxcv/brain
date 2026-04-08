---
title: "RE Practice: x86 and ARM Assembly"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - assembly
  - x86
  - arm
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: x86 and ARM Assembly

14 progressive problems. Write, read, and trace assembly in both x86-64 and ARM64.

---

## P1: x86 — Basic MOV and Arithmetic

What does this x86-64 assembly do? Trace the register values step by step.

```asm
mov rax, 5
mov rbx, 3
add rax, rbx
sub rax, 1
imul rax, rax
```

**Expected output:**
```
After mov rax, 5:   rax=5, rbx=3
After add rax, rbx: rax=8, rbx=3
After sub rax, 1:   rax=7, rbx=3
After imul rax, rax: rax=49, rbx=3
```

> [!hint]- Hint
> `mov` copies a value. `add dest, src` adds src to dest. `sub dest, src` subtracts. `imul dest, src` multiplies dest by src (result in dest).

> [!success]- Solution
> The sequence: load 5, load 3, add (5+3=8), subtract 1 (8-1=7), multiply by self (7*7=49). Final `rax=49`.

---

## P2: x86 — Conditional Jump

Trace through this code. What is the final value of `rax`?

```asm
mov rax, 10
cmp rax, 5
jg skip
mov rax, 99
skip:
mov rbx, 1
```

**Expected output:**
```
rax = 10 (the jump is taken, so 99 is never assigned)
```

> [!hint]- Hint
> `cmp a, b` sets flags based on `a - b`. `jg` (jump if greater) is taken when the signed comparison shows a > b. Since 10 > 5, the jump to `skip` is taken.

> [!success]- Solution
> `cmp rax, 5` compares 10 and 5. `jg` checks if 10 > 5 → true, so jump to `skip` is taken. The `mov rax, 99` is skipped. Final `rax=10`.

---

## P3: x86 — Function Call (cdecl-style, simplified)

Trace this code and determine the final values:

```asm
mov rcx, 4
mov rax, 1
call double
add rax, 3
; (where double is:)
; double:
;   add rax, rax
;   ret

; (continuing after call)
add rax, rcx
```

**Expected output:**
```
After call: rax = 2 (doubled), rcx = 4
After first add: rax = 5
After second add: rax = 9
```

> [!hint]- Hint
> `call` pushes the return address. `ret` pops it and jumps back. `add rax, rax` doubles rax. The result of the call is added to rcx after returning.

> [!success]- Solution
> - Load rcx=4, rax=1
> - `call` saves return address, jumps to `double`
> - `double`: `add rax, rax` → rax=2, then `ret`
> - Return to `add rax, 3` → rax=5
> - `add rax, rcx` → rax=5+4=9

---

## P4: x86 — Stack Operations

What values are on the stack after these operations?

```asm
push 0xDEAD
push 0xBEEF
push 0xCAFE
pop rax
pop rbx
pop rcx
```

**Expected output:**
```
Stack (high to low):
  After pushes: [DEAD] [BEEF] [CAFE] (rsp pointing to CAFE)
  After pops:   rax=CAFE, rbx=BEEF, rcx=DEAD
```

> [!hint]- Hint
> Push decrements rsp and writes. Pop reads and increments rsp. Stack is LIFO (Last In, First Out).

> [!success]- Solution
> Push sequence: `rsp -= 8; [rsp] = 0xDEAD` (rsp→DEAD), then BEEF (rsp→BEEF), then CAFE (rsp→CAFE).
> Pop: rax=CAFE (top), rbx=BEEF, rcx=DEAD. Stack is now empty (rsp restored).

---

## P5: x86 — Addressing Modes

Given this memory layout (all values are 8-byte qwords):

```
Address:  Value:
0x1000:   0x11
0x1008:   0x22
0x1010:   0x33
0x1018:   0x44
```

What does each instruction produce?

```asm
mov rax, [0x1000]
mov rbx, [rsp + 8]
lea rcx, [rdx + rax * 4]
```

**Expected output:**
```
mov rax, [0x1000]  ; rax = 0x11
mov rbx, [rsp + 8] ; rbx = value at rsp+8
lea rcx, [rdx + rax*4] ; rcx = rdx + rax*4 (address, not value)
```

> [!hint]- Hint
> `mov reg, [addr]` loads the value at that address. `lea reg, [expr]` computes the address expression and puts the address (not the value) into the register.

> [!success]- Solution
> - `mov rax, [0x1000]` → loads value at address 0x1000 = 0x11
> - `mov rbx, [rsp + 8]` → loads value at address `rsp + 8`
> - `lea rcx, [rdx + rax*4]` → computes address `rdx + rax*4` (no memory access, just arithmetic)

---

## P6: ARM64 — Basic MOV and Arithmetic

What does this ARM64 code do?

```asm
MOV X0, #10
MOV X1, #3
ADD X2, X0, X1    ; X2 = X0 + X1
SUB X3, X2, #1    ; X3 = X2 - 1
MUL X4, X3, X3    ; X4 = X3 * X3
```

**Expected output:**
```
X0=10, X1=3
X2=13  (10+3)
X3=12  (13-1)
X4=144 (12*12)
```

> [!hint]- Hint
> ARM64 uses X registers (64-bit). `ADD Xd, Xn, Xm` computes Xd = Xn + Xm. Immediate values use `#`.

> [!success]- Solution
> X2 = 10 + 3 = 13, X3 = 13 - 1 = 12, X4 = 12 * 12 = 144.

---

## P7: ARM64 — Conditional Branch

```asm
MOV X0, #5
CMP X0, #10
B.EQ equal       ; branch if equal (Z flag set)
MOV X1, #99      ; this runs if NOT equal
B done
equal:
MOV X1, #42
done:
MOV X0, X1
```

**Expected output:**
```
X0 = 99 (the BEQ was NOT taken because 5 != 10)
```

> [!hint]- Hint
> `CMP` sets flags. `B.EQ` branches only if Z=1 (equal/zero). 5 compared to 10 (5-10=-5) does NOT set Z, so the branch is NOT taken.

> [!success]- Solution
> X0=5. CMP 5 vs 10: 5-10 = -5 (non-zero, Z=0). B.EQ not taken. MOV X1, #99 runs. Final X0=X1=99.

---

## P8: ARM64 — Function Call

```asm
; Caller
MOV X0, #4
BL double        ; X30 = return addr, jump to double
ADD X2, X0, X1  ; X2 = return value + X1

double:
LSL X0, X0, #1  ; X0 = X0 * 2 (left shift by 1 = multiply by 2)
RET              ; BR X30
```

**Expected output:**
```
X0 initial = 4
After double: X0 = 8 (doubled)
After add: X2 = 8 + X1 (X1 is undefined/whatever was there)
```

> [!hint]- Hint
> `BL` (Branch and Link) saves the next instruction address into X30 (LR) and jumps. `RET` is `BR X30` — jumps back to saved address.

> [!success]- Solution
> BL stores return addr in X30, jumps to double. double: `LSL X0, X0, #1` → X0=8. `RET` → `BR X30`, returns to caller. Caller: `ADD X2, X0, X1` → X2 = 8 + (whatever X1 is).

---

## P9: Compare x86-64 SysV vs Windows x64 Calling Conventions

You see a function call with these arguments:
```
x86-64 SysV:  rdi=0x10, rsi=0x20, rdx=0x30, rcx=0x40, r8=0x50, r9=0x60
Windows x64:  rcx=0x10, rdx=0x20, r8=0x30, r9=0x40, stack=0x50, 0x60
```

For each convention, which argument is in which register (1st through 6th)?

**Expected output:**
```
SysV (Linux/macOS):     1st=rdi(0x10), 2nd=rsi(0x20), 3rd=rdx(0x30), 4th=rcx(0x40), 5th=r8(0x50), 6th=r9(0x60)
Windows x64:            1st=rcx(0x10), 2nd=rdx(0x20), 3rd=r8(0x30), 4th=r9(0x40), 5th=stack(0x50), 6th=stack(0x60)
```

> [!hint]- Hint
> x86-64 SysV uses `rdi, rsi, rdx, rcx, r8, r9`. Windows x64 uses `rcx, rdx, r8, r9` then stack.

> [!success]- Solution
> Both are shown above. The key difference: SysV uses `rdi` first, Windows uses `rcx` first. This matters when reverse engineering calling conventions.

---

## P10: Write x86 NASM — Hello World

Write a complete NASM assembly program that prints "Hello, x86!" to stdout using the `write` syscall, then exits cleanly.

```bash
nasm -f elf64 hello.asm -o hello.o
ld hello.o -o hello
./hello
```

**Expected output:**
```
Hello, x86!
```

> [!hint]- Hint
> Linux syscall 1 = write (fd=1 for stdout, buffer in rsi, count in rdx). Syscall 60 = exit (code in rdi).

> [!success]- Solution
> ```nasm
> section .text
> global _start
> _start:
>     mov rax, 1        ; write syscall
>     mov rdi, 1        ; stdout
>     mov rsi, msg      ; buffer address
>     mov rdx, len      ; byte count
>     syscall
>     mov rax, 60       ; exit syscall
>     xor rdi, rdi      ; exit code 0
>     syscall
> section .data
>     msg db 'Hello, x86!', 0x0A
>     len equ $ - msg
> ```

---

## P11: Cross-Compile for ARM64

Write a simple C program, compile it for ARM64, and analyze the resulting binary:

```c
// hello_arm.c
#include <stdio.h>
int main() {
    printf("Hello, ARM!\n");
    return 0;
}
```

```bash
# Compile for ARM64 (install gcc-aarch64-linux-gnu first)
aarch64-linux-gnu-gcc -o hello_arm hello_arm.c

# Analyze
file hello_arm
readelf -A hello_arm | head -20
objdump -d hello_arm | head -40
```

**Expected output:**
```
hello_arm: ELF 64-bit LSB executable, ARM aarch64
(You should see ARM-specific instructions: LDR, STR, MOVZ, MOVK, B, BL)
```

> [!hint]- Hint
> ARM64 instructions look very different from x86. Look for `LDR X0, [X1]`, `MOVZ W0, #`, `BL printf`.

> [!success]- Solution
> The output binary is ARM aarch64. The `file` command confirms architecture. `objdump -d` shows ARM opcodes — note the 4-byte fixed instruction length vs x86's variable length.

---

## P12: Read Disassembly of a Real Function

Compile and disassemble this C function:

```c
int square_sum(int a, int b) {
    int sum = a + b;
    return sum * sum;
}
```

```bash
gcc -O0 -c -o test.o test.c   # no optimization
objdump -M intel -d test.o
```

**Expected output:**
```
Shows: push rbp, mov rbp, rsp, then:
  mov eax, edi      ; a goes to eax (first int arg)
  add eax, esi      ; add b (second int arg)
  imul eax, eax     ; square it
  pop rbp
  ret
```

> [!hint]- Hint
> With -O0 (no optimization), the compiler generates very literal code matching the stack-based calling convention. With -O2, it would optimize away the intermediate variable.

> [!success]- Solution
> With -O0: `square_sum`:
> ```
> 0:  55                   push   rbp
> 1:  48 89 e5             mov    rbp,rsp
> 4:  89 7d fc             mov    DWORD PTR [rbp-4],edi   ; a
> 7:  89 75 f8             mov    DWORD PTR [rbp-8],esi   ; b
> a:  8b 45 fc             mov    eax,DWORD PTR [rbp-4]
> d:  01 45 f8             add    eax,DWORD PTR [rbp-8]   ; sum
> 10: 0f af 45 f8          imul   eax,DWORD PTR [rbp-8]  ; sum*sum
> 14: 5d                   pop    rbp
> 15: c3                   ret
> ```

---

## P13: Use LLM to Explain Unknown Assembly

Take the following x86-64 disassembly of an unknown function and ask an LLM to explain it:

```
PROMPT: "Explain this x86-64 assembly function instruction by instruction.
What does it do overall? What is the calling convention? Suggest a function name:

0:  push   rbp
1:  mov    rbp, rsp
4:  sub    rsp, 0x20
7:  cmp    DWORD PTR [rdi], 0
c:  je     1a
e:  mov    rax, QWORD PTR [rdi+0x8]
12: mov    QWORD PTR [rbp-0x18], rax
16: call   rax
1a: nop
1b: leave
1c: ret"
```

**Expected output:**
```
(LLM explains: prologue, comparing a DWORD at [rdi] against 0, if zero jump to leave,
otherwise load function pointer from [rdi+8] and call it. Likely a vtable or
function pointer dispatch pattern. Suggested name: something like "invoke_or_nil"
or "dispatch_if_not_null")
```

> [!hint]- Hint
> `[rdi]` with a DWORD compare suggests checking if a pointer's first field (maybe a flag/enum) is non-zero before calling through a function pointer at `[rdi+8]`.

> [!success]- Solution
> Example analysis: The function takes a pointer in `rdi`. It checks if the first dword (int field) at `[rdi]` is zero. If NOT zero, it loads a function pointer from `[rdi+8]` and calls it. This is a classic vtable or callback invocation pattern — possibly part of an interface dispatch or optional callback wrapper.

---

## P14: Reverse ARM64 Back to C

The following ARM64 code was extracted from a binary. Reconstruct the likely C source:

```asm
0:  SUB SP, SP, #0x10
4:  STR W0, [SP, #0xC]     ; store arg1 at [sp+12]
8:  LDR W1, [SP, #0xC]     ; load it back
c:  ADD W0, W1, W1         ; W0 = W1 + W1
10: ADD W0, W0, #1         ; W0 = W0 + 1
14: ADD SP, SP, #0x10
18: RET
```

**Expected output:**
```c
int triple_and_increment(int x) {
    return x + x + 1;  // or: 2*x + 1
}
```

> [!hint]- Hint
> W0 is the first argument (and return). W1 is used temporarily. The function doubles the input and adds 1.

> [!success]- Solution
> Likely C:
> ```c
> int func(int x) {
>     int temp = x;      // stored at [sp+12] (but same value)
>     int result = x + x; // ADD W0, W1, W1
>     result = result + 1; // ADD W0, W0, #1
>     return result;
> }
> // Or simply: return 2*x + 1;
> ```

---

## AI Power-Up: Assembly

```
PROMPT: "Translate this ARM64 assembly to equivalent x86-64 assembly.
Note any calling convention differences and register mappings:

[ARM64 assembly]"

PROMPT: "Given this assembly code from a CTF binary, what C source code
might have generated it? Be specific about data types:

[assembly code]"

PROMPT: "I'm looking at a function that takes two arguments. Here is the
disassembly. Which register holds which argument, and what is the return
value type?
[disassembly]"
```
