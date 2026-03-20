# How to Measure IPv6 Latency and Jitter

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Latency, Jitter, Performance, ping6, hping3

Description: Measure IPv6 latency and jitter using ping6, hping3, and custom Python scripts to baseline network performance and detect degradation.

## Introduction

Latency and jitter are fundamental metrics for any network. For IPv6, measuring them requires tools that explicitly use IPv6 transport. Baseline measurements before production deployment allow anomaly detection later.

## Measuring Latency with ping6

```bash
# Basic RTT measurement
ping6 -c 100 2001:4860:4860::8888

# Statistics output:
# rtt min/avg/max/mdev = 10.234/11.456/15.678/1.234 ms
# mdev = mean deviation (approximation of jitter)

# Flood ping (root required) for stress testing
ping6 -f -c 10000 2001:db8::target
```

## Measuring Jitter with hping3

```bash
# hping3 for precise jitter measurement
# --ipv6: use IPv6, --icmp: ICMP mode, --flood: send without waiting
hping3 --ipv6 --icmp -c 1000 --interval u10000 2001:db8::target
# --interval u10000 = 10ms between packets

# Output includes RTT per packet; pipe to awk for jitter calculation
hping3 --ipv6 --icmp -c 100 2001:db8::target 2>&1 | \
  awk '/rtt=/ {
    split($0, a, "rtt="); split(a[2], b, " ");
    rtt=b[1]+0;
    if (prev > 0) jitter += (rtt - prev < 0 ? prev - rtt : rtt - prev);
    prev = rtt; count++
  }
  END { printf "Avg jitter: %.3f ms\n", jitter/count }'
```

## Python Latency and Jitter Tool

```python
import subprocess
import re
import statistics

def measure_ipv6_latency(target: str, count: int = 50) -> dict:
    """
    Measure IPv6 latency and jitter to a target.
    Returns min, avg, max, stddev, and jitter.
    """
    result = subprocess.run(
        ["ping6", "-c", str(count), target],
        capture_output=True, text=True
    )

    # Extract individual RTT values
    rtts = []
    for line in result.stdout.splitlines():
        m = re.search(r'time=([\d.]+)', line)
        if m:
            rtts.append(float(m.group(1)))

    if not rtts:
        return {"error": "No RTT data collected"}

    # Jitter = mean absolute difference between consecutive samples
    jitter_values = [abs(rtts[i] - rtts[i-1]) for i in range(1, len(rtts))]

    return {
        "target": target,
        "count": len(rtts),
        "min_ms": min(rtts),
        "avg_ms": statistics.mean(rtts),
        "max_ms": max(rtts),
        "stddev_ms": statistics.stdev(rtts) if len(rtts) > 1 else 0,
        "jitter_ms": statistics.mean(jitter_values) if jitter_values else 0,
        "packet_loss": (count - len(rtts)) / count * 100,
    }

if __name__ == "__main__":
    targets = [
        "2001:4860:4860::8888",  # Google
        "2606:4700:4700::1111",  # Cloudflare
    ]
    for t in targets:
        metrics = measure_ipv6_latency(t, count=20)
        print(f"\n{metrics['target']}:")
        for k, v in metrics.items():
            if k != "target":
                print(f"  {k}: {v:.3f}" if isinstance(v, float) else f"  {k}: {v}")
```

## Continuous Monitoring with fping6

```bash
# fping6 for multi-target continuous measurement
fping6 -l -p 1000 -q 2001:4860:4860::8888 2606:4700:4700::1111 2>&1 | \
  ts '%Y-%m-%d %H:%M:%S' >> /var/log/ipv6_latency.log
# -l: loop, -p: period in ms, -q: quiet (stats only)

# Parse fping6 output for Prometheus
while read -r line; do
  target=$(echo "$line" | awk '{print $1}')
  rtt=$(echo "$line" | grep -oP 'avg=\K[\d.]+')
  echo "ipv6_latency_ms{target=\"$target\"} $rtt"
done < /var/log/ipv6_latency.log
```

## Conclusion

Use `ping6` for quick RTT snapshots, `hping3` for controlled jitter measurement, and custom Python scripts for structured baseline collection. Feed results into OneUptime to trigger alerts when latency or jitter exceeds thresholds.
