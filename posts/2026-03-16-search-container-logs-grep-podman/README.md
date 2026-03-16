# How to Search Container Logs with grep in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Logging, Debugging

Description: Learn how to effectively search Podman container logs using grep, including pattern matching, context lines, counting occurrences, and filtering live log streams.

---

> Searching logs with grep is the fastest way to find the needle in a haystack of container output.

When containers produce thousands of log lines, scrolling through them manually is impractical. Piping `podman logs` output to `grep` lets you instantly find errors, specific request IDs, or any pattern in your container logs.

---

## Basic Log Searching

```bash
# Search for a specific string in container logs
podman logs my-container 2>&1 | grep "error"

# Case-insensitive search
podman logs my-container 2>&1 | grep -i "error"

# Search in both stdout and stderr (2>&1 combines both streams)
podman logs my-container 2>&1 | grep -i "connection refused"

# Search for an exact phrase
podman logs my-container 2>&1 | grep -F "404 Not Found"
```

The `2>&1` redirect is important because `podman logs` sends stdout and stderr separately. Without it, grep only searches stdout.

## Use Regular Expressions

```bash
# Search for lines starting with ERROR
podman logs my-container 2>&1 | grep "^ERROR"

# Search for IP addresses
podman logs my-container 2>&1 | grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}'

# Search for HTTP status codes (4xx and 5xx)
podman logs my-container 2>&1 | grep -E 'HTTP/[0-9.]+" [45][0-9]{2}'

# Search for timestamps matching a specific hour
podman logs my-container 2>&1 | grep -E '2026-03-16T14:'

# Search for multiple patterns (OR)
podman logs my-container 2>&1 | grep -iE '(error|warn|critical|fatal)'
```

## Show Context Around Matches

```bash
# Show 3 lines before each match
podman logs my-container 2>&1 | grep -B 3 -i "error"

# Show 5 lines after each match
podman logs my-container 2>&1 | grep -A 5 -i "exception"

# Show 2 lines before and after (context)
podman logs my-container 2>&1 | grep -C 2 -i "timeout"

# Show line numbers with matches
podman logs my-container 2>&1 | grep -n -i "error"
```

## Count and Summarize

```bash
# Count total matches
podman logs my-container 2>&1 | grep -c -i "error"

# Count unique error messages
podman logs my-container 2>&1 | grep -i "error" | sort -u | wc -l

# Show only unique matching lines
podman logs my-container 2>&1 | grep -i "error" | sort -u

# Count occurrences of each error type
podman logs my-container 2>&1 | grep -i "error" | sort | uniq -c | sort -rn | head -20

# Count errors vs warnings
echo "Errors: $(podman logs my-container 2>&1 | grep -ci error)"
echo "Warnings: $(podman logs my-container 2>&1 | grep -ci warn)"
```

## Search with Time Constraints

```bash
# Search recent logs only (last 30 minutes)
podman logs --since 30m my-container 2>&1 | grep -i "error"

# Search a specific time window
podman logs --since "2026-03-16T14:00:00" --until "2026-03-16T15:00:00" my-container 2>&1 | grep -i "error"

# Search with timestamps for time context
podman logs --timestamps my-container 2>&1 | grep -i "error"
```

## Exclude Patterns (Inverse Grep)

```bash
# Show all lines EXCEPT those matching a pattern
podman logs my-container 2>&1 | grep -v "health check"

# Exclude multiple patterns
podman logs my-container 2>&1 | grep -v -E "(health check|ping|OPTIONS)"

# Show errors but exclude known noise
podman logs my-container 2>&1 | grep -i "error" | grep -v "expected error"

# Filter out debug and info lines, keep only warnings and errors
podman logs my-container 2>&1 | grep -v -E "^(DEBUG|INFO)"
```

## Search Live Logs

```bash
# Search in real-time log stream
podman logs -f my-container 2>&1 | grep --line-buffered -i "error"

# Highlight matches in live stream
podman logs -f my-container 2>&1 | grep --line-buffered --color=always -i "error"

# Multiple patterns in live stream
podman logs -f my-container 2>&1 | grep --line-buffered -iE "(error|warn|critical)"

# Live search with context
podman logs -f my-container 2>&1 | grep --line-buffered -A 2 -i "exception"
```

Always use `--line-buffered` with `grep` when piping from `podman logs -f`, otherwise output will be delayed by buffering.

## Search Across Multiple Containers

```bash
# Search all containers for a specific pattern
for c in $(podman ps --format '{{.Names}}'); do
  MATCHES=$(podman logs "$c" 2>&1 | grep -c -i "error")
  if [ "$MATCHES" -gt 0 ]; then
    echo "=== $c: $MATCHES errors ==="
    podman logs "$c" 2>&1 | grep -i "error" | head -5
  fi
done

# Search for a request ID across all services
REQUEST_ID="abc-123-def"
for c in $(podman ps --format '{{.Names}}'); do
  podman logs "$c" 2>&1 | grep "$REQUEST_ID" | sed "s/^/[$c] /"
done
```

## Advanced Grep Techniques

```bash
# Extract specific fields from structured logs
podman logs my-container 2>&1 | grep -oP '"status":\s*\K[0-9]+'

# Find lines with response times over 1000ms
podman logs my-container 2>&1 | grep -E 'response_time[=:]\s*[0-9]{4,}'

# Show first and last occurrence of a pattern
podman logs my-container 2>&1 | grep -i "error" | head -1
podman logs my-container 2>&1 | grep -i "error" | tail -1

# Create an error summary report
echo "=== Error Summary ==="
echo "Total errors: $(podman logs my-container 2>&1 | grep -ci error)"
echo "Total warnings: $(podman logs my-container 2>&1 | grep -ci warn)"
echo ""
echo "=== Top Errors ==="
podman logs my-container 2>&1 | grep -i error | sort | uniq -c | sort -rn | head -10
```

## Summary

Piping `podman logs` through `grep` is the most efficient way to search container logs. Always redirect stderr with `2>&1` to search the complete output. Use `-i` for case-insensitive searches, `-C` for context lines, `-c` for counting, and `-v` for exclusions. When searching live streams with `podman logs -f`, always add `--line-buffered` to grep to avoid output delays.
