# How to Use the urlencode and urldecode Functions in OpenTofu - Functions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Urlencode, Urldecode, String Functions, HCL, Infrastructure as Code, DevOps

Description: Learn how to use the urlencode and urldecode functions in OpenTofu to encode and decode URL-safe strings for use in HTTP requests and resource configurations.

---

`urlencode()` encodes a string so it can be safely included in a URL by replacing special characters with percent-encoded sequences. `urldecode()` is the reverse, converting percent-encoded sequences back to their original characters.

---

## Syntax

```hcl
urlencode(string)
urldecode(string)
```

---

## Basic Examples

```hcl
locals {
  # urlencode: make a string safe for URLs
  example1 = urlencode("hello world")           # "hello+world"
  example2 = urlencode("key=value&other=data")  # "key%3Dvalue%26other%3Ddata"
  example3 = urlencode("user@example.com")      # "user%40example.com"
  example4 = urlencode("/path/to/resource")     # "%2Fpath%2Fto%2Fresource"

  # urldecode: reverse the encoding
  example5 = urldecode("hello+world")           # "hello world"
  example6 = urldecode("user%40example.com")    # "user@example.com"
}
```

---

## Building Query String Parameters

```hcl
variable "search_query" {
  type    = string
  default = "infrastructure as code"
}

variable "filter" {
  type    = string
  default = "type=aws&status=active"
}

locals {
  # Build a properly encoded URL with query parameters
  api_url = "https://api.example.com/search?q=${urlencode(var.search_query)}&${var.filter}"
}

data "http" "search_results" {
  url = local.api_url
}
```

---

## S3 Object Keys with Special Characters

```hcl
variable "object_name" {
  type    = string
  default = "reports/2024 Q1/summary report.pdf"
}

resource "aws_s3_object" "report" {
  bucket = aws_s3_bucket.reports.bucket

  # S3 URLs need special chars encoded when used in HTTP requests
  # But the actual key should be the raw string
  key    = var.object_name

  # For the URL reference, encode the key
  source = "${path.module}/files/${var.object_name}"
}

output "object_url" {
  # Build the proper S3 URL with encoded key
  value = "https://${aws_s3_bucket.reports.bucket}.s3.amazonaws.com/${urlencode(var.object_name)}"
}
```

---

## API Gateway Base Path

```hcl
variable "api_path" {
  type    = string
  default = "/users/{userId}/settings"
}

resource "aws_api_gateway_resource" "path" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  parent_id   = aws_api_gateway_rest_api.main.root_resource_id

  # API Gateway paths should use the raw path, not encoded
  path_part = var.api_path
}

# But for the complete URL reference, encode special chars

locals {
  full_api_url = "https://${aws_api_gateway_rest_api.main.id}.execute-api.${data.aws_region.current.name}.amazonaws.com/${var.stage}${var.api_path}"
}
```

---

## Decoding Parameters from SSM or Secrets

```hcl
data "aws_ssm_parameter" "encoded_config" {
  name = "/production/api/callback-url"
}

locals {
  # If the stored value is URL-encoded, decode it
  decoded_url = urldecode(data.aws_ssm_parameter.encoded_config.value)
}
```

---

## urlencode vs Base64

```hcl
locals {
  data = "binary or special content"

  # urlencode: for URL query parameters and path components
  url_safe = urlencode(local.data)

  # base64encode: for binary data in HTTP bodies or config
  b64 = base64encode(local.data)
}
```

Use `urlencode()` for URL parameters and path components. Use `base64encode()` for binary data transmission.

---

## Summary

`urlencode(string)` converts special characters to percent-encoded sequences (spaces become `+`, `@` becomes `%40`, etc.), making strings safe for URL inclusion. `urldecode(string)` reverses this. Use `urlencode()` when constructing URLs with query parameters from variable data, building S3 object URL references, or encoding values for use in webhook endpoints. Note that the actual AWS resource names and keys should use the raw values - encoding is only needed when constructing HTTP URLs.
