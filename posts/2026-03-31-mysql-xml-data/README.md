# How to Work with XML Data in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, XML, Function, Data Type

Description: Learn how to store, query, and modify XML data in MySQL using ExtractValue, UpdateXML, and XPath expressions for practical XML data management.

---

## XML Support in MySQL

MySQL does not have a dedicated XML column type, but it provides two built-in functions for working with XML strings stored in TEXT or VARCHAR columns:

- `ExtractValue(xml, xpath)` - reads data from an XML string
- `UpdateXML(xml, xpath, new_xml)` - modifies an XML string

These functions support a subset of XPath 1.0, making it possible to query and manipulate XML directly within SQL.

## Storing XML Data

```sql
CREATE TABLE product_catalog (
  id INT AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(50),
  attributes TEXT
);

INSERT INTO product_catalog (sku, attributes) VALUES
('LAPTOP-001', '<product><name>ProBook 15</name><brand>Acme</brand><price>1299.99</price><stock>42</stock></product>'),
('PHONE-001',  '<product><name>SmartPhone X</name><brand>TechCo</brand><price>699.00</price><stock>150</stock></product>'),
('TABLET-001', '<product><name>TabPro 10</name><brand>Acme</brand><price>499.99</price><stock>0</stock></product>');
```

## Reading XML with ExtractValue

```sql
-- Extract product names
SELECT sku, ExtractValue(attributes, '/product/name') AS name
FROM product_catalog;

-- Extract price as a number for sorting
SELECT sku,
       ExtractValue(attributes, '/product/name') AS name,
       ExtractValue(attributes, '/product/price') + 0 AS price
FROM product_catalog
ORDER BY price ASC;
```

## Filtering by XML Content

```sql
-- Find products from a specific brand
SELECT sku, ExtractValue(attributes, '/product/name') AS name
FROM product_catalog
WHERE ExtractValue(attributes, '/product/brand') = 'Acme';

-- Find out-of-stock products
SELECT sku
FROM product_catalog
WHERE ExtractValue(attributes, '/product/stock') + 0 = 0;
```

## Modifying XML with UpdateXML

```sql
-- Update stock for a specific SKU
UPDATE product_catalog
SET attributes = UpdateXML(attributes, '/product/stock', '<stock>100</stock>')
WHERE sku = 'TABLET-001';

-- Apply a price discount
UPDATE product_catalog
SET attributes = UpdateXML(
  attributes,
  '/product/price',
  CONCAT('<price>', ROUND(ExtractValue(attributes, '/product/price') * 0.9, 2), '</price>')
)
WHERE ExtractValue(attributes, '/product/brand') = 'TechCo';
```

## Reading Attributes from XML Elements

```sql
SET @xml = '<items><item id="1" type="A">Widget</item><item id="2" type="B">Gadget</item></items>';

-- Extract attribute
SELECT ExtractValue(@xml, '/items/item/@id');    -- Result: 1 2
SELECT ExtractValue(@xml, '/items/item/@type');  -- Result: A B
```

## Validating XML Structure

```sql
-- Check if the expected node exists (returns empty string if missing)
SELECT sku,
  CASE
    WHEN ExtractValue(attributes, '/product/price') = '' THEN 'missing price'
    ELSE 'ok'
  END AS status
FROM product_catalog;
```

## When to Use JSON Instead

MySQL 5.7+ introduced a native JSON column type with richer support than XML:

```sql
-- JSON equivalent of the XML example above
CREATE TABLE product_catalog_json (
  id INT AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(50),
  attributes JSON
);

INSERT INTO product_catalog_json (sku, attributes) VALUES
('LAPTOP-001', '{"name":"ProBook 15","brand":"Acme","price":1299.99,"stock":42}');

-- Query with native JSON functions
SELECT sku, JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.name')) AS name
FROM product_catalog_json;
```

For new projects, JSON is preferred over XML due to better index support, richer functions, and tighter integration with MySQL's query optimizer.

## Summary

MySQL supports XML data through `ExtractValue()` and `UpdateXML()` with XPath 1.0 expressions. These functions let you read, filter, and modify XML stored in TEXT columns directly in SQL. For complex data structures or new schemas, migrate to MySQL's JSON type for better performance and more powerful querying capabilities.
