# How to Implement Request Routing with Lua in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Lua, Routing

Description: Learn how to implement custom S3 request routing logic in Ceph RGW using Lua scripts to redirect requests, enforce routing policies, and manage multi-zone traffic.

---

## Overview

While Ceph RGW handles multisite routing automatically, Lua scripts enable application-layer routing decisions within a single RGW instance. You can redirect specific users or bucket access patterns to different endpoints, enforce routing based on object metadata, or implement canary deployments at the storage layer.

## Step 1 - Basic Redirect with Lua

```lua
-- bucket_redirect.lua (preRequest context)
-- Redirect access to deprecated bucket names to new names

local BUCKET_REDIRECTS = {
  ["old-logs"] = "new-logs-bucket",
  ["legacy-data"] = "data-archive-v2",
  ["temp-uploads"] = "uploads-2026",
}

local bucket = Request.Bucket.Name or ""

if BUCKET_REDIRECTS[bucket] then
  local new_bucket = BUCKET_REDIRECTS[bucket]
  local object = Request.Object.Name or ""
  local new_url = "//" .. (Request.HTTP.Host or "") ..
                  "/" .. new_bucket
  if object ~= "" then
    new_url = new_url .. "/" .. object
  end

  RGWDebugLog("ROUTE: Redirecting " .. bucket .. " to " .. new_bucket)
  -- Return a 301 redirect response
  Response.HTTP.AddHeader("Location", new_url)
  abort(301, "BucketRedirect", "This bucket has moved to " .. new_bucket)
end
```

## Step 2 - User-Based Routing

```lua
-- user_routing.lua (preRequest context)
-- Route different user types to different endpoint responses

local user = Request.User.Id or "anonymous"
local method = Request.HTTP.Method or ""
local bucket = Request.Bucket.Name or ""

-- Route internal service accounts to a specific behavior
local function is_service_account(uid)
  return string.find(uid, "svc-", 1, true) ~= nil
end

-- Log routing decision for observability
local route = "default"
if is_service_account(user) then
  route = "service"
elseif user == "anonymous" then
  route = "public"
end

RGWDebugLog(string.format(
  "ROUTE: user=%s route=%s method=%s bucket=%s",
  user, route, method, bucket))

-- Add routing information to the response headers (postRequest)
Response.HTTP.AddHeader("X-RGW-Route", route)
```

## Step 3 - Request Forwarding with Headers

```lua
-- forwarding_headers.lua (preRequest context)
-- Add upstream routing headers for load balancers

local bucket = Request.Bucket.Name or ""
local method = Request.HTTP.Method or ""
local user = Request.User.Id or ""

-- Tag write operations for sticky routing to the same zone
if method == "PUT" or method == "POST" or method == "DELETE" then
  Response.HTTP.AddHeader("X-Upstream-Zone", "primary")
  Response.HTTP.AddHeader("X-Write-Op", "true")
else
  -- Read operations can go to any replica zone
  Response.HTTP.AddHeader("X-Upstream-Zone", "any")
  Response.HTTP.AddHeader("X-Write-Op", "false")
end

-- Tag by bucket storage class
local bucket_class = "standard"
if string.find(bucket, "archive-", 1, true) then
  bucket_class = "cold"
elseif string.find(bucket, "hot-", 1, true) then
  bucket_class = "hot"
end
Response.HTTP.AddHeader("X-Bucket-Class", bucket_class)
```

## Step 4 - Canary Routing

```lua
-- canary_routing.lua (preRequest context)
-- Route a percentage of writes to a canary bucket for testing

local method = Request.HTTP.Method or ""
local bucket = Request.Bucket.Name or ""
local CANARY_BUCKET = "canary-test-bucket"
local CANARY_PERCENTAGE = 5  -- 5% of writes

-- Skip routing for the canary bucket itself
if bucket == CANARY_BUCKET then return end

if method == "PUT" then
  -- Simple percentage check using modulo on timestamp
  local timestamp = os.time()
  local in_canary = (timestamp % 100) < CANARY_PERCENTAGE

  if in_canary then
    local object = Request.Object.Name or ""
    RGWDebugLog(string.format(
      "CANARY: Tagging write for canary routing user=%s obj=%s",
      (Request.User.Id or ""), object))
    -- Add header for a smart proxy to duplicate the write
    Response.HTTP.AddHeader("X-Canary-Write", "true")
    Response.HTTP.AddHeader("X-Canary-Bucket", CANARY_BUCKET)
  end
end
```

## Step 5 - Geo-Based Routing Tags

```lua
-- geo_routing.lua (preRequest context)
-- Tag requests with geo-routing hints based on client IP

local function get_region_from_ip(ip)
  -- Simplified IP-to-region mapping
  -- In production, use a GeoIP database via LuaSocket or an external lookup
  local region_map = {
    ["10.0.1."]  = "us-east",
    ["10.0.2."]  = "us-west",
    ["10.0.3."]  = "eu-west",
    ["10.0.4."]  = "ap-south",
  }
  for prefix, region in pairs(region_map) do
    if string.find(ip, prefix, 1, true) then
      return region
    end
  end
  return "unknown"
end

local client_ip = Request.HTTP.Environment["REMOTE_ADDR"] or ""
local region = get_region_from_ip(client_ip)

Response.HTTP.AddHeader("X-Client-Region", region)
RGWDebugLog("GEO: ip=" .. client_ip .. " region=" .. region)
```

## Step 6 - Deploy and Test Routing Logic

```bash
# Deploy the bucket redirect script
radosgw-admin script put \
  --infile=bucket_redirect.lua \
  --context=preRequest

# Test redirect
curl -v "http://rgw.example.com:7480/old-logs/" \
  -H "Authorization: AWS ${KEY}:${SIG}" 2>&1 | grep "Location:"

# Test user routing headers
aws --endpoint-url http://rgw.example.com:7480 s3 ls \
  --profile svc-account \
  --debug 2>&1 | grep "X-RGW-Route"

# Check routing decisions in logs
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=50 \
  | grep "ROUTE:\|CANARY:\|GEO:"
```

## Summary

Lua request routing scripts in Ceph RGW enable intelligent routing decisions at the storage layer. Bucket redirect rules handle legacy bucket names, user-based routing tags requests for downstream processing, and geo-routing hints aid CDN and load balancer configuration. These scripts are deployed without downtime using `radosgw-admin script put` and can be updated or removed instantly, making them ideal for operational routing policies.
