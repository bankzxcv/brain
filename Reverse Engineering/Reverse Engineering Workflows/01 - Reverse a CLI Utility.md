---
title: "RE Workflow: Reverse a CLI Utility"
date: 2026-04-07
tags:
  - reverse-engineering
  - workflow
  - project
  - cli
  - ghidra
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Workflow: Reverse a CLI Utility

## Overview

In this project, you will reverse engineer a real-world CLI (command-line interface) utility whose source code is unavailable. The goal is to understand its functionality, how it processes arguments, and document its behavior through RE alone.

**Target:** A statically compiled x86-64 CLI tool (you will create this as a practice target).

**Concepts practiced:** Static analysis, disassembly, function identification, string extraction, argument parsing, Ghidra decompilation, behavior documentation.

---

## Step-by-Step Implementation

### Step 1: Create the Target Binary

First, we'll create a realistic CLI utility to reverse. This simulates a real-world scenario.

```bash
cat > /tmp/cli_tool.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/stat.h>

#define MAX_LINE 4096
#define CONFIG_PATH "/etc/cli_tool/config"

typedef struct {
    int verbose;
    int use_color;
    char *output_file;
    char *input_file;
    int count;
} Options;

void print_banner() {
    printf("=== CLI Tool v2.1.4 ===\n");
    printf("Copyright (c) 2024 Acme Corp\n");
}

void usage(const char *prog) {
    printf("Usage: %s [options] <input_file>\n", prog);
    printf("Options:\n");
    printf("  -o FILE     Output file (default: stdout)\n");
    printf("  -c          Use color output\n");
    printf("  -v          Verbose mode\n");
    printf("  -n NUM      Count (default: 10)\n");
    printf("  -h          Show this help\n");
}

int parse_args(int argc, char **argv, Options *opts) {
    memset(opts, 0, sizeof(*opts));
    opts->count = 10;
    
    int c;
    while ((c = getopt(argc, argv, "o:cvh:n:")) != -1) {
        switch (c) {
            case 'o':
                opts->output_file = optarg;
                break;
            case 'c':
                opts->use_color = 1;
                break;
            case 'v':
                opts->verbose = 1;
                break;
            case 'n':
                opts->count = atoi(optarg);
                break;
            case 'h':
                return 0;  // signal to show help
            default:
                return -1;
        }
    }
    
    if (optind < argc) {
        opts->input_file = argv[optind];
    }
    
    return 1;
}

int process_file(Options *opts) {
    FILE *in = stdin;
    FILE *out = stdout;
    
    if (opts->verbose) {
        fprintf(stderr, "[DEBUG] Opening input: %s\n", 
                opts->input_file ? opts->input_file : "(stdin)");
    }
    
    if (opts->input_file) {
        in = fopen(opts->input_file, "r");
        if (!in) {
            fprintf(stderr, "Error: Cannot open input file: %s\n", opts->input_file);
            return 1;
        }
    }
    
    if (opts->output_file) {
        out = fopen(opts->output_file, "w");
        if (!out) {
            fprintf(stderr, "Error: Cannot open output file: %s\n", opts->output_file);
            if (in != stdin) fclose(in);
            return 1;
        }
    }
    
    char line[MAX_LINE];
    int line_num = 0;
    int printed = 0;
    
    while (fgets(line, sizeof(line), in) && printed < opts->count) {
        line_num++;
        if (opts->verbose) {
            fprintf(stderr, "[DEBUG] Processing line %d\n", line_num);
        }
        
        // Simple processing: print with optional line number
        if (opts->use_color) {
            fprintf(out, "\033[1;32m%4d:\033[0m %s", line_num, line);
        } else {
            fprintf(out, "%4d: %s", line_num, line);
        }
        printed++;
    }
    
    if (opts->verbose) {
        fprintf(stderr, "[DEBUG] Processed %d lines\n", printed);
    }
    
    if (in != stdin) fclose(in);
    if (out != stdout) fclose(out);
    
    return 0;
}

int main(int argc, char **argv) {
    Options opts;
    
    int parse_result = parse_args(argc, argv, &opts);
    
    if (parse_result == 0) {
        print_banner();
        usage(argv[0]);
        return 0;
    }
    
    if (parse_result == -1) {
        fprintf(stderr, "Error: Invalid arguments\n");
        usage(argv[0]);
        return 1;
    }
    
    return process_file(&opts);
}
EOF

gcc -static -o /tmp/cli_tool /tmp/cli_tool.c
strip -s /tmp/cli_tool
echo "Binary created:"
file /tmp/cli_tool
ls -la /tmp/cli_tool
```

### Step 2: Initial Reconnaissance

```bash
# Get basic info
file /tmp/cli_tool
ls -la /tmp/cli_tool

# Extract strings (first pass)
strings /tmp/cli_tool | head -50

# Find interesting strings
strings /tmp/cli_tool | grep -iE "(error|usage|version|config|help)"
```

**Findings to note:**
- What strings are visible?
- What does the banner say?
- What error messages are present?

### Step 3: Identify Binary Format and Sections

```bash
readelf -h /tmp/cli_tool
readelf -S /tmp/cli_tool
readelf -d /tmp/cli_tool
```

**Findings to note:**
- Is it statically or dynamically linked?
- Entry point address?
- Key sections (.text, .rodata, .data)?
- Any interesting dynamic section entries?

### Step 4: Find Main and Entry Point

```bash
# Find entry point disassembly
objdump -M intel -d /tmp/cli_tool | head -80

# Find main (if not stripped)
nm /tmp/cli_tool | grep main
# Since statically linked, main might not be exported
```

**Findings to note:**
- What does the startup code look like?
- Where is `main` (look for `call main` from startup)?

### Step 5: Load into Ghidra for Analysis

```bash
mkdir -p /tmp/ghidra_cli
analyzeHeadless /tmp/ghidra_cli -import /tmp/cli_tool -analysisTimeout 120
```

In Ghidra:
1. Open the project
2. Find `main` in the Symbol Tree
3. Decompile it
4. Navigate to interesting functions

### Step 6: Analyze Argument Parsing

In Ghidra, find and analyze the `getopt`-related code:
1. Search for "optarg", "getopt", "optind"
2. Identify the option parsing function
3. Document the flags supported

```bash
# Verify findings by running with different arguments
/tmp/cli_tool -h
/tmp/cli_tool -v /etc/hostname
/tmp/cli_tool -n 5 -c /etc/hostname -o /tmp/out.txt
cat /tmp/out.txt
```

### Step 7: Analyze File Processing

Trace from `main` → argument parsing → file processing function:
1. How does it open files?
2. How does it read lines?
3. What processing does it apply?

### Step 8: Reverse Engineer and Document

Create a comprehensive analysis document:

```markdown
## CLI Tool Analysis

### Basic Information
- File: cli_tool
- Architecture: x86-64
- Statically linked: Yes
- Entry point: 0x...

### Function Map
| Address | Name | Purpose |
|---------|------|---------|
| 0x... | main | Entry point, calls argument parser and processor |
| 0x... | parse_args | Parses command-line arguments |
| 0x... | usage | Prints help/usage message |
| 0x... | print_banner | Prints version banner |
| 0x... | process_file | Main file processing loop |

### Supported Options
| Flag | Meaning |
|------|---------|
| -h | Show help |
| -v | Verbose mode |
| -c | Color output |
| -o FILE | Output file |
| -n NUM | Number of lines |

### Behavioral Notes
- Reads from stdin if no input file specified
- Writes to stdout if no output file specified
- Uses optind to track non-option arguments
- Supports piping (stdin/stdout)
```

### Step 9: Verify by Testing

```bash
# Test all documented behaviors
/tmp/cli_tool -h 2>&1 | head -10
echo "test input" | /tmp/cli_tool -n 3
/tmp/cli_tool -n 2 /etc/hostname
/tmp/cli_tool -n 1 -c /etc/hostname -v
```

> [!success]- Full Solution

The complete RE workflow produces a full analysis documenting:
1. Binary metadata (architecture, linking, entry point)
2. Function identification and naming
3. Command-line interface documentation
4. Behavioral description
5. Verified by testing

---

## Extensions / Challenges

1. **Obfuscate and re-analyze**: Compile with UPX packing and repeat the analysis
2. **Add anti-debug and re-analyze**: Add ptrace detection, rebuild, and find the check
3. **Cross-architecture**: Compile for ARM64 and compare the analysis approach
4. **Symbol removal**: Strip more aggressively (remove `.dynsym`) and re-identify functions
5. **Write an equivalent replacement**: Write a clean C implementation matching the RE'd behavior

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Binary identification |
| `strings` | String extraction |
| `readelf` | ELF format analysis |
| `objdump` | Disassembly |
| `nm` | Symbol table |
| `Ghidra` | Decompilation and analysis |
| `strace` | Dynamic behavior verification |
