---
title: "RE Practice: ELF and PE Formats"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - elf
  - pe
  - binary-format
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: ELF and PE Formats

13 progressive problems covering ELF (Linux) and PE (Windows) binary formats.

---

## P1: Analyze ELF Header

Create and analyze an ELF header:

```bash
echo 'int main(){return 0;}' > /tmp/test.c
gcc -o /tmp/test_elf /tmp/test.c
readelf -h /tmp/test_elf
```

Identify: magic bytes, class (32/64-bit), endianness, OS/ABI, entry point, and section/program header offsets.

**Expected output:**
```
Magic: 7f 45 4c 46 (DEL + "ELF")
Class: ELF64
Data:  2's complement, little endian
Entry point: 0x401000 (or similar)
```

> [!hint]- Hint
> ELF header always starts at offset 0. Fields are at fixed offsets within the header.

> [!success]- Solution
> The `readelf -h` output shows all ELF header fields. Key ones:
> - `Magic`: 0x7F 'E' 'L' 'F'
> - `Class`: `ELF64`
> - `Data`: `2's complement, little endian`
> - `Entry point address`: `0x401000` (typical for PIE binaries)

---

## P2: Map Sections to Segments

Run `readelf -l /tmp/test_elf` and map PT_LOAD segments to their corresponding sections:

```bash
readelf -l /tmp/test_elf
readelf -S /tmp/test_elf
```

Which sections are mapped by each PT_LOAD segment? What is the difference between `FileSiz` and `MemSiz`?

**Expected output:**
```
Segment 01: .text (rx) — FileSiz=0x..., MemSiz=0x... (same)
Segment 02: .data + .bss (rw) — FileSiz=... (data) MemSiz=... (data+bss)
```

> [!hint]- Hint
> `FileSiz` is how much data is loaded from the file. `MemSiz` is how much memory is allocated (FileSiz + bss padding).

> [!success]- Solution
> PT_LOAD segments map sections into virtual memory. `.text` is typically Read+Execute (rx), `.data` is Read+Write (rw). `.bss` (uninitialized) has MemSiz > FileSiz because it's zero-initialized in memory but not stored in the file.

---

## P3: Parse ELF with Python

Write a Python script that opens any ELF binary and prints:
1. Magic
2. Class (32 or 64 bit)
3. Entry point address

```python
import struct

with open("/bin/true", "rb") as f:
    magic = f.read(4)
    if magic != b'\x7fELF':
        print("Not an ELF file")
        exit()
    
    e_class = ord(f.read(1))
    print(f"Class: {'ELF64' if e_class == 2 else 'ELF32'}")
    
    e_endian = ord(f.read(1))
    print(f"Endian: {'Little' if e_endian == 1 else 'Big'}")
    
    # Entry point (offset 24 for 64-bit, 24 for 32-bit)
    f.seek(24)
    entry = struct.unpack("<Q" if e_endian == 1 else ">Q", f.read(8))[0]
    print(f"Entry point: 0x{entry:016x}")
```

**Expected output:**
```
Magic: ELF
Class: ELF64
Endian: Little
Entry point: 0x...
```

> [!hint]- Hint
> ELF header format is defined by the spec. Offset 4 = class, 5 = endianness, 24-31 = entry point (64-bit).

> [!success]- Solution
> The script correctly parses the ELF magic, class, and entry point. For `/bin/true`, entry point is typically `0x2000` or similar.

---

## P4: Identify Symbols with nm

```bash
nm -D /bin/ls          # dynamic symbols only
nm /bin/ls | grep main # all symbols named "main"
nm -g /bin/ls          # exported (global) symbols only
```

Compare the outputs. Why does `nm -D` show fewer symbols?

**Expected output:**
```
nm -D: shows only symbols from dynamic section (.dynsym)
nm -g: shows global symbols from full symbol table (.symtab)
nm with no args: shows all symbols including locals
```

> [!hint]- Hint
> Dynamic symbols (.dynsym) are what the dynamic linker uses at runtime. The full symbol table (.symtab) has more info but is stripped in stripped binaries.

> [!success]- Solution
> `nm -D` shows `.dynsym` — only symbols needed for dynamic linking (imports/exports). `nm` without flags shows `.symtab` — all symbols including local variables and debugging symbols. Stripped binaries remove `.symtab` but keep `.dynsym`.

---

## P5: Strip a Binary and Compare

```bash
cp /bin/ls /tmp/ls_stripped
strip /tmp/ls_stripped
file /bin/ls /tmp/ls_stripped
ls -la /bin/ls /tmp/ls_stripped
diff <(nm /bin/ls) <(nm /tmp/ls_stripped 2>&1) | head -20
```

How much smaller is the stripped binary? What symbols were removed?

**Expected output:**
```
Stripped binary is significantly smaller (typically 20-50% reduction)
nm on stripped shows "no symbols"
```

> [!hint]- Hint
> `strip` removes the symbol table and debug info. File size shrinks noticeably.

> [!success]- Solution
> `strip` removes `.symtab`, `.strtab`, and debug sections. The stripped binary is smaller. `nm` on stripped binary outputs "no symbols" (but `nm -D` still works because `.dynsym` is typically preserved).

---

## P6: ELF Relocations

```bash
readelf -r /tmp/test_elf
```

What relocations are present? For each relocation:
1. What is the offset?
2. What symbol does it reference?
3. What type is it (e.g., R_X86_64_JUMP_SLOT, R_X86_64_64)?

**Expected output:**
```
Relocation section '.rela.plt':
  Offset      Type           Sym. Name
  0x...       R_X86_64_JUMP_SLOT  __libc_start_main
  0x...       R_X86_64_64        __gmon_start__
```

> [!hint]- Hint
> PLT relocations are for lazy binding of external functions. GOT relocations are for data references.

> [!success]- Solution
> The `readelf -r` output shows all relocations. R_X86_64_JUMP_SLOT = PLT entry for external function calls. R_X86_64_64 = 64-bit data relocation. R_X86_64_COPY = copy relocation.

---

## P7: ELF Sections — Find .text

```bash
readelf -S /tmp/test_elf
objdump -h /tmp/test_elf
```

Find the `.text` section. What is its:
- Virtual address
- File offset
- Size

Then disassemble it with `objdump -d`:

```bash
objdump -M intel -d /tmp/test_elf | head -30
```

**Expected output:**
```
.text section: VMA=0x401000, FileSiz=0x..., Off=0x1000
Disassembly starts at entry point with <_start>
```

> [!hint]- Hint
> `readelf -S` shows section headers. `objdump -h` shows section headers in a more readable format. `objdump -d` disassembles the `.text` section.

> [!success]- Solution
> `.text` section is the code section. `objdump -d` disassembly shows the entry point function (typically `_start` or `main` after startup code), with instructions in Intel or AT&T syntax depending on the `-M` flag.

---

## P8: PE Header Analysis (Windows-Style)

Create a simple binary and analyze its PE header (using a cross-compiler or `winedump` if on Linux):

```bash
# On Linux, use objconv or simply analyze with readelf on an ELF
# For PE practice, use this example from a CTF binary analysis:
# The PE header starts with "MZ" (0x4D 0x5A)
# At offset 0x3C: pointer to PE header (offset 0x80 typically)

# Consider a hex dump of a PE header:
# MZ signature at 0x00
# e_lfanew pointer at 0x3C -> points to 0x80
# At 0x80: PE\0\0 (0x50 0x45 0x00 0x00)
# At 0x88: Machine = 0x8664 (x86-64)
# At 0x90: NumberOfSections = 3
```

What are the machine type and number of sections?

**Expected output:**
```
Machine: x86-64 (0x8664)
NumberOfSections: 3
```

> [!hint]- Hint
> PE header offset = value at offset 0x3C. PE signature at that offset. COFF header starts 4 bytes after PE signature.

> [!success]- Solution
> - `e_lfanew` at 0x3C points to PE header location
> - PE signature = `PE\0\0` at that location
> - Machine field (IMAGE_FILE_HEADER) at offset 4 from PE sig: `0x8664` = AMD64/x86-64
> - NumberOfSections: value at offset 2 from COFF header start = 3

---

## P9: ELF — Readelf vs Objdump

Run both commands on the same binary and compare:

```bash
readelf -a /tmp/test_elf > readelf_out.txt
objdump -M intel -d -r -h /tmp/test_elf > objdump_out.txt
diff readelf_out.txt objdump_out.txt | head -5
echo "=== Same information, different format ==="
```

What information does each tool provide that the other doesn't?

**Expected output:**
```
readelf: ELF-specific, shows headers, sections, symbols, dynamic info, relocations
objdump: more focused on disassembly, shows code + headers + relocations
```

> [!hint]- Hint
> `readelf` is the standard ELF tool. `objdump` works across multiple formats (ELF, PE, Mach-O) and focuses on disassembly.

> [!success]- Solution
> - `readelf -a`: Complete ELF analysis including dynamic linking info, symbol versioning, GNU hash tables
> - `objdump -d`: Disassembly, mixed with source if available (`-S`), relocation info during disassembly
> - `objdump -d` is better for code review; `readelf` is better for format analysis

---

## P10: ELF — Section Names and Purposes

For each standard ELF section, identify its purpose:

```
.text, .data, .rodata, .bss, .got, .plt, .symtab, .strtab, .dynsym, .dynstr, .rel/.rela, .comment, .note
```

**Expected output:**
```
.text    — executable code
.data    — initialized read-write data
.rodata  — read-only data (constants)
.bss     — uninitialized data (zero-allocated)
.got     — Global Offset Table (dynamic linking)
.plt     — Procedure Linkage Table (function calls to shared libs)
.symtab  — full symbol table
.strtab  — string table for symtab
.dynsym  — dynamic linking symbol table
.dynstr  — string table for dynamic symbols
.rel/.rela — relocation entries
.comment — version control info
.note    — ABI/metadata notes
```

> [!hint]- Hint
> This is a recall exercise. The distinction between `.data` and `.bss` (file vs memory only), and between `.symtab` and `.dynsym` (full vs dynamic-only), is commonly tested.

> [!success]- Solution
> As shown above. Key distinctions:
> - `.bss` takes no file space (only `MemSiz`), it's zero-initialized at load
> - `.dynsym` is stripped but `.symtab` remains in non-stripped binaries
> - `.got` and `.plt` work together for PLT (lazy binding of shared library functions)

---

## P11: Find Symbol Size

Use `readelf` or `nm` to find the size of each function symbol:

```bash
nm -S /bin/ls | head -20
# Or:
readelf -s /bin/ls | grep FUNC | head -20
```

What is the size of `main`? How many bytes of code does it contain?

**Expected output:**
```
main: size = ... bytes
```

> [!hint]- Hint
> `nm -S` shows symbol size in hex. `readelf -s` shows similar information in a more detailed format.

> [!success]- Solution
> The `nm -S` output shows symbol sizes. For `main`, you might see a size like `0x100` (256 bytes). The size tells you how many bytes of code the function contains, useful for knowing where the function ends.

---

## P12: ELF Auxiliary Vectors (Advanced)

When the Linux kernel starts a program, it passes auxiliary vectors on the stack. Parse this simplified structure:

```
At address 0x7fffffffe108 (stack), you find:
[AT_NULL]     value=0
[AT_IGNORE]   value=0  
[AT_EXECFD]   value=2
[AT_PHDR]     value=0x400040  (pointer to program headers)
[AT_PHNUM]    value=9         (number of program headers)
[AT_ENTRY]    value=0x401000  (program entry point)
```

What does each auxv entry tell you about the running process?

**Expected output:**
```
AT_EXECFD: 2    → file descriptor of executable was 2
AT_PHDR:   0x400040 → location of program headers
AT_PHNUM:  9     → 9 program headers
AT_ENTRY:  0x401000 → entry point of the program
```

> [!hint]- Hint
> Auxv is passed by the kernel to the dynamic linker (`ld.so`) and then to `main`. It's used to find program headers, entry point, and other runtime info.

> [!success]- Solution
> AT_NULL marks end of auxv. AT_EXECFD is the exec file descriptor (rarely used). AT_PHDR/AT_PHNUM tell the dynamic linker where to find and how many program headers exist. AT_ENTRY is the program's entry point.

---

## P13: Use LLM to Analyze ELF Header

Use this prompt with a real ELF header output:

```
PROMPT: "Here is the readelf -e output of an unknown ELF binary. Analyze it
and tell me: what architecture? Is it stripped? Is it statically or dynamically
linked? What is the entry point? What notable sections does it have?

[readelf -e output]"
```

**Expected output:**
```
(LLM response with architecture identification, stripping status,
linkage type, entry point, and notable sections like .text, .data, .plt, etc.)
```

> [!hint]- Hint
> Stripped binaries won't have symbol names. Statically linked binaries won't have .plt/.got. Dynamically linked binaries will have PT_DYNAMIC segment.

> [!success]- Solution
> The LLM should correctly identify: architecture (x86-64, ARM, etc.), stripping status (whether nm works), linkage type (static = no PT_DYNAMIC, no .plt/.got; dynamic = has both), and notable sections.

---

## AI Power-Up: Binary Formats

```
PROMPT: "I'm analyzing a binary that appears to be both an ELF and something
else. Here is the hex dump of the first 256 bytes. What is it?
[hex dump]"

PROMPT: "The binary I'm analyzing claims to be a 64-bit ELF but it won't run.
Here is the readelf -h output. What is wrong with it?
[readelf -h output]"
```
