# How to Use Aggregation Pipelines with MongoDB Ruby

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Ruby, Aggregation, Pipeline, Analytics

Description: Learn how to build and execute MongoDB aggregation pipelines in Ruby to group, filter, and transform documents for analytics and reporting.

---

## Introduction

MongoDB's aggregation framework lets you process documents through a series of stages - each stage transforms the data and passes results to the next. The Ruby driver exposes this through the `collection.aggregate` method, accepting an array of pipeline stage hashes.

## Basic Setup

```ruby
require 'mongo'

client = Mongo::Client.new(['localhost:27017'], database: 'shop')
orders = client[:orders]
```

## Simple $match and $group

Count orders by status:

```ruby
pipeline = [
  { '$match' => { created_at: { '$gte' => Time.now - 86400 * 30 } } },
  { '$group' => {
      _id:   '$status',
      count: { '$sum' => 1 },
      total: { '$sum' => '$amount' }
  }},
  { '$sort' => { total: -1 } }
]

results = orders.aggregate(pipeline).to_a
results.each do |r|
  puts "#{r['_id']}: #{r['count']} orders, $#{r['total']}"
end
```

## $unwind for Array Fields

```ruby
# Flatten line items to count product sales
pipeline = [
  { '$unwind' => '$items' },
  { '$group' => {
      _id:      '$items.product_id',
      quantity: { '$sum' => '$items.qty' }
  }},
  { '$sort' => { quantity: -1 } },
  { '$limit' => 10 }
]

top_products = orders.aggregate(pipeline).to_a
```

## $lookup - Joining Collections

```ruby
pipeline = [
  { '$lookup' => {
      from:         'customers',
      localField:   'customer_id',
      foreignField: '_id',
      as:           'customer'
  }},
  { '$unwind' => '$customer' },
  { '$project' => {
      order_id:       '$_id',
      customer_name:  '$customer.name',
      amount:         1
  }}
]

enriched_orders = orders.aggregate(pipeline).to_a
```

## $project and $addFields

```ruby
pipeline = [
  { '$addFields' => {
      discount_amount: { '$multiply' => ['$amount', 0.1] }
  }},
  { '$project' => {
      amount:          1,
      discount_amount: 1,
      final_price: { '$subtract' => ['$amount', '$discount_amount'] }
  }}
]
```

## $facet for Multi-Dimension Analysis

```ruby
pipeline = [
  { '$facet' => {
      by_status: [
        { '$group' => { _id: '$status', count: { '$sum' => 1 } } }
      ],
      by_month: [
        { '$group' => {
            _id:   { '$month' => '$created_at' },
            total: { '$sum' => '$amount' }
        }}
      ]
  }}
]

facets = orders.aggregate(pipeline).first
```

## Using allowDiskUse for Large Datasets

```ruby
results = orders.aggregate(pipeline, allow_disk_use: true).to_a
```

## Summary

MongoDB aggregation pipelines in Ruby are expressed as arrays of stage hashes passed to `collection.aggregate`. Common stages include `$match` for filtering, `$group` for aggregation, `$unwind` for arrays, `$lookup` for joins, `$project` for field selection, and `$facet` for parallel sub-pipelines. Add `allow_disk_use: true` for pipelines that exceed the 100 MB memory limit.
