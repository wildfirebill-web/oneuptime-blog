# How to Enable Debug Logging with TF_LOG in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Debugging, TF_LOG, Logging, Troubleshooting, Infrastructure as Code

Description: Learn how to use the TF_LOG environment variable to enable verbose debug logging in OpenTofu, helping diagnose provider errors and unexpected behavior.

## Introduction

When `tofu plan` or `tofu apply` fails with a cryptic error, `TF_LOG` is your first tool. It activates OpenTofu's internal logging subsystem, writing detailed trace information about provider calls, state operations, and graph execution to stderr.

## Log Levels

OpenTofu supports five log levels, from least to most verbose:

| Level | Description |
|---|---|
| `ERROR` | Only fatal errors |
| `WARN` | Warnings and errors |
| `INFO` | General operational messages |
| `DEBUG` | Detailed developer-level messages |
| `TRACE` | Everything including raw API calls |

## Enabling Debug Logging

```bash
# Enable DEBUG logging for a single command
TF_LOG=DEBUG tofu plan

# Enable the most verbose TRACE logging
TF_LOG=TRACE tofu apply

# Or export for the session
export TF_LOG=DEBUG
tofu init
tofu plan
```

## Example: Diagnosing a Provider Error

Without logging:

```
Error: InvalidClientTokenId: The security token included in the request is invalid.
```

With `TF_LOG=DEBUG`:

```
2026-03-20T10:15:30.123Z [DEBUG] provider.terraform-provider-aws_v5.40.0:
  Request: GET https://ec2.us-east-1.amazonaws.com/
  Headers: Authorization: AWS4-HMAC-SHA256 Credential=AKIA...
  Response: 401 Unauthorized
  Body: <InvalidClientTokenId>The security token included in the request is invalid.</InvalidClientTokenId>
```

The debug output reveals the exact endpoint, request, and raw response, making root-cause analysis much faster.

## Filtering Log Output

The full TRACE output is very noisy. Pipe through grep to focus:

```bash
# Only show lines related to the AWS provider
TF_LOG=TRACE tofu plan 2>&1 | grep "provider.terraform-provider-aws"

# Only show error lines
TF_LOG=DEBUG tofu apply 2>&1 | grep -i "error\|fatal"

# Show HTTP request/response lines only
TF_LOG=TRACE tofu plan 2>&1 | grep -E "Request:|Response:|Body:"
```

## Saving Logs to a File

```bash
# Save debug logs to a file while also showing normal output
TF_LOG=DEBUG TF_LOG_PATH=./debug.log tofu plan

# View the log file
less ./debug.log
```

## Separating Core and Provider Logs

OpenTofu supports separate log levels for its core engine vs providers:

```bash
# Core at INFO, providers at DEBUG
export TF_LOG_CORE=INFO
export TF_LOG_PROVIDER=DEBUG
tofu plan
```

## Cleaning Up

Always unset `TF_LOG` after debugging to avoid polluting normal output:

```bash
unset TF_LOG
unset TF_LOG_PATH
unset TF_LOG_CORE
unset TF_LOG_PROVIDER
```

## Conclusion

`TF_LOG` is the fastest way to understand what OpenTofu is doing under the hood. Start with `DEBUG` for most issues and escalate to `TRACE` when you need raw HTTP traffic. Save logs with `TF_LOG_PATH` so you can analyze them after a long apply, and use `TF_LOG_CORE`/`TF_LOG_PROVIDER` to focus on the relevant subsystem.
