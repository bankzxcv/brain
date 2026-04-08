---
title: "RE Workflow: Analyze a Network Protocol"
date: 2026-04-07
tags:
  - reverse-engineering
  - workflow
  - project
  - network
  - protocol
  - wireshark
parent: "[[Reverse Engineering/Reverse Engineering Study]]"
---

# RE Workflow: Analyze a Network Protocol

## Overview

In this project, you will reverse engineer an unknown network protocol by capturing traffic, analyzing packet structure, identifying patterns, and implementing a client.

**Scenario:** You have a server binary and need to understand its protocol to write a compatible client.

**Concepts practiced:** Packet capture, protocol structure analysis, length-field detection, state machine inference, Python client implementation.

---

## Step-by-Step Implementation

### Step 1: Create a Test Server

First, create a server with a custom protocol to reverse engineer:

```bash
cat > /tmp/protocol_server.py << 'EOF'
#!/usr/bin/env python3
import socket
import struct
import random

def handle_client(conn, addr):
    print(f"Connection from {addr}")
    
    # Protocol: 
    # [1 byte: msg_type]
    # [2 bytes: sequence number (big-endian)]
    # [2 bytes: payload length (big-endian)]
    # [N bytes: payload]
    # [4 bytes: CRC32 checksum]
    
    while True:
        header = conn.recv(5)
        if not header:
            break
        
        msg_type, seq, length = struct.unpack('!BHH', header)
        
        payload = b''
        while len(payload) < length:
            chunk = conn.recv(length - len(payload))
            if not chunk:
                break
            payload += chunk
        
        # Verify checksum (skip for now)
        
        print(f"Received: type={msg_type}, seq={seq}, len={length}, payload={payload}")
        
        if msg_type == 0x01:  # HELLO
            response = struct.pack('!BHH', 0x02, seq, 0)
            conn.sendall(response)
            
        elif msg_type == 0x03:  # DATA
            # Echo back with ACK
            ack_payload = b'ACK:' + payload
            response = struct.pack('!BHH', 0x04, seq, len(ack_payload))
            response += ack_payload
            conn.sendall(response)
            
        elif msg_type == 0x05:  # QUIT
            response = struct.pack('!BHH', 0x06, seq, 0)
            conn.sendall(response)
            break
    
    conn.close()
    print(f"Client {addr} disconnected")

def main():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(('0.0.0.0', 9999))
    s.listen(5)
    print("Server listening on port 9999")
    
    while True:
        conn, addr = s.accept()
        handle_client(conn, addr)

if __name__ == '__main__':
    main()
EOF
chmod +x /tmp/protocol_server.py
```

### Step 2: Capture Traffic

Start the server and capture traffic:

```bash
# Terminal 1: Start server
python3 /tmp/protocol_server.py &

# Terminal 2: Capture traffic
sudo tcpdump -i lo -w /tmp/protocol.pcap port 9999 &

# Terminal 3: Run client interactions
sleep 1
python3 << 'PYEOF'
import socket
import struct

def send_packet(sock, msg_type, payload):
    data = struct.pack('!BHH', msg_type, random.randint(1,100), len(payload))
    data += payload
    sock.sendall(data)

sock = socket.socket()
sock.connect(('localhost', 9999))

# HELLO
send_packet(sock, 0x01, b'client_hello')
resp = sock.recv(1024)
print(f"HELLO response: {resp.hex()}")

# DATA
send_packet(sock, 0x03, b'test_data')
resp = sock.recv(1024)
print(f"DATA response: {resp.hex()}")

# QUIT
send_packet(sock, 0x05, b'')
resp = sock.recv(1024)
print(f"QUIT response: {resp.hex()}")

sock.close()
PYEOF

# Stop capture
sleep 1
kill %2 2>/dev/null
kill %1 2>/dev/null
```

### Step 3: Analyze Packets with Wireshark

Open `/tmp/protocol.pcap` in Wireshark:

```bash
wireshark /tmp/protocol.pcap &
```

Or use tshark for CLI analysis:

```bash
# Show all packets with payload
tshark -r /tmp/protocol.pcap -T fields -e data | head -10

# Follow TCP stream
tshark -r /tmp/protocol.pcap -z "follow,tcp,ascii,0"
```

**Analysis questions:**
1. What is the packet header structure?
2. How is the payload length encoded?
3. What message types exist?
4. How does the server respond?

### Step 4: Extract and Examine Payload

Write a script to extract raw payloads:

```python
import struct
from scapy.all import rdpcap, TCP

packets = rdpcap('/tmp/protocol.pcap')
for i, p in enumerate(packets):
    if TCP in p and p[TCP].payload:
        payload = bytes(p[TCP].payload)
        print(f"Packet {i}: {payload.hex()}")
        if len(payload) >= 5:
            msg_type, seq, length = struct.unpack('!BHH', payload[:5])
            print(f"  Type: 0x{msg_type:02X}, Seq: {seq}, Len: {length}")
            if len(payload) > 5:
                print(f"  Payload: {payload[5:]}")
```

### Step 5: Infer Protocol Structure

From the captured packets, determine:

| Field | Size | Offset | Description |
|-------|------|--------|-------------|
| msg_type | 1 byte | 0 | Message type (0x01=hello, etc.) |
| sequence | 2 bytes | 1 | Sequence number (big-endian) |
| length | 2 bytes | 3 | Payload length (big-endian) |
| payload | N bytes | 5 | Message data |
| checksum | 4 bytes | end | CRC32 (optional) |

### Step 6: Implement a Python Client

Based on analysis, write a client:

```python
import socket
import struct
import random
import zlib

class ProtocolClient:
    MSG_HELLO = 0x01
    MSG_HELLO_ACK = 0x02
    MSG_DATA = 0x03
    MSG_DATA_ACK = 0x04
    MSG_QUIT = 0x05
    MSG_QUIT_ACK = 0x06
    
    def __init__(self, host, port):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((host, port))
        self.seq = 0
    
    def _send(self, msg_type, payload=b''):
        header = struct.pack('!BHH', msg_type, self.seq, len(payload))
        self.sock.sendall(header + payload)
        self.seq += 1
    
    def _recv_raw(self):
        header = b''
        while len(header) < 5:
            chunk = self.sock.recv(5 - len(header))
            if not chunk:
                return None, None, None
            header += chunk
        msg_type, seq, length = struct.unpack('!BHH', header)
        payload = b''
        while len(payload) < length:
            chunk = self.sock.recv(length - len(payload))
            if not chunk:
                break
            payload += chunk
        return msg_type, seq, payload
    
    def hello(self, client_id):
        self._send(self.MSG_HELLO, client_id.encode())
        msg_type, seq, payload = self._recv_raw()
        assert msg_type == self.MSG_HELLO_ACK, f"Expected HELLO_ACK, got 0x{msg_type:02X}"
        return payload
    
    def send_data(self, data):
        self._send(self.MSG_DATA, data)
        msg_type, seq, payload = self._recv_raw()
        assert msg_type == self.MSG_DATA_ACK, f"Expected DATA_ACK, got 0x{msg_type:02X}"
        return payload
    
    def quit(self):
        self._send(self.MSG_QUIT, b'')
        self._recv_raw()
        self.sock.close()

# Test the client
client = ProtocolClient('localhost', 9999)
print(client.hello('test_client'))
print(client.send_data(b'hello world'))
client.quit()
```

### Step 7: Verify and Document

Test the client and document the protocol:

```markdown
## Protocol Analysis: Custom Binary Protocol

### Overview
TCP-based binary protocol with request-response pattern.

### Packet Structure

```
[1 byte: msg_type]
[2 bytes: sequence (big-endian unsigned short)]
[2 bytes: payload_length (big-endian unsigned short)]
[N bytes: payload]
[4 bytes: checksum (CRC32, optional)]
```

### Message Types

| Type | Name | Direction | Payload |
|------|------|----------|---------|
| 0x01 | HELLO | C→S | Client identifier string |
| 0x02 | HELLO_ACK | S→C | None |
| 0x03 | DATA | C→S | Binary data |
| 0x04 | DATA_ACK | S→C | "ACK:" + echoed data |
| 0x05 | QUIT | C→S | None |
| 0x06 | QUIT_ACK | S→C | None |

### State Machine

```
CLIENT                      SERVER
  |                            |
  |------ HELLO ------------->|
  |<----- HELLO_ACK ----------|
  |                            |
  |------ DATA -------------->|
  |<----- DATA_ACK ------------|
  |      (repeat)              |
  |                            |
  |------ QUIT -------------->|
  |<----- QUIT_ACK ------------|
  |                            |
```

### Implementation
See: protocol_client.py
```

> [!success]- Full Solution

The complete protocol reverse engineering workflow produces:
1. Captured packet samples
2. Inferred packet structure
3. Identified message types and their semantics
4. State machine diagram
5. Working Python client implementation

---

## Extensions / Challenges

1. **Obfuscated protocol**: Add XOR encryption to packets and reverse the encryption
2. **Protocol with authentication**: Server requires a login sequence before accepting commands
3. **Multipart packets**: Payloads that span multiple TCP segments
4. **Binary + text hybrid**: Some fields are binary, some are null-terminated strings
5. **Protocol versioning**: Server supports multiple protocol versions, detect which is in use

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `tcpdump` | Packet capture |
| `Wireshark` | Packet analysis |
| `tshark` | CLI packet analysis |
| `scapy` | Python packet manipulation |
| Python | Client implementation |
