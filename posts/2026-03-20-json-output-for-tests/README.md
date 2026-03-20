# JSON Output for OpenTofu Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, JSON, CI/CD, Infrastructure as Code

Description: Learn how to use JSON-formatted test output in OpenTofu to integrate test results with CI/CD pipelines and reporting tools.

## Why JSON Output for Tests?

OpenTofu's native test runner produces human-readable output by default. JSON output enables:

- Machine-readable test results for CI/CD pipeline parsing
- Integration with test reporting dashboards (JUnit, Allure, etc.)
- Automated pass/fail gates in deployment workflows
- Log aggregation and structured querying

## Enabling JSON Output

Run `tofu test` with the `-json` flag:

```bash
tofu test -json
```

This outputs each event as a newline-delimited JSON object (NDJSON).

## Sample JSON Output

```json
{"@level":"info","@message":"Starting test run","@module":"tofu.test","@timestamp":"2026-03-20T10:00:00.000Z","type":"test_run_start"}
{"@level":"info","@message":"Run \"create_vpc\"","@module":"tofu.test","@timestamp":"2026-03-20T10:00:01.000Z","type":"test_run","run":"create_vpc","file":"tests/vpc.tftest.hcl"}
{"@level":"info","@message":"Pass","@module":"tofu.test","@timestamp":"2026-03-20T10:00:15.000Z","type":"test_run_result","run":"create_vpc","result":"pass"}
{"@level":"info","@message":"1 passed, 0 failed","@module":"tofu.test","@timestamp":"2026-03-20T10:00:15.000Z","type":"test_summary","passed":1,"failed":0,"errored":0}
```

## Parsing JSON Output in Shell

```bash
tofu test -json | jq 'select(.type == "test_run_result") | {run: .run, result: .result}'
```

Output:
```json
{"run": "create_vpc", "result": "pass"}
{"run": "verify_outputs", "result": "pass"}
```

## Converting to JUnit XML for CI/CD

Many CI systems (Jenkins, GitHub Actions) accept JUnit XML. Convert with a small script:

```bash
#!/bin/bash
tofu test -json > test-results.ndjson

python3 - <<'EOF'
import json, sys
from xml.etree.ElementTree import Element, SubElement, tostring
from xml.dom import minidom

results = []
with open("test-results.ndjson") as f:
    for line in f:
        obj = json.loads(line)
        if obj.get("type") == "test_run_result":
            results.append(obj)

suite = Element("testsuite", name="OpenTofu", tests=str(len(results)))
for r in results:
    tc = SubElement(suite, "testcase", name=r["run"])
    if r["result"] == "fail":
        SubElement(tc, "failure", message=r.get("@message", ""))

xml_str = minidom.parseString(tostring(suite)).toprettyxml()
with open("test-results.xml", "w") as f:
    f.write(xml_str)
print("Wrote test-results.xml")
EOF
```

## GitHub Actions Integration

```yaml
- name: Run OpenTofu Tests
  run: tofu test -json > test-results.ndjson

- name: Parse and Report
  run: |
    FAILED=$(cat test-results.ndjson | jq '[select(.type=="test_run_result" and .result=="fail")] | length')
    echo "Failed tests: $FAILED"
    if [ "$FAILED" -gt 0 ]; then exit 1; fi

- name: Upload Test Results
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results.ndjson
```

## Filtering Specific Event Types

The JSON stream contains multiple event types:

| Event Type | Description |
|---|---|
| `test_run_start` | Test suite started |
| `test_run` | Individual test run started |
| `test_run_result` | Individual test result (pass/fail) |
| `test_summary` | Final summary of all tests |
| `log` | Log output from the test |

```bash
# Show only failures

tofu test -json | jq 'select(.type == "test_run_result" and .result == "fail")'

# Show summary
tofu test -json | jq 'select(.type == "test_summary")'
```

## Best Practices

1. **Always capture JSON output to a file** in CI/CD for post-run analysis
2. **Parse the summary event** for a quick pass/fail decision
3. **Include test result artifacts** in pipeline runs for debugging
4. **Set up dashboards** to track test trends over time
5. **Combine with `-verbose`** for detailed logs alongside structured output

## Conclusion

JSON output mode in `tofu test` transforms test results into machine-readable data that integrates cleanly with CI/CD pipelines and reporting tools. By parsing the NDJSON stream, you can build sophisticated test gates and track infrastructure test trends over time.
