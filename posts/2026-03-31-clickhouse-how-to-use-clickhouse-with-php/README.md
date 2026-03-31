# How to Use ClickHouse with PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PHP, Integration, Database

Description: Connect PHP applications to ClickHouse using the HTTP interface and available PHP clients to run queries and insert data efficiently.

---

## Overview

ClickHouse provides an HTTP interface on port 8123, making it straightforward to integrate with PHP. You can use the raw HTTP interface with cURL, or use community PHP clients for a more ergonomic experience.

## Option 1 - Using the HTTP Interface with cURL

The simplest approach requires no additional dependencies:

```php
<?php

class ClickHouseClient
{
    private string $host;
    private int $port;
    private string $user;
    private string $password;

    public function __construct(
        string $host = 'localhost',
        int $port = 8123,
        string $user = 'default',
        string $password = ''
    ) {
        $this->host = $host;
        $this->port = $port;
        $this->user = $user;
        $this->password = $password;
    }

    public function query(string $sql): array
    {
        $url = sprintf('http://%s:%d/?query=%s',
            $this->host,
            $this->port,
            urlencode($sql . ' FORMAT JSONEachRow')
        );

        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => $this->user . ':' . $this->password,
            CURLOPT_HTTPHEADER => ['Accept-Encoding: gzip'],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode !== 200) {
            throw new \RuntimeException("ClickHouse error: $response");
        }

        $rows = [];
        foreach (explode("\n", trim($response)) as $line) {
            if ($line !== '') {
                $rows[] = json_decode($line, true);
            }
        }
        return $rows;
    }

    public function insert(string $table, array $rows): void
    {
        $lines = [];
        foreach ($rows as $row) {
            $lines[] = json_encode($row);
        }

        $url = sprintf('http://%s:%d/?query=%s',
            $this->host,
            $this->port,
            urlencode("INSERT INTO $table FORMAT JSONEachRow")
        );

        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => implode("\n", $lines),
            CURLOPT_USERPWD => $this->user . ':' . $this->password,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode !== 200) {
            throw new \RuntimeException("ClickHouse insert error: $response");
        }
    }
}
```

## Option 2 - Using the smi2/phpclickhouse Library

Install via Composer:

```bash
composer require smi2/phpclickhouse
```

Basic usage:

```php
<?php
require 'vendor/autoload.php';

use ClickHouseDB\Client;

$config = [
    'host' => '127.0.0.1',
    'port' => '8123',
    'username' => 'default',
    'password' => '',
];

$db = new Client($config);
$db->database('my_database');
$db->setTimeout(1.5);
$db->setConnectTimeOut(5);

// Ping
if ($db->ping()) {
    echo "Connected to ClickHouse\n";
}
```

## Creating a Table

```php
<?php
$db->write("
    CREATE TABLE IF NOT EXISTS events (
        event_date Date,
        event_time DateTime,
        user_id UInt64,
        event_type String,
        properties String
    ) ENGINE = MergeTree()
    PARTITION BY toYYYYMM(event_date)
    ORDER BY (event_date, user_id)
");
```

## Inserting Data

```php
<?php
// Insert multiple rows efficiently
$rows = [
    [
        'event_date' => date('Y-m-d'),
        'event_time' => date('Y-m-d H:i:s'),
        'user_id' => 1001,
        'event_type' => 'page_view',
        'properties' => json_encode(['url' => '/home'])
    ],
    [
        'event_date' => date('Y-m-d'),
        'event_time' => date('Y-m-d H:i:s'),
        'user_id' => 1002,
        'event_type' => 'click',
        'properties' => json_encode(['element' => 'button'])
    ],
];

$db->insert('events', $rows, [
    'event_date', 'event_time', 'user_id', 'event_type', 'properties'
]);
```

## Querying Data

```php
<?php
// Simple query
$statement = $db->select('SELECT count() AS cnt FROM events');
$rows = $statement->rows();
echo "Total events: " . $rows[0]['cnt'] . "\n";

// Query with parameters (use prepared statement style)
$userId = 1001;
$statement = $db->select(
    "SELECT event_type, count() AS cnt
     FROM events
     WHERE user_id = :user_id
     GROUP BY event_type",
    ['user_id' => $userId]
);

foreach ($statement->rows() as $row) {
    echo sprintf("%s: %d\n", $row['event_type'], $row['cnt']);
}
```

## Using the Raw HTTP Client for Large Inserts

```php
<?php
// For large bulk inserts, stream data directly
function bulkInsert(string $table, iterable $data, int $batchSize = 5000): void
{
    $buffer = [];
    foreach ($data as $row) {
        $buffer[] = json_encode($row);
        if (count($buffer) >= $batchSize) {
            sendBatch($table, $buffer);
            $buffer = [];
        }
    }
    if (!empty($buffer)) {
        sendBatch($table, $buffer);
    }
}

function sendBatch(string $table, array $lines): void
{
    $ch = curl_init("http://localhost:8123/?query=" . urlencode("INSERT INTO $table FORMAT JSONEachRow"));
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => implode("\n", $lines),
        CURLOPT_RETURNTRANSFER => true,
    ]);
    $response = curl_exec($ch);
    curl_close($ch);
}
```

## Error Handling

```php
<?php
try {
    $result = $db->select('SELECT * FROM non_existent_table');
} catch (\ClickHouseDB\Exception\QueryException $e) {
    echo "Query failed: " . $e->getMessage() . "\n";
    echo "Error code: " . $e->getCode() . "\n";
}
```

## Summary

PHP connects to ClickHouse through the HTTP interface on port 8123, either via raw cURL requests or the `smi2/phpclickhouse` Composer package. For bulk inserts, stream data in batches of thousands of rows as JSON lines. Always use parameterized queries to prevent injection, and handle `QueryException` for proper error management in production applications.
