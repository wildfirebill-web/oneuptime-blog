# How to Create UDFs in ClickHouse with Executable Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UDF, Executable Script, Python, Shell Script

Description: Learn how to create executable UDFs in ClickHouse that run external scripts, enabling custom logic in Python, Bash, or any other language.

---

ClickHouse supports executable UDFs - functions backed by external scripts or binaries. When a query calls such a UDF, ClickHouse spawns the script as a subprocess, pipes rows to its stdin, and reads results from stdout. This enables custom logic in any language (Python, Bash, Ruby, etc.).

## How Executable UDFs Work

1. ClickHouse serializes the input columns and pipes them to the script's stdin.
2. The script reads rows line by line, processes them, and prints results to stdout.
3. ClickHouse reads the output and integrates it back into the query result.

## Defining an Executable UDF in XML

Create a file at `/etc/clickhouse-server/user_defined/my_func.xml`:

```text
<functions>
    <function>
        <name>sentiment</name>
        <type>executable</type>
        <return_type>String</return_type>
        <argument>
            <type>String</type>
            <name>text</name>
        </argument>
        <format>TabSeparated</format>
        <command>python3 /var/lib/clickhouse/user_scripts/sentiment.py</command>
        <send_chunk_header>false</send_chunk_header>
    </function>
</functions>
```

## Writing the Python Script

```bash
# /var/lib/clickhouse/user_scripts/sentiment.py
```

```text
#!/usr/bin/env python3
import sys

for line in sys.stdin:
    text = line.strip()
    if 'great' in text or 'love' in text:
        print('positive')
    elif 'bad' in text or 'hate' in text:
        print('negative')
    else:
        print('neutral')
```

Make it executable:

```bash
chmod +x /var/lib/clickhouse/user_scripts/sentiment.py
```

## Calling the UDF

```sql
SELECT
    review_id,
    sentiment(review_text) AS sentiment
FROM customer_reviews
LIMIT 10;
```

## Using Chunk Headers for Batch Processing

Enable `send_chunk_header` to send a row count before each batch, which helps scripts preallocate memory:

```text
<send_chunk_header>true</send_chunk_header>
```

The script must read the count from the first line of each chunk.

## Environment Variables

Pass environment variables to the script:

```text
<environment_variables>
    <MY_MODEL_PATH>/var/lib/clickhouse/models/classifier.pkl</MY_MODEL_PATH>
</environment_variables>
```

## Checking Registered Executable UDFs

```sql
SELECT name, origin
FROM system.functions
WHERE origin = 'ExecutableUserDefined';
```

## Performance Considerations

- Executable UDFs have subprocess startup overhead; keep scripts persistent using a loop
- Use batch processing (chunk headers) to amortize the per-row cost
- For high-throughput scenarios, consider SQL UDFs or built-in functions instead
- The script must flush stdout after each output row or batch

## Security

- Scripts run as the `clickhouse` OS user
- Restrict script permissions and avoid accepting untrusted input without sanitization

## Summary

Executable UDFs extend ClickHouse with arbitrary logic in external scripts. By defining the function in an XML config file and writing a stdin/stdout script, you can invoke Python NLP models, custom encoders, or any external tool directly from SQL queries, making ClickHouse extensible beyond its built-in function set.
