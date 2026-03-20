# How to Monitor TCP Window Size Changes During a Transfer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Networking, Monitoring, Window Size, Performance, Wireshark

Description: Monitor how TCP window size changes dynamically during a data transfer to understand flow control behavior and identify performance bottlenecks.

## Introduction

TCP window sizes are dynamic - they grow and shrink in response to buffer availability, application read speed, and congestion events. Monitoring window size changes during a live transfer reveals whether throughput is limited by the network, receiver buffer, or application processing speed.

## Monitoring with ss During a Transfer

```bash
# Start a large file transfer in the background

scp /tmp/largefile user@10.20.0.5:/dev/null &

# Monitor the connection's window sizes in real time
watch -n 0.5 'ss -tin state established "( dst 10.20.0.5 )" | grep -A2 snd_wnd'

# Key fields to watch:
# snd_wnd: sender's view of receiver's window (bytes available to send)
# rcv_space: local receive buffer allocated
# sndbuf: total send buffer size
```

## Capturing Window Size History with tcpdump

```bash
# Capture all TCP traffic during a transfer
tcpdump -i eth0 -n -w /tmp/window_monitor.pcap \
  'tcp and host 10.20.0.5 and port 8080'

# Extract window sizes over time
tshark -r /tmp/window_monitor.pcap \
  -Y "ip.dst == 10.20.0.5" \
  -T fields \
  -e frame.time_relative \
  -e tcp.window_size \
  | head -100
```

## Visualizing with Wireshark

```text
# In Wireshark: Statistics → TCP Stream Graphs → Window Scaling
# This shows the receiver's advertised window over time

# Watch for:
# - Steadily growing window: normal slow start / connection growing
# - Flat plateau: receiver buffer is limiting (window not growing further)
# - Sudden drops to 0: Zero Window (receiver buffer full)
# - Sawtooth pattern: TCP congestion control shrinking and growing window
```

## Scripted Window Size Monitoring

```bash
#!/bin/bash
# Log window sizes every 100ms during a transfer
TARGET="10.20.0.5"
PORT="8080"

echo "time_s,snd_wnd_bytes" > /tmp/window_log.csv

while ss -tin state established "( dst $TARGET dport = :$PORT )" | \
      grep -q snd_wnd; do
    WINDOW=$(ss -tin state established "( dst $TARGET dport = :$PORT )" | \
             grep -oP 'snd_wnd:\K\d+')
    TIME=$(date +%s.%N)
    echo "$TIME,$WINDOW" >> /tmp/window_log.csv
    sleep 0.1
done

echo "Logged window sizes to /tmp/window_log.csv"

# Basic analysis
awk -F, 'NR>1{sum+=$2; count++; if($2<min||min=="")min=$2; if($2>max)max=$2}
  END{print "Min:", min, "Max:", max, "Avg:", sum/count " bytes"}' \
  /tmp/window_log.csv
```

## Interpreting Window Size Patterns

```text
Pattern                  | Meaning
-------------------------|------------------------------------------
Steadily increasing      | Normal slow start, no congestion
Flat at large value      | Receiver keeping up, network saturated
Oscillating up/down      | Congestion control (CUBIC/BBR) in action
Drops to zero            | Receiver buffer full (Zero Window)
Stays at zero            | Application not reading, dead connection?
Frequent small drops     | Packet loss triggering fast recovery
Gradual decline          | Receiver getting slower (processing backlog)
```

## Conclusion

Window size monitoring during transfers provides real-time visibility into TCP flow control behavior. A steadily increasing window indicates healthy growth. A plateau means you've hit the buffer limit - consider increasing `tcp_rmem`. Zero Window events indicate receiver-side processing can't keep up with incoming data. This information directly guides whether the fix is buffer tuning, application optimization, or congestion control changes.
