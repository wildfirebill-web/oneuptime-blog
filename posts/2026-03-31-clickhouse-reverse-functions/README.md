# How to Use reverse() and reverseUTF8() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, Unicode, Analytics

Description: Learn the difference between reverse() and reverseUTF8() in ClickHouse, when each is correct, and practical uses like palindrome detection and path reversal.

---

ClickHouse provides two functions for reversing strings: `reverse()` and `reverseUTF8()`. They look similar but operate at different levels of abstraction. Choosing the wrong one for multi-byte input produces garbled output, so understanding the distinction is important for any workload that handles non-ASCII text.

## reverse() - Byte-Level Reversal

```text
reverse(str)
```

`reverse()` treats the input as a raw sequence of bytes and reverses their order. For pure ASCII text (where every character is exactly one byte) this is identical to reversing the characters. For strings that contain multi-byte UTF-8 characters, `reverse()` reverses the bytes - not the code points - which breaks the encoding of those characters.

```sql
-- ASCII: byte reversal equals character reversal
SELECT reverse('hello') AS result;
-- result: 'olleh'

-- Multi-byte: byte reversal corrupts the encoding
SELECT reverse('cafe\xCC\x81') AS result; -- 'café' as NFD
-- result: garbled bytes - the combining accent is now in the wrong position
```

Use `reverse()` when you are certain the input is ASCII-only or when you are operating on raw byte data where byte order is what matters.

## reverseUTF8() - Code Point-Level Reversal

```text
reverseUTF8(str)
```

`reverseUTF8()` decodes the string as a sequence of Unicode code points (characters) and reverses their order, then re-encodes the result as UTF-8. This produces a correctly encoded string for any input, including emoji, accented characters, Chinese, Arabic, and other multi-byte scripts.

```sql
-- Multi-byte characters are handled correctly
SELECT reverseUTF8('Привет') AS result;
-- result: 'тевирП'

SELECT reverseUTF8('こんにちは') AS result;
-- result: 'はちにんこ'

SELECT reverseUTF8('hello') AS result;
-- result: 'olleh'  (same as reverse() for ASCII)
```

For any general-purpose text processing pipeline, `reverseUTF8()` is the safe default.

## Palindrome Detection

A palindrome is a string that reads the same forwards and backwards. Reversing and comparing is a direct way to test for this property:

```sql
-- Find words that are palindromes (ASCII words)
SELECT word
FROM dictionary
WHERE lower(word) = reverse(lower(word))
  AND length(word) > 1
ORDER BY length(word) DESC
LIMIT 20;
```

```sql
-- Palindrome check for Unicode strings
SELECT
    word,
    (lowerUTF8(word) = reverseUTF8(lowerUTF8(word))) AS is_palindrome
FROM international_dictionary
WHERE lengthUTF8(word) > 1
ORDER BY is_palindrome DESC, word
LIMIT 20;
```

To check a specific value interactively:

```sql
SELECT
    'racecar' AS word,
    reverse('racecar') AS reversed,
    ('racecar' = reverse('racecar')) AS is_palindrome;
```

## Reversing File Paths and Hierarchies

Reversing segments of a file path or a dotted namespace is a common normalization technique when you want to sort or prefix-match from the deepest component outward:

```sql
-- Reverse a dot-separated Java package name for right-to-left sorting
SELECT
    class_name,
    arrayStringConcat(arrayReverse(splitByChar('.', class_name)), '.') AS reversed_package
FROM java_classes
ORDER BY reversed_package
LIMIT 10;
```

```sql
-- Reverse path segments so you can filter by the last segment (file extension area) as a prefix
SELECT
    file_path,
    arrayStringConcat(
        arrayReverse(arrayFilter(x -> x != '', splitByChar('/', file_path))),
        '/'
    ) AS reversed_path
FROM files
ORDER BY reversed_path
LIMIT 10;
```

This technique is sometimes used to index domain names in reverse (e.g., `com.example.api`) so that all subdomains of `example.com` share a common prefix in sort order.

## Reversing Domain Names for Prefix Indexing

Storing domain names in reverse order (TLD first) enables prefix-based queries over subdomain hierarchies:

```sql
-- Reverse a domain name: 'api.example.com' -> 'com.example.api'
SELECT
    domain,
    arrayStringConcat(arrayReverse(splitByChar('.', domain)), '.') AS reversed_domain
FROM hostnames
ORDER BY reversed_domain
LIMIT 10;
```

```sql
-- Find all subdomains of example.com using prefix match on reversed domain
SELECT domain
FROM (
    SELECT
        domain,
        arrayStringConcat(arrayReverse(splitByChar('.', domain)), '.') AS reversed_domain
    FROM hostnames
)
WHERE reversed_domain LIKE 'com.example%'
ORDER BY reversed_domain
LIMIT 20;
```

## Combining reverse() with String Operations

`reverse()` composes with other string functions for byte-level tricks:

```sql
-- Extract the file extension by reversing, taking the first segment, then reversing back
SELECT
    file_name,
    reverse(splitByChar('.', reverse(file_name))[1]) AS extension
FROM files
LIMIT 10;
```

A simpler alternative is `replaceRegexpOne(file_name, '.*\\.', '')`, but the reverse-based approach illustrates how the function can substitute for right-anchored operations.

## Performance Notes

Both `reverse()` and `reverseUTF8()` are O(n) in the length of the input string. `reverse()` has lower overhead because it simply reverses raw bytes without any decoding step. `reverseUTF8()` must decode each multi-byte sequence, reverse at the code point level, and re-encode, which is more work per byte but still linear.

For a primary key or ORDER BY column, reversing strings at ingestion time (via a materialized column) is preferable to reversing at query time:

```sql
ALTER TABLE hostnames
    ADD COLUMN reversed_domain String
    MATERIALIZED arrayStringConcat(arrayReverse(splitByChar('.', domain)), '.');
```

## Summary

`reverse(str)` reverses byte order and is correct for ASCII strings and raw binary data. `reverseUTF8(str)` reverses at the Unicode code point level and is correct for any UTF-8 encoded text including multi-byte characters. Use `reverseUTF8()` as the default when your data may contain non-ASCII content. Common applications include palindrome detection, domain name normalization for prefix-based indexing, and right-anchored substring extraction via the reverse-split-reverse pattern.
