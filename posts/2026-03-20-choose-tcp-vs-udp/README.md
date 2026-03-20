# How to Choose Between TCP and UDP for Your Application

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, UDP, Protocol Selection, Networking, Application Design

Description: Apply a structured decision framework to choose between TCP and UDP based on your application's requirements for reliability, latency, ordering, and delivery guarantees.

## Introduction

Every networked application must choose between TCP and UDP. The choice is not about which protocol is "better" - it is about which properties your application needs. Making the wrong choice means either building complex reliability logic on top of UDP unnecessarily, or accepting TCP's latency when your application would benefit from UDP's speed.

## Decision Framework

```text
Question 1: Does every message need to arrive?
  YES → Strongly prefer TCP (reliability built-in)
  NO  → UDP is viable; continue to question 2

Question 2: Does message ordering matter?
  YES → Strongly prefer TCP (ordering built-in)
  NO  → UDP is viable; continue to question 3

Question 3: Is latency more important than reliability?
  YES → Use UDP (possibly with application-level FEC or retry)
  NO  → Use TCP

Question 4: Do you need broadcast or multicast?
  YES → Must use UDP (TCP is unicast only)

Question 5: Are messages short (< 1 packet) and infrequent?
  YES → UDP is more efficient (no handshake overhead)
  NO  → TCP's persistent connection amortizes setup cost
```

## By Application Category

```bash
# === Financial Transactions ===

# Requirement: every order must arrive, in order, exactly once
# Choice: TCP
# Reason: TCP reliability and ordering match requirement exactly

# === DNS Queries ===
# Requirement: fast, single request/response, tolerate retry at app layer
# Choice: UDP (with TCP fallback for large responses)
# Reason: sub-millisecond UDP query vs 1.5 RTT TCP handshake

# === VoIP / Real-time Audio ===
# Requirement: low latency, tolerate occasional packet loss (better gap than delay)
# Choice: UDP (typically with RTP)
# Reason: late audio packet is worse than missing audio packet

# === Video Streaming (live) ===
# Requirement: real-time, some loss acceptable, low latency
# Choice: UDP (QUIC/RTP); sometimes TCP with small buffers
# Reason: stale video frames are useless; better to skip and continue

# === File Transfer ===
# Requirement: all bytes must arrive, correct order, checksummed
# Choice: TCP (SFTP, HTTP, etc.)
# Reason: TCP reliability is exactly what file transfer needs

# === Game State Updates ===
# Requirement: latest state matters, old states can be dropped
# Choice: UDP
# Reason: retransmitting stale position data wastes bandwidth

# === Database Queries ===
# Requirement: every query and response must arrive, in order
# Choice: TCP
# Reason: data integrity is paramount

# === Service Discovery (mDNS, SSDP) ===
# Requirement: broadcast/multicast to all devices on network
# Choice: UDP (multicast)
# Reason: TCP cannot multicast
```

## Performance Characteristics

```bash
# Measure TCP connection overhead vs UDP
# TCP: requires 3-way handshake before first byte
# RTT cost: 1.5 RTT minimum before application data (SYN, SYN-ACK, ACK+Data)

# For 1ms RTT:
# TCP minimum: 1.5ms before first byte (TLS adds more)
# UDP minimum: 0ms - first packet IS the request

# This matters for short-lived, high-frequency connections
# For persistent connections (HTTP/1.1 keep-alive, HTTP/2): TCP amortizes this cost

# Throughput (without congestion):
# TCP: rate-limited by CWND and retransmit behavior
# UDP: can fill link capacity but no flow control → potential for loss

# For high-throughput bulk transfer over reliable networks: TCP wins
# For controlled-rate streaming: UDP with rate limiting wins
```

## Hybrid Approach: QUIC

```bash
# QUIC = UDP + reliability + ordering + congestion control + TLS
# Best of both worlds for web traffic:
# - UDP base: no kernel TCP state, faster handshake (0-RTT reconnect)
# - Reliability per stream (not for all streams)
# - Stream independence: no head-of-line blocking (unlike TCP)

# Check if your service supports QUIC (HTTP/3):
curl -I --http3 https://example.com 2>/dev/null | grep -i "alt-svc\|http/3"

# For new protocol design: consider QUIC or similar (WebRTC, DCCP)
```

## Conclusion

The choice is mechanical once you know your requirements. If all messages must arrive in order: use TCP. If you can tolerate loss and need low latency: use UDP. If you need broadcast/multicast: you must use UDP. For modern high-performance applications that want reliability with lower latency than TCP, QUIC (UDP-based) is the right answer. There is no universal winner - the right choice depends entirely on the application's reliability, ordering, and latency requirements.
