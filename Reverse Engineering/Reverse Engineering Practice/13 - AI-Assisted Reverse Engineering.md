---
title: "RE Practice: AI-Assisted Reverse Engineering"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - ai
  - llm
  - opencode
  - mcp
  - automation
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: AI-Assisted Reverse Engineering

12 progressive problems using LLMs, OpenCode, and MCP integration for reverse engineering automation.

---

## P1: Use OpenCode to Analyze Disassembly

Run OpenCode with this prompt to analyze unknown assembly:

```
PROMPT: "Explain this x86-64 assembly function. For each instruction, describe
what it does. Then explain the overall function purpose. Suggest a better
function name based on the behavior. Also identify any security concerns.

Assembly:
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
(OpenCode/an LLM explains: checks if pointer's first field is non-zero,
loads function pointer from second field, calls it. Suggested name:
"invoke_if_valid" or similar. Security note: no null check on function pointer result.)
```

> [!hint]- Hint
> OpenCode and LLMs can process assembly and provide structured analysis with suggestions for naming and security concerns.

> [!success]- Solution
> The LLM correctly identifies:
> 1. Takes pointer in `rdi`, checks first dword at `[rdi]` against 0
> 2. If non-zero, loads function pointer from `[rdi+8]` and calls it
> 3. Pattern: optional callback invocation (common in event systems)
> 4. Security: `call rax` with no validation of `rax` after `mov rax, [rdi+8]` — potential call hijacking if `rax` could be controlled

---

## P2: Generate Ghidra Script with LLM

Prompt an LLM to write a Ghidra Python script:

```
PROMPT: "Write a Ghidra Python script that:
1. Finds all functions that reference the string 'password'
2. For each such function, prints its address, name, and decompiled output
3. Saves the results to a JSON file

Use the Ghidra Scripting API."
```

**Expected output:**
```python
# Ghidra Python script
import json
from ghidra.program.model.symbol import Symbol
from ghidra.app.script import GhidraScript

class FindPasswordFunctions(GhidraScript):
    def run(self):
        results = []
        for func in getAllFunctions(True):
            for ref in getReferencesTo(func.entryPoint):
                refAddr = ref.getFromAddress()
                # Check if any string references contain 'password'
                # ... collect results
        with open("password_funcs.json", "w") as f:
            json.dump(results, f, indent=2)
```

> [!hint]- Hint
> LLMs can generate code that uses Ghidra's API. The generated script may need minor fixes but provides a solid starting point.

> [!success]- Solution
> The LLM generates a Ghidra Python script that uses the `GhidraScript` base class, iterates functions, finds references to strings containing "password", and exports results to JSON.

---

## P3: Automate Crackme Solving with LLM

You have a crackme binary. Use an LLM to analyze the solution path:

```
PROMPT: "Here is the Ghidra decompilation of a crackme's check function.
Analyze it and tell me what input string would make it return 1 (success):

Decompilation:
int check(char *input) {
    char computed[32];
    int sum = 0;
    for (int i = 0; input[i]; i++) {
        sum += input[i];
    }
    if (sum == 1337) {
        return 1;
    }
    return 0;
}
```

**Expected output:**
```
(LLM: The function sums all ASCII values of input characters.
It returns 1 if the sum equals 1337.
Many inputs satisfy this (e.g., 'A' = 65, so 20*'A' = 1300, need 37 more.)
```

> [!hint]- Hint
> The crackme checks if the sum of ASCII values equals 1337. The LLM identifies this constraint.

> [!success]- Solution
> The constraint is: `sum(input[i]) = 1337`. Many solutions exist:
> - `'A'` (65) × 20 = 1300, need 37 more (e.g., `'&'`=38) → "AAAAAAAAAAAAAAAAAAAA&"
> - Or any combination summing to 1337
> The LLM identifies the check and provides a solving strategy.

---

## P4: Use LLM to Understand Obfuscated Code

```
PROMPT: "This decompiled function is heavily obfuscated with control flow
flattening. Simplify it into readable C pseudocode. Identify the actual logic
(the real if/else structure, loops, etc.):

[Obfuscated decompiled code with dispatcher loop and switch]"
```

**Expected output:**
```
(LLM simplifies the flattened control flow back to structured if-else.
Identifies: main condition, branches, and the actual computation.)
```

> [!hint]- Hint
> LLMs can work backwards from the flattened dispatch pattern to reconstruct the original structured logic.

> [!success]- Solution
> The LLM identifies:
> 1. The state variable controlling the dispatch
> 2. Which basic blocks correspond to which original branches
> 3. The original conditional logic
> 4. Outputs simplified pseudocode

---

## P5: Write a Deobfuscation Script

Ask an LLM to write a deobfuscation script for XOR strings:

```
PROMPT: "Write a Python script that:
1. Reads a binary file
2. Scans for encrypted strings at a known offset (0x403000)
3. Decrypts them using single-byte XOR with key 0x55
4. Patches the binary by replacing encrypted bytes with decrypted bytes
5. Reports all decrypted string addresses
"
```

**Expected output:**
```python
import sys

def decrypt_xor(data, key=0x55):
    return bytes(b ^ key for b in data)

def deobfuscate_strings(binary_path, offset, count):
    with open(binary_path, 'r+b') as f:
        results = []
        for i in range(count):
            addr = offset + i * 64  # 64 bytes per string
            f.seek(addr)
            encrypted = f.read(64)
            # Find null terminator
            null_pos = encrypted.find(b'\x00')
            if null_pos > 0:
                encrypted = encrypted[:null_pos]
            decrypted = decrypt_xor(encrypted)
            print(f"0x{addr:08X}: {decrypted}")
            results.append((addr, decrypted))
        return results
```

> [!hint]- Hint
> LLMs can generate practical deobfuscation scripts that work directly on binary files.

> [!success]- Solution
> The LLM generates a working Python script that decrypts XOR-obfuscated strings and patches the binary in-place.

---

## P6: Analyze Memory Dump with LLM

```
PROMPT: "I used GDB to dump 256 bytes of memory at address 0x7fff0000.
Here is the hex dump. Analyze it and tell me:
1. What data structures do you see?
2. Are there any recognizable strings or patterns?
3. What type of data might this be?

[hex dump of memory region]"
```

**Expected output:**
```
(LLM identifies potential data structures, strings, pointers, and suggests
what memory region this might be — stack, heap, global data, etc.)
```

> [!hint]- Hint
> LLMs can analyze raw memory dumps and identify patterns, strings, and data structures.

> [!success]- Solution
> The LLM analyzes the hex dump, identifies:
> - ASCII strings if present
> - Pointer values (addresses that look like valid memory locations)
> - Repeated patterns (arrays, buffers)
> - Suggests the memory region type (stack frame, heap allocation, etc.)

---

## P7: Set Up MCP for GDB Integration

Configure OpenCode or Claude to use GDB via MCP:

```bash
# Example MCP server setup for GDB
# (requires mcp-server-gdb or similar)

# Configuration in Claude Code or OpenCode:
# {
#   "mcpServers": {
#     "gdb": {
#       "command": "python3",
#       "args": ["-m", "mcp_gdb", "--port", "8765"]
#     }
#   }
# }

# Once connected, you can:
# - Read registers via AI
# - Set breakpoints via AI
# - Analyze memory via AI
```

How would you use an MCP-connected GDB?

```
PROMPT: "Use the connected GDB to set a breakpoint at the 'check_license'
function. Then run the binary with input 'AAAABBBB'. When it hits the
breakpoint, show me all registers and the top 20 bytes of the stack."
```

**Expected output:**
```
(With MCP: AI sends GDB commands, receives results, analyzes them.
Shows register state and stack at the breakpoint.)
```

> [!hint]- Hint
> MCP (Model Context Protocol) connects AI agents to external tools. GDB MCP server lets the AI control debugging.

> [!success]- Solution
> With MCP-connected GDB:
> 1. AI sends `break check_license` via GDB
> 2. AI sends `run AAAABBBB`
> 3. AI receives breakpoint hit notification
> 4. AI sends `info registers` and `x/20x $rsp`
> 5. AI analyzes the output and explains the state

---

## P8: Generate IDAPython Script

```
PROMPT: "Write an IDAPython script (for IDA Pro) that:
1. Iterates all functions in the IDB
2. For each function, checks if it references any of these APIs: GetProcAddress, LoadLibraryA, VirtualAlloc
3. If found, prints the function name and the API call
4. Creates a list of potential runtime-linked functions"
```

**Expected output:**
```python
import idaapi
import idautils
import idc

for func in idautils.Functions():
    func_name = idc.get_func_name(func)
    for ref in idautils.XrefsTo(func):
        if ref.type == fl_CN or ref.type == fl_CF:
            target = idc.get_name(ref.to)
            if target in ['GetProcAddress', 'LoadLibraryA', 'VirtualAlloc']:
                print(f"{func_name} calls {target}")
```

> [!hint]- Hint
> LLMs can generate IDAPython scripts that use the IDA SDK APIs to analyze binaries.

> [!success]- Solution
> The LLM generates an IDAPython script that uses `idautils.Functions()`, `idautils.XrefsTo()`, and `idc.get_name()` to find functions that dynamically resolve APIs.

---

## P9: Automated Vulnerability Analysis

```
PROMPT: "I'm analyzing a binary for vulnerabilities. Here is the Ghidra
decompilation of a function that handles user input. Check for:
1. Buffer overflow (strcpy, sprintf, gets without bounds)
2. Format string vulnerability (printf with user-controlled format)
3. Integer overflow
4. Use-after-free (pointers used after free)
5. Command injection (system, exec with user input)

Decompilation:
void process_input(char *input) {
    char buf[64];
    strcpy(buf, input);
    sprintf(buf, input);  // second use of buf
    printf(buf);
}"
```

**Expected output:**
```
(LLM identifies:
1. strcpy: no bounds check on input — buffer overflow if input > 63 chars
2. sprintf(buf, input): second overflow if input contains % specifiers
3. printf(buf): format string vulnerability — %n can write to memory)
```

> [!hint]- Hint
> LLMs can identify multiple vulnerability classes in decompiled code.

> [!success]- Solution
> Three vulnerabilities identified:
> 1. `strcpy(buf, input)` — input unbounded, overflow if > 63 chars
> 2. `sprintf(buf, input)` — if input has format specifiers, writes beyond buffer
> 3. `printf(buf)` — direct format string (user controls format, could use `%n`)

---

## P10: Reverse Engineer a Protocol with LLM Assistance

```
PROMPT: "I captured these network packets from an unknown protocol. Each
packet is shown as hex bytes. Analyze the structure:

Packet 1 (client->server): 01 00 01 00 0A 00 00 00 48 65 6C 6C 6F 00
Packet 2 (server->client): 01 00 02 00 0D 00 00 00 57 6F 72 6C 64 21 0A 00
Packet 3 (client->server): 02 00 03 00 05 00 00 00 54 65 73 74

Identify:
1. The packet header structure
2. Message types
3. Length fields
4. How the server response relates to client message
"
```

**Expected output:**
```
(LLM infers:
Header: [1 byte: msg_type][2 bytes: unknown][2 bytes: length][payload]
Type 01 = message, Type 02 = command
Length field at offset 4
Server echoes back client's length but different content)
```

> [!hint]- Hint
> LLMs can identify patterns across multiple packets and infer protocol structure.

> [!success]- Solution
> LLM identifies:
> ```
> [1 byte: type][2 bytes: seq?][2 bytes: length][payload]
> ```
> Type 01 = string message, 02 = command. Server responds with type 01. The sequence number increments (01 → 02 → 03). Payload length matches the length field.

---

## P11: Generate Decompiler Comments

Use an LLM to add explanatory comments to decompiled code:

```
PROMPT: "Add detailed comments to this decompiled code explaining what each
section does. Also add a header comment explaining the function's purpose:

void process_packet(byte *data, int len) {
    byte type = data[0];
    int *state = (int*)(data + 4);
    void (*handler)(int) = (void(*)(int))data[8];
    if (*state == 1) {
        handler(*state);
    }
    memcpy(g_buf, data + 12, len - 12);
}"
```

**Expected output:**
```c
// process_packet - handles incoming network packets
//   data: raw packet bytes
//   len: total packet length
//   type: extracted from byte 0 (packet type)
//   state: pointer to global state (4 bytes at offset 4)
//   handler: function pointer extracted from offset 8, called if state==1
//   remaining bytes copied to global buffer for processing
```

> [!hint]- Hint
> LLMs can add context, type information, and explanation to decompiled code.

> [!success]- Solution
> The LLM adds comprehensive comments:
> - Function header: purpose and parameters
> - Each line: detailed explanation of what it does and why
> - Security notes: handler is called without validation (potential issue)
> - Data flow: how bytes are extracted and used

---

## P12: Compare Binaries with LLM

```
PROMPT: "I have two versions of the same binary (v1.0 and v1.1). Here are
the functions that differ between them. Version 1.1 has a security fix.
Analyze the differences and identify what vulnerability was likely fixed:

v1.0 check_password:
  char buf[64];
  strcpy(buf, user_input);
  if (strcmp(buf, "secret") == 0) return 1;
  return 0;

v1.1 check_password:
  char buf[64];
  strcpy(buf, user_input);  // still vulnerable!
  if (strcmp(buf, "secret") == 0) return 1;
  return 0;
  
Wait, both look the same... check the other functions..."
```

**Expected output:**
```
(LLM: The check_password function looks identical. Let me check other changed functions...
It finds a new function in v1.1 that validates input length before calling check_password.
The vulnerability was not in check_password itself but in a caller that didn't validate input length.)
```

> [!hint]- Hint
> Sometimes the fix isn't in the obvious function. LLMs can compare across all changed functions.

> [!success]- Solution
> The LLM notices that `check_password` itself looks unchanged, then investigates other changed functions. It finds a new validation function in v1.1 that checks input length before calling `check_password`, revealing that the actual fix was adding length validation at the caller level.

---

## AI Power-Up: Advanced AI RE Workflows

```
PROMPT: "I want to automate my RE workflow using OpenCode. Create a workflow
that:
1. Opens a binary in Ghidra and auto-analyzes it
2. Exports all decompiled functions to a JSON file
3. Feeds each function to an LLM for analysis
4. Collects vulnerability findings and suspicious patterns
5. Generates a report

Show me the OpenCode agent configuration and the prompt chain."
```

**Expected output:**
```
(LLM outlines a multi-step OpenCode workflow with:
- Tool calls to run analyzeHeadless Ghidra
- Python script to export decompiled functions
- LLM analysis of each function
- Result aggregation
- Report generation)
```

> [!hint]- Hint
> OpenCode can orchestrate complex multi-step RE workflows using tool calls and chained prompts.

> [!success]- Solution
> The LLM outlines a comprehensive automated RE pipeline:
> 1. `ghidra.analyze()` headlessly
> 2. Export via Ghidra Python script
> 3. For each function, call LLM with analysis prompt
> 4. Aggregate findings
> 5. Generate markdown report

---

## AI Power-Up: Final Practice

```
PROMPT: "You are a reverse engineering assistant. I will give you various
binary RE artifacts (disassembly, decompilation, hex dumps, strace output,
memory layouts) and you will:
1. Explain what you see
2. Identify key functions and their purpose
3. Note security implications
4. Suggest next steps for analysis
5. Write automation scripts when relevant

Ready for your first artifact. Provide any binary RE output and I will analyze it."
```
