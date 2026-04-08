---
title: "RE Practice: Dynamic Analysis"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - strace
  - ltrace
  - dynamic-analysis
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Dynamic Analysis

13 progressive problems using strace, ltrace, Wireshark, and behavioral analysis.

---

## P1: Trace System Calls with strace

```bash
cat > /tmp/hello.c << 'EOF'
#include <stdio.h>
int main() {
    printf("Hello, RE!\n");
    return 0;
}
EOF
gcc -o /tmp/hello /tmp/hello.c
strace -o /tmp/strace.log /tmp/hello
cat /tmp/strace.log
```

What system calls does `printf` ultimately make?

**Expected output:**
```
write(1, "Hello, RE!\n", 12) = 12
exit_group(0)
```

> [!hint]- Hint
> `printf` calls `write` internally through libc. `strace` shows the actual syscalls, not the C library calls.

> [!success]- Solution
> `printf("Hello, RE!\n")` ends up calling the `write` syscall:
> ```
> write(1, "Hello, RE!\n", 12) = 12
> exit_group(0)
> ```
> `strace` reveals that the C library function wraps a kernel system call. The `1` is stdout's file descriptor.

---

## P2: Trace Library Calls with ltrace

```bash
ltrace -o /tmp/ltrace.log /tmp/hello
cat /tmp/ltrace.log
```

What library calls does ltrace show that strace doesn't?

**Expected output:**
```
printf("Hello, RE!\n") = 12
exit(0)
```

> [!hint]- Hint
> `ltrace` shows shared library function calls (PLT calls). `strace` shows kernel syscalls. The same operation (printing) shows different things in each.

> [!success]- Solution
> `ltrace` shows the libc function calls: `printf("Hello, RE!\n")` instead of the underlying `write` syscall. This is useful for understanding the program flow at a higher level.

---

## P3: Compare strace and ltrace Side by Side

Run both on the same program:

```bash
strace -f -e trace=write /tmp/hello 2>&1
ltrace -f -e printf /tmp/hello 2>&1
```

What's the difference between tracing syscalls vs library calls?

**Expected output:**
```
strace: write(1, ...) — kernel-level I/O
ltrace: printf(...) — libc wrapper function
```

> [!hint]- Hint
> `ltrace` intercepts calls to shared library functions (PLT/GOT). `strace` intercepts kernel system calls. `printf` internally calls `write`.

> [!success]- Solution
> - `strace` with `-f` follows forks and traces into syscalls
> - `ltrace` with `-e` filters to specific library functions
> - For understanding program behavior, both are useful: syscall level (what the kernel sees) vs library level (what the program calls)

---

## P4: Trace a Network Program

Create and trace a network client:

```bash
cat > /tmp/net.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in server;
    server.sin_family = AF_INET;
    server.sin_port = htons(8080);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);
    
    connect(sock, (struct sockaddr *)&server, sizeof(server));
    
    char *msg = "Hello, server!";
    send(sock, msg, strlen(msg), 0);
    
    char buf[100];
    int n = recv(sock, buf, 100, 0);
    buf[n] = '\0';
    printf("Received: %s\n", buf);
    
    close(sock);
    return 0;
}
EOF
gcc -o /tmp/net /tmp/net.c
strace -f -e trace=socket,connect,send,recv,close /tmp/net 2>&1
```

**Expected output:**
```
socket(AF_INET, SOCK_STREAM, 0) = 3
connect(3, {sin_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS
send(3, "Hello, server!", 13, 0) = 13
recv(3, ..., 100, 0) = -1 ECONNREFUSED (because no server running)
close(3) = 0
```

> [!hint]- Hint
> The connection will fail (ECONNREFUSED) because no server is running on port 8080. But the strace output still shows all the network syscalls.

> [!success]- Solution
> The strace output shows the exact sequence: socket creation → connect attempt → send → (recv fails) → close. Even without a server, the syscall sequence reveals the protocol.

---

## P5: Filter strace to Find File Operations

```bash
strace -f -e trace=open,read,write,close /tmp/hello 2>&1
```

What files does `hello` open? (It shouldn't open any explicitly — printf uses existing stdout)

**Expected output:**
```
(No openat/close for files — stdout is already open)
write(1, "Hello, RE!\n", 12) = 12
```

> [!hint]- Hint
> `hello` only uses stdout (fd 1) which is already open from the kernel. No additional files are opened.

> [!success]- Solution
> The simple `printf` program doesn't open any files — stdout (fd 1), stderr (fd 2), and stdin (fd 0) are inherited from the parent process (shell). The only operations are `write` to stdout.

---

## P6: Observe File Creation

```bash
cat > /tmp/create.c << 'EOF'
#include <stdio.h>
int main() {
    FILE *f = fopen("/tmp/test_output.txt", "w");
    fprintf(f, "Created by RE practice\n");
    fclose(f);
    return 0;
}
EOF
gcc -o /tmp/create /tmp/create.c
strace -f -e trace=openat,write,close /tmp/create 2>&1
ls -la /tmp/test_output.txt
```

**Expected output:**
```
openat(AT_FDCWD, "/tmp/test_output.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 3
write(3, "Created by RE practice\n", 23) = 23
close(3) = 0
```

> [!hint]- Hint
> `fopen` calls `openat` under the hood. The `strace` shows the exact flags used.

> [!success]- Solution
> The strace reveals the exact syscalls:
> - `openat` with `O_WRONLY|O_CREAT|O_TRUNC` creates or truncates the file
> - `write` writes 23 bytes
> - `close` closes the fd
> File permissions `0644` are also visible in the syscall.

---

## P7: Network Capture with tcpdump

```bash
# In one terminal, start a simple server:
nc -l 8080 &

# In another terminal, run our network client:
/tmp/net &

# Capture traffic
sudo tcpdump -i lo -w /tmp/capture.pcap
# or capture with wireshark:
wireshark &
# filter: tcp.port == 8080
```

**Expected output:**
```
Packet capture shows TCP handshake (SYN, SYN-ACK, ACK)
Then data packets with "Hello, server!"
```

> [!hint]- Hint
> tcpdump captures packets. Wireshark analyzes them. For localhost, use interface `lo`.

> [!success]- Solution
> The capture shows the full TCP flow: 3-way handshake, then the data payload "Hello, server!" in the packet. This is the foundation of protocol reverse engineering.

---

## P8: Analyze Behavior of an Unknown Binary

You receive a binary without source. Use dynamic analysis to determine what it does:

```bash
# Create a test unknown binary
cat > /tmp/unknown.sh << 'EOF'
#!/bin/bash
echo "I'm an unknown binary!"
ls -la /tmp
cat /etc/hostname
EOF
chmod +x /tmp/unknown.sh

# Use strace
strace /tmp/unknown.sh 2>&1 | head -30

# Use ltrace
ltrace /tmp/unknown.sh 2>&1 | head -20
```

**Expected output:**
```
strace shows: execve, then syscalls for echo, ls, cat
ltrace shows: library calls if dynamically linked
```

> [!hint]- Hint
> `strace` shows the first syscall `execve` (running the script), then all subsequent syscalls as the shell interprets and runs commands.

> [!success]- Solution
> - `strace`: `execve("/tmp/unknown.sh", ...)`, then `write("I'm an unknown binary!")`, `openat("/tmp", ...)`, `newfstatat("/etc/hostname", ...)`, etc.
> - This reveals the binary's full behavior without source code

---

## P9: Detect Filename Encryption (Runtime Decryption)

A binary encodes filenames at runtime. Use ltrace to detect the pattern:

```bash
cat > /tmp/obfuscated.c << 'EOF'
#include <stdio.h>
#include <string.h>

char encode(char c) { return c ^ 0x55; }

void reveal(const char *input, char *output) {
    while (*input) {
        *output++ = encode(*input++);
    }
    *output = '\0';
}

int main() {
    char hidden[] = {'H', 'i', 'd', 'd', 'e', 'n', '\0'};
    char revealed[100];
    reveal(hidden, revealed);
    printf("Found: %s\n", revealed);
    return 0;
}
EOF
gcc -o /tmp/obfuscated /tmp/obfuscated.c
ltrace /tmp/obfuscated 2>&1
```

**Expected output:**
```
ltrace shows: encode(0x48) = 0x1d (H ^ 0x55 = 0x1d)
reveal(...) = "Hidden"
```

> [!hint]- Hint
> `ltrace` intercepts all function calls, including custom ones like `encode`. Each call is logged with arguments and return values.

> [!success]- Solution
> `ltrace` shows each call to `encode` with its input and output. By tracing the sequence: `encode('H')` → `0x1d`, `encode('i')` → `?`, etc., you can determine the XOR key (0x55) and reconstruct the obfuscation logic.

---

## P10: Behavioral Analysis — Full Workflow

Perform a complete behavioral analysis of a binary:

1. Run `strace` to see system interactions
2. Run `ltrace` to see library call patterns
3. Check file system changes with `diff`
4. Analyze network behavior
5. Summarize findings

```bash
# Create test binary
cat > /tmp/behave.c << 'EOF'
#include <stdio.h>
#include <unistd.h>
#include <sys/stat.h>

int main() {
    mkdir("/tmp/re_test", 0755);
    FILE *f = fopen("/tmp/re_test/data.txt", "w");
    fprintf(f, "ID: %d\n", getpid());
    fclose(f);
    return 0;
}
EOF
gcc -o /tmp/behave /tmp/behave.c

# Stage 1: strace
strace -f -e trace=mkdir,openat,write,getpid /tmp/behave 2>&1

# Stage 2: before/after filesystem check
mkdir -p /tmp/before
find /tmp -newer /tmp/behave -type f 2>/dev/null > /tmp/after
cat /tmp/after
```

**Expected output:**
```
mkdir("/tmp/re_test", 0755) = 0
openat(AT_FDCWD, "/tmp/re_test/data.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 3
write(3, "ID: 12345\n", 11) = 11
getpid() = 12345
```

> [!hint]- Hint
> Combining multiple dynamic analysis techniques gives a complete picture of what a program does.

> [!success]- Solution
> The combined approach reveals:
> - Creates directory `/tmp/re_test`
> - Creates file `/tmp/re_test/data.txt`
> - Writes PID to the file
> - No network activity
> - No additional syscalls

---

## P11: Compare strace Output of Two Binaries

Compare a statically linked vs dynamically linked binary:

```bash
cat > /tmp/static_test.c << 'EOF'
#include <stdio.h>
int main() { printf("hi\n"); return 0; }
EOF
gcc -o /tmp/dynamic /tmp/static_test.c
gcc -static -o /tmp/static /tmp/static_test.c

strace /tmp/dynamic 2>&1 | grep -E '^(execve|write|exit'
strace /tmp/static 2>&1 | grep -E '^(execve|write|exit'
```

How do the strace outputs differ?

**Expected output:**
```
Dynamic: fewer syscalls (libc handles some internally)
Static: many more syscalls (libc code linked in, more internal calls)
```

> [!hint]- Hint
> Static binaries include all library code, so they make more syscalls for things that dynamic binaries handle in user space via libc.

> [!success]- Solution
> The dynamic binary shows: `execve`, `write`, `exit_group`. The static binary shows many more syscalls because libc is compiled in and makes direct syscall calls for buffering, etc. The dynamic binary's libc calls go through the PLT, which may use fewer direct syscalls.

---

## P12: Use LLM to Analyze strace Output

Take a real strace output and ask an LLM to summarize behavior:

```
PROMPT: "Here is the strace output of an unknown binary. Summarize what the
program does step by step. Identify any suspicious activities (file access,
network, process forking, etc.):

[strace output of a binary]"
```

**Expected output:**
```
(LLM summarizes: file operations, network connections, process behavior,
any concerning patterns like spawning shells, connecting to external IPs, etc.)
```

> [!hint]- Hint
> LLMs can quickly parse and summarize large strace outputs, identifying patterns that would take a human much longer to spot.

> [!success]- Solution
> The LLM correctly identifies: syscalls in order, what each syscall does, what files are accessed, what network connections are made, and flags any suspicious behavior (like `execve("/bin/sh")` or connecting to external IPs).

---

## AI Power-Up: Dynamic Analysis

```
PROMPT: "I have a strace and ltrace of a program that I suspect is malware.
Here are the outputs. Classify it as: benign, suspicious, or malicious.
Explain your reasoning and identify any IOCs:

[strace + ltrace output]"

PROMPT: "The program is making an outbound connection. Based on this strace
output, what is the target IP/port? Can I find the configuration or
hardcoded address that determines the connection?
[strace showing network syscalls]"
```
