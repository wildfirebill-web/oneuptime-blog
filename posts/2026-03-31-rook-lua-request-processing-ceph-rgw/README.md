# How to Write Request Processing Scripts with Lua in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Lua, Request Processing

Description: Learn how to write Lua scripts that process and inspect S3 API requests in Ceph RGW, enabling dynamic request validation and transformation.

---

## Overview

Lua request processing scripts in Ceph RGW run at two stages: before the request is handled (preRequest) and after the response is ready (postRequest). These hooks let you validate request parameters, block unauthorized operations, add response headers, and collect telemetry without modifying RGW source code.

## Step 1 - Request Inspection Basics

```lua
-- inspect_request.lua
-- Comprehensive request inspection example

local function log_request()
  local info = {
    method  = Request.HTTP.Method,
    bucket  = Request.Bucket.Name or "none",
    object  = Request.Object.Name or "none",
    user    = Request.User.Id or "anonymous",
    host    = Request.HTTP.Host,
    uri     = Request.HTTP.URI,
  }

  local log_line = "REQUEST"
  for k, v in pairs(info) do
    log_line = log_line .. " " .. k .. "=" .. tostring(v)
  end

  RGWDebugLog(log_line)
end

log_request()
```

## Step 2 - Validate Request Parameters

```lua
-- validate_upload.lua
-- Block uploads larger than 1 GiB from non-admin users

local MAX_SIZE_BYTES = 1073741824  -- 1 GiB
local method = Request.HTTP.Method
local user = Request.User.Id or ""
local content_length = Request.HTTP.ContentLength or 0

-- Check if this is an upload operation
if method == "PUT" or method == "POST" then
  -- Allow admin users to bypass the limit
  if user ~= "admin" then
    if content_length > MAX_SIZE_BYTES then
      RGWDebugLog("BLOCKED: Upload too large for user=" .. user ..
                  " size=" .. tostring(content_length))
      -- Abort the request with a 403 response
      abort(403, "EntityTooLarge",
            "Upload size exceeds the maximum allowed for this account.")
    end
  end
end
```

Upload and test:

```bash
radosgw-admin script put --infile=validate_upload.lua --context=preRequest

# Test: try to upload a small file (should succeed)
aws --endpoint-url http://rgw.example.com:7480 \
  s3 cp /etc/hosts s3://mybucket/hosts.txt

# Check logs
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=20 | grep BLOCKED
```

## Step 3 - Enforce Naming Conventions

```lua
-- enforce_naming.lua
-- Require object keys to start with a date prefix (YYYY-MM-DD/)

local method = Request.HTTP.Method
local object_key = Request.Object.Name or ""

if method == "PUT" and object_key ~= "" then
  -- Check for date prefix pattern YYYY-MM-DD/
  local date_prefix = string.match(object_key, "^%d%d%d%d%-%d%d%-%d%d/")
  if not date_prefix then
    RGWDebugLog("BLOCKED: Object key missing date prefix: " .. object_key)
    abort(400, "InvalidKeyPrefixError",
          "Object key must start with a date prefix (YYYY-MM-DD/).")
  end
end
```

## Step 4 - Inspect Request Metadata

```lua
-- metadata_inspector.lua
-- Log all custom metadata on PUT operations

local method = Request.HTTP.Method

if method == "PUT" then
  local metadata_found = false
  for k, v in pairs(Request.HTTP.Metadata) do
    RGWDebugLog("METADATA " .. k .. "=" .. v)
    metadata_found = true
  end

  if not metadata_found then
    RGWDebugLog("PUT with no custom metadata: key=" ..
                (Request.Object.Name or ""))
  end

  -- Require a mandatory metadata field
  local owner = Request.HTTP.Metadata["x-amz-meta-owner"]
  if not owner or owner == "" then
    abort(400, "MissingMetadataError",
          "Objects must include the x-amz-meta-owner metadata.")
  end
end
```

## Step 5 - Post-Request Response Modification

```lua
-- add_response_headers.lua
-- Add custom headers to every response (postRequest context)

-- Add a server identifier header
Response.HTTP.AddHeader("X-Storage-Backend", "Ceph-RGW")
Response.HTTP.AddHeader("X-Cluster-ID", "prod-ceph-01")

-- Add CORS headers for browser applications
if Request.HTTP.Method == "OPTIONS" then
  Response.HTTP.AddHeader("Access-Control-Allow-Origin", "*")
  Response.HTTP.AddHeader("Access-Control-Allow-Methods",
                          "GET, PUT, DELETE, HEAD, POST")
  Response.HTTP.AddHeader("Access-Control-Max-Age", "3600")
end
```

```bash
# Upload the postRequest script
radosgw-admin script put \
  --infile=add_response_headers.lua \
  --context=postRequest
```

## Step 6 - Error Handling in Lua Scripts

```lua
-- safe_script.lua
-- Best practice: wrap all logic in pcall for safety

local ok, err = pcall(function()
  local method = Request.HTTP.Method
  local bucket = Request.Bucket.Name

  if not bucket then return end

  -- Your processing logic here
  if method == "DELETE" and bucket == "protected-bucket" then
    abort(403, "BucketProtected",
          "This bucket is protected from deletion.")
  end
end)

if not ok then
  RGWDebugLog("Lua script error: " .. tostring(err))
  -- Do NOT abort here - log and continue to avoid blocking valid requests
end
```

## Summary

Lua request processing scripts in Ceph RGW provide a powerful, zero-overhead hook into the S3 request lifecycle. Using preRequest scripts you can validate object sizes, enforce naming conventions, check required metadata fields, and block unauthorized operations via the `abort()` function. postRequest scripts modify outgoing response headers for CORS, observability, and custom branding. Wrapping logic in `pcall` prevents script errors from impacting normal request processing.
