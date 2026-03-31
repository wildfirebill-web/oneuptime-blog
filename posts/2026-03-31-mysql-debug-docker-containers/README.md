# How to Debug MySQL in Docker Containers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Debugging, Container, Log

Description: Learn how to debug MySQL running in Docker containers using logs, exec commands, and diagnostic queries to resolve common issues quickly.

---

Debugging MySQL inside a Docker container requires a slightly different approach than debugging a native installation. The database process runs inside an isolated filesystem and network namespace, so you need to know where to look and which tools to use.

## Checking MySQL Container Logs

The first step in any MySQL debugging session is checking the container logs. Docker captures stdout and stderr from the MySQL process, including startup errors, InnoDB initialization messages, and replication warnings.

```bash
docker logs mysql-container
docker logs --tail 100 mysql-container
docker logs --since 10m mysql-container
```

For persistent storage of logs, mount a host directory and configure MySQL to write to a log file:

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - ./mysql-logs:/var/log/mysql
    command: --general-log=1 --general-log-file=/var/log/mysql/general.log --slow-query-log=1 --slow-query-log-file=/var/log/mysql/slow.log --long-query-time=2
```

## Running Diagnostic Commands Inside the Container

Use `docker exec` to open a shell or run MySQL commands directly inside the container:

```bash
# Open an interactive shell
docker exec -it mysql-container bash

# Connect to MySQL directly
docker exec -it mysql-container mysql -uroot -p

# Run a one-off SQL command
docker exec mysql-container mysql -uroot -psecret -e "SHOW STATUS LIKE 'Threads_connected';"
```

Once inside, run these diagnostic queries to understand the current state:

```sql
-- Check running queries
SHOW FULL PROCESSLIST;

-- Check InnoDB status for deadlocks and lock waits
SHOW ENGINE INNODB STATUS\G

-- Check table status
SHOW TABLE STATUS FROM mydb\G

-- Check error log messages
SHOW VARIABLES LIKE 'log_error';
```

## Inspecting the Container Configuration

Verify how MySQL was started and what configuration files are active:

```bash
# View container environment variables
docker inspect mysql-container | jq '.[0].Config.Env'

# Check which my.cnf is loaded
docker exec mysql-container mysql -uroot -psecret -e "SHOW VARIABLES LIKE 'datadir';"
docker exec mysql-container cat /etc/mysql/my.cnf
docker exec mysql-container mysql --verbose --help 2>&1 | grep -A 1 "Default options"
```

You can also mount a custom configuration file for debugging purposes:

```bash
docker run -d \
  --name mysql-debug \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v $(pwd)/custom.cnf:/etc/mysql/conf.d/custom.cnf \
  mysql:8.0
```

Where `custom.cnf` contains:

```text
[mysqld]
general_log = 1
slow_query_log = 1
long_query_time = 0
log_output = FILE
```

## Checking Network and Port Connectivity

If applications cannot connect to MySQL in Docker, check network configuration:

```bash
# Check exposed ports
docker port mysql-container

# Check Docker network
docker network inspect bridge

# Test connectivity from another container
docker exec app-container bash -c "apt-get install -y mysql-client && mysql -h mysql-container -uroot -psecret -e 'SELECT 1;'"
```

## Examining the Error Log File

InnoDB crash recovery details and table corruption messages appear in the MySQL error log, not just stdout:

```bash
docker exec mysql-container bash -c "cat \$(mysql -uroot -psecret -sse \"SELECT @@log_error;\")"
```

## Summary

Debugging MySQL in Docker containers relies on three key approaches: reading container logs with `docker logs`, using `docker exec` to run diagnostic SQL commands, and inspecting container configuration and network settings. Mounting custom config files and log directories makes it easier to capture detailed output during troubleshooting sessions. Always check `SHOW ENGINE INNODB STATUS` when investigating deadlocks or lock contention, and use `SHOW FULL PROCESSLIST` to identify long-running queries blocking other transactions.
