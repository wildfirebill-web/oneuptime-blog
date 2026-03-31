# How to Use encodeXMLComponent() and decodeXMLComponent() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, XML, Data Transformation, SQL

Description: Learn how encodeXMLComponent() escapes special XML characters in ClickHouse and decodeXMLComponent() reverses it, enabling safe XML generation from SQL queries.

---

Generating or parsing XML from a database is occasionally necessary when integrating with legacy systems, SOAP APIs, feed formats like RSS and Atom, or reporting tools that consume XML. ClickHouse provides `encodeXMLComponent()` to escape the five special XML characters into their entity references, and `decodeXMLComponent()` to convert entity references back to their original characters. Using these functions ensures that data values containing `&`, `<`, `>`, `"`, or `'` do not corrupt the XML structure you are building.

## Function Signatures

```text
encodeXMLComponent(str)   -- escapes XML special characters in str
decodeXMLComponent(str)   -- unescapes XML entity references in str
```

The characters that `encodeXMLComponent` handles are listed below.

```text
Character  | Encoded as
-----------+------------
&          | &amp;
<          | &lt;
>          | &gt;
"          | &quot;
'          | &apos;
```

## Basic Encoding

Any string that may contain XML-special characters should be passed through `encodeXMLComponent` before embedding it inside XML tags.

```sql
SELECT
    raw_value,
    encodeXMLComponent(raw_value) AS escaped
FROM (
    SELECT arrayJoin([
        'Hello & World',
        '<script>alert(1)</script>',
        '5 > 3 and 2 < 4',
        '"quoted" value',
        'it''s a test'
    ]) AS raw_value
)
```

```text
raw_value                    | escaped
-----------------------------+----------------------------------
Hello & World                | Hello &amp; World
<script>alert(1)</script>    | &lt;script&gt;alert(1)&lt;/script&gt;
5 > 3 and 2 < 4              | 5 &gt; 3 and 2 &lt; 4
"quoted" value               | &quot;quoted&quot; value
it's a test                  | it&apos;s a test
```

## Generating XML Output from a SQL Query

ClickHouse can produce an XML document fragment by concatenating strings with `encodeXMLComponent` applied to data values. The following example generates `<product>` elements from a products table.

```sql
SELECT concat(
    '<product id="', encodeXMLComponent(toString(product_id)), '">',
    '<name>', encodeXMLComponent(product_name), '</name>',
    '<description>', encodeXMLComponent(description), '</description>',
    '<price>', toString(price), '</price>',
    '</product>'
) AS xml_element
FROM products
WHERE is_active = 1
LIMIT 10
```

Wrapping this in a higher-level query allows building a full XML document.

```sql
SELECT concat(
    '<?xml version="1.0" encoding="UTF-8"?>',
    '<catalog>',
    arrayStringConcat(groupArray(xml_element), ''),
    '</catalog>'
) AS xml_document
FROM (
    SELECT concat(
        '<item>',
        '<id>', encodeXMLComponent(toString(product_id)), '</id>',
        '<name>', encodeXMLComponent(product_name), '</name>',
        '</item>'
    ) AS xml_element
    FROM products
    WHERE is_active = 1
)
```

## Generating RSS Feed Items

RSS feeds are XML documents. Building them from a ClickHouse blog or news table requires escaping titles and descriptions.

```sql
SELECT concat(
    '<item>',
    '<title>', encodeXMLComponent(title), '</title>',
    '<link>', encodeXMLComponent(url), '</link>',
    '<description>', encodeXMLComponent(summary), '</description>',
    '<pubDate>', formatDateTime(published_at, '%a, %d %b %Y %H:%M:%S +0000'), '</pubDate>',
    '</item>'
) AS rss_item
FROM blog_posts
WHERE published_at >= now() - INTERVAL 30 DAY
ORDER BY published_at DESC
LIMIT 20
```

## Decoding XML Entity References Stored in Columns

If your table contains values that were previously XML-escaped before storage (common when ingesting data from XML sources), `decodeXMLComponent` recovers the original strings.

```sql
SELECT
    raw_xml_value,
    decodeXMLComponent(raw_xml_value) AS decoded_value
FROM xml_import_staging
LIMIT 20
```

## Parsing XML Content Stored in Text Columns

When XML fragments are stored as plain text, you can combine `decodeXMLComponent` with `extract()` to pull out specific field values and then decode their content.

```sql
SELECT
    xml_column,
    decodeXMLComponent(
        extract(xml_column, '<title>([^<]+)</title>')
    ) AS title,
    decodeXMLComponent(
        extract(xml_column, '<description>([^<]+)</description>')
    ) AS description
FROM xml_feed_items
WHERE event_date = today()
LIMIT 50
```

## Cleaning Escaped HTML in Text Fields

`decodeXMLComponent` also handles HTML entity references that overlap with XML entities, making it useful for cleaning text fields that were HTML-escaped before being stored.

```sql
SELECT
    raw_text,
    decodeXMLComponent(raw_text) AS clean_text
FROM user_comments
WHERE raw_text LIKE '%&amp;%'
   OR raw_text LIKE '%&lt;%'
   OR raw_text LIKE '%&gt;%'
LIMIT 100
```

## Summary

`encodeXMLComponent()` escapes the five XML special characters (`&`, `<`, `>`, `"`, `'`) into their entity reference forms, making any string safe to embed inside XML elements or attributes. `decodeXMLComponent()` reverses the process. These functions enable constructing valid XML documents entirely within SQL queries and cleaning XML-escaped data during ingestion. They are compact complements to ClickHouse's text processing toolkit whenever XML interchange is part of the data pipeline.
