---
title: "RE Workflow: Patch a License Check"
date: 2026-04-07
tags:
  - reverse-engineering
  - workflow
  - project
  - patching
  - license
  - crackme
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Workflow: Patch a License Check

## Overview

In this project, you will reverse engineer a binary with a software license check, identify the validation logic, and patch the binary to accept any license key.

**Scenario:** You have a trial version of a tool that expires after 30 days. You need to patch it to remove the expiration check.

**Concepts practiced:** Ghidra decompilation, function identification, conditional jump patching, hex editing, verification testing.

---

## Step-by-Step Implementation

### Step 1: Create the Target Binary

Create a binary with a license check:

```bash
cat > /tmp/licensed_app.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define LICENSE_KEY "TRIAL-2024-ABC123"

typedef struct {
    int is_licensed;
    int days_used;
    char license_key[64];
    time_t install_date;
} AppState;

AppState g_state = {0};

int verify_license(const char *key) {
    // Simulated license verification
    // In real software, this would check against a server or verify a signature
    if (key == NULL) return 0;
    
    // Simple string comparison
    if (strcmp(key, LICENSE_KEY) == 0) {
        return 1;
    }
    
    // Trial period check
    if (strncmp(key, "TRIAL-", 6) == 0) {
        // Parse trial expiration
        // For demo: any TRIAL- key is valid for 30 days from install
        return 1;
    }
    
    return 0;
}

int check_expiration() {
    time_t now = time(NULL);
    double days_elapsed = difftime(now, g_state.install_date) / (60 * 60 * 24);
    
    if (days_elapsed > 30) {
        printf("[ERROR] Trial period expired! Please purchase a license.\n");
        return 0;
    }
    
    if (days_elapsed > 20) {
        printf("[WARNING] Trial expires in %d days!\n", (int)(30 - days_elapsed));
    }
    
    return 1;
}

void show_license_info() {
    if (g_state.is_licensed) {
        printf("License: LICENSED (%s)\n", g_state.license_key);
    } else {
        printf("License: UNLICENSED (Trial: %d days used)\n", g_state.days_used);
    }
}

int activate_license(const char *key) {
    if (verify_license(key)) {
        g_state.is_licensed = 1;
        strncpy(g_state.license_key, key, sizeof(g_state.license_key) - 1);
        printf("[SUCCESS] License activated!\n");
        return 1;
    }
    printf("[ERROR] Invalid license key!\n");
    return 0;
}

int main(int argc, char **argv) {
    g_state.install_date = time(NULL);  // Set install date
    
    if (argc < 2) {
        printf("Usage: %s [activate <key> | info | use | expire-test]\n", argv[0]);
        return 1;
    }
    
    if (strcmp(argv[1], "activate") == 0) {
        if (argc < 3) {
            printf("Usage: %s activate <key>\n", argv[0]);
            return 1;
        }
        return activate_license(argv[2]) ? 0 : 1;
    }
    
    if (strcmp(argv[1], "info") == 0) {
        show_license_info();
        return 0;
    }
    
    if (strcmp(argv[1], "use") == 0) {
        if (!g_state.is_licensed) {
            if (!check_expiration()) {
                return 1;
            }
        }
        printf("Using premium feature...\n");
        printf("Feature X: COMPLETED\n");
        printf("Feature Y: COMPLETED\n");
        return 0;
    }
    
    if (strcmp(argv[1], "expire-test") == 0) {
        // For testing: set install date to 31 days ago
        g_state.install_date = time(NULL) - (31 * 24 * 60 * 60);
        printf("Simulating expired trial...\n");
        return check_expiration() ? 0 : 1;
    }
    
    printf("Unknown command: %s\n", argv[1]);
    return 1;
}
EOF

gcc -O0 -o /tmp/licensed_app /tmp/licensed_app.c
echo "Binary created:"
file /tmp/licensed_app
```

### Step 2: Initial Analysis

```bash
# Check strings
strings /tmp/licensed_app | grep -iE "(license|error|trial|expire|valid)"

# Run it
/tmp/licensed_app info
/tmp/licensed_app activate TRIAL-2024-XYZ789
/tmp/licensed_app use
/tmp/licensed_app expire-test
```

### Step 3: Load into Ghidra

```bash
mkdir -p /tmp/ghidra_license
analyzeHeadless /tmp/ghidra_license -import /tmp/licensed_app -analysisTimeout 60
```

In Ghidra, analyze:
1. Find `main`
2. Find `verify_license`
3. Find `check_expiration`

### Step 4: Analyze the License Check

Find the `verify_license` function in Ghidra:

```c
// Ghidra decompilation of verify_license:
int verify_license(char *key) {
    if (key == NULL) return 0;
    if (strcmp(key, "TRIAL-2024-ABC123") == 0) return 1;
    if (strncmp(key, "TRIAL-", 6) == 0) return 1;
    return 0;
}
```

Find the call site in `main`:

```c
// In main, after parsing "activate" command:
if (verify_license(key)) {
    g_state.is_licensed = 1;
    // ...
} else {
    printf("[ERROR] Invalid license key!\n");
}
```

### Step 5: Identify the Patch Point

The simplest patch is to make `verify_license` always return 1.

Find the function in assembly:

```bash
objdump -M intel -d /tmp/licensed_app | grep -A20 "<verify_license>:"
```

The function starts at some address like `0x4012a0`. To make it always return 1:

**Option 1: Patch the return value**
- Change `xor eax, eax` (return 0) to `mov eax, 1; ret` (return 1)

**Option 2: NOP the comparison**
- NOP out the conditional jumps that check for failure

**Option 3: Patch the entry point**
- Change the function prologue to immediately return 1

### Step 6: Apply the Patch

```bash
# Find the verify_license address
VERIFY_ADDR=$(objdump -M intel -d /tmp/licensed_app | grep -m1 "<verify_license>:" | awk '{print $1}')
echo "verify_license at: $VERIFY_ADDR"

# Copy the binary for patching
cp /tmp/licensed_app /tmp/licensed_app_patched

# Patch strategy: change the 'return 0' at the end of verify_license
# The function looks like:
# 4012a0: ... (prologue)
# 4012b0: xor eax, eax    <- return 0
# 4012b2: add rsp, ...    <- epilogue  
# 4012b5: ret

# Find the exact bytes to patch
objdump -M intel -d /tmp/licensed_app | grep -A5 "<verify_license>:" | tail -6

# Use radare2 for clean patching
rizin -w /tmp/licensed_app_patched << 'RIZIN'
# Seek to verify_license
s verify_license
# Show current instruction
pd 3
# The 'xor eax,eax' followed by 'ret' means return 0
# Change it to: mov eax, 1; nop; ret
# xor eax,eax = 31 C0 (2 bytes)
# mov eax, 1 = B8 01 00 00 00 (5 bytes)
# Since we need same byte count, let's patch differently

# Instead: change the first instruction to immediately return 1
# Seek to function start
s verify_license
# Overwrite prologue to just: mov eax, 1; ret
"wa mov eax, 1" 
"wa nop"
"wa ret"
pd 3
q
RIZIN
```

### Step 7: Verify the Patch

```bash
echo "=== Original binary ==="
/tmp/licensed_app info
/tmp/licensed_app activate ANYTHING
/tmp/licensed_app use

echo ""
echo "=== Patched binary ==="
/tmp/licensed_app_patched info
/tmp/licensed_app_patched activate ANYTHING
/tmp/licensed_app_patched use
```

**Expected result:**
- Original: Only `TRIAL-2024-ABC123` or `TRIAL-*` keys work
- Patched: Any key is accepted

### Step 8: Alternative Patch — NOP the Check

If the first approach doesn't work, try patching the comparison:

```bash
# Alternative: NOP out the strcmp call and its comparison
# Find the address of the strcmp call in main
objdump -M intel -d /tmp/licensed_app_patched | grep -B3 "call.*verify_license"

# NOP the conditional jump after the call
# je 0x4013xx (jump if license invalid)
# Change to: jmp past the error message
```

> [!success]- Full Solution

The complete patch workflow:
1. Binary analysis to understand license check logic
2. Identification of the vulnerable check point
3. Multiple patch strategies (patch return value, NOP comparison, redirect flow)
4. Testing to verify patch works without breaking functionality

---

## Extensions / Challenges

1. **Signature verification**: The real check uses a digital signature. Find the signature and create a keygen.
2. **Encrypted check**: The license string is encrypted. Find the decryption function.
3. **Online verification**: The binary calls home to verify. Patch the network check.
4. **Obfuscated strings**: License strings are XOR'd. Write a deobfuscation script first.
5. **Anti-debug**: Binary detects debuggers. Bypass the anti-debug before patching.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `strings` | Initial string extraction |
| `objdump` | Disassembly and function finding |
| `Ghidra` | Decompilation and analysis |
| `rizin` | Precise byte-level patching |
| `printf + dd` | Simple byte patching |
| Testing | Verification of patch effectiveness |
