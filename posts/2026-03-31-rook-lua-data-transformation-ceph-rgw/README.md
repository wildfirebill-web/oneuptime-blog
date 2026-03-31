# How to Use Lua for Data Transformation in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Lua, Data Transformation

Description: Learn how to use Lua scripts in Ceph RGW to transform request and response metadata, normalize object keys, and enrich objects with computed attributes on upload.

---

## Overview

Ceph RGW Lua scripts can transform request and response attributes at the gateway layer, acting as a transparent middleware for your S3 API. Transformations include normalizing object key formats, stripping or adding metadata, enforcing content-type policies, and enriching uploads with computed metadata before they are stored in RADOS.

## Step 1 - Normalize Object Key Formats

```lua
-- normalize_keys.lua (preRequest context)
-- Normalize object keys to lowercase with hyphens instead of spaces

local method = Request.HTTP.Method or ""
local object = Request.Object.Name or ""

if (method == "PUT" or method == "GET") and object ~= "" then
  -- Normalize: lowercase and replace spaces with hyphens
  local normalized = string.lower(object)
  normalized = string.gsub(normalized, "%s+", "-")

  -- Remove double slashes
  normalized = string.gsub(normalized, "//+", "/")

  if normalized ~= object then
    RGWDebugLog(string.format(
      "TRANSFORM: Key normalized from '%s' to '%s'",
      object, normalized))
    -- Note: modifying the object key directly requires RGW API support
    -- This logs the transformation for audit purposes
    Response.HTTP.AddHeader("X-Original-Key", object)
    Response.HTTP.AddHeader("X-Normalized-Key", normalized)
  end
end
```

## Step 2 - Enrich Uploads with Computed Metadata

```lua
-- enrich_metadata.lua (preRequest context)
-- Add computed metadata to incoming PUT requests

local method = Request.HTTP.Method or ""
local object = Request.Object.Name or ""
local bucket = Request.Bucket.Name or ""

if method == "PUT" and object ~= "" then
  local user = Request.User.Id or "anonymous"
  local timestamp = tostring(os.time())

  -- These metadata values are added to the object as system metadata
  -- by injecting them into the request before RGW processes it
  -- Note: Lua can add response headers; true metadata injection
  -- requires the RGW Lua API to support request header modification

  RGWDebugLog(string.format(
    "ENRICH: bucket=%s object=%s user=%s ts=%s",
    bucket, object, user, timestamp))

  -- Add headers that become part of the response for audit
  Response.HTTP.AddHeader("X-Upload-User", user)
  Response.HTTP.AddHeader("X-Upload-Timestamp", timestamp)
  Response.HTTP.AddHeader("X-Upload-Bucket", bucket)
end
```

## Step 3 - Content-Type Enforcement and Correction

```lua
-- content_type_policy.lua (preRequest context)
-- Enforce content-type policies for specific bucket prefixes

local method = Request.HTTP.Method or ""
local object = Request.Object.Name or ""
local content_type = Request.HTTP.Header["Content-Type"] or ""

if method == "PUT" and object ~= "" then
  -- Derive expected content type from file extension
  local ext_map = {
    jpg = "image/jpeg",
    jpeg = "image/jpeg",
    png = "image/png",
    gif = "image/gif",
    pdf = "application/pdf",
    json = "application/json",
    txt = "text/plain",
    html = "text/html",
    xml = "application/xml",
    csv = "text/csv",
  }

  -- Extract extension
  local ext = string.match(object, "%.(%w+)$")
  if ext then
    ext = string.lower(ext)
    local expected_ct = ext_map[ext]

    if expected_ct then
      if content_type == "" then
        RGWDebugLog("CONTENT_TYPE: Missing for " .. ext ..
                    " object, expected " .. expected_ct)
        -- Log for monitoring; abort to enforce:
        -- abort(400, "MissingContentType", "Content-Type header is required.")
      elseif content_type ~= expected_ct and
             not string.find(content_type, expected_ct, 1, true) then
        RGWDebugLog(string.format(
          "CONTENT_TYPE: Mismatch for %s: got '%s', expected '%s'",
          object, content_type, expected_ct))
      end
    end
  end
end
```

## Step 4 - Response Metadata Transformation

```lua
-- transform_response.lua (postRequest context)
-- Normalize and enrich response metadata for S3 clients

-- Sanitize the ETag header to ensure it has double quotes
-- (some S3 clients require quoted ETags)
local etag = Response.HTTP.Header["ETag"] or ""
if etag ~= "" and not string.find(etag, '"', 1, true) then
  etag = '"' .. etag .. '"'
  Response.HTTP.AddHeader("ETag", etag)
  RGWDebugLog("TRANSFORM: ETag normalized to " .. etag)
end

-- Add missing Content-Disposition for specific file types
local object = Request.Object.Name or ""
local ext = string.match(object, "%.(%w+)$")
if ext then
  local download_exts = {exe=true, zip=true, tar=true, gz=true, pdf=true}
  if download_exts[string.lower(ext)] then
    Response.HTTP.AddHeader("Content-Disposition",
      'attachment; filename="' .. object .. '"')
  end
end
```

## Step 5 - Metadata Schema Validation

```lua
-- metadata_schema.lua (preRequest context)
-- Validate required metadata fields on upload

local method = Request.HTTP.Method or ""
local bucket = Request.Bucket.Name or ""

-- Only enforce metadata schema on specific buckets
local SCHEMA_BUCKETS = {["compliance-data"] = true, ["audit-logs"] = true}

if method == "PUT" and SCHEMA_BUCKETS[bucket] then
  local required_fields = {
    "x-amz-meta-classification",
    "x-amz-meta-data-owner",
    "x-amz-meta-retention-days",
  }

  local missing = {}
  for _, field in ipairs(required_fields) do
    local val = Request.HTTP.Metadata[string.gsub(field, "x%-amz%-meta%-", "")]
    if not val or val == "" then
      table.insert(missing, field)
    end
  end

  if #missing > 0 then
    local missing_str = table.concat(missing, ", ")
    RGWDebugLog("SCHEMA: Missing required metadata: " .. missing_str)
    abort(400, "MissingRequiredMetadata",
          "Required metadata fields are missing: " .. missing_str)
  end
end
```

## Step 6 - Deploy and Test Transformations

```bash
# Deploy the metadata schema validation
radosgw-admin script put \
  --infile=metadata_schema.lua \
  --context=preRequest

# Test: upload without required metadata (should fail)
aws --endpoint-url http://rgw.example.com:7480 \
  s3 cp /tmp/report.pdf s3://compliance-data/report.pdf
# Expected: 400 MissingRequiredMetadata

# Test: upload with required metadata (should succeed)
aws --endpoint-url http://rgw.example.com:7480 \
  s3 cp /tmp/report.pdf s3://compliance-data/report.pdf \
  --metadata "classification=confidential,data-owner=finance,retention-days=2555"

# Check transformation log output
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=30 \
  | grep -E "TRANSFORM:|SCHEMA:|ENRICH:|CONTENT_TYPE:"
```

## Summary

Lua data transformation scripts in Ceph RGW act as a transparent middleware layer for the S3 API, enabling key normalization, metadata schema enforcement, content-type validation, and response header enrichment. These transformations run at request time without modifying stored data, making them suitable for enforcing data governance policies, ensuring client compatibility, and maintaining metadata standards across large-scale object storage deployments.
