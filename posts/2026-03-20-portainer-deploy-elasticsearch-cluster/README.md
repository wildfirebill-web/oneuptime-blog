# How to Deploy Elasticsearch Cluster via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Elasticsearch, Cluster, High Availability, Self-Hosted

Description: Deploy a multi-node Elasticsearch cluster via Portainer for high availability and distributed search with automatic shard allocation and replication.

## Introduction

A production Elasticsearch deployment requires multiple nodes for high availability and query performance. This guide deploys a 3-node Elasticsearch cluster using Portainer stacks, providing shard replication and automatic failover.

## Prerequisites

- Docker host with at least 8GB RAM (for 3-node cluster)
- Portainer installed

## System Configuration

```bash
# Required for all nodes - increase VM map count
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elasticsearch.conf
```

## Deploy the Elasticsearch Cluster

In Portainer, create a stack named `elasticsearch-cluster`:

```yaml
version: "3.8"

services:
  es-node1:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: es-node1
    environment:
      - node.name=es-node1
      - cluster.name=production-cluster
      - discovery.seed_hosts=es-node2,es-node3
      - cluster.initial_master_nodes=es-node1,es-node2,es-node3
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=elastic_cluster_password
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - bootstrap.memory_lock=false
    volumes:
      - es1_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic-network
    restart: unless-stopped

  es-node2:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: es-node2
    environment:
      - node.name=es-node2
      - cluster.name=production-cluster
      - discovery.seed_hosts=es-node1,es-node3
      - cluster.initial_master_nodes=es-node1,es-node2,es-node3
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=elastic_cluster_password
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - bootstrap.memory_lock=false
    volumes:
      - es2_data:/usr/share/elasticsearch/data
    ports:
      - "9201:9200"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic-network
    restart: unless-stopped

  es-node3:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: es-node3
    environment:
      - node.name=es-node3
      - cluster.name=production-cluster
      - discovery.seed_hosts=es-node1,es-node2
      - cluster.initial_master_nodes=es-node1,es-node2,es-node3
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=elastic_cluster_password
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - bootstrap.memory_lock=false
    volumes:
      - es3_data:/usr/share/elasticsearch/data
    ports:
      - "9202:9200"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic-network
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=["http://es-node1:9200","http://es-node2:9200","http://es-node3:9200"]
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=kibana_password
    ports:
      - "5601:5601"
    networks:
      - elastic-network
    depends_on:
      - es-node1
    restart: unless-stopped

networks:
  elastic-network:
    driver: bridge

volumes:
  es1_data:
  es2_data:
  es3_data:
```

## Verify Cluster Health

After deployment (allow 60-90 seconds for full initialization):

```bash
# Check cluster health (should show: green, 3 nodes)
curl -s "http://localhost:9200/_cluster/health?pretty" \
  -u elastic:elastic_cluster_password

# Check nodes
curl -s "http://localhost:9200/_cat/nodes?v" \
  -u elastic:elastic_cluster_password
```

## Create a Replicated Index

```bash
# Create an index with replication (1 replica = 2 copies total)
curl -X PUT "http://localhost:9200/logs" \
  -u elastic:elastic_cluster_password \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "number_of_shards": 3,     # Spread across 3 nodes
      "number_of_replicas": 1    # 1 replica per shard
    }
  }'
```

## Simulate Node Failure

```bash
# Stop one node to test failover
docker stop es-node3

# Check cluster status (should rebalance to yellow, then recover)
curl -s "http://localhost:9200/_cluster/health?pretty" \
  -u elastic:elastic_cluster_password

# Restart node
docker start es-node3

# Cluster should return to green
curl -s "http://localhost:9200/_cluster/health?pretty" \
  -u elastic:elastic_cluster_password
```

## Conclusion

A 3-node Elasticsearch cluster via Portainer provides high availability and distributed search at the cost of higher resource requirements. The cluster automatically handles shard allocation and replication, and recovers from node failures with minimal downtime. Portainer's stack management makes rolling upgrades straightforward by updating one node at a time.
