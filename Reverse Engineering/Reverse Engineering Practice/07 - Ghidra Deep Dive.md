---
title: "RE Practice: Ghidra Deep Dive"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - ghidra
  - decompiler
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Ghidra Deep Dive

12 progressive problems covering Ghidra decompilation, scripting, and analysis techniques.

---

## P1: Import and Analyze a Binary

```bash
# Launch Ghidra headless to analyze our test binary
mkdir -p /tmp/ghidra_proj
analyzeHeadless /tmp/ghidra_proj -import /tmp/simple -analysisTimeout 60
```

Open the result in Ghidra GUI and:
1. Browse the Symbol Tree → Functions → find `square` and `main`
2. Double-click `main` to see the decompiled output
3. Note the function signature and local variables

**Expected output:**
```
main: int main(void) { return square(5); }
square: int square(int x) { return x * x; }
```

> [!hint]- Hint
> Ghidra auto-analyzes on import. Use the CodeBrowser window. Press `Tab` to toggle between disassembly and decompiler view.

> [!success]- Solution
> Ghidra decompiles to clean C code. The function signatures and local variables are extracted from debug info or inferred from code patterns.

---

## P2: Rename and Annotate

In Ghidra, perform these operations on the `square` function:

1. Rename `x` (parameter) to `input_value` — press `L` with cursor on the variable
2. Add a comment: `// This function squares the input` — press `;`
3. Rename the function to `square_value` — press `L` on the function name
4. Export the changes

What did the decompiler show before vs after?

**Expected output:**
```
Before: int square(int x) { return x * x; }
After:  int square_value(int input_value) { return input_value * input_value; }
```

> [!hint]- Hint
> `L` = rename (symbol). `;` = comment. `Tab` = toggle decompiler/disassembly. `Esc` = cancel.

> [!success]- Solution
> Renaming makes the decompilation more readable. The actual machine code is unchanged — this is purely for the analyst's convenience in the Ghidra database.

---

## P3: Define a Structure

You encounter this struct in a binary:

```c
struct packet {
    int type;        // offset 0, 4 bytes
    int length;     // offset 4, 4 bytes
    char *data;     // offset 8, 8 bytes (pointer)
    int checksum;   // offset 16, 4 bytes
};
```

In Ghidra:
1. Open Data Type Manager
2. Create a new Structure
3. Add fields with correct offsets and data types
4. Apply it to a memory location in the binary

```bash
# Or use Ghidra Script:
# DataTypeManager dtm = currentProgram.getDataTypeManager();
# Structure packet = new StructureDataType("packet", 0);
# packet.add(new IntegerDataType(), 4, "type", "");
# etc.
```

**Expected output:**
```
Structure defined in Data Type Manager
Applied to memory locations reveals field names
```

> [!hint]- Hint
> Right-click in Data Type Manager → New → Structure. Then add fields with correct sizes. Apply by selecting bytes in the listing and pressing `T`.

> [!success]- Solution
> Defining structures turns raw hex into meaningful named fields. When you apply a struct to a pointer dereference, the decompiler shows `packet->type` instead of `*(int*)(ptr+0)`.

---

## P4: Analyze a Function with Multiple Return Paths

Create and analyze:

```bash
cat > /tmp/multi.c << 'EOF'
int classify(int x) {
    if (x < 0) return -1;
    if (x == 0) return 0;
    if (x < 10) return 1;
    return 2;
}
EOF
gcc -O0 -o /tmp/multi /tmp/multi.c
# Analyze in Ghidra
```

How does Ghidra's decompiler handle the multiple return paths? What does it show about the control flow?

**Expected output:**
```
Ghidra decompilation shows cascaded if-else returning -1, 0, 1, or 2
The decompiler preserves the branching logic as clean if statements
```

> [!hint]- Hint
> Ghidra's decompiler handles multiple returns well. It shows the logic as a series of if-else statements, sometimes simplified to a switch or min/max expression.

> [!success]- Solution
> Ghidra produces:
> ```c
> int classify(int x) {
>   if (x < 0) return -1;
>   if (x == 0) return 0;
>   if (x < 10) return 1;
>   return 2;
> }
> ```
> The decompiler preserves the branching structure cleanly.

---

## P5: Find Cross-References to a Function

In Ghidra:
1. Select `printf` in the Symbol Tree
2. Right-click → "Find References to" → "References to printf"
3. List all places where `printf` is called

How many call sites does it find? What strings are passed to `printf`?

**Expected output:**
```
Shows all CALL instructions referencing printf's PLT/GOT entry
Lists calling functions and the strings used as format arguments
```

> [!hint]- Hint
> XREFs (cross-references) are listed in the Symbol Tree when you select a symbol. The decompiler view also shows references with clickable links.

> [!success]- Solution
> All call sites are found. The references show: which function calls `printf`, what format string is used, and what arguments are passed. This is useful for finding interesting behavior (e.g., error messages, logging, etc.).

---

## P6: Write a Ghidra Script — Auto-Rename by Pattern

Write a Ghidra script that renames all functions containing "str" in their name to something more descriptive:

```java
import ghidra.app.script.GhidraScript;

public class RenameStringFunctions extends GhidraScript {
    public void run() throws Exception {
        for (Function func : getAllFunctions(true)) {
            String name = func.getName();
            if (name.contains("str") || name.contains("str_")) {
                println("Found: " + name);
                // Rename based on patterns
                if (name.contains("cmp")) {
                    func.setName("compare_" + name.hashCode(), SourceType.ANALYSIS);
                }
            }
        }
    }
}
```

```bash
# Run the script headlessly
analyzeHeadless /tmp/ghidra_proj -import /tmp/simple -scriptPath . \
  -postScript RenameStringFunctions.java
```

**Expected output:**
```
Found functions listed, renamed where patterns match
```

> [!hint]- Hint
> Ghidra scripts use Java or Python. The `getAllFunctions(true)` iterates all functions. `func.setName()` renames with a source type.

> [!success]- Solution
> The script iterates all functions, finds those with "str" in the name, and renames them based on additional patterns (like "cmp" → "compare"). This demonstrates programmatic analysis at scale.

---

## P7: Decompile with Ghidra — Loop Analysis

Create and analyze a loop:

```bash
cat > /tmp/loop.c << 'EOF'
int sum_array(int *arr, int len) {
    int sum = 0;
    for (int i = 0; i < len; i++) {
        sum += arr[i];
    }
    return sum;
}
EOF
gcc -O0 -o /tmp/loop /tmp/loop.c
# Analyze in Ghidra
```

How does Ghidra's decompiler show the loop? Does it recognize the pattern?

**Expected output:**
```
Decompilation shows a for loop with index variable i
Array access shown as arr[i] with proper type info
```

> [!hint]- Hint
> Ghidra's decompiler recognizes common loop idioms (for, while, do-while) and can represent them naturally.

> [!success]- Solution
> Ghidra produces:
> ```c
> int sum_array(int *arr, int len) {
>   int sum = 0;
>   for (int i = 0; i < len; i++) {
>     sum += arr[i];
>   }
>   return sum;
> }
> ```
> The loop structure is preserved. The pointer `arr` is recognized, and `arr[i]` is shown as array indexing with proper bounds awareness.

---

## P8: Side-by-Side x86 and ARM Analysis

Open both the x86 and ARM versions of the same binary in Ghidra (two separate projects). Compare:
1. Function signatures
2. Local variable storage
3. Register usage

```bash
# Compile both versions
gcc -o /tmp/x86_binary /tmp/simple.c
aarch64-linux-gnu-gcc -o /tmp/arm_binary /tmp/simple.c

# Analyze both in Ghidra
# Use Window > Diff to compare
```

What differences do you observe between the architectures?

**Expected output:**
```
Same C code produces different assembly but similar decompilation
ARM shows X0-X7 for args, x86 shows RDI, RSI, etc.
Local variables laid out differently
```

> [!hint]- Hint
> Ghidra's decompiler normalizes architecture differences. The decompiled output should look nearly identical despite the different machine code.

> [!success]- Solution
> Both decompilations show the same C-level logic: `square(int x)` returning `x*x`. The differences are in:
> - Argument registers (X0 vs RDI)
> - Return register (X0 vs RAX)
> - Stack frame layout
> - Instruction encoding

---

## P9: Decompile a Struct Function

Analyze a binary that passes structs by value:

```bash
cat > /tmp/struct_fn.c << 'EOF'
#include <string.h>

struct Point {
    int x;
    int y;
};

int distance(struct Point p) {
    return p.x * p.x + p.y * p.y;
}
EOF
gcc -O0 -o /tmp/struct_fn /tmp/struct_fn.c
# Analyze in Ghidra
```

How does Ghidra handle the struct passed by value? Does it use stack or registers?

**Expected output:**
```
x86-64: struct passed via registers (rdi=x, rsi=y) or on stack
Ghidra shows struct field access as p.x and p.y
```

> [!hint]- Hint
> Small structs (up to 16 bytes on x86-64) are passed in registers. Larger structs go on the stack.

> [!success]- Solution
> With -O0, the struct fields are stored on the stack at `[rbp-X]` offsets. Ghidra recognizes the struct layout and shows `p.x` and `p.y` field accesses rather than raw stack offsets.

---

## P10: Use Ghidra to Analyze a CTF Binary

Download a simple CTF binary (or use a provided one). Use Ghidra to:
1. Find the `main` function
2. Identify any comparison operations
3. Find interesting strings (Search → For Strings)
4. Trace from string references to the code that uses them
5. Identify the input needed to reach the "win" function

```bash
# Example: a simple crackme
# Try: https://github.com/antonio-b/ctf-challs/raw/master/reverse/kraken/kraken
```

**Expected output:**
```
String: "Password OK" or similar
Code checks input against expected value
Win function reachable when check passes
```

> [!hint]- Hint
> In CTF binaries, the "win" condition is often a separate function called only when the check passes. Find the comparison, trace to the conditional jump, and determine what input makes it take the win path.

> [!success]- Solution
> Standard CTF workflow:
> 1. Find strings → `printf("Enter password:")`
> 2. Find where the string is referenced → it's passed to `scanf`
> 3. Compare against hardcoded value
> 4. Identify the win function (often named `win`, `success`, `print_flag`)
> 5. Determine input that makes the check pass

---

## P11: Ghidra Python Script — Dump All Strings

```python
# Ghidra Python script to dump all strings
from ghidra.program.model.symbol import Symbol
from ghidra.util.task import ConsoleTaskMonitor

# Get all defined strings
listing = currentProgram.getListing()
data = listing.getDefinedData(True)

strings_found = []
for dataItem in data:
    if dataItem.isString():
        addr = dataItem.getAddress()
        val = dataItem.getValue()
        strings_found.append(f"{addr}: {val}")

for s in sorted(strings_found):
    println(s)
```

Save as `DumpStrings.py` and run headlessly:

```bash
analyzeHeadless /tmp/ghidra_proj -import /tmp/simple \
  -scriptPath . -postScript DumpStrings.java
```

**Expected output:**
```
Lists all strings in the binary with their addresses
```

> [!hint]- Hint
> Ghidra Python uses the same API as Java scripts. `currentProgram.getListing()` gets the code listing. `getDefinedData(True)` iterates all defined data.

> [!success]- Solution
> The script extracts all strings from the binary's defined data sections. For `/tmp/simple` (which uses libc), this would include libc strings. For real binaries, it reveals interesting strings like error messages, URLs, configuration data, etc.

---

## P12: Use LLM to Analyze Ghidra Decompilation

Take a Ghidra decompilation of an unknown function and ask an LLM to analyze it:

```
PROMPT: "Here is the Ghidra decompilation of an unknown function from a CTF
binary. Analyze and explain:
1. What does this function do?
2. What is the vulnerability (if any)?
3. What input would trigger the vulnerability?

Decompilation:
int process_input(char *buf, int len) {
    char temp[64];
    if (len > 64) return -1;
    memcpy(temp, buf, len);
    if (temp[0] == 'A' && temp[1] == 'B' && temp[2] == 'C') {
        ((void (*)(void))temp[8])();
    }
    return 0;
}"
```

**Expected output:**
```
(LLM identifies: buffer overflow + code execution via function pointer in temp buffer.
The attacker can overwrite temp[8] with a shellcode address or use ROP.)
```

> [!hint]- Hint
> The `memcpy` has a bounds check (len > 64), but the buffer is only 64 bytes. If len is exactly 64, bytes 64-71 (temp[0]-temp[7]) overwrite the saved return address. Also, `temp[8]` is cast to a function pointer and called!

> [!success]- Solution
> Two vulnerabilities:
> 1. `memcpy(temp, buf, len)` copies `len` bytes into 64-byte buffer. `len > 64` check uses unsigned int comparison, so `len = -1` (which is very large) passes the check!
> 2. `((void (*)(void))temp[8])()` — bytes at offset 8 are treated as a function pointer and called. An attacker can overwrite this with a shellcode address or use ROP.

---

## AI Power-Up: Ghidra

```
PROMPT: "I'm analyzing a malware sample with Ghidra. The decompiled main
function shows this. What is it likely doing? What are the IOCs (Indicators of
Compromise)?

[decompiled malware main function]"

PROMPT: "Write a Ghidra Python script that:
1. Finds all functions that call GetProcAddress
2. For each, extracts the hash or name parameter passed to it
3. Resolves the imported function name
4. Exports to JSON"
```
