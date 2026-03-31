# How to Use Aggregation Pipelines with MongoDB PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, PHP, Aggregation, Pipeline, Library

Description: Learn how to build and execute MongoDB aggregation pipelines in PHP using the official MongoDB PHP Library with practical stage examples.

---

## Overview

The MongoDB PHP Library exposes aggregation via the `aggregate()` method on a collection. Pipelines are expressed as PHP arrays where each element is an associative array representing a stage document - the same syntax as writing a pipeline in mongosh.

## Setup

```bash
composer require mongodb/mongodb
```

```php
<?php
require 'vendor/autoload.php';

use MongoDB\Client;

$client = new Client('mongodb://localhost:27017');
$orders = $client->shopdb->orders;
```

## Match and Group

```php
$pipeline = [
    ['$match' => ['status' => 'completed']],
    ['$group' => [
        '_id'          => '$category',
        'totalRevenue' => ['$sum' => '$amount'],
        'orderCount'   => ['$sum' => 1],
        'avgOrder'     => ['$avg' => '$amount'],
    ]],
    ['$sort' => ['totalRevenue' => -1]],
    ['$limit' => 5],
];

$results = $orders->aggregate($pipeline);

foreach ($results as $row) {
    echo "{$row['_id']}: \${$row['totalRevenue']} ({$row['orderCount']} orders)\n";
}
```

## Unwind Stage

```php
$pipeline = [
    ['$unwind' => '$tags'],
    ['$group' => [
        '_id'   => '$tags',
        'count' => ['$sum' => 1],
    ]],
    ['$sort' => ['count' => -1]],
    ['$limit' => 10],
];

$topTags = $orders->aggregate($pipeline);
```

## Project Stage

```php
$pipeline = [
    ['$match' => ['category' => 'electronics']],
    ['$project' => [
        'name'         => 1,
        'price'        => 1,
        'priceWithTax' => ['$multiply' => ['$price', 1.2]],
        '_id'          => 0,
    ]],
];
```

## Lookup (Left Outer Join)

```php
$pipeline = [
    ['$match' => ['status' => 'pending']],
    ['$lookup' => [
        'from'         => 'customers',
        'localField'   => 'customerId',
        'foreignField' => '_id',
        'as'           => 'customerInfo',
    ]],
    ['$unwind' => '$customerInfo'],
    ['$project' => [
        'orderId'       => 1,
        'amount'        => 1,
        'customerName'  => '$customerInfo.name',
        'customerEmail' => '$customerInfo.email',
    ]],
];

$enriched = $orders->aggregate($pipeline);
foreach ($enriched as $order) {
    echo "Order {$order['orderId']} for {$order['customerName']}: \${$order['amount']}\n";
}
```

## Date-Based Grouping

```php
$pipeline = [
    ['$match' => ['status' => 'completed']],
    ['$group' => [
        '_id' => [
            'year'  => ['$year'  => '$createdAt'],
            'month' => ['$month' => '$createdAt'],
        ],
        'revenue' => ['$sum' => '$amount'],
    ]],
    ['$sort' => ['_id.year' => 1, '_id.month' => 1]],
];

$monthly = $orders->aggregate($pipeline);
foreach ($monthly as $row) {
    echo "{$row['_id']['year']}-{$row['_id']['month']}: \${$row['revenue']}\n";
}
```

## Pagination in Aggregation

```php
$page = 0;
$size = 20;

$pipeline = [
    ['$match' => ['status' => 'active']],
    ['$sort'  => ['createdAt' => -1]],
    ['$skip'  => $page * $size],
    ['$limit' => $size],
];
```

## Aggregation with Options

```php
$options = [
    'allowDiskUse' => true,   // spill to disk for large pipelines
    'maxTimeMS'    => 30000,  // 30 second timeout
    'typeMap'      => ['root' => 'array', 'document' => 'array'],
];

$results = $orders->aggregate($pipeline, $options);
```

## AddFields Stage

```php
$pipeline = [
    ['$addFields' => [
        'fullName' => ['$concat' => ['$firstName', ' ', '$lastName']],
        'isVip'    => ['$gte' => ['$totalSpent', 1000]],
    ]],
];
```

## Facet for Multi-Dimensional Analytics

```php
$pipeline = [
    ['$facet' => [
        'byCategory' => [
            ['$group' => ['_id' => '$category', 'count' => ['$sum' => 1]]],
        ],
        'byPriceRange' => [
            ['$bucket' => [
                'groupBy'    => '$price',
                'boundaries' => [0, 50, 100, 500],
                'default'    => '500+',
                'output'     => ['count' => ['$sum' => 1]],
            ]],
        ],
    ]],
];
```

## Summary

MongoDB aggregation pipelines in PHP are plain nested arrays passed to `$collection->aggregate()`. Each array element is a stage document using the standard `$match`, `$group`, `$project`, `$lookup`, `$unwind`, `$sort`, `$skip`, `$limit`, `$addFields`, and `$facet` operators. Set `allowDiskUse` in options for large datasets and `typeMap` to control whether results decode to PHP arrays or BSON objects.
