---
title: "Reverse Engineering Study"
date: 2026-04-07
tags:
  - reverse-engineering
  - security
  - study
  - course
aliases:
  - RE Study
  - Reverse Engineering Course
status: in-progress
---

# Reverse Engineering Study

> [!tip] What is Reverse Engineering?
> Reverse engineering is the process of analyzing a compiled/binary system to understand how it works, when source code is unavailable. You disassemble, debug, trace, and reconstruct behavior from the outside in.

> [!abstract] Course Structure
> **Reference** - This note covers all core concepts with deep explanations, tool guides, and gotchas.
> **Practice** - 13 topic drills with 12-15 progressive problems each. See [[#Practice Drills]].
> **Workflows** - 5 real-world projects combining all concepts. See [[#Workflow Projects]].
> **AI-Assisted RE** - LLM/OpenCode/MCP integration woven into every topic and covered deeply in Practice 13.

---

## Table of Contents

### Practice Drills
1. [[#Binary Fundamentals]]
2. [[#x86 and ARM Assembly]]
3. [[#ELF and PE Formats]]
4. [[#Disassembly and Static Analysis]]
5. [[#Debugging with GDB]]
6. [[#Stack and Calling Conventions]]
7. [[#Ghidra Deep Dive]]
8. [[#Dynamic Analysis]]
9. [[#Patching and Binary Modification]]
10. [[#Anti-Reverse Engineering]]
11. [[#Protocol Reverse Engineering]]
12. [[#Obfuscation and Deobfuscation]]
13. [[#AI-Assisted Reverse Engineering]]

### Workflow Projects
1. [[Reverse Engineering/Reverse Engineering Workflows/01 - Reverse a CLI Utility|Rever]] - Reverse a CLI Utility
2. [[Reverse Engineering/Reverse Engineering Workflows/02 - Analyze a Network Protocol|Analyze a Network Protocol]]
3. [[Reverse Engineering/Reverse Engineering Workflows/03 - Patch a License Check|Patch a License Check]]
4. [[Reverse Engineering/Reverse Engineering Workflows/04 - Deobfuscate a Malware Sample|Deobfuscate a Malware Sample]]
5. [[Reverse Engineering/Reverse Engineering Workflows/05 - Reconstruct Source from Stripped Binary|Reconstruct Source from Stripped Binary]]

---

## 1. Binary Fundamentals

### Hex, Endianness, and Number Systems

Computers store everything as bytes — 8-bit sequences representable as two hexadecimal digits (0x00–0xFF).

```
Decimal: 0    1    10   100   255
Hex:     0x00 0x01 0x0A 0x64  0xFF
Binary:  0000 0001 1010 0110 1111 1111
```

**Big Endian** — Most significant byte first. Network byte order is big endian.
**Little Endian** — Least significant byte first. x86/x64 and ARM (usually) use this.

```
0x01234567 in memory (address 0x1000):
  Big Endian:    [01][23][45][67]  at 0x1000,0x1001,0x1002,0x1003
  Little Endian: [67][45][23][01]  at 0x1000,0x1001,0x1002,0x1003
```

### ASCII and Strings

```c
// ASCII table (key values)
'A' = 0x41 = 65  'Z' = 0x5A = 90
'a' = 0x61 = 97  'z' = 0x7A = 122
'0' = 0x30 = 48  '9' = 0x39 = 57
'\0' = 0x00 = 0  (null terminator)
'\n' = 0x0A = 10  (newline)
```

Strings in memory are null-terminated (`\0`) in C. Pascal strings use a length prefix.

### File Formats

**Binary files** are just byte sequences. The same bytes can be interpreted as:
- An executable (ELF/PE/Mach-O)
- An image (JPEG starts with `0xFF 0xD8`)
- A sound file
- Raw protocol packets

Magic bytes identify file types:

```
ELF:        0x7F 'E' 'L' 'F'
PE:         'M' 'Z' (DOS header) then 'PE\0\0'
JPEG:       0xFF 0xD8
PNG:        0x89 'P' 'N' 'G'
ZIP:        'P' 'K' (PK)
Mach-O:     0xFE 0xED 0xFA 0xCE (64-bit)
```

### Useful Commands

```bash
# View file type
file binary_name

# Hex dump
xxd binary_name            # default hex+ASCII
xxd -b binary_name        # binary
xxd -g 1 binary_name       # group by 1 byte
xxd -s 0x100 binary_name  # skip to offset 0x100

# Hex dump with less
hexdump -C binary_name    # canonical hex+ASCII
hexdump -d binary_name    # decimal
od -A x -t x1z binary_name # octal dump with hex

# Strings extraction
strings binary_name               # all strings
strings -n 8 binary_name          # min 8 chars
strings -e l binary_name          # 16-bit little-endian (often used in Windows PE)
strings -e b binary_name          # 16-bit big-endian

# Entropy analysis (high entropy = likely compressed/encrypted)
# See: https://github.com/jreisinger/checkov
```

### AI Power-Up: Binary Fundamentals

```
PROMPT: "Here is the hex dump of an unknown binary file. Identify the file type,
guess the architecture, and identify any interesting strings or structures:
[paste hexdump output]"
```

---

## 2. x86 and ARM Assembly

### x86/x64 Register Overview

**General Purpose Registers (x86-64):**

| 64-bit | 32-bit | 16-bit | 8-bit (low) | Purpose |
|--------|--------|--------|-------------|---------|
| `rax` | `eax` | `ax` | `al` | Return value, accumulator |
| `rbx` | `ebx` | `bx` | `bl` | Base, data |
| `rcx` | `ecx` | `cx` | `cl` | Counter (loop, string ops) |
| `rdx` | `edx` | `dx` | `dl` | Data, I/O |
| `rsi` | `esi` | `si` | `sil` | Source index |
| `rdi` | `edi` | `di` | `dil` | Destination index |
| `rbp` | `ebp` | `bp` | `bpl` | Base pointer (frame) |
| `rsp` | `esp` | `sp` | `spl` | Stack pointer |
| `r8`–`r15` | `r8d`–`r15d` | `r8w`–`r15w` | `r8b`–`r15b` | Extra |

**Special Registers:**
- `rip` — Instruction pointer (program counter)
- `rflags` / `eflags` — Status flags (ZF, CF, OF, SF, etc.)
- `cs`, `ds`, `es`, `fs`, `gs`, `ss` — Segment selectors

### Common x86/x64 Instructions

```
MOV dest, src        ; Copy src to dest
ADD dest, src        ; dest = dest + src
SUB dest, src        ; dest = dest - src
IMUL dest, src       ; dest = dest * src (signed)
DIV src              ; rdx:rax / src
AND dest, src        ; bitwise AND
OR dest, src         ; bitwise OR
XOR dest, src        ; bitwise XOR
NOT dest             ; bitwise NOT

CMP a, b             ; set flags based on a - b
TEST a, b            ; set flags based on a & b

JMP label            ; unconditional jump
JE/JZ label          ; jump if equal/zero (ZF=1)
JNE/JNZ label        ; jump if not equal/zero (ZF=0)
JG/JNLE label        ; jump if greater (signed)
JL/JNGE label        ; jump if less (signed)
JA/JNBE label        ; jump if above (unsigned)
JB/JNAE label        ; jump if below (unsigned)
JMP [reg]            ; indirect jump via register

CALL label           ; push return address, jump to label
CALL [reg]           ; indirect call via register
RET                  ; pop return address, jump back

PUSH reg             ; rsp -= 8; [rsp] = reg
POP reg              ; reg = [rsp]; rsp += 8

LEA dest, [src]      ; Load Effective Address (compute address, not value)
                     ; LEA rax, [rbx+rcx*4+8] -> rax = rbx + rcx*4 + 8
```

### x86 Addressing Modes

```
[reg]                ; direct: address in reg
[reg+offset]         ; base + displacement
[reg+reg*scale]      ; scaled index: scale = 1,2,4,8
[reg+reg*scale+off]  ; full addressing: base + index*scale + displacement
```

### ARM (AArch64) Register Overview

| Register | Purpose |
|---|---|
| `X0`–`X30` | General purpose (64-bit) |
| `W0`–`W30` | Lower 32 bits of X registers |
| `XZR` | Zero register (reads as 0) |
| `SP` | Stack pointer |
| `PC` | Program counter |
| `LR` (X30) | Link register (return address) |
| `FPSR`, `FPCR` | Floating point status/control |

**Callee-saved:** X19–X29, SP
**Caller-saved:** X0–X18, LR (X30)

### Common ARM (AArch64) Instructions

```
MOV X0, X1           ; Copy register
MOV X0, #42         ; Move immediate
ADD X0, X1, X2      ; X0 = X1 + X2
SUB X0, X1, #1      ; X0 = X1 - 1
MUL X0, X1, X2      ; X0 = X1 * X2 (signed)
LSL X0, X1, #3      ; Logical shift left (X1 << 3)
LSR X0, X1, #3      ; Logical shift right
AND X0, X1, X2      ; Bitwise AND
ORR X0, X1, X2      ; Bitwise OR
EOR X0, X1, X2      ; Bitwise XOR
CMP X0, #0          ; Set flags based on X0 - 0
CBZ X0, label       ; Compare and branch if zero
CBNZ X0, label      ; Compare and branch if not zero
B label             ; Unconditional branch
BL label            ; Branch and link (call)
BR X0               ; Branch via register (indirect)
RET                 ; Return (BR LR)
STR X0, [SP, #-16]! ; Store X0 to stack (pre-decrement)
LDR X0, [SP], #16   ; Load X0 from stack (post-increment)
```

### Condition Flags (both x86 and ARM)

| Flag | Name | Set when |
|---|---|---|
| ZF | Zero | Result is zero |
| CF | Carry | Unsigned overflow / borrow |
| SF | Sign | Result is negative |
| OF | Overflow | Signed overflow |
| PF | Parity | Even number of 1 bits |

### Writing Assembly: NASM (x86) Example

```nasm
section .text
global _start

_start:
    mov rax, 1        ; syscall number for write
    mov rdi, 1        ; fd 1 = stdout
    mov rsi, msg      ; pointer to message
    mov rdx, len      ; message length
    syscall

    mov rax, 60       ; syscall number for exit
    xor rdi, rdi      ; exit code 0
    syscall

section .data
    msg db 'Hello, RE!', 0x0A
    len equ $ - msg
```

```bash
nasm -f elf64 hello.asm -o hello.o
ld hello.o -o hello
./hello
```

### Cross-Compiling ARM on x86

```bash
# Install cross-compiler (macOS)
brew install arm-none-eabi-gcc

# Linux
sudo apt install gcc-aarch64-linux-gnu

# Compile for ARM64
aarch64-linux-gnu-gcc -o arm_binary source.c

# Compile for ARM32 (hard float)
arm-linux-gnueabihf-gcc -o arm32_binary source.c

# Inspect the binary
file arm_binary
readelf -A arm_binary    # view ARM attributes
```

### AI Power-Up: Assembly

```
PROMPT: "Explain this x86 assembly instruction by instruction. What does this
function do overall? Focus on control flow and data transformation:
[paste assembly here]"

PROMPT: "Translate this ARM64 assembly to equivalent x86-64 assembly and
vice versa. Note any calling convention differences:
[paste assembly]"

PROMPT: "Given this assembly code, what C source code might have generated it?
Explain your reasoning:
[paste assembly]"
```

---

## 3. ELF and PE Formats

### ELF (Executable and Linkable Format) — Linux/Unix

```
ELF Header (52/64 bytes)
  ├── Magic: 0x7F 'E' 'L' 'F'
  ├── Class: 32-bit or 64-bit
  ├── Endianness: Little or Big
  ├── OS/ABI: System V, Linux, BSD, etc.
  └── Entry point address

Program Headers (segments to load into memory)
  └── Type: PT_LOAD, PT_DYNAMIC, PT_INTERP, etc.

Section Headers (linker view)
  ├── .text    — executable code
  ├── .rodata  — read-only data
  ├── .data    — initialized rw data
  ├── .bss     — uninitialized data (zero-alloc'd)
  ├── .got     — Global Offset Table
  ├── .plt     — Procedure Linkage Table
  ├── .symtab  — symbol table
  ├── .strtab  — string table
  ├── .dynsym  — dynamic symbols
  └── .dynstr  — dynamic strings

Section Content
  └── .text: raw machine code bytes
```

### ELF Loading Process

1. Kernel reads ELF header, validates magic and class
2. Program headers are parsed — PT_LOAD segments are mapped into virtual memory
3. Entry point (`e_entry`) is set as initial `rip`
4. Dynamic linker (`ld.so`) handles shared library resolution if PT_DYNAMIC present
5. PLT/GOT entries are resolved lazily via `_dl_runtime_resolve`

### ELF Commands

```bash
# Full info
readelf -a binary

# Program headers (segments)
readelf -l binary

# Section headers
readelf -S binary

# Symbol table
readelf -s binary

# Dynamic section
readelf -d binary

# Relocations
readelf -r binary

# Headers (ELF + Program)
readelf -e binary

# Disassemble .text
objdump -d binary
objdump -M intel -d binary    # Intel syntax

# Ghidra's objdump equivalent
readelf -x .text binary

# Symbols
nm binary                     # all symbols
nm -D binary                 # dynamic symbols only
nm -g binary                  # defined external (exported)
readelf --syms binary

# Strip a binary
strip binary                  # removes symbol table
strip -s binary              # removes all
strip --strip-debug binary   # keep symbol table, remove debug
```

### PE (Portable Executable) — Windows

```
DOS Header (MZ)
  └── e_lfan pointer to PE Header

PE Header (IMAGE_NT_HEADERS)
  ├── Signature: 'PE\0\0'
  ├── File Header (IMAGE_FILE_HEADER)
  │     ├── Machine: x86/x64/ARM
  │     ├── NumberOfSections
  │     └── TimeDateStamp
  └── Optional Header (IMAGE_OPTIONAL_HEADER)
        ├── AddressOfEntryPoint
        ├── ImageBase
        ├── SectionAlignment
        ├── FileAlignment
        └── DataDirectory[]
              ├── Export Table
              ├── Import Table
              ├── Resource Table
              └── TLS Directory

Section Table (IMAGE_SECTION_HEADER)
  ├── .text    — code
  ├── .data    — initialized data
  ├── .rdata   — read-only data
  ├── .bss     — uninitialized
  ├── .idata   — import table
  ├── .edata   — export table
  ├── .reloc   — base relocations
  └── .rsrc    — resources

Sections
  └── Raw bytes mapped from file into memory
```

### PE Commands

```bash
# On Linux with wine/wine-tools
winedump -h binary.exe        # header dump
winedump -R binary.exe        # resources

# On Windows
dumpbin /ALL binary.exe       # MSVC tool
```

### Stripped Binaries

Stripping removes symbol table (`.symtab`, `.strtab`). Debug info (`DWARF`) may also be removed.

- **Fully stripped**: No symbols, no debug info — hardest to RE
- **Partially stripped**: Symbol table removed but dynamic symbols remain (use `nm -D`)
- **With symbols**: Full symbol table intact — easiest to RE

```bash
file stripped_binary
# stripped_binary: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped

# What a non-stripped binary looks like:
# hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), not stripped
```

### AI Power-Up: Binary Formats

```
PROMPT: "Analyze this ELF/PE header output and explain: entry point, sections,
segments, libraries linked, and any notable characteristics:
[paste readelf -e or objdump output]"

PROMPT: "I'm analyzing a stripped binary. Based on these strings and imports,
what do you think this program does?
[paste strings -n 8 and nm/dumpbin output]"
```

---

## 4. Disassembly and Static Analysis

### Static vs Dynamic Analysis

| Aspect | Static Analysis | Dynamic Analysis |
|---|---|---|
| **Method** | Analyze binary without running | Observe program during execution |
| **Tools** | objdump, radare2, Ghidra, IDA | GDB, strace, ltrace, x64dbg |
| **Pros** | Full code coverage, safe | Real behavior, runtime data |
| **Cons** | No concrete execution path, obfuscation can mislead | Only observed paths covered |

### Disassembly Strategies

**Linear Sweep** — Disassemble every byte sequentially. Prone to skipping data.

**Recursive Descent** — Follows control flow (jumps, calls, branches). More accurate for code sections. Does not handle indirect jumps well.

### objdump

```bash
# Disassemble all code
objdump -d binary

# Disassemble specific function
objdump -d binary --start-address=0x401000 --stop-address=0x401100

# Intel syntax (more readable)
objdump -M intel -d binary

# Show source mixed with assembly (if compiled with -g)
objdump -S binary

# Show disassembly with hex (before instruction)
objdump -b binary -D --prefix-addresses

# Show relocations
objdump -R binary
```

### radare2 / rizin

```bash
# Open binary
r2 -A binary          # auto-analyze on open
r2 -d binary          # open with debugger

# Basic commands
aaa                   ; analyze all (autoanalysis)
afl                   ; list functions
pdf @ main            ; disassemble function
px @ 0x401000         ; hexdump at address
xx @ 0x401000         ; hex view with bytes
axt @ sym.imp.strlen  ; find cross-references (who calls strlen)
axf @ 0x401000        ; find cross-references from address
s main                ; seek to main
VV                    ; visual graph of function
```

### Cross-References (XREFs)

```
CODE XREF: sub_401000+5D ↓    ; called from here
DATA XREF: [rdata+0x10] ↑      ; references data
```

- **CODE XREF**: Code that references this address (call or jump)
- **DATA XREF**: Data that contains this address (pointers)

### Function Identification

How to find `main` in a stripped binary:

```bash
# Entry point
objdump -d binary | head -30

# Look for common main signatures:
# x86: push rbp; mov rbp, rsp; sub rsp, ...
# ARM: stp x29, x30, [sp, #-...]

# Or use radare2:
r2 -c 'aaa; f~main' binary

# symbols from dynamic table (if not stripped)
nm -D binary | grep main
```

### Control Flow Graph (CFG)

Visual representation of all possible execution paths through a function. Jumps, calls, and branches create edges. Essential for understanding complex logic.

### AI Power-Up: Static Analysis

```
PROMPT: "I'm reversing a stripped binary. Here is the disassembly of an
unknown function. Give it a name based on behavior, explain the control
flow, identify any interesting patterns (loops, comparisons, function
calls), and suggest what the original C code might look like:
[paste disassembly]"

PROMPT: "From this objdump output, identify the calling convention being
used, which registers hold arguments, and what the function returns:
[paste objdump]"
```

---

## 5. Debugging with GDB

### GDB Basics

```bash
gdb ./binary          # start debugging
gdb -q ./binary       # quiet mode (less banner)
gdb -ex "run" ./binary # one-shot run

# In GDB
(gdb) break main          # set breakpoint at main
(gdb) break *0x401000     # breakpoint at address
(gdb) run [args]          # start program
(gdb) continue            # continue to next breakpoint
(gdb) next                # step over (next line)
(gdb) step                # step into (follow calls)
(gdb) stepi               # step one instruction
(gdb) nexti               # step one instruction, over calls
(gdb) print $rax          # print register
(gdb) x/10x $rsp          # examine 10 hex words at stack pointer
(gdb) x/s $rdi            # examine as string at rdi
(gdb) info registers       # all registers
(gdb) info breakpoints     # all breakpoints
(gdb) delete 1            # delete breakpoint 1
(gdb) disassemble          # disassemble current function
(gdb) backtrace            # show call stack (bt)
(gdb) watch *(int*)0x601080 # watch memory location
(gdb) p * (struct_t*) $rdi # cast and print as struct
```

### GDB with Pwndbg/GEF (Recommended)

```bash
# Install GEF (GDB Enhanced Features)
# https://github.com/hugsy/gef
# One-liner:
bash -c "$(wget -qO- https://github.com/gef-osx/gef/raw/master/gef.sh)"

# Install pwndbg
# https://github.com/pwndbg/pwndbg

# Then in GDB, you get:
# - Context-aware register/state display
# - Better disassembly
# - Heap analysis (for CTFs)
# - ROP gadget finder
# - Enhanced breakpoints
```

```bash
# In GEF/pwndbg
context             # full state view (registers + stack + code)
telescope $rsp      # follow stack pointer
stack 20            # show 20 stack frames
regs                # all registers
goto 0x401000       # jump to address
ropgrep "pop rdi"   # find ROP gadgets
ropper --file binary # full ROP gadget search
```

### Debugging ARM Binaries

```bash
# On Linux, use qemu-user for ARM
qemu-arm -g 1234 ./arm_binary   # listen on port 1234
gdb-multiarch ./arm_binary
(gdb) target remote localhost:1234

# Or use IDA's gdbserver
# Or use ARM DS-5
```

### GDB Scripting

```gdb
# .gdbinit example
set disassembly-flavor intel
set pagination off
define hook-run
echo \n=== RUNNING ===\n
end
define hook-stop
echo \n=== STOPPED ===\n
info registers
end

# Python GDB scripting (requires gdb compiled with python)
python
import gdb
gdb.execute("break main")
gdb.execute("run")
print gdb.parse_and_eval("$rax")
end
```

### Conditional Breakpoints

```gdb
break main if $rdi > 10
watch *(int*)0x601080 if * (int*)0x601080 == 0x1337
```

### Attaching to Running Processes

```bash
gdb -p $(pidof binary_name)
# or
gdb
(gdb) attach $(pidof binary_name)
```

### AI Power-Up: Debugging

```
PROMPT: "I'm debugging a binary with GDB. The program just crashed with this
register state and stack trace. What does this tell me about the crash?
[paste register dump and backtrace]"

PROMPT: "I need to find the right breakpoint location. Here is the disassembly
around a function call I'm interested in. Where should I set a breakpoint to
catch when a specific string is passed as an argument?
[paste disassembly]"

PROMPT: "My program crashes with a segfault. Here is the state right before
the crash. What is the likely root cause?
[paste register/state dump]"
```

---

## 6. Stack and Calling Conventions

### Stack Basics

The stack grows **downward** (high to low addresses). `push` decrements `rsp`, then writes. `pop` reads, then increments `rsp`.

```
High Addr:  [........][........][........]
            arg2      arg1      return addr
Low Addr:   [========][ rsp ->  ][         ]
                          rsp after push
```

### x86-64 System V Calling Convention (Linux/macOS)

**Arguments (in order):** `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`, then stack

**Return:** `rax` (and `rdx:rax` for 128-bit values)

**Callee-saved:** `rbx`, `rbp`, `r12`, `r13`, `r14`, `r15`, `rsp`
**Caller-saved:** All others (including `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8–r11`)

### x86-64 Microsoft x64 Calling Convention (Windows)

**Arguments:** `rcx`, `rdx`, `r8`, `r9`, then stack

**Shadow space:** Caller must allocate 32 bytes (4 * 8) of shadow space on stack before call (even if not passing args on stack)

**Callee-saved:** `rbx`, `rbp`, `rdi`, `rsi`, `r12`, `r13`, `r14`, `r15`, `xmm6–xmm15`
**Caller-saved:** All others

### x86 (32-bit) Calling Conventions

**cdecl** (C declaration — most common on Linux x86):
- Arguments pushed right-to-left on stack
- **Caller** cleans up the stack after call
- Allows `varargs` (like `printf`)

**stdcall** (Microsoft Win32 API):
- Arguments pushed right-to-left
- **Callee** cleans up the stack (`ret N` pops and discards `N` bytes)
- Used by Win32 API functions

**fastcall** (Microsoft, various forms):
- First 2 args in `ecx` and `edx`, rest on stack

### ARM64 (AAPCS64 — ARM Procedure Call Standard)

**Arguments:** `x0`–`x7` (or `w0`–`w7` for 32-bit)
**Return:** `x0` (and `x1` for 128-bit)
**Callee-saved:** `x19`–`x29`, `sp`, `lr` (actually `x30`)
**Caller-saved:** `x0`–`x18`, `lr`

### Stack Frame Layout (x86-64)

```
RSP ->  [return address]
        [saved RBP]        <- RBP points here
        [local vars]
        [spilled regs]
        [outargs / shadow] <- if called function has arguments
```

### Frame Pointer Omission (FPO)

With `-fomit-frame-pointer` (common optimization), `rbp` is free for use as a general register. Frame is instead tracked via `rsp` offsets.

### Return Address

`CALL` pushes return address onto stack. `RET` pops it and jumps. This is the basis of stack-smashing attacks.

### AI Power-Up: Calling Conventions

```
PROMPT: "I'm reversing a function call. Here is the disassembly showing the
CALL instruction and the next few instructions. Identify the calling convention
and which register holds the first argument:
[paste assembly]"

PROMPT: "I see a function prologue 'push rbp; mov rbp, rsp; sub rsp, N'.
How many bytes of local variable space is allocated? What would the equivalent
C local variable declarations look like?
[paste prologue]"
```

---

## 7. Ghidra Deep Dive

### Getting Started

```bash
# Install Ghidra (free, Java-based)
# Download from: https://github.com/NationalSecurityAgency/ghidra/releases
# Extract and run:
./ghidra_2024/ghidraRun

# Headless mode for scripting
analyzeHeadless /path/to/project -import binary -scriptPath /my/scripts -postScript MyScript.java
```

### Key Ghidra Concepts

| Concept | Description |
|---|---|
| **Listing** | Main code/disassembly view |
| **Decompiler** | Pseudocode C-like output from Ghidra's decompiler |
| **Symbol Tree** | Functions, classes, globals, namespaces |
| **Data Type Manager** | Structures, unions, enums, typedefs |
| **Function Graph** | Visual control flow graph |
| **交叉引用 (XREFs)** | Where code/data is referenced from/to |
| **Defined Data** | Telling Ghidra that bytes are int, pointer, string, etc. |

### Key Shortcuts

| Key | Action |
|---|---|
| `G` | Go to address |
| `F` | Create function |
| `U` | Undefine (treat as bytes) |
| `D` | Define as byte, word, etc. |
| `A` | Define as ASCII string |
| `Esc` | Cancel / go back |
| `Tab` | Toggle Disassembly ↔ Decompiler |
| `Shift+E` | Edit data type |
| `;` | Add comment |
| `:` | Set bookmark |

### Ghidra Scripting

```java
// Ghidra Script (Java)
import ghidra.app.script.GhidraScript;

public class MyFirstScript extends GhidraScript {
    public void run() throws Exception {
        // Get current function
        Function f = getCurrentFunction();
        println("Analyzing: " + f.getName());

        // Rename function based on patterns
        for (Symbol s : f.getGlobalSymbolNameSpace().getSymbols()) {
            if (s.getName().contains("str")) {
                println("Found string ref: " + s.getName());
            }
        }

        // Get all functions
        for (Function func : getGlobalFunctions("main")) {
            println("Function: " + func.getName() + " at " + func.getEntryPoint());
        }
    }
}
```

### Python Scripting (via Ghidra's Python 3 support)

```python
# Run in Ghidra's Python console or via headless
import ghidra.program.model.symbol as Symbol

# Rename all functions with "str" in name to have prefix
for func in getAllFunctions():
    name = func.getName()
    if "str" in name.lower():
        # Make more descriptive
        rename_func = getFirstSymbol()
        pass

# Find all cross-references to a specific address
addr = toAddr(0x00401000)
refs = getReferencesTo(addr)
for ref in refs:
    print(f"From: {ref.getFrom()}")

# Apply structure to memory location
# Define a struct:
struct_mgr = currentProgram.dataTypeManager
struct = struct_mgr.getDataType("/structures/my_struct")
data = createData(addr, struct)
```

### Side-by-Side x86 and ARM Analysis

1. Open x86 binary in one Ghidra project
2. Use "Window > Diff" to compare with ARM binary
3. Compare function signatures and structures

### LLM + Ghidra Workflow

```python
# Pseudo-workflow: export Ghidra decompilation, feed to LLM
# 1. In Ghidra script, export decompiled function to text:
decompiled = getFunctionAt(entry).getSignature().getPrototypeString()
body = getFunctionAt(entry).getBody()
# Save to file

# 2. Send to LLM:
PROMPT = """Here is the Ghidra decompilation of an unknown function.
Analyze and explain what it does, suggest a better function name, and
identify any potential security issues:

[decompiled output]"""
```

### AI Power-Up: Ghidra

```
PROMPT: "Here is the Ghidra decompilation of a function. Explain what it does
in plain English, suggest a better function name, and identify any notable
security implications:
[paste decompiled function]"

PROMPT: "I'm looking at what appears to be a buffer overflow vulnerability in
this decompiled code. Can you identify the vulnerable line and explain the
exploitation primitive?
[paste decompiled function]"
```

---

## 8. Dynamic Analysis

### strace — Trace System Calls

```bash
# Trace all system calls
strace ./binary

# Trace specific syscalls
strace -e read,write ./binary

# Trace and time syscalls
strace -T ./binary

# Trace child processes (fork/vfork)
strace -f ./binary

# Save to file
strace -o output.txt ./binary

# Filter by result (failed syscalls only)
strace -e trace=read -f ./binary 2>&1 | grep -v "= -1"

# Trace TCP connections
strace -e trace=network ./binary

# Hex dump of read/write buffers
strace -s 100 ./binary   # show first 100 bytes of strings
```

### ltrace — Trace Library Calls

```bash
# Trace library (shared object) calls
ltrace ./binary

# Trace specific library functions
ltrace -e malloc,free,printf ./binary

# Print library returns
ltrace -S ./binary

# Filter by library
ltrace -e libc.so.* ./binary
```

### Combining strace and ltrace

```bash
# Both at once
strace -f -e trace=write,read ./binary 2>&1 | grep -E '(write|read)\('
```

### Wireshark — Network Analysis

```bash
# Capture on interface
wireshark &

# CLI capture
tshark -i eth0 -w capture.pcap

# Read and filter
tshark -r capture.pcap -Y "tcp.port == 8080" -T fields -e data

# Export objects from pcap
foremost -i capture.pcap -o output/   # extract files from pcap
```

### API Monitoring (Windows)

```bash
# Procmon alternatives on Linux:
# - sysdig: sudo sysdig
# - bpftrace: for syscall tracing
# - auditd: Linux Audit system
```

### Behavioral Analysis Workflow

1. Run `strace` to see system interactions (file ops, network, process)
2. Run `ltrace` to see library call patterns
3. Run in VM/sandbox, observe file system changes
4. Use `strings` on binary vs runtime-observed behavior to spot discrepancies

### AI Power-Up: Dynamic Analysis

```
PROMPT: "Here is an strace output of an unknown binary. Summarize what system
calls it makes, what files it opens, and what network connections it attempts.
Identify any suspicious behavior:
[strace output]"

PROMPT: "This program makes several network connections. Based on this strace
output showing send/recv calls, what protocol might it be using?
[strace output]"

PROMPT: "The program is making an unexpected outbound connection. Here is the
strace and ltrace output. Can you identify the configuration or string that
determines the connection target?
[strace + ltrace output]"
```

---

## 9. Patching and Binary Modification

### Byte Patching

The goal: modify binary behavior without recompiling from source.

**Types:**
1. **NOP padding** — Replace unwanted instructions with NOP (`0x90` on x86)
2. **Byte overwrite** — Directly modify opcodes/data
3. **Jump patching** — Redirect control flow (short jump `EB xx`, near jump `E9 xx`)
4. **Data modification** — Change strings, constants, pointers

### Tools

```bash
# Hex editor (interactive)
hexedit binary

# Command-line patch
# Change byte at offset 0x1A3 from 74 to EB:
printf '\xEB' | dd of=binary bs=1 seek=$((0x1A3)) count=1 conv=notrunc

# Using radare2
r2 -w binary
[0x00000000]> o+                  # open in read-write
[0x00000000]> pd 5                ; disassemble 5 instructions
[0x00000000]> wa mov eax, 0       ; write assembly
[0x00000000]> wx 9090            ; write hex bytes
[0x00000000]> wop O a            ; show physical offset of current

# Using Ghidra
# Load binary -> open in listing -> right-click -> Patch Instruction
# Or: File -> Export -> Modified program
```

### NOPing a Comparison

```bash
# Original: JNZ (74) -> jump if not zero
# NOP it: replace 74 with 90 (x86 NOP)
# Or use two-byte NOP: 66 90 (x87) for alignment

# Find the offset of the byte to patch:
objdump -d binary | grep -A2 "<function_name>"

# Or in GDB:
disassemble function_name
# find the JE/JNE instruction, note its address
```

### Redirecting Control Flow

```bash
# Short jump (relative, 1-byte offset, range -128 to +127)
# Opcode: EB xx (xx = relative offset from next instruction)
# Example: at address 0x401000, replace 'JE .+5' (74 03) with NOP NOP (90 90)

# Near jump (32-bit relative offset)
# Opcode: E9 xx xx xx xx

# Absolute jump via register
# Example: JMP EDI -> FF E7 (FF D7 = JMP EDX)
```

### Handling Jump Offset Calculation

```
# Original: JZ target at address 0x401020 (next instruction is 0x401002)
# Current instruction at 0x401000: 74 00 -> JE $+2 (would go to 0x401002)
# Want to redirect to 0x401050

# Near jump: E9 [offset]
# Offset = target - (current_addr + 5)  [5 = near jmp opcode size]
#        = 0x401050 - (0x401000 + 5)
#        = 0x401050 - 0x401005
#        = 0x4B

# New bytes: E9 4B 00 00 00
```

### Patching Strings

Strings are easy targets — null-terminated ASCII in the `.data` or `.rdata` section.

```bash
# Find string offsets
strings -n 8 binary | grep "license"
# Or in radare2:
/ license
# Or in Ghidra: Search -> For Strings

# Patch in place (careful about length)
# If "UNLICENSED" -> "LICENSED":
# New string must be same length or shorter with null padding

# Example: "UNLICENSED" (10 bytes) -> "LICENSED\0\0"
printf "LICENSED" | dd of=binary bs=1 seek=$((string_offset)) count=8 conv=notrunc
```

### Binary Diffing

```bash
# BinDiff (IDA plugin) — compare two binaries
# DarunGrim — original binary diffing tool
# Diaphora (Ghidra plugin) — free, excellent
# yagi (Ghidra plugin) — yet another Ghidra Integration

# Quick diff with diff:
diff <(objdump -d binary1) <(objdump -d binary2)
```

### AI Power-Up: Patching

```
PROMPT: "I need to patch a binary to bypass this license check. The function
disassembly is below. Which bytes do I need to change to make it always take
the 'success' path? Show the before/after bytes:
[paste disassembly of check function]"

PROMPT: "I want to redirect this conditional jump to always take the true
branch. The instruction is at offset 0x1000 and currently is '74 05' (JE
+5). What should I replace it with to always jump, and what is the new byte
sequence?"
```

---

## 10. Anti-Reverse Engineering Techniques

### Detection and Evasion

**Anti-debug:**
- `ptrace(PTRACE_TRACEME)` — Linux; if another tracer already attached, fails
- `IsDebuggerPresent()` / `CheckRemoteDebuggerPresent()` — Windows
- Timing attacks: `rdtsc` before/after operations; debuggers slow execution
- Self-debugging: fork child to do work, parent traces child (makes attaching external debugger hard)

**Anti-disassembly:**
- Junk bytes inserted between valid instructions
- Jump to middle of instructions (disassembler confusion)
- Opaque predicates: conditional jumps that always evaluate the same way but confuse linear sweep disassemblers
- Function overlapping: overlapping function prologues

**Packing:**
- UPX: compresses and encrypts the original binary
  ```bash
  upx -9 binary          # pack
  upx -d binary          # unpack
  ```
- Custom packers: encryption layer + decryption stub + original binary

**Obfuscation:**
- String encryption: strings XOR'd with key, decrypted at runtime
- Control flow flattening: replace natural control flow with a loop + switch
- Indirect calls via register (call `[rbx+rax*4]`)
- Import table hashing: resolve API addresses via hash lookup instead of names

### Identifying Packing

```bash
# High entropy sections
# Normal: ~6.5-7.0 bits/byte
# Packed: ~7.5-8.0 bits/byte
# Use: binwalk -E binary (entropy)

# Packed program entry point not in .text
readelf -l binary | grep -A1 LOAD
# Check if entry point maps to a known section

# Check for UPX signature
strings binary | grep UPX

# Very small raw section sizes
readelf -S binary
```

### Unpacking

1. Set breakpoint at original entry point (OEP) — often after unpacking stub
2. Let unpacker stub run, dump memory at OEP
3. Fix import table (often stored encrypted in packed section)
4. Tools: ` ScyllaHide` (Windows), `un UPX` (Linux), `Frida` (scriptable unpacking)

### AI Power-Up: Anti-RE

```
PROMPT: "I'm dealing with an obfuscated binary. Here are some interesting
strings and the disassembly around a suspicious function. Does this look like
a packer, and if so, what is the unpacking strategy?
[strings + disassembly]"

PROMPT: "The binary is checking for a debugger. Here is the relevant code.
How can I patch around this check?
[anti-debug code]"

PROMPT: "The strings in this binary are all encrypted. I can see the
decryption loop. Can you help me write a script to decrypt all strings offline?
[decryption function disassembly]"
```

---

## 11. Protocol Reverse Engineering

### Approach

1. Capture traffic (Wireshark, tcpdump)
2. Identify patterns (packet sizes, sequence numbers, repeated structures)
3. Extract payloads
4. Replay to probe behavior
5. Hypothesis + test iteratively
6. Implement parser/reconstructor

### Wireshark

```bash
# Capture
sudo wireshark &          # GUI
sudo tcpdump -i eth0 -w capture.pcap

# Analyze
tshark -r capture.pcap -Y "tcp" -T fields -e ip.src -e ip.dst -e tcp.srcport -e tcp.dstport -e data
# Export raw data from specific packet
tshark -r capture.pcap -Y "frame.number == 5" -x

# Follow stream
tshark -r capture.pcap -Y "tcp.stream eq 0" -T fields -e data | tr -d '\n'
```

### Common Protocol Patterns

| Pattern | Likely Meaning |
|---|---|
| Fixed header size | Header contains length/type fields |
| Length-prefixed strings | First N bytes = string length |
| Magic bytes at start | Protocol version/identifier |
| Incremental counters | Sequence numbers, transaction IDs |
| Checksum/CRC at end | Integrity verification |

### Netcat for Probing

```bash
# Connect to server
nc localhost 8080

# Send hex data
cat payload.bin | nc localhost 8080

# Hex input mode
# Some versions: nc -C (send CRLF)
# Use: xxd -r -p < hex.txt | nc host port
```

### Replay Attacks

```python
# Python replay script
import socket

# Send binary packet
packet = bytes.fromhex("01 02 00 0a 48 65 6c 6c 6f")
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("localhost", 9000))
s.send(packet)
response = s.recv(4096)
print(response.hex())
```

### AI Power-Up: Protocol RE

```
PROMPT: "Here is a hex dump of several sequential packets from an unknown
protocol. Identify the structure, field boundaries, and likely data types for
each field. Look for magic bytes, length fields, and repeating patterns:
[hex dump of packets]"

PROMPT: "I'm trying to reverse engineer a network protocol. Here is a packet
capture of the login sequence. Based on the payloads, what do you think the
authentication flow looks like?
[packet payloads]"
```

---

## 12. Obfuscation and Deobfuscation

### Types of Obfuscation

| Type | Description | Example |
|---|---|---|
| **String encryption** | Strings XOR'd or AES-encrypted | `"hello"` → `"\x1F\x2E..."` |
| **Control flow flattening** | Natural if/else → switch inside loop | Blocks reordered unpredictably |
| **Dead code injection** | Junk code that never executes | Unreachable basic blocks |
| **Instruction substitution** | `a + b` → `a - (-b)` | Harder to read |
| **Opaque predicates** | Always-true/false conditions | `x * 0 == 0` |
| **Import hashing** | APIs resolved by hash lookup | `GetProcAddress(addr_by_hash)` |
| **Virtualization** | Custom bytecode + VM interpreter | Themida, VMProtect |

### Deobfuscation Strategies

1. **Automated** — Use deobfuscators where available (UPX, deVMut, etc.)
2. **Scripted** — Write Ghidra/IDAPython/Rizin scripts to simplify
3. **Dynamic** — Let it run in controlled environment, observe real behavior
4. **Brute force** — Pattern-specific (XOR search, entropy mapping)

### String Decryption Scripts

```python
# Simple XOR decryption
def decrypt_xor(data: bytes, key: int) -> bytes:
    return bytes(b ^ key for b in data)

# Find XOR key (known plaintext attack)
# If you know a string that appears in the decrypted output:
# XOR encrypted with key K, known plaintext P
# K = encrypted[0] ^ P[0]
def find_xor_key(encrypted: bytes, known_plaintext: bytes) -> int:
    return encrypted[0] ^ known_plaintext[0]

# Then decrypt all
data = open("encrypted_strings.bin", "rb").read()
key = find_xor_key(data, b"password")
decrypted = decrypt_xor(data, key)
print(decrypted)
```

### Ghidra Script for String Decryption

```java
// Decrypt XOR strings in Ghidra
import ghidra.app.script.GhidraScript;

public class DecryptXorStrings extends GhidraScript {
    public void run() throws Exception {
        long key = 0x42; // XOR key (find via analysis)
        Address stringPtr = addr(0x00403000);
        
        for (int i = 0; i < 50; i++) {
            byte b = getByte(stringPtr.add(i));
            if (b == 0) break; // null terminator
            byte plain = (byte)(b ^ key);
            printf("%c", plain);
        }
        println("");
    }
}
```

### Control Flow Deobfuscation

Often requires symbolic execution or abstract interpretation. Tools:
- **angr** (Python): Symbolic execution framework, can simplify paths
- **Triton** (Python): Symbolic execution + taint analysis
- **IDA Pro with Hex-Rays** (paid): Best decompiler for messy code
- **Ghidra**: Decompiler + scripting can help flatten switch-isms

### AI Power-Up: Obfuscation

```
PROMPT: "I see a function that appears to decrypt a string at runtime. The
decryption loop is shown below. Write a Python script to decrypt all such
strings in this binary given the key:
[decryption function disassembly]"

PROMPT: "This binary uses a custom obfuscation scheme. The control flow
graph looks like spaghetti. Can you analyze this decompiled output and
simplify it into a more readable form? Identify the actual logic paths:
[decompiled obfuscated function]"
```

---

## 13. AI-Assisted Reverse Engineering

> [!tip] AI as a Force Multiplier
> Large Language Models are not just for writing code — they can analyze code, suggest names, identify patterns, generate scripts, and explain behavior. This topic covers using AI/LLMs as a first-class RE tool alongside Ghidra, GDB, and radare2.

### Tool Landscape

| Tool | Description | Use Case |
|---|---|---|
| **OpenCode** | Local AI coding agent with tool access | Interactive RE sessions, script writing, multi-file analysis |
| **Claude (API)** | Anthropic's LLM via API | Decompilation analysis, vulnerability research |
| **GPT-4o** | OpenAI's model | Code explanation, script generation |
| **Ollama** | Local LLM server (Llama, Mistral, etc.) | Offline/sensitive RE work |
| **llama.cpp** | CPU-based LLM inference | Local model serving |
| **Ghidra + LLM bridges** | Connect LLM to Ghidra scripting | Auto-rename, auto-type, deobfuscation |
| **radare2 + LLM** | r2 plugin for AI analysis | Interactive RE with AI |
| **MCP (Model Context Protocol)** | Standardized AI ↔ tool protocol | Connect AI to GDB, Ghidra, Wireshark, etc. |

### OpenCode for RE

OpenCode can be used for:
- Analyzing decompiled output and explaining behavior
- Writing Ghidra scripts / IDAPython scripts
- Generating patch code
- Identifying vulnerability patterns
- Writing deobfuscation scripts
- Analyzing memory dumps and crash reports

### MCP Integration

MCP enables AI agents to interact with RE tools directly:

```json
// Example MCP server config for GDB
{
  "mcpServers": {
    "gdb": {
      "command": "python3",
      "args": ["-m", "mcp_gdb", "--port", "8765"]
    },
    "ghidra": {
      "command": "python3", 
      "args": ["-m", "mcp_ghidra", "--port", "8766"]
    }
  }
}
```

### Prompt Engineering for RE

**Good prompts for RE tasks:**

```
PROMPT: "Act as a reverse engineer. Analyze this disassembly/decompilation
and provide:
1. What the function does (plain English)
2. A suggested function name
3. Any security concerns
4. The likely C equivalent source code

[code]"

PROMPT: "I'm writing a Ghidra script to automate [task]. Here is my current
script. Fix the bugs and complete the implementation:
[script]"

PROMPT: "Given this binary's behavior from strace and this partial
decompilation, what is the configuration/control flow that determines the
behavior?
[strace output + decompilation]"
```

### Local LLMs for Sensitive RE

When binaries are sensitive/proprietary and can't be uploaded to cloud LLMs:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model
ollama pull llama3.2
ollama pull codellama

# Serve locally
ollama serve

# Use with OpenCode: configure to use local endpoint
```

### AI-Assisted Workflows

**1. Decompilation → Explanation**
```
Analyze this Ghidra decompilation, explain what it does, identify security
issues, and suggest a better function name:

[decompiled function]
```

**2. Script Generation**
```
Write a Ghidra Python script that:
1. Finds all functions that reference the string "password"
2. Exports their addresses and decompiled output to a JSON file
3. Runs headlessly
```

**3. Vulnerability Hunting**
```
Review this decompiled function for common vulnerability patterns:
- Buffer overflow
- Use-after-free
- Integer overflow
- Command injection
- Path traversal

[class/function]
```

**4. Deobfuscation**
```
This binary uses XOR with a single-byte key for string encryption. Here
is the decryption loop. Write a Python script that dumps all decrypted
strings and their addresses to a file:
[decryption loop disassembly]
```

**5. Binary Diffing**
```
I have two versions of the same binary (v1.0 and v1.1). Version 1.1 has
a security fix. Here are the functions that differ. Analyze what changed
and what vulnerability was likely fixed:

[v1.0 function] vs [v1.1 function]
```

### LLM Limitations in RE

| Limitation | Mitigation |
|---|---|
| Hallucination | Always verify LLM output against actual disassembly/debugger |
| Context length | Feed focused snippets, not entire binaries |
| Novel patterns | May miss novel obfuscation — use as helper, not replacement |
| No runtime | Can't observe actual execution — combine with strace/GDB |
| Math/hex | Double-check any offset calculations — LLMs can miscalculate |

### Best Practice: LLM + Tools Loop

```
Human/Agent
  ↓ feed disassembly
LLM → Analysis + Suggestions
  ↓ apply via Ghidra script / GDB
Tool → New insight / behavior
  ↓ feed back to LLM
LLM → Deeper analysis
  ↓ iterate
```

---

## Practice Drills

| # | Topic | File |
|---|---|---|
| 01 | Binary Fundamentals | [[Reverse Engineering/Reverse Engineering Practice/01 - Binary Fundamentals]] |
| 02 | x86 and ARM Assembly | [[Reverse Engineering/Reverse Engineering Practice/02 - x86 and ARM Assembly]] |
| 03 | ELF and PE Formats | [[Reverse Engineering/Reverse Engineering Practice/03 - ELF and PE Formats]] |
| 04 | Disassembly and Static Analysis | [[Reverse Engineering/Reverse Engineering Practice/04 - Disassembly and Static Analysis]] |
| 05 | Debugging with GDB | [[Reverse Engineering/Reverse Engineering Practice/05 - Debugging with GDB]] |
| 06 | Stack and Calling Conventions | [[Reverse Engineering/Reverse Engineering Practice/06 - Stack and Calling Conventions]] |
| 07 | Ghidra Deep Dive | [[Reverse Engineering/Reverse Engineering Practice/07 - Ghidra Deep Dive]] |
| 08 | Dynamic Analysis | [[Reverse Engineering/Reverse Engineering Practice/08 - Dynamic Analysis]] |
| 09 | Patching and Binary Modification | [[Reverse Engineering/Reverse Engineering Practice/09 - Patching and Binary Modification]] |
| 10 | Anti-Reverse Engineering | [[Reverse Engineering/Reverse Engineering Practice/10 - Anti-Reverse Engineering]] |
| 11 | Protocol Reverse Engineering | [[Reverse Engineering/Reverse Engineering Practice/11 - Protocol Reverse Engineering]] |
| 12 | Obfuscation and Deobfuscation | [[Reverse Engineering/Reverse Engineering Practice/12 - Obfuscation and Deobfuscation]] |
| 13 | AI-Assisted Reverse Engineering | [[Reverse Engineering/Reverse Engineering Practice/13 - AI-Assisted Reverse Engineering]] |

---

## Workflow Projects

| # | Project | File |
|---|---|---|
| 01 | Reverse a CLI Utility | [[Reverse Engineering/Reverse Engineering Workflows/01 - Reverse a CLI Utility]] |
| 02 | Analyze a Network Protocol | [[Reverse Engineering/Reverse Engineering Workflows/02 - Analyze a Network Protocol]] |
| 03 | Patch a License Check | [[Reverse Engineering/Reverse Engineering Workflows/03 - Patch a License Check]] |
| 04 | Deobfuscate a Malware Sample | [[Reverse Engineering/Reverse Engineering Workflows/04 - Deobfuscate a Malware Sample]] |
| 05 | Reconstruct Source from Stripped Binary | [[Reverse Engineering/Reverse Engineering Workflows/05 - Reconstruct Source from Stripped Binary]] |
