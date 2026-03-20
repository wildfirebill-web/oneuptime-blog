# How to Forward Podman Container Logs to Fluentd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Logging, Fluentd

Description: Learn how to forward Podman container logs to Fluentd for centralized log aggregation, using file tailing, journald integration, and Fluent Bit as a lightweight collector.

---

> Fluentd transforms Podman container logs into structured data that can be routed to any destination, from Elasticsearch to S3.

Fluentd is a popular open-source log aggregator that can collect, transform, and forward logs to numerous destinations. Unlike Docker, Podman does not have a native Fluentd log driver, but there are several effective approaches to get container logs into Fluentd.

---

## Approach 1: Fluent Bit Sidecar Tailing Log Files

Run Fluent Bit alongside Podman to tail container log files.

```bash
# Step 1: Create a Fluent Bit configuration

mkdir -p /etc/fluent-bit

cat > /etc/fluent-bit/fluent-bit.conf << 'EOF'
[SERVICE]
    Flush        5
    Log_Level    info

[INPUT]
    Name         tail
    Path         /var/lib/containers/storage/overlay-containers/*/userdata/ctr.log
    Tag          podman.*
    Parser       k8s-file
    Refresh_Interval 5

[PARSER]
    Name         k8s-file
    Format       regex
    Regex        ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
    Time_Key     time
    Time_Format  %Y-%m-%dT%H:%M:%S.%L%z

[OUTPUT]
    Name         forward
    Match        podman.*
    Host         fluentd-server.example.com
    Port         24224
EOF

# Step 2: Run Fluent Bit as a container
podman run -d \
  --name fluent-bit \
  -v /var/lib/containers/storage:/var/lib/containers/storage:ro \
  -v /etc/fluent-bit:/fluent-bit/etc:ro \
  fluent/fluent-bit:latest
```

## Approach 2: journald Input with Fluentd

Use the journald log driver and have Fluentd read from the systemd journal.

```bash
# Step 1: Run containers with journald driver
podman run -d \
  --log-driver journald \
  --log-opt tag="myapp" \
  --name web \
  nginx:latest

# Step 2: Create a Fluentd configuration for journald input
cat > /etc/fluentd/fluentd.conf << 'EOF'
<source>
  @type systemd
  tag podman
  read_from_head false

  <storage>
    @type local
    path /var/log/fluentd/journald-cursor.json
  </storage>

  <entry>
    fields_strip_underscores true
    fields_lowercase true
  </entry>
</source>

<filter podman.**>
  @type grep
  <regexp>
    key container_name
    pattern /.+/
  </regexp>
</filter>

<match podman.**>
  @type forward
  <server>
    host fluentd-aggregator.example.com
    port 24224
  </server>

  <buffer>
    @type file
    path /var/log/fluentd/buffer
    flush_interval 10s
  </buffer>
</match>
EOF

# Step 3: Run Fluentd with the journald plugin
podman run -d \
  --name fluentd \
  -v /etc/fluentd:/fluentd/etc:ro \
  -v /var/log/journal:/var/log/journal:ro \
  -v /run/log/journal:/run/log/journal:ro \
  -v /var/log/fluentd:/var/log/fluentd \
  fluent/fluentd:latest
```

## Approach 3: Pipe Logs to Fluentd via TCP

Forward logs directly to Fluentd's TCP input.

```bash
# Step 1: Configure Fluentd with a TCP input
cat > /etc/fluentd/tcp-input.conf << 'EOF'
<source>
  @type tcp
  tag podman.logs
  port 5170
  <parse>
    @type json
  </parse>
</source>

<match podman.**>
  @type elasticsearch
  host elasticsearch.example.com
  port 9200
  index_name podman-logs
  <buffer>
    @type file
    path /var/log/fluentd/buffer
    flush_interval 10s
  </buffer>
</match>
EOF

# Step 2: Forward container logs as JSON to Fluentd's TCP port
podman logs -f --timestamps my-container 2>&1 | while IFS= read -r line; do
  TS=$(echo "$line" | awk '{print $1}')
  MSG=$(echo "$line" | cut -d' ' -f2- | sed 's/"/\\"/g')
  echo "{\"container\":\"my-container\",\"timestamp\":\"$TS\",\"message\":\"$MSG\"}" | \
    nc -q0 localhost 5170
done &
```

## Approach 4: Fluent Bit Sidecar in a Pod

Run Fluent Bit inside the same pod as your application.

```bash
# Step 1: Create a pod
podman pod create --name app-pod -p 8080:80

# Step 2: Create Fluent Bit config for shared volume
mkdir -p ./fluent-bit-config
cat > ./fluent-bit-config/fluent-bit.conf << 'EOF'
[SERVICE]
    Flush        5
    Log_Level    info

[INPUT]
    Name         tail
    Path         /app/logs/*.log
    Tag          app
    Refresh_Interval 5

[OUTPUT]
    Name         forward
    Match        *
    Host         fluentd-server.example.com
    Port         24224

[OUTPUT]
    Name         stdout
    Match        *
EOF

# Step 3: Run the app container (writes logs to shared volume)
podman run -d \
  --pod app-pod \
  --name app \
  -v app-logs:/app/logs \
  my-app:latest

# Step 4: Run Fluent Bit in the same pod
podman run -d \
  --pod app-pod \
  --name fluent-bit \
  -v app-logs:/app/logs:ro \
  -v ./fluent-bit-config:/fluent-bit/etc:ro \
  fluent/fluent-bit:latest
```

## Configure Log Enrichment

Add metadata to logs before forwarding.

```bash
# Fluent Bit config with record modification
cat > /etc/fluent-bit/enrich.conf << 'EOF'
[SERVICE]
    Flush        5

[INPUT]
    Name         tail
    Path         /var/lib/containers/storage/overlay-containers/*/userdata/ctr.log
    Tag          podman.*

[FILTER]
    Name         record_modifier
    Match        *
    Record       hostname ${HOSTNAME}
    Record       environment production
    Record       source podman

[FILTER]
    Name         modify
    Match        *
    Add          cluster my-cluster
    Add          region us-east-1

[OUTPUT]
    Name         forward
    Match        *
    Host         fluentd-server.example.com
    Port         24224
EOF
```

## Verify Log Forwarding

```bash
# Test with a log-generating container
podman run -d \
  --log-driver journald \
  --log-opt tag="fluentd-test" \
  --name fluentd-test \
  alpine sh -c 'while true; do echo "{\"level\":\"info\",\"msg\":\"test at $(date)\"}"; sleep 5; done'

# Check Fluent Bit output (if using stdout output)
podman logs fluent-bit | tail -20

# Check if Fluentd received the logs
podman exec fluentd cat /var/log/fluentd/buffer/* 2>/dev/null | head -5

# Clean up test container
podman rm -f fluentd-test
```

## Summary

Forwarding Podman container logs to Fluentd can be done through file tailing with Fluent Bit, journald integration with the systemd input plugin, direct TCP forwarding, or sidecar containers sharing log volumes. Fluent Bit is the lightest approach for collection, while Fluentd provides rich routing and transformation capabilities. Use the journald approach for the simplest integration, or the file-tailing approach for the most reliable log capture.
