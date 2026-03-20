# How to Implement Reliable Communication Over UDP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Reliability, ARQ, Sequence Numbers, Retransmission, Protocol Design

Description: Implement reliable delivery over UDP using sequence numbers, acknowledgments, and retransmission logic for applications that need custom reliability semantics.

## Introduction

UDP provides no reliability guarantees, but you can add exactly as much reliability as your application needs. This gives you more control than TCP: you can choose which messages need acknowledgment, how long to wait before retransmitting, and what to do with out-of-order arrivals. This is why protocols like QUIC and game networking libraries are built on UDP rather than TCP.

## Basic Reliable UDP Pattern

```python
#!/usr/bin/env python3
# reliable_udp.py - Simple stop-and-wait reliable UDP

import socket
import struct
import time
import threading

TIMEOUT = 0.5  # 500ms retransmit timeout
MAX_RETRIES = 5

# Packet format: 4-byte seq_num + 1-byte flags + data

# Flags: 0x01 = ACK, 0x02 = DATA
HEADER_FORMAT = '!IB'  # network byte order: uint32 seq_num, uint8 flags
HEADER_SIZE = struct.calcsize(HEADER_FORMAT)

def pack_packet(seq_num, flags, data=b''):
    header = struct.pack(HEADER_FORMAT, seq_num, flags)
    return header + data

def unpack_packet(packet):
    header = packet[:HEADER_SIZE]
    seq_num, flags = struct.unpack(HEADER_FORMAT, header)
    data = packet[HEADER_SIZE:]
    return seq_num, flags, data

class ReliableUDP:
    DATA = 0x02
    ACK = 0x01

    def __init__(self, local_addr):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.sock.bind(local_addr)
        self.sock.settimeout(TIMEOUT)
        self.next_seq = 0

    def send_reliable(self, data, remote_addr):
        """Send data and wait for ACK, retransmitting if needed."""
        seq = self.next_seq
        packet = pack_packet(seq, self.DATA, data)
        self.next_seq = (self.next_seq + 1) % (2**32)

        for attempt in range(MAX_RETRIES):
            self.sock.sendto(packet, remote_addr)
            try:
                resp, _ = self.sock.recvfrom(65535)
                ack_seq, flags, _ = unpack_packet(resp)
                if flags & self.ACK and ack_seq == seq:
                    return True  # ACKed
            except socket.timeout:
                print(f"Timeout, retrying (attempt {attempt+1}/{MAX_RETRIES})")

        return False  # Failed after max retries

    def recv_reliable(self):
        """Receive data and send ACK."""
        while True:
            try:
                packet, addr = self.sock.recvfrom(65535)
                seq, flags, data = unpack_packet(packet)
                if flags & self.DATA:
                    # Send ACK
                    ack = pack_packet(seq, self.ACK)
                    self.sock.sendto(ack, addr)
                    return data, addr
            except socket.timeout:
                continue
```

## Sliding Window for Higher Throughput

```python
# Stop-and-wait limits throughput to 1 packet per RTT
# Sliding window allows N packets in flight simultaneously

# Window size determines throughput:
# throughput = window_size * packet_size / RTT

# For 100ms RTT and 1KB packets:
# window=1:   8 Kbps
# window=10:  80 Kbps
# window=100: 800 Kbps

class SlidingWindowSender:
    def __init__(self, window_size=10):
        self.window_size = window_size
        self.sent = {}       # seq -> (data, timestamp)
        self.next_seq = 0
        self.ack_base = 0

    def can_send(self):
        # Number of unacknowledged packets must be < window_size
        return (self.next_seq - self.ack_base) < self.window_size

    def send(self, sock, data, addr):
        if not self.can_send():
            return False  # Window full, wait

        seq = self.next_seq
        packet = pack_packet(seq, ReliableUDP.DATA, data)
        sock.sendto(packet, addr)
        self.sent[seq] = (data, time.time())
        self.next_seq += 1
        return True

    def process_ack(self, ack_seq):
        # Cumulative ACK: ack_seq means "received all up to ack_seq"
        if ack_seq in self.sent:
            # Remove all acknowledged packets
            for seq in list(self.sent.keys()):
                if seq <= ack_seq:
                    del self.sent[seq]
            self.ack_base = ack_seq + 1

    def retransmit_expired(self, sock, addr, timeout=0.5):
        """Retransmit packets that haven't been ACKed within timeout."""
        now = time.time()
        for seq, (data, sent_time) in list(self.sent.items()):
            if now - sent_time > timeout:
                packet = pack_packet(seq, ReliableUDP.DATA, data)
                sock.sendto(packet, addr)
                self.sent[seq] = (data, now)  # Reset timer
                print(f"Retransmitting seq={seq}")
```

## When to Use Custom Reliable UDP vs TCP

```text
Use custom reliable UDP when:
  ✓ You need selective reliability (some messages need ACK, others don't)
  ✓ You want message-level framing (not byte-stream like TCP)
  ✓ You need custom retransmit timing (e.g., give up after 1 retry)
  ✓ You're implementing a protocol for IoT or embedded with constrained stack
  ✓ Head-of-line blocking in TCP would hurt you (game state vs chat messages)

Use TCP or QUIC when:
  ✓ You need full reliability and ordering for all data
  ✓ You're building on top of HTTP or existing protocols
  ✓ Development speed matters more than control
```

## Conclusion

Reliable UDP is not about reimplementing TCP - it is about implementing exactly the reliability your application needs. Stop-and-wait works for low-rate request/response. Sliding window enables higher throughput. Selective acknowledgment (only ACK important packets) works for game state. The key advantage over TCP is granular control: you decide which messages need reliability, how long to wait for ACKs, and how many times to retry before giving up.
