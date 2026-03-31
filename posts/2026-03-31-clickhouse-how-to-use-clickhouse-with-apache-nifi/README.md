# How to Use ClickHouse with Apache NiFi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apache NiFi, Data Flow, ETL, Ingestion, Integration

Description: Build automated data flow pipelines with Apache NiFi that ingest, transform, and load data into ClickHouse using HTTP processors and custom scripting.

---

Apache NiFi is a data flow automation platform with a drag-and-drop interface. Its built-in processors handle HTTP requests, file I/O, and scripting, making it straightforward to build pipelines that stream data into ClickHouse.

## Architecture Overview

NiFi communicates with ClickHouse via its HTTP interface. The typical flow is:

```text
Source -> [Fetch/Generate Processor] -> [Transform Processor] -> [InvokeHTTP Processor] -> ClickHouse HTTP API
```

## Setting Up the InvokeHTTP Processor

ClickHouse accepts inserts via HTTP POST. Configure the `InvokeHTTP` processor:

- **HTTP Method**: POST
- **Remote URL**: `http://your-clickhouse-host:8123/?query=INSERT+INTO+default.events+FORMAT+JSONEachRow`
- **Content-Type**: `application/json`
- Headers if authentication is needed:
  - `X-ClickHouse-User: default`
  - `X-ClickHouse-Key: your_password`

## Generating Test Data

Use the `GenerateFlowFile` processor to create test records:

```text
Processor: GenerateFlowFile
- File Size: 0B
- Custom Text:
{"event_id":"${UUID()}","user_id":"user_${random():mod(100)}","event_type":"click","created_at":"${now():format('yyyy-MM-dd HH:mm:ss')}"}
```

## Transforming Data with ExecuteScript

Use the `ExecuteScript` processor with Groovy to reshape data:

```groovy
import org.apache.commons.io.IOUtils
import java.nio.charset.StandardCharsets
import groovy.json.JsonSlurper
import groovy.json.JsonOutput

def flowFile = session.get()
if (!flowFile) return

flowFile = session.write(flowFile) { inputStream, outputStream ->
    def content = IOUtils.toString(inputStream, StandardCharsets.UTF_8)
    def slurper = new JsonSlurper()
    def record = slurper.parseText(content)
    record['processed_at'] = new Date().format("yyyy-MM-dd HH:mm:ss")
    outputStream.write(JsonOutput.toJson(record).getBytes(StandardCharsets.UTF_8))
}

session.transfer(flowFile, REL_SUCCESS)
```

## Batch Insert with MergeContent

For better throughput, batch multiple FlowFiles before sending to ClickHouse:

1. Add a `MergeContent` processor before `InvokeHTTP`.
2. Set Merge Strategy to "Bin-Packing Algorithm."
3. Set Minimum Number of Entries to 100 or Minimum Group Size to 1 MB.
4. Set Delimiter Strategy to "Text" with newline as the separator.

This produces line-delimited JSON (JSONEachRow format) for bulk insert.

## Querying ClickHouse from NiFi

Use `InvokeHTTP` with a GET request to run a SELECT query:

- URL: `http://your-clickhouse-host:8123/?query=SELECT+count()+FROM+default.events+FORMAT+JSON`
- Method: GET

The response JSON can then be routed to downstream processors for alerting or logging.

## Summary

Apache NiFi's HTTP processors make it easy to stream data into ClickHouse without custom code. For simple inserts, `InvokeHTTP` is sufficient. For richer transformations, use `ExecuteScript`. Batching with `MergeContent` dramatically improves insert throughput by reducing the number of HTTP round trips.
