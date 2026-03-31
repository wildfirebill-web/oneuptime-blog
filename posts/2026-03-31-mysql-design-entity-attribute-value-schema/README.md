# How to Design an Entity-Attribute-Value (EAV) Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EAV, Schema Design, Flexible Attribute

Description: Learn when to use the EAV pattern in MySQL, how to build it correctly, and how to query it efficiently despite its trade-offs.

---

The Entity-Attribute-Value (EAV) pattern stores metadata as rows rather than columns. It is useful when you have a large or unpredictable set of attributes per entity, such as product specifications in an e-commerce catalog where a TV has different attributes than a shirt.

## Core Schema

```sql
CREATE TABLE entities (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    entity_type VARCHAR(50) NOT NULL,
    name        VARCHAR(255) NOT NULL,
    created_at  DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_type (entity_type)
);

CREATE TABLE attributes (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name       VARCHAR(100) NOT NULL,
    data_type  ENUM('text', 'number', 'date', 'boolean') NOT NULL DEFAULT 'text',
    PRIMARY KEY (id),
    UNIQUE KEY uq_attr_name (name)
);

CREATE TABLE entity_attribute_values (
    entity_id    INT UNSIGNED NOT NULL,
    attribute_id INT UNSIGNED NOT NULL,
    value_text   VARCHAR(1000) NULL,
    value_number DECIMAL(18,4) NULL,
    value_date   DATE          NULL,
    value_bool   TINYINT(1)    NULL,
    PRIMARY KEY (entity_id, attribute_id),
    KEY idx_attr_entity (attribute_id, entity_id),
    CONSTRAINT fk_eav_entity FOREIGN KEY (entity_id)
        REFERENCES entities (id) ON DELETE CASCADE,
    CONSTRAINT fk_eav_attr   FOREIGN KEY (attribute_id)
        REFERENCES attributes (id) ON DELETE CASCADE
);
```

Storing typed values in separate columns avoids the notorious "everything as VARCHAR" anti-pattern.

## Inserting Data

```sql
INSERT INTO entities (entity_type, name) VALUES ('product', 'Samsung 65" TV');
SET @eid = LAST_INSERT_ID();

INSERT INTO attributes (name, data_type) VALUES
    ('screen_size_inches', 'number'),
    ('resolution',         'text'),
    ('smart_tv',           'boolean');

INSERT INTO entity_attribute_values (entity_id, attribute_id, value_number) VALUES (@eid, 1, 65);
INSERT INTO entity_attribute_values (entity_id, attribute_id, value_text)   VALUES (@eid, 2, '4K UHD');
INSERT INTO entity_attribute_values (entity_id, attribute_id, value_bool)   VALUES (@eid, 3, 1);
```

## Querying EAV With PIVOT-Style Joins

```sql
SELECT
    e.name,
    size_attr.value_number  AS screen_size,
    res_attr.value_text     AS resolution,
    smart_attr.value_bool   AS is_smart_tv
FROM entities e
LEFT JOIN entity_attribute_values size_attr
    ON size_attr.entity_id = e.id AND size_attr.attribute_id = 1
LEFT JOIN entity_attribute_values res_attr
    ON res_attr.entity_id  = e.id AND res_attr.attribute_id  = 2
LEFT JOIN entity_attribute_values smart_attr
    ON smart_attr.entity_id = e.id AND smart_attr.attribute_id = 3
WHERE e.entity_type = 'product';
```

## Filtering by Attribute Value

```sql
SELECT e.name
FROM entities e
JOIN entity_attribute_values eav ON eav.entity_id = e.id
JOIN attributes a ON a.id = eav.attribute_id
WHERE a.name = 'screen_size_inches'
  AND eav.value_number >= 55;
```

## When to Avoid EAV

EAV adds complexity and makes queries verbose. Prefer it only when:
- The attribute set is truly dynamic or user-defined
- The number of possible attributes is too large for columns
- You are building a generic platform (e.g., CMS, form builder)

For a fixed schema, use standard columns instead.

## Summary

EAV stores attributes as rows rather than columns, enabling flexible schemas at the cost of query complexity. Use typed value columns rather than a single VARCHAR. Index the `(attribute_id, entity_id)` pair for filter queries and use LEFT JOINs to pivot EAV rows back into a flat result set.
