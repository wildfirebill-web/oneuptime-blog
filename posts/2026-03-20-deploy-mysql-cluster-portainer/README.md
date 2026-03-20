# How to Deploy a MySQL Cluster with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, MySQL, Database Cluster, Docker Swarm, High Availability, Replication

Description: Learn how to deploy a highly available MySQL cluster using Group Replication or primary-replica setup via Portainer stacks.

---

MySQL clustering in Docker requires careful orchestration of replication, shared networking, and health checks. Portainer stacks make it straightforward to define and deploy a multi-node MySQL setup.

## Architecture Overview

```mermaid
graph TD
    App[Application] --> Primary[MySQL Primary :3306]
    Primary --> Replica1[MySQL Replica 1]
    Primary --> Replica2[MySQL Replica 2]
    Replica1 --> App2[Read Replicas]
    Replica2 --> App2
```

## Primary-Replica Setup

Deploy a primary MySQL node with binary logging enabled:

```yaml
version: "3.8"

services:
  mysql-primary:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
    command: >
      --server-id=1
      --log-bin=mysql-bin
      --binlog-format=ROW
      --gtid-mode=ON
      --enforce-gtid-consistency=ON
    volumes:
      - mysql_primary_data:/var/lib/mysql
    networks:
      - mysql_cluster
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 5

  mysql-replica:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    command: >
      --server-id=2
      --log-bin=mysql-bin
      --binlog-format=ROW
      --gtid-mode=ON
      --enforce-gtid-consistency=ON
      --read-only=ON
    volumes:
      - mysql_replica_data:/var/lib/mysql
    networks:
      - mysql_cluster
    depends_on:
      mysql-primary:
        condition: service_healthy

volumes:
  mysql_primary_data:
  mysql_replica_data:

networks:
  mysql_cluster:
    driver: bridge
```

## Configuring Replication

After the stack starts, configure replication from the primary node:

```bash
# On the primary: create a replication user

docker exec -it $(docker ps -qf name=mysql-primary) mysql -uroot -prootpassword -e "
CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'replpassword';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
"

# On the replica: point it at the primary
docker exec -it $(docker ps -qf name=mysql-replica) mysql -uroot -prootpassword -e "
CHANGE MASTER TO
  MASTER_HOST='mysql-primary',
  MASTER_USER='replicator',
  MASTER_PASSWORD='replpassword',
  MASTER_AUTO_POSITION=1;
START SLAVE;
"
```

## Verifying Replication Status

Check that the replica is in sync:

```bash
# Check replica status
docker exec -it $(docker ps -qf name=mysql-replica) mysql -uroot -prootpassword -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master"

# Expected output:
# Slave_IO_Running: Yes
# Slave_SQL_Running: Yes
# Seconds_Behind_Master: 0
```

## Adding a ProxySQL Load Balancer

Route writes to primary and reads to replicas automatically:

```yaml
  proxysql:
    image: proxysql/proxysql:latest
    ports:
      - "6033:6033"   # MySQL protocol port
      - "6032:6032"   # Admin port
    volumes:
      - ./proxysql.cnf:/etc/proxysql.cnf
    networks:
      - mysql_cluster
    depends_on:
      - mysql-primary
      - mysql-replica
```

A minimal `proxysql.cnf` routes writes (hostgroup 10) to primary and reads (hostgroup 20) to replicas:

```ini
mysql_servers =
(
  { address="mysql-primary", port=3306, hostgroup=10, max_connections=100 },
  { address="mysql-replica",  port=3306, hostgroup=20, max_connections=100, weight=1000 }
)

mysql_query_rules =
(
  { rule_id=1, match_digest="^SELECT", destination_hostgroup=20, apply=1 },
  { rule_id=2, match_digest=".*", destination_hostgroup=10, apply=1 }
)
```

## Scaling Read Replicas in Swarm

In Docker Swarm mode use the `deploy` key to run multiple replicas:

```yaml
  mysql-replica:
    image: mysql:8.0
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
    # ... rest of config
```

## Monitoring with OneUptime

OneUptime can monitor the cluster endpoint and alert if the primary becomes unavailable, or if replica lag exceeds a threshold via a custom probe script.
