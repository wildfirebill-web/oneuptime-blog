# How to Deploy Elasticsearch via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Elasticsearch, Search, Docker, Deployment

Description: Learn how to deploy Elasticsearch via Portainer with security enabled, persistent storage, and proper JVM memory configuration for a single-node or small cluster setup.

## Single-Node Elasticsearch via Portainer

**Stacks → Add Stack → elasticsearch**

```yaml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    restart: unless-stopped
    environment:
      - node.name=es01
      - cluster.name=docker-cluster
      - discovery.type=single-node    # Single node mode
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false    # Disable for internal use
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"    # REST API
      - "9300:9300"    # Inter-node communication
    healthcheck:
      test: ["CMD-SHELL", "curl -s -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health | grep -qE '\"status\":\"(green|yellow)\"'"]
      interval: 20s
      timeout: 10s
      retries: 5

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    restart: unless-stopped
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
    ports:
      - "5601:5601"
    depends_on:
      elasticsearch:
        condition: service_healthy

volumes:
  elasticsearch_data:
```

## Environment Variables

```text
ELASTIC_PASSWORD = your-elastic-password
KIBANA_PASSWORD = kibana-system-password
```

## Pre-Deployment: Set vm.max_map_count

Elasticsearch requires a higher value for virtual memory maps:

```bash
# On the Docker host

sudo sysctl -w vm.max_map_count=262144

# Make permanent
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Configure Kibana System User

After Elasticsearch starts, set the Kibana system user password:

```bash
# Via Portainer exec on Elasticsearch container
curl -u elastic:${ELASTIC_PASSWORD} \
  -X POST "http://localhost:9200/_security/user/kibana_system/_password" \
  -H "Content-Type: application/json" \
  -d '{"password": "kibana-system-password"}'
```

## Verify Elasticsearch

```bash
# Check cluster health
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health?pretty

# Check nodes
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cat/nodes?v

# Create a test index
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "http://localhost:9200/test-index" \
  -H "Content-Type: application/json" \
  -d '{"settings": {"number_of_shards": 1, "number_of_replicas": 0}}'
```

## JVM Memory Sizing

Set JVM heap to 50% of available container memory:

| Available RAM | JVM Heap Setting |
|--------------|-----------------|
| 2GB | `-Xms1g -Xmx1g` |
| 4GB | `-Xms2g -Xmx2g` |
| 8GB | `-Xms4g -Xmx4g` |

```yaml
environment:
  - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
```

## Conclusion

Elasticsearch via Portainer works well for development and small production deployments with the `single-node` discovery type. The key requirements are setting `vm.max_map_count` on the host, allocating appropriate JVM memory, and using Portainer's environment variable management for the Elastic password. Once running, Kibana provides the browser-based UI for index management and queries.
