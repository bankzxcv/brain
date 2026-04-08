---
title: "RE Workflow: Reconstruct Source from Stripped Binary"
date: 2026-04-07
tags:
  - reverse-engineering
  - workflow
  - project
  - reconstruction
  - stripped
  - ghidra
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Workflow: Reconstruct Source from Stripped Binary

## Overview

In this project, you will perform comprehensive reverse engineering on a fully stripped binary to reconstruct equivalent high-level source code.

**Scenario:** You have a production binary with no symbols, no debug info, and some runtime protections. You need to recover its functionality to fix a critical bug that only manifests in production.

**Concepts practiced:** Symbol recovery, type reconstruction, structure recovery, function naming, pseudocode generation, C source reconstruction.

---

## Step-by-Step Implementation

### Step 1: Create the Target Binary

Create a realistic stripped binary to practice on:

```bash
cat > /tmp/stripped_app.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_INPUT 1024
#define MAX_NAME 256

typedef struct {
    char name[MAX_NAME];
    int age;
    int id;
    float salary;
    char department[64];
} Employee;

typedef struct {
    int total;
    int count;
    Employee *employees;
} Company;

void init_company(Company *c) {
    c->total = 100;
    c->count = 0;
    c->employees = calloc(c->total, sizeof(Employee));
}

int add_employee(Company *c, const char *name, int age, int id, 
                 float salary, const char *dept) {
    if (c->count >= c->total) {
        // Expand capacity
        c->total *= 2;
        c->employees = realloc(c->employees, c->total * sizeof(Employee));
    }
    
    Employee *e = &c->employees[c->count];
    strncpy(e->name, name, MAX_NAME - 1);
    e->name[MAX_NAME - 1] = '\0';
    e->age = age;
    e->id = id;
    e->salary = salary;
    strncpy(e->department, dept, 63);
    e->department[63] = '\0';
    
    c->count++;
    return c->count;
}

float get_average_salary(Company *c) {
    if (c->count == 0) return 0.0;
    
    float total = 0;
    for (int i = 0; i < c->count; i++) {
        total += c->employees[i].salary;
    }
    return total / c->count;
}

Employee* find_by_id(Company *c, int id) {
    for (int i = 0; i < c->count; i++) {
        if (c->employees[i].id == id) {
            return &c->employees[i];
        }
    }
    return NULL;
}

Employee* find_by_name(Company *c, const char *name) {
    for (int i = 0; i < c->count; i++) {
        if (strcasecmp(c->employees[i].name, name) == 0) {
            return &c->employees[i];
        }
    }
    return NULL;
}

void print_employee(Employee *e) {
    if (!e) return;
    printf("ID: %d, Name: %s, Age: %d, Salary: $%.2f, Dept: %s\n",
           e->id, e->name, e->age, e->salary, e->department);
}

void print_company_stats(Company *c) {
    printf("\n=== Company Statistics ===\n");
    printf("Total Employees: %d\n", c->count);
    printf("Average Salary: $%.2f\n", get_average_salary(c));
    printf("========================\n\n");
}

int parse_employee_line(const char *line, char *name, int *age, 
                        int *id, float *salary, char *dept) {
    // Format: "Name,Age,ID,Salary,Department"
    char *copy = strdup(line);
    char *token = strtok(copy, ",");
    if (!token) { free(copy); return 0; }
    strncpy(name, token, MAX_NAME - 1);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    *age = atoi(token);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    *id = atoi(token);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    *salary = atof(token);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    strncpy(dept, token, 63);
    dept[63] = '\0';
    
    free(copy);
    return 1;
}

int main(int argc, char **argv) {
    if (argc < 2) {
        printf("Usage: %s <employee_data_file>\n", argv[0]);
        printf("Employee file format: Name,Age,ID,Salary,Department\n");
        return 1;
    }
    
    Company company;
    init_company(&company);
    
    FILE *f = fopen(argv[1], "r");
    if (!f) {
        fprintf(stderr, "Error: Cannot open file: %s\n", argv[1]);
        return 1;
    }
    
    char line[MAX_INPUT];
    char name[MAX_NAME];
    int age, id;
    float salary;
    char dept[64];
    
    while (fgets(line, sizeof(line), f)) {
        // Remove newline
        line[strcspn(line, "\n")] = 0;
        
        if (parse_employee_line(line, name, &age, &id, &salary, dept)) {
            add_employee(&company, name, age, id, salary, dept);
        }
    }
    
    fclose(f);
    
    // Demonstrate functionality
    print_company_stats(&company);
    
    printf("All Employees:\n");
    for (int i = 0; i < company.count; i++) {
        printf("%d. ", i + 1);
        print_employee(&company.employees[i]);
    }
    
    // Test find functions
    if (company.count > 0) {
        printf("\nSearch by ID %d:\n", company.employees[0].id);
        Employee *found = find_by_id(&company, company.employees[0].id);
        if (found) print_employee(found);
    }
    
    free(company.employees);
    return 0;
}
EOF

# Create sample data
cat > /tmp/employees.txt << 'EOF'
Alice Smith,30,1001,75000.00,Engineering
Bob Johnson,45,1002,95000.00,Management
Carol Williams,28,1003,65000.00,Engineering
David Brown,35,1004,80000.00,Sales
Eve Davis,42,1005,85000.00,HR
EOF

gcc -O2 -o /tmp/stripped_app /tmp/stripped_app.c
strip -s /tmp/stripped_app  # Full strip
echo "Stripped binary created:"
file /tmp/stripped_app
ls -la /tmp/stripped_app

# Test it works
/tmp/stripped_app /tmp/employees.txt
```

### Step 2: Initial Analysis — What Do We Have?

```bash
# Basic info
file /tmp/stripped_app
ls -la /tmp/stripped_app

# Check if fully stripped
nm /tmp/stripped_app 2>&1 | head -5
# Should say "no symbols"

# Check dynamic symbols (can't be stripped fully)
nm -D /tmp/stripped_app

# Check for interesting strings
strings /tmp/stripped_app | grep -v "U\|GLIBC" | head -30
```

### Step 3: Load into Ghidra

```bash
mkdir -p /tmp/ghidra_stripped
analyzeHeadless /tmp/ghidra_stripped -import /tmp/stripped_app \
  -analysisTimeout 120 -processor x86:64 -cspec gcc
```

In Ghidra:
1. Let auto-analysis run (it will take time to find functions)
2. Look at the Symbol Tree — how many functions were auto-identified?
3. Check what imports are used (File → Set Data Types → windows → Edit → Windows Specs)

### Step 4: Find and Name Main

In Ghidra:
1. Look for the entry point (`_start` → `__libc_start_main`)
2. Navigate to the function passed to `__libc_start_main` — that's `main`
3. Rename it to `main`

```c
// Ghidra shows something like:
undefined8 main(int argc, char **argv)
{
    // code here
}
```

### Step 5: Analyze Data Structures

Look for global variables accessed in `main`:

```c
// In Ghidra, look for global accesses
// [DAT_00404000] or similar

// This is the Company struct
// Identify its fields by analyzing access patterns
```

### Step 6: Reconstruct the Employee and Company Structs

By analyzing how the binary accesses memory:

| Offset | Field | Evidence |
|--------|-------|----------|
| +0x00 | name[256] | `memcpy` to/from this area, strncpy |
| +0x100 | age (int) | Accessed as DWORD, initialized to 0 |
| +0x104 | id (int) | Compared, stored |
| +0x108 | salary (float) | Accessed as float (XMM register) |
| +0x10C | department[64] | strncpy destination |

### Step 7: Reconstruct Functions

For each function you identify in Ghidra, document:

1. **Function name** — based on behavior
2. **Parameters** — based on register usage at call sites
3. **Local variables** — based on stack frame analysis
4. **Return type** — based on register used for return (RAX)
5. **Pseudocode** — from Ghidra's decompiler

**Expected reconstructed functions:**

```c
// Reconstructed from binary:
typedef struct {
    char name[256];      // offset 0x00
    int age;             // offset 0x100
    int id;              // offset 0x104
    float salary;        // offset 0x108
    char department[64]; // offset 0x10C
} Employee;

typedef struct {
    int total;           // capacity
    int count;           // current count
    Employee *employees; // heap pointer
} Company;

// Functions:
void init_company(Company *c);                    // 0x4012a0
int add_employee(Company *c, char *name, ...);   // 0x4013d0
float get_average_salary(Company *c);           // 0x401520
Employee* find_by_id(Company *c, int id);        // 0x4015f0
Employee* find_by_name(Company *c, char *name); // 0x401680
void print_employee(Employee *e);                // 0x401730
void print_company_stats(Company *c);            // 0x4017a0
int parse_employee_line(char *line, ...);        // 0x4018c0
```

### Step 8: Write the Reconstructed Source

Based on Ghidra analysis, write equivalent C source:

```c
/*
 * RECONSTRUCTED SOURCE
 * Original: stripped_app
 * Architecture: x86-64 Linux
 * 
 * RECONSTRUCTED BY: RE Practice Workflow
 * DATE: 2026-04-07
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_INPUT 1024
#define MAX_NAME 256

// === RECONSTRUCTED DATA STRUCTURES ===

typedef struct {
    char name[MAX_NAME];
    int age;
    int id;
    float salary;
    char department[64];
} Employee;

typedef struct {
    int total;       // capacity
    int count;        // current count
    Employee *employees;
} Company;

// === RECONSTRUCTED FUNCTIONS ===

void init_company(Company *c) {
    c->total = 100;
    c->count = 0;
    c->employees = calloc(c->total, sizeof(Employee));
}

int add_employee(Company *c, const char *name, int age, int id,
                 float salary, const char *dept) {
    if (c->count >= c->total) {
        c->total *= 2;
        c->employees = realloc(c->employees, c->total * sizeof(Employee));
    }
    
    Employee *e = &c->employees[c->count];
    strncpy(e->name, name, MAX_NAME - 1);
    e->name[MAX_NAME - 1] = '\0';
    e->age = age;
    e->id = id;
    e->salary = salary;
    strncpy(e->department, dept, 63);
    e->department[63] = '\0';
    
    c->count++;
    return c->count;
}

float get_average_salary(Company *c) {
    if (c->count == 0) return 0.0;
    
    float total = 0;
    for (int i = 0; i < c->count; i++) {
        total += c->employees[i].salary;
    }
    return total / c->count;
}

Employee* find_by_id(Company *c, int id) {
    for (int i = 0; i < c->count; i++) {
        if (c->employees[i].id == id) {
            return &c->employees[i];
        }
    }
    return NULL;
}

Employee* find_by_name(Company *c, const char *name) {
    for (int i = 0; i < c->count; i++) {
        if (strcasecmp(c->employees[i].name, name) == 0) {
            return &c->employees[i];
        }
    }
    return NULL;
}

void print_employee(Employee *e) {
    if (!e) return;
    printf("ID: %d, Name: %s, Age: %d, Salary: $%.2f, Dept: %s\n",
           e->id, e->name, e->age, e->salary, e->department);
}

void print_company_stats(Company *c) {
    printf("\n=== Company Statistics ===\n");
    printf("Total Employees: %d\n", c->count);
    printf("Average Salary: $%.2f\n", get_average_salary(c));
    printf("========================\n\n");
}

int parse_employee_line(const char *line, char *name, int *age,
                        int *id, float *salary, char *dept) {
    char *copy = strdup(line);
    char *token = strtok(copy, ",");
    if (!token) { free(copy); return 0; }
    strncpy(name, token, MAX_NAME - 1);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    *age = atoi(token);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    *id = atoi(token);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    *salary = atof(token);
    
    token = strtok(NULL, ",");
    if (!token) { free(copy); return 0; }
    strncpy(dept, token, 63);
    dept[63] = '\0';
    
    free(copy);
    return 1;
}

int main(int argc, char **argv) {
    if (argc < 2) {
        printf("Usage: %s <employee_data_file>\n", argv[0]);
        return 1;
    }
    
    Company company;
    init_company(&company);
    
    FILE *f = fopen(argv[1], "r");
    if (!f) {
        fprintf(stderr, "Error: Cannot open file: %s\n", argv[1]);
        return 1;
    }
    
    char line[MAX_INPUT];
    char name[MAX_NAME];
    int age, id;
    float salary;
    char dept[64];
    
    while (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\n")] = 0;
        if (parse_employee_line(line, name, &age, &id, &salary, dept)) {
            add_employee(&company, name, age, id, salary, dept);
        }
    }
    
    fclose(f);
    print_company_stats(&company);
    
    for (int i = 0; i < company.count; i++) {
        printf("%d. ", i + 1);
        print_employee(&company.employees[i]);
    }
    
    free(company.employees);
    return 0;
}
```

### Step 9: Verify Reconstruction

```bash
# Compile the reconstructed source
gcc -O2 -o /tmp/reconstructed_app /tmp/reconstructed_app.c

# Compare outputs
echo "=== Original ==="
/tmp/stripped_app /tmp/employees.txt

echo ""
echo "=== Reconstructed ==="
/tmp/reconstructed_app /tmp/employees.txt

# Should be identical!
diff <(/tmp/stripped_app /tmp/employees.txt) \
      <(/tmp/reconstructed_app /tmp/employees.txt)
echo "Diff result: $?"
```

> [!success]- Full Solution

The complete source reconstruction workflow:
1. Initial analysis and binary intelligence gathering
2. Ghidra loading and auto-analysis
3. Function identification (entry point → main → called functions)
4. Data structure reconstruction through memory access analysis
5. Pseudocode generation via decompiler
6. Manual source code reconstruction
7. Verification by comparing original and reconstructed outputs

---

## Extensions / Challenges

1. **Partially stripped**: Only strip dynamic symbols and re-identify functions using patterns
2. **Obfuscated control flow**: Add obfuscation and still reconstruct logic
3. **Different optimization levels**: Compare -O0 vs -O2 reconstruction difficulty
4. **C++ with vtables**: Reconstruct C++ classes and virtual functions
5. **Multi-file binary**: Handle multiple compilation units

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Binary identification |
| `nm` | Symbol analysis |
| `strings` | String extraction |
| `objdump` | Disassembly |
| `Ghidra` | Decompilation and analysis |
| `diff` | Verification comparison |
