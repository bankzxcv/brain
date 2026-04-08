---
title: "RE Practice: Protocol Reverse Engineering"
date: 2026-04-07
tags:
  - reverse-engineering
  - practice
  - protocol
  - wireshark
  - network
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Practice: Protocol Reverse Engineering

12 progressive problems covering packet capture, protocol structure analysis, and reconstruction.

---

## P1: Capture and Examine Packets

```bash
# Start a simple server
nc -l -p 9999 > /tmp/server.log &

# Connect client
echo "hello" | nc localhost 9999

# Capture with tcpdump
sudo tcpdump -i lo -w /tmp/capture.pcap port 9999

# Examine
tcpdump -r /tmp/capture.pcap -X
```

What does the packet hex look like?

**Expected output:**
```
Packet data contains "hello" with length prefix
TCP handshake visible (SYN, SYN-ACK, ACK)
```

> [!hint]- Hint
> tcpdump shows packet headers and payload. The `-X` flag shows hex + ASCII.

> [!success]- Solution
> The capture shows:
> - TCP SYN → SYN-ACK → ACK (3-way handshake)
> - Data packet with "hello"
> - ACK from server
> The hex dump shows the raw bytes including TCP/IP headers.

---

## P2: Extract Payload from PCAP

```python
from scapy.all import rdpcap, TCP

packets = rdpcap('/tmp/capture.pcap')
for p in packets:
    if TCP in p and p[TCP].payload:
        print(f"From {p[TCP].sport} -> {p[TCP].dport}:")
        print(bytes(p[TCP].payload))
```

What is the extracted payload?

**Expected output:**
```
b'hello\n'
```

> [!hint]- Hint
> Scapy can parse PCAP files and extract TCP payloads.

> [!success]- Solution
> The Python script extracts the raw TCP payload bytes. For simple protocols, the payload is just the application data. For more complex protocols, this is where the protocol data begins.

---

## P3: Identify Fixed Header Structure

You capture several packets from an unknown protocol:

```
Packet 1: 01 00 00 0A 48 65 6C 6C 6F
Packet 2: 01 00 00 0D 57 6F 72 6C 64 21 0A 00 00
Packet 3: 02 00 00 05 54 65 73 74
```

What structure can you infer?

**Expected output:**
```
Byte 0: type (01 or 02)
Bytes 1-2: unknown (00 00)
Bytes 3: length of payload
Bytes 4+: payload

01 = string message, 0A = length 10, "Hello"
02 = command, 05 = length 5, "Test"
```

> [!hint]- Hint
> Compare packets byte-by-byte. The first 3 bytes look similar across packets. The 4th byte (0-indexed) seems to be a length field.

> [!success]- Solution
> Proposed structure:
> ```
> [1 byte: type][2 bytes: flags/unknown][1 byte: length][N bytes: payload]
> ```
> Packet 1: type=0x01, length=0x0A=10, payload="Hello" (padded to 10)
> Packet 2: type=0x01, length=0x0D=13, payload="World!\n" + padding
> Packet 3: type=0x02, length=0x05=5, payload="Test"

---

## P4: Identify Length-Prefixed Strings

From the packets in P3, can you find where the string ends?

```
Packet 1: 01 00 00 0A 48 65 6C 6C 6F 00 00 00 00 00
Packet 2: 01 00 00 0D 57 6F 72 6C 64 21 0A 00 00 00
```

What does the protocol look like?

**Expected output:**
```
Length prefix (1 byte): length = total packet length or payload length?
String is null-terminated after the ASCII text
Padding bytes follow the string
```

> [!hint]- Hint
> The null bytes in packets might indicate null-terminated strings. The length byte might be the total packet length including header.

> [!success]- Solution
> Alternative structure:
> ```
> [1 byte: total length][1 byte: type][payload][null padding]
> ```
> Packet 1: total=0x0A=10 bytes, type=0x01, payload="Hello" (5 chars) + 4 nulls
> Packet 2: total=0x0D=13 bytes, type=0x01, payload="World!\n" (7 chars) + 5 nulls
> The length is total packet length, strings are null-terminated, padded to fixed size.

---

## P5: Replay Attack

You capture a login sequence:

```
Client -> Server: 05 01 48 65 6C 6C 6F
Server -> Client: 06 01 4F 4B 00 00 00 00 00
```

Replay the login packet to test for authentication bypass:

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('localhost', 9999))

# Replay the login packet
packet = bytes.fromhex("05 01 48 65 6C 6C 6F")
s.send(packet)
response = s.recv(1024)
print(response.hex())
```

**Expected output:**
```
Server response may be "OK" or authentication accepted
This tests if the server validates sequence/ freshness
```

> [!hint]- Hint
> If the server accepts a replay of the same packet, it means no freshness validation (replay attack possible).

> [!success]- Solution
> If the server responds with "OK" on replay, the protocol lacks freshness validation. No nonce, timestamp, or sequence number protects against replay attacks.

---

## P6: Protocol State Machine

You observe a multi-step protocol:

```
1. Client -> Server: HELLO (client greeting)
2. Server -> Client: CHALLENGE (random nonce)
3. Client -> Server: AUTH response (proves client identity)
4. Server -> Client: ACCEPT/REJECT
```

Draw the state machine. What are the valid transitions?

**Expected output:**
```
States: INITIAL -> CHALLENGED -> AUTHENTICATED/REJECTED
Invalid: starting with AUTH (no challenge first)
```

> [!hint]- Hint
> The protocol requires a challenge-response exchange. Clients must receive CHALLENGE before sending AUTH.

> [!success]- Solution
> State machine:
> ```
> INITIAL: wait for HELLO
>   <- HELLO -> CHALLENGED
> CHALLENGED: wait for AUTH
>   <- AUTH (verify response)
>     -> ACCEPT (if correct) -> AUTHENTICATED
>     -> REJECT (if wrong)
> ```
> Invalid transitions: AUTH without HELLO, ACCEPT without AUTH, etc.

---

## P7: Wireshark Follow TCP Stream

```bash
# Start a TCP server that speaks a custom protocol
python3 -c "
import socket
s = socket.socket()
s.bind(('', 9000))
s.listen(1)
c, a = s.accept()
data = c.recv(1024)
print('Received:', data)
c.send(b'ECHO:' + data)
c.close()
" &

# Send some packets
echo "test" | nc localhost 9000
echo "hello" | nc localhost 9000

# Capture and follow stream
sudo tcpdump -i lo -w /tmp/tcp.pcap port 9000
# Then use:
# wireshark -r /tmp/tcp.pcap -Y "tcp.stream eq 0" -T fields -e data
```

**Expected output:**
```
Wireshark shows bidirectional conversation:
Client: test
Server: ECHO:test
Client: hello
Server: ECHO:hello
```

> [!hint]- Hint
> Wireshark's "Follow TCP Stream" reconstructs the full conversation.

> [!success]- Solution
> The Wireshark follow-stream feature shows the full bidirectional TCP conversation, making it easy to see the request-response pattern and infer the protocol structure.

---

## P8: Binary Protocol Parsing

A binary protocol packet looks like:

```
[2 bytes: magic = 0xCAFE]
[2 bytes: sequence number]
[1 byte: message type]
[2 bytes: payload length]
[N bytes: payload]
[4 bytes: CRC32 checksum]
```

Write a parser:

```python
import struct

def parse_packet(data):
    if len(data) < 11:
        return None
    magic, seq, msg_type, length = struct.unpack('<HHBB', data[:6])
    if magic != 0xCAFE:
        return None
    payload = data[6:6+length]
    checksum = struct.unpack('<I', data[6+length:10+length])[0]
    return {'magic': magic, 'seq': seq, 'type': msg_type,
            'length': length, 'payload': payload, 'checksum': checksum}

# Test
packet = struct.pack('<HHBB', 0xCAFE, 1, 0x01, 5) + b'Hello' + struct.pack('<I', 0)
print(parse_packet(packet))
```

**Expected output:**
```
{'magic': 51966, 'seq': 1, 'type': 1, 'length': 5, 'payload': b'Hello', 'checksum': 0}
```

> [!hint]- Hint
> Binary protocols use fixed offsets and specific byte sizes. Little-endian (`<`) is common on x86.

> [!success]- Solution
> The parser correctly extracts all fields from the binary packet structure. CRC32 can be verified against the payload using `zlib.crc32()`.

---

## P9: Infer Protocol from NetCat Traffic

```bash
# Capture traffic between client and server
# Analyze with xxd
tcpdump -r /tmp/tcp.pcap -x | head -40

# Or extract ASCII from packets
tcpdump -r /tmp/tcp.pcap -A | head -40
```

What ASCII strings can you identify?

**Expected output:**
```
ASCII strings like "GET / HTTP/1.1" for HTTP
Or custom protocol commands like "AUTH", "DATA", etc.
```

> [!hint]- Hint
> ASCII strings in packet captures are often protocol commands, headers, or data. Filter by port and look for patterns.

> [!success]- Solution
> ASCII extraction reveals:
> - HTTP: "GET / HTTP/1.1", "Host:", "User-Agent:"
> - Custom: "AUTH", "CHALLENGE", "DATA", "OK", "ERROR"
> The ASCII portions reveal the protocol's message types and structure.

---

## P10: Fuzz the Protocol

```python
import socket

# Known good packet
good = bytes.fromhex("01 00 00 0A 48 65 6C 6C 6F")

# Fuzz variations
for i in range(len(good)):
    for val in [0x00, 0xFF, 0x41]:
        mutated = bytearray(good)
        mutated[i] = val
        try:
            s = socket.socket()
            s.settimeout(1)
            s.connect(('localhost', 9999))
            s.send(bytes(mutated))
            resp = s.recv(1024)
            print(f"Offset {i} = 0x{val:02x}: resp={resp.hex()}")
            s.close()
        except Exception as e:
            print(f"Offset {i} = 0x{val:02x}: {e}")
```

What do you learn from fuzzing?

**Expected output:**
```
Some mutations cause crash/timeout (crashes server)
Some cause error responses (server handles gracefully)
Some have no effect
```

> [!hint]- Hint
> Fuzzing reveals how the server handles malformed input. This helps understand what fields the server parses and how.

> [!success]- Solution
> Fuzzing results:
> - Mutations at length field offset → server truncates or ignores
> - Mutations at magic field → server rejects (good security)
> - Mutations in payload → varies by server implementation
> - Crashes → vulnerabilities to investigate further

---

## P11: Implement Protocol Client

Based on your analysis, implement a client for the unknown protocol:

```python
import socket
import struct

class ProtocolClient:
    def __init__(self, host, port):
        self.sock = socket.socket()
        self.sock.connect((host, port))
    
    def send_message(self, msg_type, payload):
        packet = struct.pack('<HHBB', 0xCAFE, self.seq, msg_type, len(payload))
        packet += payload
        packet += struct.pack('<I', self._crc32(packet))
        self.sock.send(packet)
        self.seq += 1
        return self._recv_response()
    
    def _recv_response(self):
        data = self.sock.recv(4096)
        return data
    
    def _crc32(self, data):
        import zlib
        return zlib.crc32(data) & 0xffffffff

client = ProtocolClient('localhost', 9000)
client.send_message(0x01, b'Hello')
```

**Expected output:**
```
Client successfully communicates with server
Packet structure validated
```

> [!hint]- Hint
> Once the protocol is reverse-engineered, implementing a client is straightforward. This is useful for testing or interacting with proprietary servers.

> [!success]- Solution
> The client correctly implements the inferred protocol and can communicate with the server. This validates the reverse-engineering analysis and provides a reusable tool.

---

## AI Power-Up: Protocol RE

```
PROMPT: "I'm reverse engineering an unknown network protocol. Here is a hex dump
of several sequential packets in both directions (client -> server and server ->
client). Identify the packet structure, field boundaries, and suggest what each
field means based on patterns:

[packet hex dumps]"
```
