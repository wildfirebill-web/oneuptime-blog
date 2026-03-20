# How to Forward Container Logs to Fluentd via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Fluentd, Log Forwarding, Docker, Log Aggregation, Centralized Logging

Description: Learn how to configure Docker containers managed by Portainer to forward logs to Fluentd using the Docker Fluentd log driver.

---

The Docker Fluentd log driver sends container logs directly to a Fluentd instance over TCP. Fluentd can then route logs to Elasticsearch, S3, Loki, syslog, or any other output. This makes it a flexible aggregation hub.

## Deploying Fluentd

First, deploy a Fluentd collector as a Portainer stack:

```yaml
version: "3.8"

services:
  fluentd:
    image: fluent/fluentd:v1.16-debian-1
    volumes:
      - ./fluent.conf:/fluentd/etc/fluent.conf:ro
      - fluentd_buffer:/fluentd/log/buffer
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    networks:
      - logging_net

volumes:
  fluentd_buffer:

networks:
  logging_net:
    driver: bridge
    name: logging_net
```

## Basic Fluentd Configuration

Create `fluent.conf` to receive Docker logs and write to files:

```xml
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

# Parse JSON logs from application containers
<filter docker.**>
  @type parser
  key_name log
  reserve_data true
  remove_key_name_field false
  <parse>
    @type json
    time_key time
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</filter>

# Write all logs to files, one per container tag
<match docker.**>
  @type file
  path /fluentd/log/docker/${tag}
  append true
  <format>
    @type json
  </format>
  <buffer tag, time>
    timekey 1d
    timekey_use_utc true
    flush_interval 30s
  </buffer>
</match>
```

## Routing Logs to Multiple Destinations

Route different services to different outputs:

```xml
<match docker.my-app.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  index_name docker-my-app
  logstash_format true
  <buffer>
    flush_interval 5s
  </buffer>
</match>

<match docker.my-other-app.**>
  @type s3
  aws_key_id YOUR_AWS_KEY
  aws_sec_key YOUR_AWS_SECRET
  s3_bucket my-log-bucket
  s3_region us-east-1
  path logs/my-other-app/
  <buffer time>
    @type file
    timekey 1h
    timekey_wait 10m
  </buffer>
</match>
```

## Configuring Application Containers to Use Fluentd

Set the Fluentd log driver in application stacks:

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: "localhost:24224"   # Fluentd on the same host
        tag: "docker.my-app.api"
        fluentd-async: "true"               # Non-blocking; won't stall container if Fluentd is down
        fluentd-retry-wait: "1s"
        fluentd-max-retries: "30"

  worker:
    image: my-worker:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: "localhost:24224"
        tag: "docker.my-app.worker"
        fluentd-async: "true"
```

## Setting Fluentd as Default Log Driver

To apply Fluentd logging to all containers on a host, set it in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "fluentd",
  "log-opts": {
    "fluentd-address": "localhost:24224",
    "tag": "docker.{{.Name}}",
    "fluentd-async": "true"
  }
}
```

Individual services can override this by specifying a different `logging` block.

## Tag Naming Conventions

Use hierarchical tags for flexible routing in Fluentd:

```
docker.{stack}.{service}    — e.g., docker.my-app.api
docker.{environment}.{service}  — e.g., docker.production.api
```

Fluentd `<match>` directives support wildcards:

```xml
<match docker.production.**>   # All production services
<match docker.**.api>          # API services in all environments
<match docker.my-app.*>        # All services in my-app stack
```

## Structured Log Forwarding

Fluentd preserves JSON-structured logs. Use structured logging in your application for richer filtering:

```python
import json, logging, sys

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "time": self.formatTime(record),
            "level": record.levelname,
            "msg": record.getMessage(),
            "service": "api",
            "request_id": getattr(record, 'request_id', None)
        })

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
```
