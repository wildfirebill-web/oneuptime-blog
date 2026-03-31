# How to Use toDecimalString() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Type Conversion, Decimal, String Formatting, Report

Description: Learn how to use toDecimalString() in ClickHouse to convert Decimal values to strings with a specific number of decimal places for reports and exports.

---

`toDecimalString(x, scale)` converts a `Decimal` (or any numeric) value to a `String` with exactly `scale` decimal places. Unlike `toString()` which uses the column's stored scale, `toDecimalString` lets you control the output precision explicitly. This is useful for generating human-readable reports, exporting financial data with consistent formatting, and producing output that must match a specific decimal format.

## Basic Usage

```sql
-- Convert a Decimal value to a string with 2 decimal places
SELECT toDecimalString(toDecimal64('99.99', 4), 2) AS formatted;
-- Returns: '99.99'

-- Control the number of decimal places in the output
SELECT toDecimalString(toDecimal64('3.14159', 5), 2) AS two_dp;
-- Returns: '3.14'

SELECT toDecimalString(toDecimal64('100', 0), 2) AS padded;
-- Returns: '100.00'
```

## Formatting Financial Amounts for Reports

When generating reports or exporting financial data, amounts must be formatted with a consistent number of decimal places.

```sql
-- Format revenue with exactly 2 decimal places
SELECT
    product_id,
    sum(amount) AS total_revenue,
    toDecimalString(sum(amount), 2) AS formatted_revenue
FROM sales
GROUP BY product_id
ORDER BY total_revenue DESC
LIMIT 10;
```

## Exporting Decimal Values as Strings

When exporting data to formats that represent all values as strings, use `toDecimalString` to ensure consistent decimal formatting.

```sql
-- Export financial records with consistent decimal format
SELECT
    invoice_id,
    toDecimalString(subtotal, 2)    AS subtotal_str,
    toDecimalString(tax_amount, 2)  AS tax_str,
    toDecimalString(total, 2)       AS total_str
FROM invoices
ORDER BY invoice_id
LIMIT 20;
```

## Comparing toDecimalString vs toString

`toString()` on a Decimal column uses the column's stored scale. `toDecimalString` lets you override it.

```sql
-- A Decimal64(4) column stores 4 decimal places
SELECT
    toString(toDecimal64('99.9', 4))         AS tostring_result,    -- '99.9000'
    toDecimalString(toDecimal64('99.9', 4), 2) AS decimal_string_2dp, -- '99.90'
    toDecimalString(toDecimal64('99.9', 4), 0) AS decimal_string_0dp; -- '100'
```

## Rounding Behavior

`toDecimalString` rounds the value to the specified number of decimal places.

```sql
-- Observe rounding behavior
SELECT
    toDecimalString(toDecimal64('3.145', 3), 2) AS rounded,    -- '3.15' (rounds up)
    toDecimalString(toDecimal64('3.144', 3), 2) AS truncated;  -- '3.14' (rounds down)
```

## Using toDecimalString with Aggregate Results

```sql
-- Format aggregate results for a summary report
SELECT
    category,
    count()                                  AS order_count,
    toDecimalString(sum(price), 2)           AS total_revenue,
    toDecimalString(avg(price), 2)           AS avg_price,
    toDecimalString(min(price), 2)           AS min_price,
    toDecimalString(max(price), 2)           AS max_price
FROM orders
GROUP BY category
ORDER BY sum(price) DESC
LIMIT 10;
```

## Zero-Padding for Consistent Output

`toDecimalString` always produces the exact number of decimal places specified, padding with zeros if needed.

```sql
-- All outputs have exactly 4 decimal places
SELECT
    toDecimalString(toDecimal64('1', 0), 4)    AS whole_number,   -- '1.0000'
    toDecimalString(toDecimal64('1.5', 1), 4)  AS one_dp,         -- '1.5000'
    toDecimalString(toDecimal64('1.23', 2), 4) AS two_dp;         -- '1.2300'
```

## Building Formatted Output Strings

Use `toDecimalString` inside `concat` to build formatted output.

```sql
-- Build a formatted price string
SELECT
    product_name,
    currency_code,
    concat(currency_code, ' ', toDecimalString(price, 2)) AS formatted_price
FROM products
LIMIT 10;

-- Example output: 'USD 99.99', 'EUR 149.50'
```

## Summary

`toDecimalString(x, scale)` converts a Decimal or numeric value to a String with exactly `scale` decimal places, rounding as needed. It is the correct tool for formatting monetary values in reports and exports where the number of decimal places must be consistent. Unlike `toString()`, it lets you control the output precision independently of the stored column scale. Use it inside `concat()` to build formatted output strings for financial reports, invoices, and data exports.
