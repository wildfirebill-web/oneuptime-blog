# How to Use Canal for MySQL CDC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Change Data Capture, Canal, Replication, Event

Description: Learn how to set up Alibaba Canal to parse MySQL binary logs and deliver row-level change events to downstream consumers for real-time data integration.

---

Canal is an open-source CDC tool developed by Alibaba that simulates a MySQL replica to read binary log events. It is widely used in the Java ecosystem and supports delivering change events to Kafka, RocketMQ, TCP clients, and flat files. Canal parses the binlog at the byte level for high throughput.

## Enabling MySQL Binary Logging

Canal requires row-based binary logging. Edit `my.cnf` and restart MySQL:

```ini
[mysqld]
server-id      = 1
log-bin        = mysql-bin
binlog_format  = ROW
binlog_row_image = FULL
```

Create a replication user for Canal:

```sql
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal_password';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

## Installing Canal Server

Download and extract the Canal server package:

```bash
mkdir -p /opt/canal && cd /opt/canal
wget https://github.com/alibaba/canal/releases/download/canal-1.1.7/canal.deployer-1.1.7.tar.gz
tar -xzf canal.deployer-1.1.7.tar.gz
```

## Configuring Canal Instance

Each Canal instance corresponds to one MySQL data source. Edit the instance configuration:

```bash
vi /opt/canal/conf/example/instance.properties
```

Set the MySQL connection details:

```properties
canal.instance.master.address=127.0.0.1:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal_password
canal.instance.connectionCharset=UTF-8
canal.instance.filter.regex=myapp\\..*
```

The `filter.regex` value uses `database\\.table` pattern syntax. The above captures all tables in the `myapp` schema.

## Starting the Canal Server

```bash
/opt/canal/bin/startup.sh
```

Verify it is running:

```bash
tail -f /opt/canal/logs/canal/canal.log
```

You should see lines confirming the instance connected to MySQL as a replica.

## Reading Events with the Java Client

Canal provides a Java client library. Add the dependency to your Maven project:

```xml
<dependency>
  <groupId>com.alibaba.otter</groupId>
  <artifactId>canal.client</artifactId>
  <version>1.1.7</version>
</dependency>
```

Connect and consume events:

```java
CanalConnector connector = CanalConnectors.newSingleConnector(
    new InetSocketAddress("127.0.0.1", 11111), "example", "", "");

connector.connect();
connector.subscribe("myapp\\.orders");

while (true) {
    Message message = connector.getWithoutAck(100);
    long batchId = message.getId();
    List<CanalEntry.Entry> entries = message.getEntries();

    for (CanalEntry.Entry entry : entries) {
        if (entry.getEntryType() == CanalEntry.EntryType.ROWDATA) {
            CanalEntry.RowChange rowChange =
                CanalEntry.RowChange.parseFrom(entry.getStoreValue());
            System.out.printf("Event: %s%n", rowChange.getEventType());
            for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
                rowData.getAfterColumnsList().forEach(col ->
                    System.out.printf("  %s = %s%n", col.getName(), col.getValue()));
            }
        }
    }
    connector.ack(batchId);
}
```

## Routing to Kafka

Canal supports Kafka output via the MQ adapter. Configure `canal.properties`:

```properties
canal.serverMode=kafka
kafka.bootstrap.servers=broker1:9092
canal.mq.topic=canal_myapp
canal.mq.partitionsNum=3
canal.mq.partitionHash=myapp.orders:id
```

Restart Canal and events from `myapp.orders` will appear in the `canal_myapp` Kafka topic, partitioned by the `id` column.

## Summary

Canal simulates a MySQL replica to read binary log events with minimal overhead on the source database. You enable row-based replication, create a dedicated user, configure an instance pointing to your schema, and then consume events either via the Java client or Kafka output. Its mature Java ecosystem integration and high-throughput binlog parsing make it a solid choice for real-time data synchronization within JVM-based architectures.
