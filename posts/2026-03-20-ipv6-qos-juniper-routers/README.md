# How to Configure IPv6 QoS Policies on Juniper Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Juniper, IPv6, QoS, DSCP, Class of Service, Junos, Router

Description: Configure IPv6 Quality of Service on Juniper routers using Class of Service (CoS), including DSCP rewriting, forwarding classes, scheduler policies, and interface application.

---

Juniper JunOS uses Class of Service (CoS) for QoS configuration. CoS handles IPv6 and IPv4 uniformly since both use the same DSCP field (IPv6 Traffic Class), making IPv6 QoS configuration identical to IPv4 in most cases.

## Juniper CoS Classifier for IPv6

```text
# Juniper JunOS CoS Configuration

# Define DSCP classifier (applies to both IPv4 and IPv6)

set class-of-service classifiers dscp IPv6-DSCP-CLASSIFIER \
  forwarding-class VOIP-MEDIA loss-priority low code-point 101110  # EF
set class-of-service classifiers dscp IPv6-DSCP-CLASSIFIER \
  forwarding-class VOIP-SIGNAL loss-priority low code-point 101000  # CS5
set class-of-service classifiers dscp IPv6-DSCP-CLASSIFIER \
  forwarding-class VIDEO loss-priority low code-point 100010  # AF41
set class-of-service classifiers dscp IPv6-DSCP-CLASSIFIER \
  forwarding-class DATA loss-priority low code-point 001010  # AF11
set class-of-service classifiers dscp IPv6-DSCP-CLASSIFIER \
  forwarding-class BEST-EFFORT loss-priority low code-point 000000  # CS0

commit
```

## Forwarding Classes and Queues

```text
# Define forwarding classes (mapped to queues)
set class-of-service forwarding-classes queue 0 BEST-EFFORT
set class-of-service forwarding-classes queue 1 DATA
set class-of-service forwarding-classes queue 2 VIDEO
set class-of-service forwarding-classes queue 3 VOIP-SIGNAL
set class-of-service forwarding-classes queue 7 VOIP-MEDIA

commit
```

## Scheduler Policy for IPv6 QoS

```text
# Define scheduler for each forwarding class
set class-of-service schedulers SCHED-VOIP-MEDIA \
  transmit-rate percent 30 \
  priority strict-high \
  buffer-size temporal 5ms

set class-of-service schedulers SCHED-VOIP-SIGNAL \
  transmit-rate percent 5 \
  buffer-size percent 5

set class-of-service schedulers SCHED-VIDEO \
  transmit-rate percent 25 \
  buffer-size percent 25 \
  random-early-detection medium

set class-of-service schedulers SCHED-DATA \
  transmit-rate percent 20 \
  buffer-size percent 20

set class-of-service schedulers SCHED-BEST-EFFORT \
  transmit-rate remainder \
  buffer-size remainder

# Map schedulers to forwarding classes
set class-of-service scheduler-maps WAN-SCHED-MAP \
  forwarding-class VOIP-MEDIA scheduler SCHED-VOIP-MEDIA
set class-of-service scheduler-maps WAN-SCHED-MAP \
  forwarding-class VOIP-SIGNAL scheduler SCHED-VOIP-SIGNAL
set class-of-service scheduler-maps WAN-SCHED-MAP \
  forwarding-class VIDEO scheduler SCHED-VIDEO
set class-of-service scheduler-maps WAN-SCHED-MAP \
  forwarding-class DATA scheduler SCHED-DATA
set class-of-service scheduler-maps WAN-SCHED-MAP \
  forwarding-class BEST-EFFORT scheduler SCHED-BEST-EFFORT

commit
```

## Rewrite Rules for IPv6 DSCP

```text
# Define rewrite rules (for remarking IPv6 Traffic Class)
set class-of-service rewrite-rules dscp IPv6-DSCP-REWRITE \
  forwarding-class VOIP-MEDIA loss-priority low code-point 101110  # EF
set class-of-service rewrite-rules dscp IPv6-DSCP-REWRITE \
  forwarding-class VIDEO loss-priority low code-point 100010  # AF41
set class-of-service rewrite-rules dscp IPv6-DSCP-REWRITE \
  forwarding-class BEST-EFFORT loss-priority low code-point 000000  # CS0

commit
```

## Applying CoS to IPv6 Interface

```text
# Apply CoS to interface (handles both IPv4 and IPv6)
set class-of-service interfaces ge-0/0/0 \
  scheduler-map WAN-SCHED-MAP
set class-of-service interfaces ge-0/0/0 \
  unit 0 classifiers dscp IPv6-DSCP-CLASSIFIER
set class-of-service interfaces ge-0/0/0 \
  unit 0 rewrite-rules dscp IPv6-DSCP-REWRITE

# Verify interface CoS configuration
run show class-of-service interface ge-0/0/0

commit
```

## Monitoring IPv6 QoS on Juniper

```text
# Check queue statistics
show class-of-service interface ge-0/0/0 comprehensive

# View per-queue statistics
show interfaces ge-0/0/0 detail | find "Queue counters"

# Check DSCP classification
show class-of-service classifier name IPv6-DSCP-CLASSIFIER

# Monitor real-time queue usage
monitor interface ge-0/0/0

# Check CoS applied to interface
show class-of-service interface ge-0/0/0

# Verify forwarding class assignment for a test packet
run test class-of-service dscp-classifier IPv6-DSCP-CLASSIFIER \
  forwarding-class VOIP-MEDIA code-point 101110
```

## Firewall Filter for IPv6 QoS (Alternative Approach)

```text
# Using firewall filter to classify and mark IPv6 traffic
set firewall family inet6 filter IPV6-QOS-MARK \
  term VOIP-MEDIA from protocol udp \
  term VOIP-MEDIA from destination-port 10000-20000 \
  term VOIP-MEDIA then dscp ef \
  term VOIP-MEDIA then count VOIP-COUNTER \
  term VOIP-MEDIA then accept

set firewall family inet6 filter IPV6-QOS-MARK \
  term DEFAULT then accept

# Apply filter to interface input
set interfaces ge-0/0/1 unit 0 \
  family inet6 filter input IPV6-QOS-MARK

commit
```

Juniper's CoS framework handles IPv6 and IPv4 QoS uniformly since DSCP classification uses the same bit patterns in both the IPv4 ToS field and IPv6 Traffic Class field, requiring only a single classifier and scheduler policy for both protocols.
