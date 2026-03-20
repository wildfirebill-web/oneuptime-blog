# How to Forward Container Logs to Syslog via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Syslog, Logging, Log Forwarding, System Logs

Description: Configure Docker's syslog log driver to forward container logs to a syslog server, enabling integration with existing syslog infrastructure, SIEM systems, and rsyslog configurations.

## Introduction

Many organizations have existing syslog infrastructure — rsyslog, syslog-ng, or cloud SIEM systems — and need Docker container logs to flow into the same pipeline. Docker's built-in syslog log driver makes this straightforward: configure the driver per container or globally, and logs flow directly to your syslog receiver over UDP, TCP, or TLS. This guide covers configuring the syslog log driver for Portainer-managed containers.

## Step 1: Configure Syslog Log Driver Globally

```json
// /etc/docker/daemon.json - Send all container logs to syslog
{
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "udp://syslog.internal.com:514",
    "syslog-format": "rfc5424",
    "tag": "docker/{{.Name}}/{{.ID}}"
  }
}
```

```bash
# Apply changes
sudo systemctl restart docker

# Verify the driver is set
docker info | grep "Logging Driver"
# Returns: Logging Driver: syslog

# Test: run a container and verify syslog receives it
docker run --rm alpine echo "test syslog message"
# Check syslog server for this message
```

## Step 2: Per-Container Syslog Configuration

```yaml
# docker-compose.yml - Syslog logging per service
version: "3.8"

services:
  api:
    image: myapp/api:latest
    logging:
      driver: syslog
      options:
        # Syslog server address
        syslog-address: "udp://syslog.internal.com:514"
        # Or TCP for reliability
        # syslog-address: "tcp://syslog.internal.com:514"
        # Or TCP+TLS for security
        # syslog-address: "tcp+tls://syslog.internal.com:6514"
        # Syslog facility and severity
        syslog-facility: "daemon"
        # RFC5424 format (recommended, includes structured data)
        syslog-format: "rfc5424"
        # Tag identifies this container in syslog
        tag: "api/{{.Name}}"
        # Add hostname
        syslog-hostname: "docker-host-1"

  nginx:
    image: nginx:alpine
    logging:
      driver: syslog
      options:
        syslog-address: "udp://syslog.internal.com:514"
        syslog-format: "rfc5424"
        tag: "nginx/{{.Name}}"

  database:
    image: postgres:15-alpine
    # Keep database logs local (noisy, different retention needs)
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"
```

## Step 3: Deploy a Local Syslog Server

```yaml
# docker-compose.yml - Local rsyslog server
version: "3.8"

services:
  rsyslog:
    image: rsyslog/syslog_appliance_alpine:latest
    container_name: rsyslog
    restart: unless-stopped
    volumes:
      - ./rsyslog.conf:/etc/rsyslog.conf
      - rsyslog_logs:/var/log/rsyslog
    ports:
      - "514:514/udp"   # Standard syslog UDP
      - "514:514/tcp"   # TCP for reliability
      - "6514:6514/tcp" # TLS syslog

volumes:
  rsyslog_logs:
```

```bash
# rsyslog.conf - Configure rsyslog to receive Docker logs

# Load modules
module(load="imudp")
module(load="imtcp")

# Listen on UDP 514 and TCP 514
input(type="imudp" port="514")
input(type="imtcp" port="514")

# Template for Docker log files organized by container tag
template(name="DockerLogFile" type="string"
  string="/var/log/rsyslog/%SYSLOGTAG:F,47:1%/%$YEAR%-%$MONTH%-%$DAY%.log")

# Write Docker logs to organized file structure
if $syslogtag startswith "docker/" then {
  action(
    type="omfile"
    DynaFile="DockerLogFile"
    template="RSYSLOG_TraditionalFileFormat"
  )
  stop  # Don't process further
}

# Default: write all other messages to messages file
*.* /var/log/rsyslog/messages
```

## Step 4: Forward to Multiple Syslog Destinations

```yaml
# Send logs to both local syslog AND a SIEM
services:
  api:
    image: myapp/api:latest
    logging:
      driver: syslog
      options:
        # Primary: local rsyslog
        syslog-address: "tcp://rsyslog.internal.com:514"
        syslog-format: "rfc5424"
        tag: "myapp/api/{{.Name}}"
```

```bash
# rsyslog.conf - Forward to SIEM while keeping local copy

# Receive from Docker
module(load="imtcp")
input(type="imtcp" port="514")

# Local storage
*.* /var/log/docker/all.log

# Forward to SIEM (Splunk, QRadar, etc.)
*.* action(
  type="omfwd"
  target="splunk.corp.com"
  port="9514"
  protocol="tcp"
  queue.type="LinkedList"
  queue.size="10000"
  queue.saveonshutdown="on"
  action.resumeRetryCount="-1"
)
```

## Step 5: TLS-Secured Syslog

```bash
# Generate TLS certificates for secure syslog
openssl req -x509 -nodes -days 3650 \
  -keyout /etc/rsyslog-certs/key.pem \
  -out /etc/rsyslog-certs/cert.pem \
  -subj "/CN=syslog.internal.com"

# rsyslog.conf - TLS configuration
module(load="imtcp")
module(load="gtls" streamdriver.name="gtls")

input(
  type="imtcp"
  port="6514"
  streamDriver.name="gtls"
  streamDriver.mode="1"     # TLS mode
  streamDriver.authMode="x509/name"
)

global(
  DefaultNetstreamDriverCAFile="/etc/rsyslog-certs/ca.pem"
  DefaultNetstreamDriverCertFile="/etc/rsyslog-certs/cert.pem"
  DefaultNetstreamDriverKeyFile="/etc/rsyslog-certs/key.pem"
)
```

```yaml
# Container using TLS syslog
services:
  api:
    image: myapp/api:latest
    logging:
      driver: syslog
      options:
        syslog-address: "tcp+tls://syslog.internal.com:6514"
        syslog-tls-cert: /etc/docker/syslog-certs/client-cert.pem
        syslog-tls-key: /etc/docker/syslog-certs/client-key.pem
        syslog-tls-ca-cert: /etc/docker/syslog-certs/ca.pem
```

## Step 6: Verify Syslog Integration

```bash
# Test syslog reception
docker run --rm \
  --log-driver=syslog \
  --log-opt syslog-address=udp://syslog.internal.com:514 \
  --log-opt tag="test/container" \
  alpine \
  echo "Testing syslog from Docker container"

# Check rsyslog received it
tail -f /var/log/rsyslog/test/$(date +%Y-%m-%d).log
# Should show the test message

# Check with logger (manual syslog test)
docker exec rsyslog logger -t "docker/api" "Manual test message"
```

## Conclusion

Docker's syslog log driver integrates seamlessly with existing syslog infrastructure. Organizations running rsyslog, syslog-ng, or cloud SIEM systems can route container logs through the same pipelines used for host OS logs. RFC5424 format with structured data includes container metadata like the tag and hostname in a machine-parseable format. Use TCP instead of UDP for production to ensure delivery, and TLS for environments where log data is sensitive. Portainer's compose YAML makes it straightforward to configure different logging destinations per service, so noisy services like databases can stay local while application services ship to the central SIEM.
