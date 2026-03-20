# How to Set Up NeuVector Syslog Integration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Syslog, SIEM, Log Management, Container Security

Description: Configure NeuVector to forward security events to a syslog server or SIEM for centralized log management and security monitoring.

## Introduction

Integrating NeuVector with your syslog infrastructure enables centralized log collection and SIEM (Security Information and Event Management) analysis. NeuVector can forward all security events, audit logs, and compliance findings to a syslog destination in CEF (Common Event Format) or standard syslog format. This guide covers configuration for various syslog targets including Splunk, Elastic Stack, and standard syslog servers.

## Prerequisites

- NeuVector installed and running
- A syslog server or SIEM with a listening endpoint
- Network connectivity from NeuVector controller to syslog server
- NeuVector Manager access

## Step 1: Configure Syslog Output

```bash
# Configure syslog forwarding
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "syslog_ip": "syslog.company.com",
      "syslog_ip_proto": "udp",
      "syslog_port": 514,
      "syslog_level": "Warning",
      "syslog_status": true,
      "syslog_categories": ["event", "security-event", "audit"],
      "syslog_in_json": false,
      "single_cve_per_syslog": false
    }
  }'
```

### Use TCP for Reliable Delivery

```bash
# Configure syslog with TCP (recommended for production)
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "syslog_ip": "syslog.company.com",
      "syslog_ip_proto": "tcp",
      "syslog_port": 1514,
      "syslog_level": "Info",
      "syslog_status": true,
      "syslog_in_json": true
    }
  }'
```

### Use TLS for Encrypted Syslog

```bash
# Configure TLS syslog (syslog over TLS, port 6514)
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "syslog_ip": "syslog.company.com",
      "syslog_ip_proto": "tcp",
      "syslog_port": 6514,
      "syslog_level": "Info",
      "syslog_status": true,
      "syslog_tls_verify": true
    }
  }'
```

## Step 2: Configure Syslog in the UI

1. Navigate to **Settings** > **Configuration**
2. Scroll to **Syslog** section
3. Configure:
   - **Enable**: Toggle on
   - **Syslog Server**: IP or hostname
   - **Port**: 514 (UDP) or 1514 (TCP)
   - **Protocol**: UDP or TCP
   - **Level**: Info, Warning, Error, Critical
   - **Format**: JSON or Text
   - **Categories**: Select which event types to forward

## Step 3: Configure Log Categories

Control which events are forwarded to syslog:

```bash
# Forward all security events and audit logs
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "syslog_categories": [
        "event",
        "security-event",
        "audit",
        "incident",
        "violation"
      ]
    }
  }'
```

Available categories:
- `event` — System and operational events
- `security-event` — Security violations and threats
- `audit` — User actions and configuration changes
- `incident` — Correlated security incidents
- `violation` — Policy violation details

## Step 4: Configure for Splunk

Configure a Splunk HTTP Event Collector (HEC) receiver to accept NeuVector logs:

```bash
# In Splunk: Create an HEC token
# Settings > Data Inputs > HTTP Event Collector > New Token
# Note: Enable a standard syslog listener if using native syslog

# Configure NeuVector to send to Splunk Universal Forwarder syslog port
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "syslog_ip": "splunk-indexer.company.com",
      "syslog_port": 1514,
      "syslog_ip_proto": "tcp",
      "syslog_level": "Info",
      "syslog_status": true,
      "syslog_in_json": true
    }
  }'
```

Splunk search for NeuVector events:
```
index=security sourcetype=neuvector
| where level="Critical" OR level="High"
| stats count by source_container, event_type, action
```

## Step 5: Configure for Elastic Stack (ELK)

Set up Logstash to receive NeuVector syslog and index to Elasticsearch:

```ruby
# logstash-neuvector.conf
input {
  syslog {
    port => 1514
    type => "neuvector"
  }
}

filter {
  if [type] == "neuvector" {
    # Parse JSON NeuVector events
    json {
      source => "message"
      target => "neuvector"
    }

    # Add enrichment fields
    mutate {
      add_field => {
        "[@metadata][index]" => "neuvector-security"
      }
    }
  }
}

output {
  if [type] == "neuvector" {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "%{[@metadata][index]}-%{+YYYY.MM.dd}"
    }
  }
}
```

## Step 6: Verify Syslog Delivery

Test that events are reaching your syslog server:

```bash
# Generate a test event in NeuVector
# (Trigger a test scan or security event)

# Check syslog server for received messages
tail -f /var/log/syslog | grep -i neuvector

# Or use tcpdump to verify packets are sent
tcpdump -i any -n "host syslog.company.com and port 514"
```

## Step 7: Configure Log Levels

Filter log verbosity to avoid overwhelming your SIEM:

| Level | Events Included |
|---|---|
| Critical | Only critical security events |
| Error | Critical + errors |
| Warning | Critical + errors + warnings (recommended) |
| Notice | All except debug |
| Info | All events |

```bash
# Set to Warning level for production
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/system/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{"config": {"syslog_level": "Warning"}}'
```

## Conclusion

Syslog integration connects NeuVector's container security events to your existing SIEM and log management infrastructure. This enables correlation of container security events with other infrastructure logs, automated alerting through your SIEM rules, and long-term retention for compliance auditing. For production environments, use TCP with TLS to ensure reliable, encrypted event delivery.
