# How to Stream Portainer Logs to Syslog

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Logging, Syslog, Docker, Log Management, SIEM, Monitoring

Description: Learn how to stream Portainer application logs to a syslog server using Docker's log driver for centralized log management and SIEM integration.

---

Streaming Portainer logs to a syslog server centralizes them with your other infrastructure logs, enables real-time alerting on authentication failures, and satisfies compliance requirements for audit logging.

## Using Docker's syslog Log Driver

The simplest approach is using Docker's built-in syslog log driver when running the Portainer container:

```bash
docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  # Configure Docker to send container logs to syslog
  --log-driver syslog \
  --log-opt syslog-address=udp://syslog-server.example.com:514 \
  --log-opt syslog-facility=daemon \
  --log-opt syslog-severity=info \
  --log-opt tag="portainer" \
  portainer/portainer-ce:latest \
  --log-mode NOCOLOR         # Plain text is easier to parse in syslog
```

## Using TCP for Reliable Delivery

UDP syslog can lose messages under load. Use TCP for reliable delivery:

```bash
docker run -d \
  --name portainer \
  --log-driver syslog \
  --log-opt syslog-address=tcp://syslog-server.example.com:514 \
  --log-opt syslog-format=rfc5424 \       # Modern syslog format
  --log-opt tag="portainer/{{.Name}}" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --log-level INFO \
  --log-mode NOCOLOR
```

## JSON Logs to Fluentd

For log aggregation pipelines (ELK, Loki), use JSON format with the Fluentd log driver:

```bash
docker run -d \
  --name portainer \
  --log-driver fluentd \
  --log-opt fluentd-address=tcp://fluentd.example.com:24224 \
  --log-opt tag="portainer" \
  --log-opt fluentd-async=true \       # Non-blocking - don't let log failures stop Portainer
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --log-level INFO \
  --log-mode JSON                      # JSON for structured log parsing
```

## Docker Compose with Logging

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    command:
      - --log-level=INFO
      - --log-mode=JSON
    logging:
      driver: syslog
      options:
        syslog-address: udp://syslog-server.example.com:514
        tag: "portainer"
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

## Sample Syslog Queries

After logs are in your SIEM, useful queries:

```bash
# Failed login attempts
tag:portainer AND "authentication failed"

# Stack deployments
tag:portainer AND "stack" AND ("deployed" OR "updated")

# Agent connection events
tag:portainer AND "agent" AND ("connected" OR "disconnected")
```
