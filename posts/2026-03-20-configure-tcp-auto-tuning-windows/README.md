# How to Configure TCP Auto-Tuning on Windows Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Windows Server, Auto-Tuning, Performance, Netsh, PowerShell

Description: Learn how to configure and verify TCP receive window auto-tuning on Windows Server to maximize throughput for high-latency and high-bandwidth network connections.

## What Is TCP Auto-Tuning on Windows?

Windows implements TCP receive window auto-tuning (RFC 1323) which dynamically adjusts the TCP receive window size based on network conditions. This feature was introduced in Vista/Server 2008 and is enabled by default.

The receive window scaling level controls how aggressively Windows expands the TCP window:

| Level | Description | Max Window |
|---|---|---|
| disabled | 64 KB fixed window | 64 KB |
| highlyrestricted | Very conservative scaling | ~256 KB |
| restricted | Conservative scaling | ~1 MB |
| normal | Standard auto-tuning (default) | ~16 MB |
| experimental | Maximum scaling | ~1 GB |

## Step 1: Check Current Auto-Tuning Level

```cmd
REM Check from Command Prompt (run as Administrator)
netsh interface tcp show global

REM Or from PowerShell
Get-NetTCPSetting | Select-Object SettingName, AutoTuningLevelLocal, CongestionProvider
```

## Step 2: Enable or Adjust Auto-Tuning

```powershell
# PowerShell (run as Administrator)

# Check current level

Get-NetTCPSetting -SettingName InternetCustom

# Set to Normal (recommended for most scenarios)
Set-NetTCPSetting -SettingName InternetCustom -AutoTuningLevelLocal Normal

# Set to Experimental for high-bandwidth, high-latency links
Set-NetTCPSetting -SettingName InternetCustom -AutoTuningLevelLocal Experimental

# Or with netsh (legacy method)
netsh interface tcp set global autotuninglevel=normal
```

## Step 3: Enable BBR-Equivalent on Windows (CUBIC)

Windows uses CUBIC or CTCP (Compound TCP) congestion control. Optimize it:

```powershell
# View available congestion providers
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider

# Set CUBIC (best for most environments)
Set-NetTCPSetting -SettingName InternetCustom -CongestionProvider CUBIC

# Enable ECN (Explicit Congestion Notification)
Set-NetTCPSetting -SettingName InternetCustom -EcnCapability Enabled

# Set initial RTO
Set-NetTCPSetting -SettingName InternetCustom -InitialCongestionWindow 10
```

## Step 4: Configure Chimney Offload and RSS

```powershell
# Enable Receive Side Scaling (RSS)
netsh interface tcp set global rss=enabled

# Check RSS state
netsh interface tcp show global | findstr "RSS"

# Enable TCP Chimney Offload (if NIC supports it)
netsh interface tcp set global chimney=enabled

# Enable NetDMA for reduced CPU usage
netsh interface tcp set global netdma=enabled
```

## Step 5: Tune with Group Policy (Enterprise)

For domain-joined Windows servers, use Group Policy:

```text
Computer Configuration
  → Administrative Templates
    → Network
      → QoS Packet Scheduler
        → Set Default DSCP Marking
      → TCP/IP Settings
        → Parameters
          → EnableTCPChimney = 1
          → EnableRSS = 1
          → EnableTCPA = 1
```

## Step 6: Test and Verify Throughput

```powershell
# Install iperf3 on Windows
# Download from: https://iperf.fr/iperf-download.php

# Test throughput
.\iperf3.exe -c server-ip -t 30 -P 4

# Before experimental:
# [SUM] 0.00-30.00 sec  3.36 GBytes   962 Mbits/sec   sender

# After experimental auto-tuning:
# [SUM] 0.00-30.00 sec  3.58 GBytes  1.02 Gbits/sec   sender
```

## Step 7: Troubleshoot When Auto-Tuning Causes Issues

Some applications or firewalls break with large receive windows. Restrict auto-tuning:

```cmd
REM Restrict if Windows Update or browsing is slow
netsh interface tcp set global autotuninglevel=restricted

REM Or disable completely (not recommended)
netsh interface tcp set global autotuninglevel=disabled

REM Re-enable when issue is resolved
netsh interface tcp set global autotuninglevel=normal
```

## Conclusion

Windows TCP auto-tuning should remain at `normal` or `experimental` for high-performance servers. Use `Get-NetTCPSetting` to inspect and `Set-NetTCPSetting` to configure. Enable RSS for multi-queue NIC support. For high-latency WAN connections (datacenter-to-datacenter), `experimental` mode allows the largest receive windows and typically improves throughput by 20-50% versus the default `normal` setting.
