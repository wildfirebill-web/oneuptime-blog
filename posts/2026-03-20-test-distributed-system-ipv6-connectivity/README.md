# How to Test Distributed System IPv6 Connectivity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Distributed Systems, Testing, Connectivity, Network Testing, DevOps

Description: Systematically test IPv6 connectivity between distributed system components using network tools, custom health check scripts, and automated test suites.

---

Testing IPv6 connectivity in distributed systems requires verifying not just basic reachability but also that each component's IPv6-specific ports, TLS, and application-layer protocols function correctly.

## Layer-by-Layer Testing Approach

```bash
# Test from outermost (application) to innermost (network)

# Layer 1: Network - can we ping over IPv6?
ping6 -c 3 2001:db8::1

# Layer 2: Transport - can we connect on the right port?
nc -6 -w 3 2001:db8::1 9042    # Cassandra
nc -6 -w 3 2001:db8::1 6379    # Redis
nc -6 -w 3 2001:db8::1 9092    # Kafka

# Layer 3: Application - does the service respond correctly?
redis-cli -h 2001:db8::1 PING  # Should return PONG
```

## Automated Connectivity Test Script

```bash
#!/bin/bash
# test_distributed_ipv6.sh - Comprehensive connectivity test

declare -A SERVICES=(
  ["etcd"]="2001:db8::1:2379"
  ["kafka"]="2001:db8::2:9092"
  ["redis"]="2001:db8::3:6379"
  ["cassandra"]="2001:db8::4:9042"
  ["consul"]="2001:db8::5:8500"
)

PASS=0
FAIL=0

test_tcp_connection() {
  local name="$1"
  local host="$2"
  local port="$3"

  if nc -6 -w 3 "$host" "$port" < /dev/null > /dev/null 2>&1; then
    echo "PASS: $name TCP connection to [$host]:$port"
    ((PASS++))
    return 0
  else
    echo "FAIL: $name TCP connection to [$host]:$port"
    ((FAIL++))
    return 1
  fi
}

for service in "${!SERVICES[@]}"; do
  IFS=':' read -r host port <<< "${SERVICES[$service]}"
  test_tcp_connection "$service" "$host" "$port"
done

echo ""
echo "Results: $PASS passed, $FAIL failed"
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

## Testing Kafka over IPv6

```bash
# Test Kafka producer and consumer connectivity over IPv6
kafka-topics.sh \
  --bootstrap-server [2001:db8::1]:9092 \
  --list 2>&1 | grep -v "Error" && echo "Kafka: PASS" || echo "Kafka: FAIL"

# Test with authentication if needed
kafka-console-producer.sh \
  --bootstrap-server [2001:db8::1]:9092 \
  --topic connectivity-test \
  --timeout 5000 < /dev/null 2>&1
```

## Testing Database Connectivity over IPv6

```bash
# PostgreSQL
psql -h 2001:db8::1 -p 5432 -U testuser -c "SELECT 1" && echo "PostgreSQL: PASS"

# MySQL
mysql -h 2001:db8::1 -P 3306 -u testuser -ptestpass \
  -e "SELECT 1" && echo "MySQL: PASS"

# MongoDB
mongo --host "[2001:db8::1]:27017" --eval "db.adminCommand('ping')" && echo "MongoDB: PASS"

# Redis
redis-cli -h 2001:db8::1 PING | grep -q PONG && echo "Redis: PASS"

# Elasticsearch
curl -6 -s http://[2001:db8::1]:9200/_cluster/health | \
  python3 -c "import sys,json; d=json.load(sys.stdin); \
  print('Elasticsearch:', 'PASS' if d['status']!='red' else 'FAIL')"
```

## TLS Certificate Verification over IPv6

```bash
# Verify TLS certs for each service
check_tls() {
  local host="$1"
  local port="$2"
  local name="$3"

  result=$(echo | timeout 5 openssl s_client \
    -connect "[$host]:$port" \
    -servername "$host" 2>/dev/null | \
    openssl x509 -noout -checkend 86400 2>/dev/null)

  if echo "$result" | grep -q "not expire"; then
    echo "TLS PASS: $name cert valid for > 1 day"
  else
    echo "TLS FAIL: $name cert expires within 1 day or unreachable"
  fi
}

check_tls "2001:db8::1" 2379 "etcd"
check_tls "2001:db8::1" 8200 "vault"
```

## Kubernetes IPv6 Pod Connectivity Test

```bash
# Deploy a test pod to verify IPv6 connectivity in Kubernetes
kubectl run ipv6-test --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- sh -c '
    echo "Testing service connectivity over IPv6..."
    curl -6 http://kafka-service:9092 2>&1 | head -5
    curl -6 http://redis-service:6379 2>&1 | head -5
    echo "Done"
  '
```

## Measuring Latency and Throughput over IPv6

```bash
# Measure Redis latency over IPv6
redis-cli -h 2001:db8::1 --latency --latency-history

# Measure database query latency
time psql -h 2001:db8::1 -c "SELECT COUNT(*) FROM large_table" > /dev/null

# Network throughput test
iperf3 -6 -c 2001:db8::1 -t 10 -P 4
```

Systematic IPv6 connectivity testing from network layer through application layer ensures distributed systems components can communicate reliably before deploying production workloads on IPv6 infrastructure.
