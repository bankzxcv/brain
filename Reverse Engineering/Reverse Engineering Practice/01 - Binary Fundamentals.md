---
title: "RE Practice: Binary Fundamentals"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - binary
  - hex
  - endianness
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Binary Fundamentals

14 progressive problems. Each problem builds on the last — from reading hex to analyzing file structures.

---

## P1: Read Hex Dump

You are analyzing a network packet capture stored as raw hex. What do these bytes represent?

```
48 65 6c 6c 6f 20 57 6f 72 6c 64 21 0a
```

**Expected output:**
```
Hello World!
```

> [!hint]- Hint
> ASCII — each byte is a character code. 'H' = 0x48, 'e' = 0x65.

> [!success]- Solution
> `48 65 6c 6c 6f` = "Hello", `20` = space, `6f 72 6c 64 21` = "World!", `0a` = newline

---

## P2: Convert Between Bases

Convert the following values between decimal, hex, and binary:

- `0xDEAD` → decimal and binary
- `48879` → hex
- `11010110` (binary) → hex and decimal

**Expected output:**
```
0xDEAD = 57005 decimal = 1101111010101101 binary
48879 = 0xBEEF
11010110 binary = 0xD6 = 214 decimal
```

> [!hint]- Hint
> 0xDEAD is a hex "word" — 4 hex digits. 0xD = 1101, 0xE = 1110, 0xA = 1010, 0xD = 1101.

> [!success]- Solution
> - `0xDEAD` = D(13)×16³ + E(14)×16² + A(10)×16¹ + D(13)×16⁰ = 53248+3584+160+13 = 57005
>   Binary: D=1101, E=1110, A=1010, D=1101 → `1101111010101101`
> - `48879` ÷ 16 = 3054 rem 15(F), 3054 ÷ 16 = 190 rem 14(E), 190 ÷ 16 = 11 rem 14(E), 11 = B → `0xBEEF`
> - `11010110` = 0xD6 = 128+64+4+2 = 214

---

## P3: Endianness Conversion

The bytes `67 45 23 01` are read from memory at address `0x1000`. Are they stored in big-endian or little-endian? What integer do they represent?

**Expected output:**
```
Little-endian: 0x01234567 = 19088743
Big-endian: 0x67452301 = 1734989569
```

> [!hint]- Hint
> Little-endian stores the least significant byte first. x86/x64 uses little-endian.

> [!success]- Solution
> - Little-endian: bytes `67 45 23 01` → integer `0x01234567` → 19088743
> - Big-endian: bytes `67 45 23 01` → integer `0x67452301` → 1734989569

---

## P4: Identify File Type by Magic Bytes

You receive these hex dumps. Identify the file type for each:

```
A: 7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
B: 89 50 4e 47 0d 0a 1a 0a
C: 4d 5a 90 00 03 00 00 00 04 00 00 00 ff ff 00 00
D: 50 4b 03 04 14 00 00 00 08 00
E: ff d8 ff e0 00 10 4a 46 49 46 00 01
```

**Expected output:**
```
A: ELF (Executable and Linkable Format)
B: PNG (Portable Network Graphics)
C: PE (Portable Executable, Windows)
D: ZIP (or DOCX/XLSX/Zip archive)
E: JPEG (JFIF)
```

> [!hint]- Hint
> Magic bytes at the start of files are unique identifiers. PNG always starts with `89 50 4e 47`. ELF starts with `7F 'E' 'L' 'F'`.

> [!success]- Solution
> - A: `7F 45 4C 46` = DEL + "ELF" → ELF (Linux/Unix executable)
> - B: `89 50 4E 47` = PNG signature
> - C: `4D 5A` = "MZ" (DOS header) → PE/EXE (Windows)
> - D: `50 4B` = "PK" → ZIP archive
> - E: `FF D8 FF` = JPEG SOI marker

---

## P5: Extract Strings

Run `strings` on a real binary. Use `file` to find a binary on your system, then extract all strings with length >= 6.

```bash
# Find a binary
file /bin/ls
strings -n 6 /bin/ls | head -30
```

**Expected output:**
```
(Your output will vary, but should show readable ASCII strings from the binary)
```

> [!hint]- Hint
> `strings` extracts sequences of printable ASCII characters. The `-n 6` flag requires at least 6 characters.

> [!success]- Solution
> ```bash
> file /bin/ls
> # /bin/ls: ELF 64-bit LSB executable, x86-64
> strings -n 6 /bin/ls | head -30
> #可能出现: /lib64/ld-linux-x86-64.so.2, libselinux.so.1, ...
> ```

---

## P6: Hex Dump Analysis

You are given a hex dump of an ELF header from offset 0. Use `readelf -h` to verify your analysis.

```bash
# Create a simple binary and hex dump it
echo 'int main(){return 0;}' > /tmp/test.c
gcc -o /tmp/test /tmp/test.c
xxd /tmp/test | head -10
readelf -h /tmp/test
```

**Expected output:**
```
Magic: 7f 45 4c 46 (ELF)
Class: 64-bit
Entry: 0x401000 (or similar)
```

> [!hint]- Hint
> ELF header starts at offset 0. The first 16 bytes are the ELF magic + class info.

> [!success]- Solution
> The ELF header shows: `7F ELF` at bytes 0-3, class 2 = 64-bit, data 1 = little-endian, type 2 = executable, entry point address.

---

## P7: Binary Arithmetic in Hex

Perform these hex calculations by hand, then verify with Python:

- `0xFF + 0x01`
- `0x10 * 0x10`
- `0x100 - 0xFF`
- `0xDEAD + 0xBEEF`

**Expected output:**
```
0xFF + 0x01 = 0x100
0x10 * 0x10 = 0x100
0x100 - 0xFF = 0x1
0xDEAD + 0xBEEF = 0x19D9C (decimal 105164)
```

> [!hint]- Hint
> Hex arithmetic follows the same rules as decimal, base 16. `0xF + 0x1 = 0x10` (carry at 16).

> [!success]- Solution
> ```python
> print(hex(0xFF + 0x01))   # 0x100
> print(hex(0x10 * 0x10))   # 0x100
> print(hex(0x100 - 0xFF))  # 0x1
> print(hex(0xDEAD + 0xBEEF))  # 0x19d9c
> ```

---

## P8: Analyze PE Header from Hex

You captured this hex dump from a Windows executable at offset 0x100 (right after the DOS header):

```
4d 5a 90 00 03 00 00 00 04 00 00 00 ff ff 00 00
b8 00 00 00 00 00 00 00 40 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 d0 00 00 00
0e 1f ba 0e 00 b4 09 cd 21 b8 01 4c cd 21 54 68
69 73 20 70 72 6f 67 72 61 6d 20 6d 75 73 74 20
62 65 20 72 75 6e 20 69 6e 20 44 4f 53 20 6d 6f
64 65 2e 0d 0d 0a 24 00 00 00 00 00 00 00 50 45
```

What is the PE entry point RVA? (Look at offset 0x18 within this PE header block, it's a 4-byte field)

**Expected output:**
```
Entry point RVA: 0x4000 (at file offset 0x118)
```

> [!hint]- Hint
> PE header starts with "PE\0\0" (0x50 0x45 0x00 0x00). The Entry Point RVA is at offset 0x28 within the COFF header. The value `40 00 00 00` in little-endian is `0x40`.

> [!success]- Solution
> The PE header starts at offset 0x100. "PE\0\0" at bytes 0x104-0x107. Entry Point RVA is at offset 0x108 (0x28 into PE header) = `40 00 00 00` = 0x4000.

---

## P9: ELF Program Header Analysis

Compile and analyze a simple binary:

```bash
echo 'int main(){return 0;}' > /tmp/simple.c
gcc -o /tmp/simple /tmp/simple.c
readelf -l /tmp/simple
```

Identify the LOAD segments. How many bytes of each segment are loaded?

**Expected output:**
```
Type: PT_LOAD (multiple)
You should see at least 2 PT_LOAD segments — one for code (.text), one for data
```

> [!hint]- Hint
> PT_LOAD segments are mapped into virtual memory. File offset + virtual address tell you where bytes go.

> [!success]- Solution
> `readelf -l /tmp/simple` shows Program Headers:
> - Type `PT_LOAD` — maps `.text` (code) into memory at `0x401000`
> - Type `PT_LOAD` — maps `.data` and `.bss` into memory at `0x404000`
> The `readelf -l` output shows `FileSiz` and `MemSiz` for each segment.

---

## P10: Calculate File Offset from Virtual Address

In an ELF binary with the following layout:

```
Program header at file offset 0x40:
  Offset: 0x00000   VirtAddr: 0x04000   FileSiz: 0x1000
```

If a function is at virtual address `0x401234`, what is the corresponding file offset?

**Expected output:**
```
File offset: 0x1234
```

> [!hint]- Hint
> For PT_LOAD segments, `file_offset = virtual_address - virtaddr + offset`

> [!success]- Solution
> `file_offset = (0x401234 - 0x4000) + 0 = 0x1234`
> Formula: `offset = vaddr - virtaddr + offset = 0x401234 - 0x4000 + 0 = 0x1234`

---

## P11: Stripped vs Non-Stripped Binary

Create two binaries, one stripped and one not, then compare:

```bash
echo 'int main(){return 0;}' > /tmp/test.c
gcc -o /tmp/test_ns /tmp/test.c
gcc -s -o /tmp/test_s /tmp/test.c
file /tmp/test_ns /tmp/test_s
nm /tmp/test_ns | head -10
nm /tmp/test_s | head -10
```

**Expected output:**
```
test_ns: not stripped
test_s: stripped (no symbols)
nm on test_s should show "no symbols"
```

> [!hint]- Hint
> `gcc -s` invokes `strip`, removing the symbol table. `nm` reads the symbol table.

> [!success]- Solution
> - `file test_ns` → `not stripped`
> - `file test_s` → `stripped`
> - `nm test_ns` shows symbols like `main`, `_start`
> - `nm test_s` shows `nm: test_s: no symbols`

---

## P12: Entropy Analysis

You have a compressed file and an uncompressed file. Use Python to calculate Shannon entropy for each byte position (simplified):

```python
import math
def entropy(data):
    freq = [0] * 256
    for b in data: freq[b] += 1
    ent = 0
    for f in freq:
        if f: ent -= (f/len(data)) * math.log2(f/len(data))
    return ent

# Normal text (low entropy ~4.5)
text = b"hello world hello world" * 100
# Compressed (high entropy ~7.9)
import zlib
compressed = zlib.compress(text)
print(f"Text entropy: {entropy(text):.2f}")
print(f"Compressed entropy: {entropy(compressed):.2f}")
```

**Expected output:**
```
Text entropy: ~4.5
Compressed entropy: ~7.9
```

> [!hint]- Hint
> High entropy (close to 8.0) indicates compressed or encrypted data. Low entropy indicates structured/plain data.

> [!success]- Solution
> Text has predictable patterns → lower entropy (~4-5 bits/byte). Compressed data looks random → higher entropy (~7.5-8.0 bits/byte).

---

## P13: Parse a Simple Binary Structure

Write a Python script that reads the first 64 bytes of `/bin/true` and prints:
1. Magic bytes (4 bytes)
2. Class (1 byte)
3. Endianness (1 byte)
4. Version (1 byte)
5. OS/ABI (1 byte)
6. Entry point address (8 bytes, little-endian for 64-bit)

```python
import struct

with open("/bin/true", "rb") as f:
    data = f.read(64)

magic = data[0:4]
e_class = data[4]
e_endian = data[5]
e_version = data[6]
e_osabi = data[7]
e_entry = struct.unpack("<Q", data[24:32])[0]  # little-endian u64

print(f"Magic: {magic}")
print(f"Class: {'64-bit' if e_class == 2 else '32-bit'}")
print(f"Endian: {'Little' if e_endian == 1 else 'Big'}")
print(f"Entry: 0x{e_entry:016x}")
```

**Expected output:**
```
Magic: b'\x7fELF'
Class: 64-bit
Endian: Little
Entry: 0x0000000000002000 (or similar)
```

> [!hint]- Hint
> ELF header fields are at fixed offsets. Magic at 0, class at 4, entry point at 24 (for 64-bit ELF).

> [!success]- Solution
> The script above parses the ELF header. For `/bin/true` you should see `0x7fELF`, 64-bit class, little-endian, and an entry point around `0x2000` or similar.

---

## P14: Use LLM to Analyze Unknown Hex

You encounter this hex data in a binary:

```
0a 13 4f 42 4a 45 43 54 5f 48 41 4e 44 4c 45 52
01 01 01 01 01 01 01 01 01 01 01 01 01 01 01 01
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
01 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00
```

Use an LLM with this prompt:

```
PROMPT: "Analyze this hex dump. Identify any magic bytes, strings, or structures.
What type of data or file format might this be? Explain your reasoning:
[hex data above]"
```

**Expected output:**
```
(LLM response identifying OBJECT_BALLET_HANDLER signature, repeating 01 byte pattern,
and data structure fields — depending on what the actual data represents)
```

> [!hint]- Hint
> Look for magic bytes at the start, repeating patterns, and null padding that suggests struct alignment.

> [!success]- Solution
> Example analysis: The bytes `0a 13 4f 42 4a 45 43 54 5f 48 41 4e 44 4c 45 52` decode to ASCII "OBJECT_BALLET_HANDLER" — likely a magic signature. The repeating `01` bytes suggest flags or bitfields. Null padding suggests alignment.

---

## AI Power-Up: Binary Fundamentals

```
PROMPT: "I'm analyzing an unknown binary format. Here is the hex dump of the
first 128 bytes. Identify the file type, extract any strings, and hypothesize
the structure of the header:
[hex dump]"

PROMPT: "A binary I am analyzing has two versions that differ in size. Version
1 is 12KB and version 2 is 8KB. The only difference I can see in the hex is
at offset 0x100. What might this indicate?"
```
