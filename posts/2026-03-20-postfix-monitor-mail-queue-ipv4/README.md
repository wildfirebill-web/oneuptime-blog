# How to Monitor Postfix Mail Queue from IPv4 Sources

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, Mail Queue, Monitoring, IPv4, Linux, Email, Operation

Description: Learn how to monitor the Postfix mail queue to identify stuck messages, per-source statistics, and delivery failures from IPv4 senders.

---

The Postfix mail queue holds messages awaiting delivery. Monitoring queue depth, identifying stuck messages from specific IPv4 sources, and understanding delivery failures is essential for keeping a mail server healthy.

## Viewing the Queue

```bash
# Show a brief summary of the queue

mailq
# or equivalently:
postqueue -p

# Count queued messages
mailq | grep -c "^[A-F0-9]"

# Count messages currently deferred (retry later)
postqueue -p | grep "^[A-F0-9]" | wc -l
```

## Filtering the Queue by Source IPv4

```bash
# Show queue entries from a specific sender IPv4 address
postqueue -p | grep -B2 "192.168.1.50"

# Show all entries containing a specific sender domain
postqueue -p | grep "sender@domain.com"

# List sender IPs in the queue (from mail log, more reliable)
grep "client=unknown\[" /var/log/mail.log | grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn
```

## Queue Statistics

```bash
# Summary of queue status and mail statistics
postfix -p   # Process status

# Get detailed mail transport statistics
pflogsumm /var/log/mail.log | head -50

# Install pflogsumm if not available
apt install pflogsumm -y
```

## Inspecting Individual Messages

```bash
# View the headers of a queued message by queue ID
postcat -q QUEUE_ID

# Example: view the message with ID 3F8D21234
postcat -q 3F8D21234 | head -30
```

## Flushing or Deleting Stuck Messages

```bash
# Attempt immediate delivery of all queued messages
postqueue -f
# or
postfix flush

# Delete all queued messages (use with caution!)
postsuper -d ALL

# Delete messages in the deferred queue only
postsuper -d ALL deferred

# Delete a specific message by queue ID
postsuper -d 3F8D21234

# Requeue all deferred messages for immediate retry
postsuper -r ALL deferred
```

## Monitoring Queue Size Over Time

```bash
#!/bin/bash
# queue_monitor.sh - Log queue size every minute
while true; do
    QUEUE=$(mailq | grep -c "^[A-F0-9]" 2>/dev/null || echo 0)
    echo "$(date '+%Y-%m-%d %H:%M:%S') queue_size=$QUEUE" >> /var/log/postfix_queue.log
    sleep 60
done
```

## Alerting with OneUptime

For production environments, expose the queue size as a metric and configure an alert.

```bash
# Script to push queue size to a monitoring system
QUEUE_SIZE=$(mailq | grep -c "^[A-F0-9]" 2>/dev/null || echo 0)
curl -s "https://your-oneuptime-instance.com/api/monitor/MONITOR_ID/log" \
  -H "Content-Type: application/json" \
  -d "{\"queue_size\": $QUEUE_SIZE, \"source_ip\": \"$(hostname -I | awk '{print $1}')\"}"
```

## Key Takeaways

- `postqueue -p` (or `mailq`) shows the full queue; filter by IP or sender with `grep`.
- `postcat -q QUEUE_ID` inspects individual message headers and content.
- `postsuper -d ALL deferred` clears stuck deferred messages without touching the active queue.
- Parse `/var/log/mail.log` with `pflogsumm` for daily summaries of delivery statistics per source IPv4.
