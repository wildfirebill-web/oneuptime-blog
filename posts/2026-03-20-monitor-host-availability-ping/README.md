# How to Continuously Monitor Host Availability with Ping

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ping, Monitoring, Linux, Bash, Networking, Availability

Description: Build continuous host availability monitoring using ping scripts that detect outages, measure uptime percentages, and send alerts when hosts go down.

Continuous ping monitoring detects outages in real time and builds a history of host availability. A simple bash script can replace expensive monitoring tools for small environments.

## Simple Continuous Monitor

```bash
#!/bin/bash
# monitor.sh — Alert when host goes down or comes up

HOST="${1:-8.8.8.8}"
INTERVAL=5
DOWN=false

echo "Monitoring $HOST (Ctrl+C to stop)..."

while true; do
    if ping -c 1 -W 2 "$HOST" > /dev/null 2>&1; then
        if $DOWN; then
            echo "$(date '+%Y-%m-%d %H:%M:%S') RECOVERED: $HOST is back UP"
            DOWN=false
        fi
    else
        if ! $DOWN; then
            echo "$(date '+%Y-%m-%d %H:%M:%S') ALERT: $HOST is DOWN"
            DOWN=true
        fi
    fi
    sleep "$INTERVAL"
done
```

## Monitor with Timestamps and Logging

```bash
#!/bin/bash
# monitor-log.sh — Log all state changes with timestamps

HOST="${1:-192.168.1.1}"
LOGFILE="/var/log/ping-monitor-${HOST}.log"
INTERVAL=10

echo "Logging availability of $HOST to $LOGFILE"

PREV_STATE=""

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

    if ping -c 2 -W 3 "$HOST" > /dev/null 2>&1; then
        STATE="UP"
    else
        STATE="DOWN"
    fi

    if [ "$STATE" != "$PREV_STATE" ]; then
        echo "$TIMESTAMP [$STATE] $HOST" | tee -a "$LOGFILE"
        PREV_STATE="$STATE"
    fi

    sleep "$INTERVAL"
done
```

## Calculate Uptime Percentage

```bash
#!/bin/bash
# uptime-check.sh — Measure uptime over N pings

HOST="${1:-8.8.8.8}"
COUNT=100  # Number of pings

echo "Testing availability of $HOST ($COUNT pings)..."

RESULT=$(ping -c "$COUNT" -i 0.5 "$HOST" 2>&1)
SENT=$(echo "$RESULT" | grep -oP '\d+(?= packets transmitted)')
RECV=$(echo "$RESULT" | grep -oP '\d+(?= received)')
LOSS=$(echo "$RESULT" | grep -oP '\d+(?=% packet loss)')

UPTIME=$((100 - LOSS))

echo "Packets sent: $SENT"
echo "Packets received: $RECV"
echo "Packet loss: ${LOSS}%"
echo "Availability: ${UPTIME}%"
```

## Monitor Multiple Hosts

```bash
#!/bin/bash
# multi-host-monitor.sh — Monitor multiple hosts in parallel

HOSTS=("192.168.1.1" "192.168.1.10" "8.8.8.8" "1.1.1.1")

monitor_host() {
    HOST="$1"
    while true; do
        if ping -c 1 -W 2 "$HOST" > /dev/null 2>&1; then
            STATUS="UP"
        else
            STATUS="DOWN"
        fi
        echo "$(date '+%H:%M:%S') $HOST: $STATUS"
        sleep 30
    done
}

# Start monitoring each host in background
for host in "${HOSTS[@]}"; do
    monitor_host "$host" &
done

# Wait for all background processes
wait
```

## Send Email Alert on Outage

```bash
#!/bin/bash
# alert-monitor.sh — Email alert when host goes down

HOST="192.168.1.1"
ALERT_EMAIL="ops@example.com"
DOWN=false

while true; do
    if ! ping -c 2 -W 3 "$HOST" > /dev/null 2>&1; then
        if ! $DOWN; then
            echo "Host $HOST is DOWN as of $(date)" \
              | mail -s "ALERT: $HOST Unreachable" "$ALERT_EMAIL"
            DOWN=true
        fi
    else
        if $DOWN; then
            echo "Host $HOST recovered at $(date)" \
              | mail -s "RESOLVED: $HOST Back Online" "$ALERT_EMAIL"
            DOWN=false
        fi
    fi
    sleep 60
done
```

## Run as a Systemd Service

```ini
# /etc/systemd/system/ping-monitor.service

[Unit]
Description=Ping Availability Monitor
After=network.target

[Service]
ExecStart=/opt/scripts/monitor-log.sh 192.168.1.1
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable ping-monitor
sudo systemctl start ping-monitor
sudo journalctl -u ping-monitor -f
```

Continuous ping monitoring is the simplest form of availability monitoring — reliable, low-resource, and easy to extend with alerting and logging as your needs grow.
