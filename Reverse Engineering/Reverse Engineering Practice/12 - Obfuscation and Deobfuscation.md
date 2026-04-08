---
title: "RE Practice: Obfuscation and Deobfuscation"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - obfuscation
  - deobfuscation
  - xor
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Obfuscation and Deobfuscation

12 progressive problems covering XOR encryption, control flow flattening, string decryption, and script-based deobfuscation.

---

## P1: Identify XOR Obfuscation Pattern

You find these bytes in the binary at a location referenced by a decryption function:

```
Encrypted: 0x1F 0x15 0x12 0x1A 0x5D 0x5F 0x58 0x12
Expected plaintext: "password"
```

What is the XOR key?

**Expected output:**
```
0x1F ^ 'p'(0x70) = 0x6F
0x15 ^ 'a'(0x61) = 0x74
0x12 ^ 's'(0x73) = 0x61
...
Key: 0x6F 0x74 0x61 ... or single byte?
```

> [!hint]- Hint
> If the same key byte is used repeatedly, the XOR is single-byte. Try XORing each byte with the corresponding ASCII character of the known plaintext.

> [!success]- Solution
> - `0x1F ^ 'p'(0x70)` = `0x6F` = 'o'
> - `0x15 ^ 'a'(0x61)` = `0x74` = 't'
> - `0x12 ^ 's'(0x73)` = `0x61` = 'a'
> - `0x1A ^ 's'(0x73)` = `0x69` = 'i'
> - ...
> Looks like repeating key "otai" (4 bytes) or single byte key varies.
> If all equal: `0x1F ^ 0x70 = 0x6F` but `0x15 ^ 0x61 = 0x74` — different, so multi-byte key.

---

## P2: Single-Byte XOR Brute Force

Write a script to brute-force a single-byte XOR:

```python
def brute_xor(encrypted: bytes) -> list:
    results = []
    for key in range(256):
        decrypted = bytes(b ^ key for b in encrypted)
        # Check if result looks like ASCII text
        if all(0x20 <= c < 0x7F or c in [0x0A, 0x0D] for c in decrypted):
            results.append((key, decrypted))
    return results

encrypted = bytes.fromhex("1F15121A5D5F5812")
for key, dec in brute_xor(encrypted):
    print(f"Key=0x{key:02X}: {dec}")
```

**Expected output:**
```
One key produces readable output: "password"
```

> [!hint]- Hint
> If the plaintext is ASCII, brute-forcing single-byte XOR will reveal it as the only result that produces printable ASCII.

> [!success]- Solution
> Running the script: the key that produces valid ASCII output is the correct key. Any other key produces garbage bytes (non-ASCII). This is a classic single-byte XOR cipher.

---

## P3: Multi-Byte XOR Key Recovery

```python
def find_multibyte_key(encrypted: bytes, plaintext: bytes) -> bytes:
    return bytes(e ^ p for e, p in zip(encrypted, plaintext))

# Known: plaintext starts with "Hello, "
encrypted = bytes.fromhex("4F 4A 48 48 5B 3F 20 57 ...")
key = find_multibyte_key(encrypted[:7], b"Hello, ")
print(f"Key: {key}")
```

**Expected output:**
```
Key: b"Secret"
```

> [!hint]- Hint
> Known plaintext attack: XOR the known plaintext bytes with the encrypted bytes to recover the key.

> [!success]- Solution
> If you know the beginning of the plaintext (e.g., from dynamic analysis or a leaked configuration), XORing it with the ciphertext reveals the key. For "Hello, " (7 bytes), you recover 7 bytes of the key.

---

## P4: Deobfuscate Strings in Ghidra Script

Write a Ghidra Python script to decrypt strings:

```python
# Ghidra Python script
# Assumes XOR key is 0x55, strings are at address 0x00403000

addr = 0x00403000
key = 0x55
decrypted = []

for i in range(100):  # up to 100 bytes
    b = getByte(toAddr(addr + i))
    if b == 0:  # null terminator
        break
    decrypted.append(chr(b ^ key))

print("Decrypted:", "".join(decrypted))
```

Run it in Ghidra (Window → Script Manager).

**Expected output:**
```
Decrypted string printed in console
```

> [!hint]- Hint
> Ghidra scripts have access to `getByte()`, `toAddr()`, and other APIs. The script manager lets you run Python/Java scripts against the loaded binary.

> [!success]- Solution
> The script reads encrypted bytes, XORs each with the key, and prints the decrypted string. For more complex encryption, add decryption logic (loop, key schedule, etc.).

---

## P5: Detect Control Flow Flattening

You see this pattern in a function's CFG (visual graph):

```
+------+
| B0   |  <- entry
+------+
   |
   v
+------+
| B1   |  <- switch variable init
+------+
   |
   v
+------+
| B2   |  <- dispatch
+------+
   |
 +--+--+--+
 |  |  |
 v  v  v
+--+--+--+
|B3|B4|B5|  <- all basic blocks jump to dispatch
+------+--+
   |
   v
+------+
| B6   |  <- exit
+------+
```

What does this tell you?

**Expected output:**
```
Control flow flattening — natural if-else tree replaced with loop+switch.
Original hierarchical CFG turned into flat dispatch pattern.
```

> [!hint]- Hint
> A natural if-else tree has a branching structure. This pattern is a flat loop with a switch/d dispatch. It's a signature of obfuscation.

> [!success]- Solution
> Control flow flattening shows:
> - A dispatcher loop at B2 checks a state variable
> - All basic blocks (B3, B4, B5) jump back to the dispatcher
> - This is the flattened form of what was originally a structured if-else or switch
> Tools like `angr`'s CFG can detect this pattern automatically

---

## P6: Deobfuscate String Encryption with Python

Extract and decrypt all encrypted strings from a binary:

```python
import subprocess

def get_encrypted_strings(binary_path, address_range):
    """Use objdump/strings to find encrypted data"""
    result = subprocess.run(
        ['strings', '-n', '4', binary_path],
        capture_output=True, text=True
    )
    # Filter for strings at specific offsets
    return [s for s in result.stdout.split('\n') if 'str' in s.lower()]

def decrypt_xor(data: bytes, key: int) -> bytes:
    return bytes(b ^ key for b in data)

# Extract encrypted bytes from binary at known offset
with open('/tmp/obfuscated_binary', 'rb') as f:
    f.seek(0x403000)  # known string table offset
    encrypted_data = f.read(256)

# Try each key 0-255
for key in range(256):
    dec = decrypt_xor(encrypted_data, key)
    # Check for readable ASCII
    if all(0x20 <= c < 0x7F or c in [0x0A, 0x0D, 0x00] for c in dec):
        print(f"Key {key}: {dec[:50]}")
```

**Expected output:**
```
Key that produces readable output is the correct key
Shows decrypted strings
```

> [!hint]- Hint
> If you don't know the key, try all 256 possible single-byte XOR keys. Only the correct key produces printable ASCII.

> [!success]- Solution
> The script brute-forces the XOR key by checking which one produces valid ASCII output. The correct key is identified as the one that makes the decrypted data look like normal text.

---

## P7: Unpack a Simple Packer

```python
import struct

def unpack_self(binary_path):
    """Simple self-extracting packer stub"""
    with open(binary_path, 'rb') as f:
        # Read packed data from end of file
        f.seek(-4, 2)  # go to 4 bytes before EOF
        packed_size = struct.unpack('<I', f.read(4))[0]
        
        f.seek(-4 - packed_size, 2)  # go to start of packed data
        packed_data = f.read(packed_size)
    
    # Simple XOR unpack
    key = 0x55
    decrypted = bytes(b ^ key for b in packed_data)
    return decrypted

data = unpack_self('/tmp/packed_binary')
with open('/tmp/unpacked.bin', 'wb') as f:
    f.write(data)
```

**Expected output:**
```
Unpacked binary written to file
Can be analyzed with objdump/Ghidra
```

> [!hint]- Hint
> Self-extracting packers store the compressed/encrypted original at a known location (e.g., end of file) and unpack at runtime.

> [!success]- Solution
> The simple self-extracting packer:
> 1. Reads packed data from the end of the file
> 2. Decrypts with XOR (key 0x55)
> 3. Writes the original binary
> Real packers use compression (zlib, LZMA) instead of just XOR.

---

## P8: Recognize VMProtect-Style Obfuscation

You encounter this in a malware binary:

```asm
; VM entry
vm_main:
    movzx eax, byte [rbp+0x10]      ; EIP = bytecode[PC]
    movzx ebx, byte [vm_handlers]    ; handler table base
    movzx ebx, byte [ebx+eax*4]     ; handler = handlers[opcode]
    jmp ebx                          ; dispatch

; Handlers look like:
handler_00:  ; ADD
    movzx eax, byte [rbp+0x12]
    movzx ebx, byte [rbp+0x14]
    add eax, ebx
    mov byte [rbp+0x16], al
    jmp vm_main
    
handler_01:  ; LOAD
    movzx eax, byte [rbp+0x12]      ; index
    mov ebx, [rbp+0x100]            ; memory base
    movzx eax, byte [ebx+eax]       ; mem[index]
    jmp vm_main
```

What does this indicate?

**Expected output:**
```
Custom virtual machine interprets bytecode.
Each "handler" is a VM instruction implementation.
Bytecode at rbp+0x10 is the program.
```

> [!hint]- Hint
> A dispatch loop + handler table + virtual state = custom VM. This is VMProtect-style protection.

> [!success]- Solution
> VMProtect-style VM:
> 1. Bytecode at `[rbp+0x10]` contains the protected program
> 2. `vm_handlers` is a table of handler addresses
> 3. Each handler implements one instruction (ADD, LOAD, STORE, etc.)
> 4. To RE: decode the bytecode, write a disassembler, understand the VM architecture

---

## P9: Decrypt API Hash Table

A binary uses hashed API names. The hash function is:

```c
int hash_api_name(const char *name) {
    int hash = 0;
    while (*name) {
        hash = (hash * 31) + *name;
        name++;
    }
    return hash;
}
```

You find these hashes in the import resolver:

```
0xA2D4E8F1  -> ?
0x5D8C3F2A  -> ?
0x3E1B9C4D  -> ?
```

Find which APIs they correspond to:

```python
import itertools

def hash_name(s):
    h = 0
    for c in s:
        h = (h * 31 + ord(c)) & 0xFFFFFFFF
    return h

# Brute force common API names
apis = ['GetProcAddress', 'LoadLibraryA', 'VirtualAlloc', 
        'CreateFileA', 'WriteFile', 'ExitProcess']

target_hashes = [0xA2D4E8F1, 0x5D8C3F2A, 0x3E1B9C4D]

for name in apis:
    h = hash_name(name)
    if h in target_hashes:
        print(f"{name}: 0x{h:08X}")
```

**Expected output:**
```
Matching API names printed for found hashes
```

> [!hint]- Hint
> Brute-force known API names against observed hashes. The hash function is deterministic.

> [!success]- Solution
> The script checks common API names against the observed hashes. If a match is found, we've identified the API. For unknown hashes, brute-force over common Windows API names.

---

## P10: Deobfuscation with Symbolic Execution (angr)

```python
import angr

# Load binary
p = angr.Project('/tmp/obfuscated_binary')

# Find the decryption function
cfg = p.analyses.CFGFast()
decrypt_func = cfg.functions.by_name('decrypt_strings')  # or find by address

# Create initial state
state = p.factory.blank_state(decrypt_func.addr)

# Run symbolic execution to find inputs that produce known outputs
# (more advanced: use angr's symbolic execution to simplify the program)
```

**Expected output:**
```
angr can symbolically execute the decryption function
Can find inputs that satisfy constraints
```

> [!hint]- Hint
> angr is a Python framework for binary analysis. It can perform symbolic execution to understand obfuscated code paths.

> [!success]- Solution
> angr's symbolic execution:
> 1. Loads the binary
> 2. Builds CFG
> 3. Executes functions symbolically
> 4. Can find inputs that reach specific code paths
> 5. Can simplify complex obfuscated logic

---

## P11: Detect Dead Code Injection

```bash
# Create binary with dead code
cat > /tmp/dead.c << 'EOF'
int main() {
    // Real code
    int x = 5;
    
    // Dead code (never executed)
    int y = 999;
    if (y == 0) {
        // unreachable
    }
    
    return x;
}
EOF
gcc -o /tmp/dead /tmp/dead.c

# Check with objdump
objdump -d /tmp/dead | grep -E "999|dead"

# Check for unreachable functions
objdump -d /tmp/dead | grep -B2 -A5 "<dead_func>:"
```

What does the disassembly show?

**Expected output:**
```
Dead code appears in disassembly but never executed
Unreachable functions present in binary
```

> [!hint]- Hint
> Dead code increases binary size without contributing to functionality. It's a common obfuscation technique.

> [!success]- Solution
> The dead code (y=999, if(y==0)) appears in the binary but never executes. This can confuse disassemblers and increase analysis time.

---

## AI Power-Up: Obfuscation

```
PROMPT: "I'm analyzing a malware sample that uses string encryption. The
decryption function is below. Write a Python script to:
1. Extract all encrypted string locations from the binary
2. Decrypt each string using the algorithm shown
3. Output a list of (address, decrypted_string) pairs

[decryption function disassembly]"

PROMPT: "This binary uses multi-byte XOR encryption with a repeating key. The
encrypted strings are at offset 0x403000. I know the plaintext for the first
string is 'kernel32.dll'. Extract the key and decrypt all strings at that
offset."
```
