# How to Use Fluentd to Route Logs by IPv4 Source Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fluentd, IPv4, Log Routing, Log Aggregation, Observability

Description: Configure Fluentd to parse IPv4 addresses from log fields and route log records to different destinations based on source IP or subnet, enabling tenant-based or environment-based log separation.

## Introduction

Fluentd's `grep` and `record_transformer` filters allow routing decisions based on parsed IPv4 addresses. Combined with `copy` and conditional outputs, you can send logs from different subnets to different Elasticsearch indices, S3 buckets, or monitoring systems.

## Parse Nginx Logs and Extract IP

```xml
<!-- /etc/fluent/fluent.conf -->

<source>
  @type tail
  path /var/log/nginx/access.log
  pos_file /var/log/fluent/nginx-access.pos
  tag nginx.access
  <parse>
    @type nginx
  </parse>
</source>
```

The built-in `nginx` parser extracts `remote` (client IP) field automatically.

## Route Based on IPv4 Subnet with grep Filter

```xml
<!-- Route internal (10.x.x.x) to internal store -->
<match nginx.access>
  @type route
  <route 10.**>
    copy
    add_tag_prefix internal.
  </route>
  <route **>
    copy
    add_tag_prefix external.
  </match>
</match>
```

Note: `route` plugin routing uses record field values, not tag patterns. Use `rewrite_tag_filter` for field-based routing:

## Rewrite Tag Based on IP (rewrite_tag_filter)

```xml
<filter nginx.access>
  @type record_transformer
  <record>
    ip_class ${record["remote"].start_with?("10.") || record["remote"].start_with?("192.168.") ? "internal" : "external"}
  </record>
</filter>

<match nginx.access>
  @type rewrite_tag_filter
  <rule>
    key     ip_class
    pattern ^internal$
    tag     nginx.internal
  </rule>
  <rule>
    key     ip_class
    pattern ^external$
    tag     nginx.external
  </rule>
</match>
```

## Route to Different Elasticsearch Indices

```xml
<match nginx.internal>
  @type elasticsearch
  host elasticsearch.internal
  port 9200
  index_name nginx-internal-%Y.%m.%d
  <buffer time>
    timekey 86400
    timekey_wait 10m
  </buffer>
</match>

<match nginx.external>
  @type elasticsearch
  host elasticsearch.internal
  port 9200
  index_name nginx-external-%Y.%m.%d
  <buffer time>
    timekey 86400
    timekey_wait 10m
  </buffer>
</match>
```

## Filter by Specific IPv4 Range (grep filter)

```xml
<!-- Drop log records from known crawlers/scrapers -->
<filter nginx.access>
  @type grep
  <exclude>
    key    remote
    pattern /^(66\.249\.|40\.77\.|13\.66\.)/
  </exclude>
</filter>
```

## Record Transformer - Add Subnet Tag

```xml
<filter nginx.access>
  @type record_transformer
  enable_ruby true
  <record>
    source_subnet ${
      ip = record["remote"].split(".")
      "#{ip[0]}.#{ip[1]}.#{ip[2]}.0/24"
    }
    environment ${
      record["remote"].start_with?("10.64.") ? "aws-prod" :
      record["remote"].start_with?("10.128.") ? "azure-prod" :
      "external"
    }
  </record>
</filter>
```

## Conclusion

Fluentd routes logs by IPv4 source address using `record_transformer` to add classification fields, `rewrite_tag_filter` to create routing tags, and separate `match` blocks for each destination. Use Ruby string methods in `record_transformer` for subnet matching; for precise CIDR matching embed a small Ruby `IPAddr.include?` check in the `enable_ruby true` block.
