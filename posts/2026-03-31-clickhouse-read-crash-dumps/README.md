# How to Read ClickHouse Crash Dumps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Crash Dump, Core Dump, Debugging, Stack Trace, Signal

Description: Learn how to locate, enable, and read ClickHouse crash dumps and core files to diagnose server crashes and segmentation faults.

---

## When ClickHouse Crashes

ClickHouse can crash due to:
- Segmentation faults in native code
- Memory corruption from codec bugs
- OOM killer termination
- Signals (SIGSEGV, SIGABRT, SIGBUS)

When a crash occurs, ClickHouse writes a crash log to its log file and optionally a core dump to disk.

## Finding Crash Information in Logs

ClickHouse logs crashes in its server log (`/var/log/clickhouse-server/clickhouse-server.log`):

```bash
grep -A 50 "Received signal\|Terminate\|std::terminate\|Signal" /var/log/clickhouse-server/clickhouse-server.log | tail -100
```

Look for lines like:

```text
Signal 11. Received 1 times. More likely there is a bug in ClickHouse.
Stack trace (use flamegraph.pl or addr2line):
0x0000000005c3a1f0
0x0000000005c3a3f0
...
```

## Enabling Core Dumps

Enable core dump generation on Linux:

```bash
# Set unlimited core file size
ulimit -c unlimited

# Configure core dump location
echo '/tmp/cores/core.%e.%p.%t' > /proc/sys/kernel/core_pattern
mkdir -p /tmp/cores
```

For systemd-managed ClickHouse, configure in the service unit:

```ini
[Service]
LimitCORE=infinity
```

## Reading a Core Dump with GDB

Install debug symbols:

```bash
apt-get install clickhouse-common-static-dbg
```

Analyze the core dump:

```bash
gdb /usr/bin/clickhouse-server /tmp/cores/core.clickhouse-se.12345.1700000000
```

Inside GDB:

```text
(gdb) bt
(gdb) thread apply all bt
(gdb) info threads
```

## Decoding Stack Trace Addresses

ClickHouse logs raw addresses. Decode them with `addr2line`:

```bash
addr2line -e /usr/bin/clickhouse-server -f -C 0x0000000005c3a1f0
```

Or use the `clickhouse-symbolizer` tool bundled with ClickHouse:

```bash
cat crash_trace.txt | clickhouse-symbolizer
```

## Reading Sanitizer Reports

If ClickHouse was built with AddressSanitizer or UBSan, crash output is more descriptive:

```text
=================================================================
==12345==ERROR: AddressSanitizer: use-after-free on address 0x...
READ of size 8 at 0x... thread T0
    #0 0x5f2a3b in DB::SomeClass::someMethod ...
```

## Checking for OOM Kills

If ClickHouse disappears without a crash log, check for OOM kills:

```bash
dmesg | grep -i "oom\|killed process\|clickhouse" | tail -20
```

Or:

```bash
journalctl -k | grep -i "oom\|clickhouse" | tail -20
```

## Summary

Read ClickHouse crash dumps by first checking the server log for signal information and stack traces, then enabling core dumps via ulimit or systemd configuration. Use GDB with debug symbols to inspect core files, `addr2line` or `clickhouse-symbolizer` to decode addresses, and `dmesg` to detect OOM kills that leave no crash log.
