# How to Add Custom Headers with Lua in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Lua, Header

Description: Learn how to add, modify, and remove HTTP response headers in Ceph RGW using Lua postRequest scripts for CORS, security, and observability.

---

## Overview

Ceph RGW Lua scripts running in the postRequest context can add and modify HTTP response headers before they are sent to the client. This enables adding security headers (HSTS, CSP), custom CORS headers, cache control directives, observability metadata, and routing information without modifying RGW source code.

## Step 1 - Basic Header Addition

```lua
-- add_basic_headers.lua (postRequest context)
-- Add standard security and server identification headers

-- Server identification
Response.HTTP.AddHeader("X-Storage-System", "Ceph-RGW")
Response.HTTP.AddHeader("X-Cluster-Region", "us-east-1")

-- Add a request trace ID
local bucket = Request.Bucket.Name or "global"
local method = Request.HTTP.Method
local timestamp = tostring(os.time())
Response.HTTP.AddHeader("X-Request-ID",
  method .. "-" .. bucket .. "-" .. timestamp)
```

## Step 2 - Add Security Headers

```lua
-- security_headers.lua (postRequest context)
-- Add standard web security headers for browser clients

-- Strict Transport Security (1 year)
Response.HTTP.AddHeader("Strict-Transport-Security",
  "max-age=31536000; includeSubDomains")

-- Prevent MIME sniffing
Response.HTTP.AddHeader("X-Content-Type-Options", "nosniff")

-- Clickjacking protection
Response.HTTP.AddHeader("X-Frame-Options", "DENY")

-- Content Security Policy for the S3 Console
Response.HTTP.AddHeader("Content-Security-Policy",
  "default-src 'self'; img-src 'self' data:")

-- Referrer policy
Response.HTTP.AddHeader("Referrer-Policy", "strict-origin-when-cross-origin")

RGWDebugLog("Security headers added for " .. (Request.HTTP.Method or "?"))
```

## Step 3 - Dynamic CORS Headers

```lua
-- dynamic_cors.lua (postRequest context)
-- Add CORS headers based on the request origin

local ALLOWED_ORIGINS = {
  ["https://app.example.com"] = true,
  ["https://admin.example.com"] = true,
  ["http://localhost:3000"] = true,
}

local origin = Request.HTTP.Header["Origin"] or ""

if ALLOWED_ORIGINS[origin] then
  Response.HTTP.AddHeader("Access-Control-Allow-Origin", origin)
  Response.HTTP.AddHeader("Access-Control-Allow-Credentials", "true")
  Response.HTTP.AddHeader("Access-Control-Allow-Methods",
    "GET, PUT, POST, DELETE, HEAD, OPTIONS")
  Response.HTTP.AddHeader("Access-Control-Allow-Headers",
    "Content-Type, Authorization, X-Amz-Date, X-Api-Key, X-Amz-Security-Token")
  Response.HTTP.AddHeader("Access-Control-Max-Age", "7200")
  Response.HTTP.AddHeader("Vary", "Origin")
  RGWDebugLog("CORS headers added for origin: " .. origin)
elseif origin ~= "" then
  RGWDebugLog("CORS: Rejected unknown origin: " .. origin)
end
```

## Step 4 - Cache Control Headers

```lua
-- cache_control.lua (postRequest context)
-- Set cache control based on bucket and object type

local bucket = Request.Bucket.Name or ""
local object = Request.Object.Name or ""
local method = Request.HTTP.Method

-- Only add cache headers for GET requests
if method ~= "GET" then return end

-- Determine cache policy based on bucket type
local cache_ttl = 0

if string.find(bucket, "static-", 1, true) then
  -- Static assets - cache for 1 year
  cache_ttl = 31536000
elseif string.find(bucket, "media-", 1, true) then
  -- Media files - cache for 1 week
  cache_ttl = 604800
elseif string.find(bucket, "api-", 1, true) then
  -- API responses - no caching
  cache_ttl = 0
end

if cache_ttl > 0 then
  Response.HTTP.AddHeader("Cache-Control",
    "public, max-age=" .. tostring(cache_ttl))
  Response.HTTP.AddHeader("CDN-Cache-Control",
    "public, max-age=" .. tostring(cache_ttl))
else
  Response.HTTP.AddHeader("Cache-Control",
    "no-store, no-cache, must-revalidate")
end
```

## Step 5 - Observability and Tracing Headers

```lua
-- tracing_headers.lua (postRequest context)
-- Add distributed tracing headers for observability pipelines

local method = Request.HTTP.Method
local bucket = Request.Bucket.Name or ""
local object = Request.Object.Name or ""
local user = Request.User.Id or "anonymous"

-- Propagate or generate a trace ID
local trace_id = Request.HTTP.Header["X-Trace-ID"]
if not trace_id or trace_id == "" then
  -- Generate a simple trace ID from timestamp + user hash
  trace_id = string.format("%x-%x",
    os.time(),
    #user * 31 + #bucket * 17)
end

Response.HTTP.AddHeader("X-Trace-ID", trace_id)
Response.HTTP.AddHeader("X-RGW-User", user)
Response.HTTP.AddHeader("X-RGW-Operation", method)
Response.HTTP.AddHeader("X-RGW-Bucket", bucket)

if object ~= "" then
  Response.HTTP.AddHeader("X-RGW-Object", object)
end
```

## Step 6 - Deploy and Validate

```bash
# Upload the postRequest script
radosgw-admin script put \
  --infile=security_headers.lua \
  --context=postRequest

# Test that headers appear in responses
curl -v http://rgw.example.com:7480/mybucket/ \
  -H "Authorization: AWS ${ACCESS_KEY}:${SIGNATURE}" \
  2>&1 | grep -E "X-|Strict-|Access-Control|Cache-Control"

# Expected output includes the custom headers
# < X-Storage-System: Ceph-RGW
# < Strict-Transport-Security: max-age=31536000; includeSubDomains
# < X-Content-Type-Options: nosniff

# Check logs for any script errors
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=20 | grep -i "lua\|header"
```

## Summary

Lua postRequest scripts in Ceph RGW provide full control over HTTP response headers, enabling security hardening (HSTS, CSP, X-Frame-Options), dynamic CORS with per-origin allowlists, cache control policies keyed by bucket type, and distributed tracing header injection. These scripts require no RGW recompilation and can be updated live using `radosgw-admin script put`, making them ideal for operational policies that change frequently.
