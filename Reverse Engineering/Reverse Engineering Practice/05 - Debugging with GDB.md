---
title: "RE Practice: Debugging with GDB"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - gdb
  - debugging
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Debugging with GDB

12 progressive problems covering GDB, GEF/pwndbg, and runtime analysis.

---

## P1: Set Breakpoints and Inspect Registers

Compile and debug a simple program:

```bash
cat > /tmp/debug.c << 'EOF'
int square(int x) {
    return x * x;
}

int main() {
    int result = square(5);
    return result;
}
EOF
gcc -g -o /tmp/debug /tmp/debug.c
```

Start GDB and trace the execution:

```bash
gdb -q /tmp/debug
(gdb) break square
(gdb) run
(gdb) info registers
(gdb) print x = $rdi    # x86-64 first argument
(gdb) continue
(gdb) print $rax         # return value
```

**Expected output:**
```
Breakpoint 1 at square
Breakpoint 1, square (x=5) at 0x...
rax = 25 (5*5)
```

> [!hint]- Hint
> In x86-64 SysV, the first integer argument is in `$rdi`. The return value is in `$rax`.

> [!success]- Solution
> At the breakpoint inside `square`:
> - `$rdi = 5` (the argument)
> - After stepping through `imul edi, edi` (or `imul rdi, rdi`), `$rax = 25`

---

## P2: Step Through Code Instruction by Instruction

```bash
gdb -q /tmp/debug
(gdb) break main
(gdb) run
(gdb) disassemble
(gdb) nexti     # step one instruction
(gdb) nexti
(gdb) info registers rax rdi
```

What do you see in the register window as you step through?

**Expected output:**
```
Instruction pointer (rip) advances
rax changes from 5 to 25 after the imul instruction
```

> [!hint]- Hint
> `nexti` steps one machine instruction, not one source line. `stepi` follows into function calls.

> [!success]- Solution
> Each `nexti` executes one instruction. Watch `rip` increment, watch the multiplication happen in `rax`, see `ret` pop the return address.

---

## P3: Watch Memory Changes

```bash
gdb -q /tmp/debug
(gdb) break square
(gdb) run
(gdb) watch $rax
(gdb) continue
(gdb) info watchpoints
```

What triggers the watchpoint? What is the old and new value?

**Expected output:**
```
Hardware watchpoint 1: $rax
Old value = 5
New value = 25
```

> [!hint]- Hint
> A watchpoint triggers when the watched memory location changes. Since `$rax` holds the return value, it changes from the input (5) to the result (25).

> [!success]- Solution
> The watchpoint fires when `imul` writes the result (25) into `$rax`, replacing the input value (5). `info watchpoints` shows the triggered watchpoint with old/new values.

---

## P4: Backtrace (Stack Trace)

```bash
gdb -q /tmp/debug
(gdb) break square
(gdb) run
(gdb) backtrace
(gdb) frame 1
(gdb) info locals
```

What does the backtrace show? What is `main`'s stack frame?

**Expected output:**
```
#0  square (x=5) at debug.c:3
#1  0x... in main () at debug.c:7
```

> [!hint]- Hint
> Backtrace shows the call chain. Frame 0 is the current function (`square`). Frame 1 is the caller (`main`).

> [!success]- Solution
> - `backtrace` (or `bt`) shows: `#0 square`, `#1 main`
> - `frame 1` switches to main's context
> - `info locals` in main shows `result` variable with its address on the stack

---

## P5: Examine Stack Memory

```bash
gdb -q /tmp/debug
(gdb) break square
(gdb) run
(gdb) x/10x $rsp        # 10 hex words from stack pointer
(gdb) x/s $rdi          # examine argument as string
(gdb) x/20b $rsp        # 20 bytes from stack pointer
```

What do you see at the stack? Where is the return address?

**Expected output:**
```
Stack contains: return address (pointing into main's code), saved RBP, etc.
```

> [!hint]- Hint
> When `main` calls `square`, the stack has: `[return address]` (where to go after square returns), then `[saved RBP]` if frame pointer is used.

> [!success]- Solution
> The stack at the start of `square`:
> - `$rsp` → return address (address of the instruction after `call square` in main)
> - `$rsp+8` → saved RBP (if prologue pushes it)
> - `$rsp+16` → first local variable of main (if any)

---

## P6: Use GEF (GDB Enhanced Features)

```bash
# Install GEF
bash -c "$(curl -fsSL https://github.com/gef-osx/gef/raw/master/gef.sh)"

# Or pwndbg
# https://github.com/pwndbg/pwndbg

gdb -q /tmp/debug
gef➤  context         # full context display
gef➤  registers       # all registers with color coding
gef➤  xinfo $rsp      # memory info around stack
gef➤  got             # show GOT entries
```

Compare the GEF display to vanilla GDB. What extra information does GEF provide?

**Expected output:**
```
GEF shows: color-coded register state, stack canary detection, ASLR status, security checks (NX, PIE, etc.)
```

> [!hint]- Hint
> GEF adds context-aware highlighting, security feature detection, and smarter display of complex data structures.

> [!success]- Solution
> GEF provides: security checks (NX, PIE, stack canary, RELRO), color-coded registers (callee-saved vs caller-saved), better disassembly display, heap analysis, and ROP gadget finder.

---

## P7: Conditional Breakpoints

```bash
gdb -q /tmp/debug
(gdb) break square if $rdi > 10
(gdb) run     # won't stop since 5 is not > 10
# Expected: program exits without stopping at square
```

Create a version with multiple calls and test:

```bash
cat > /tmp/debug2.c << 'EOF'
int square(int x) { return x * x; }
int main() {
    int a = square(3);   // won't trigger (3 < 10)
    int b = square(5);   // won't trigger (5 < 10)
    int c = square(12);  // WILL trigger (12 > 10)
    return c;
}
EOF
gcc -g -o /tmp/debug2 /tmp/debug2.c
gdb -q /tmp/debug2
(gdb) break square if $rdi > 10
(gdb) run
(gdb) print $rdi    # should be 12
```

**Expected output:**
```
Breakpoint triggers only when x > 10 (at the call with x=12)
```

> [!hint]- Hint
> Conditional breakpoints only fire when the condition is true. This saves time compared to stepping through every iteration.

> [!success]- Solution
> The conditional `break square if $rdi > 10` causes GDB to only stop when `square(12)` is called, skipping the first two calls entirely.

---

## P8: Disassemble with Mixed Source

```bash
gcc -g -o /tmp/debug /tmp/debug.c
gdb -q /tmp/debug
(gdb) disas main
(gdb) disassemble /m square   # /m = mixed source + assembly
```

What does mixed source/assembly show? How does it relate source lines to machine code?

**Expected output:**
```
Each source line is followed by its corresponding assembly instructions
```

> [!hint]- Hint
> The `/m` modifier to `disassemble` interleaves source code lines with their corresponding assembly instructions when debug info is available.

> [!success]- Solution
> `disassemble /m` shows:
> ```
> 3:	    return x * x;
>    0x0000000000401129 <+0>:	mov	eax, edi
>    0x000000000040112b <+2>:	imul	eax, edi
> 4:	}
>    0x000000000040112e <+5>:	ret
> ```

---

## P9: Attach to a Running Process

```bash
# Terminal 1: run the program in a loop
./tmp/debug &

# Terminal 2: attach GDB to it
gdb -q -p $(pidof /tmp/debug)
(gdb) break square
(gdb) continue
```

How do you attach GDB to an already-running process? What are the tradeoffs vs starting fresh?

**Expected output:**
```
gdb -p PID attaches to running process
Program is already executing, breakpoints take effect immediately
```

> [!hint]- Hint
> `gdb -p $(pidof binary)` or `gdb` then `attach PID`. The process must be your own (or you need proper permissions).

> [!success]- Solution
> - `gdb -p PID` attaches directly
> - Or `gdb` → `attach PID`
> - `detach` to stop debugging and let it run freely
> - Tradeoff: can't control the initial startup, but useful for analyzing loops/server processes

---

## P10: Use GDB Python API for Automation

Write a GDB Python script to automate analysis:

```python
import gdb

gdb.execute("file /tmp/debug")
gdb.execute("break square")
gdb.execute("run")

# Read registers at breakpoint
frame = gdb.newest_frame()
print(f"Function: {frame.name()}")
print(f"Argument (rdi): {frame.read_register('rdi')}")

# Step one instruction
gdb.execute("nexti")
print(f"Return value (rax): {frame.read_register('rax')}")
```

```bash
gdb -q -ex "source script.py" /tmp/debug
```

**Expected output:**
```
Function: square
Argument (rdi): 5
Return value (rax): 25
```

> [!hint]- Hint
> GDB's Python API lets you script complex analysis, automate repetitive tasks, and extract information programmatically.

> [!success]- Solution
> The script breaks at `square`, reads the argument from `$rdi` (5), steps one instruction, reads the result from `$rax` (25). This demonstrates programmatic GDB control for RE automation.

---

## P11: Debug ARM Binary with QEMU

Debug an ARM64 binary compiled for a different architecture:

```bash
# Compile for ARM64
aarch64-linux-gnu-gcc -o /tmp/debug_arm /tmp/debug.c

# Run with QEMU in debug mode (waits for GDB)
qemu-arm -g 1234 /tmp/debug_arm &

# In another terminal, connect GDB
gdb-multiarch /tmp/debug_arm
(gdb) target remote localhost:1234
(gdb) break square
(gdb) continue
(gdb) info registers
```

**Expected output:**
```
Registers shown in ARM format (X0-X30, SP, PC)
Same debugging concepts apply, different register names
```

> [!hint]- Hint
> QEMU can run ARM binaries on x86 and exposes a GDB stub on a TCP port. `gdb-multiarch` supports multiple architectures.

> [!success]- Solution
> The GDB session shows ARM registers (X0 for first argument instead of RDI, X30 for LR instead of RA, etc.). The debugging workflow is identical — breakpoints, stepping, memory inspection all work the same way.

---

## P12: Use LLM to Interpret GDB State

After debugging a complex function, ask an LLM to interpret the state:

```
PROMPT: "I'm debugging a binary with GDB. Right before a crash, I see this state:

Registers:
rax = 0x0000000000000000
rbx = 0x00007fffffffd8e0
rcx = 0x0000000000000000
rsp = 0x00007fffffffd890
rbp = 0x00007fffffffd8c0
rip = 0x0000000000401187

Stack dump (rsp to rsp+0x40):
0x7fffffffd890: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffd8a0: 0x00000000  0x00000000  0x41414141  0x41414141
0x7fffffffd8b0: 0x41414141  0x41414141  0x41414141  0x41414141

What happened? Is this a buffer overflow?"
```

**Expected output:**
```
(LLM identifies: stack buffer overflow with 0x41 (ASCII 'A') filling the stack.
The return address area is overwritten with A's. This is a classic stack smashing exploit attempt.)
```

> [!hint]- Hint
> The pattern `0x41414141` is highly suspicious — 0x41 is the hex value of ASCII 'A'. Multiple consecutive 0x41414141 values indicate overflow.

> [!success]- Solution
> The LLM correctly identifies: buffer overflow detected — the value 0x41414141 (ASCII 'A') has overwritten the stack, including potentially the return address. The `rip` value of `0x00401187` is inside the binary, but if the overflow had reached the return address on the stack, control flow could have been redirected.

---

## AI Power-Up: Debugging

```
PROMPT: "I'm debugging a program that crashes with a segfault. Here is the GDB
output right at the crash. What is the likely cause?

(gdb) bt
#0  0x0000000000401187 in ?? ()
#1  0x4141414141414141 in ?? ()
#2  0x4141414141414141 in ?? ()
Backtrace stopped: frame did not save stack"

PROMPT: "I need to find where a specific global variable is being written to.
What GDB commands would I use to set a watchpoint on a specific memory address?
Give me the exact commands."
```
