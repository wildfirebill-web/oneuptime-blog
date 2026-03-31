# How to Debug Lua Scripts in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Lua, Debug

Description: Learn how to debug Lua scripts in Ceph RGW using log analysis, safe error handling, unit testing patterns, and incremental deployment techniques.

---

## Overview

Debugging Lua scripts in Ceph RGW requires understanding how scripts run in-process and how to capture their output. Since RGW scripts cannot be attached to a debugger, effective debugging relies on the `RGWDebugLog` function, progressive testing, error capture with `pcall`, and careful log analysis.

## Step 1 - Enable Debug Logging

```bash
# Enable RGW debug logging at level 10 or higher
ceph config set client.rgw.my-store debug_rgw 10

# For Rook-managed RGW
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- \
  ceph config set client.rgw.my-store debug_rgw 10

# Restart RGW to apply
kubectl -n rook-ceph rollout restart deployment/rook-ceph-rgw-my-store-a

# Stream Lua debug output from logs
kubectl -n rook-ceph logs -l app=rook-ceph-rgw -f \
  | grep -i "lua\|script"
```

## Step 2 - Add Systematic Debug Output

```lua
-- debug_template.lua
-- Template for adding debug output to any Lua script

-- Step 1: Check that the script is running at all
RGWDebugLog("DEBUG: Script started - context=preRequest")

local ok, err = pcall(function()
  -- Step 2: Log all available request fields
  RGWDebugLog("DEBUG: method=" .. tostring(Request.HTTP.Method))
  RGWDebugLog("DEBUG: bucket=" .. tostring(Request.Bucket.Name))
  RGWDebugLog("DEBUG: object=" .. tostring(Request.Object.Name))
  RGWDebugLog("DEBUG: user=" .. tostring(Request.User.Id))
  RGWDebugLog("DEBUG: host=" .. tostring(Request.HTTP.Host))
  RGWDebugLog("DEBUG: uri=" .. tostring(Request.HTTP.URI))

  -- Step 3: Log all request headers
  for k, v in pairs(Request.HTTP.Header) do
    RGWDebugLog("DEBUG: header " .. k .. "=" .. v)
  end

  -- Step 4: Log all metadata
  for k, v in pairs(Request.HTTP.Metadata) do
    RGWDebugLog("DEBUG: metadata " .. k .. "=" .. v)
  end

  -- Your actual script logic here
  RGWDebugLog("DEBUG: Logic executing...")
end)

if not ok then
  RGWDebugLog("DEBUG: Script ERROR: " .. tostring(err))
end

RGWDebugLog("DEBUG: Script completed - ok=" .. tostring(ok))
```

## Step 3 - Unit Testing Lua Scripts Locally

```lua
-- test_harness.lua
-- Run this with the standalone lua interpreter to test logic

-- Mock the RGW API for testing
Request = {
  HTTP = {
    Method = "PUT",
    URI = "/mybucket/myobject.txt",
    Host = "rgw.example.com",
    ContentLength = 1024,
    Header = {["Content-Type"] = "text/plain"},
    Metadata = {["x-amz-meta-owner"] = "alice"},
  },
  Bucket = {Name = "mybucket"},
  Object = {Name = "myobject.txt"},
  User = {Id = "alice", DisplayName = "Alice Smith"},
}

Response = {
  HTTP = {
    AddHeader = function(k, v) print("RESPONSE HEADER: " .. k .. "=" .. v) end,
  }
}

function RGWDebugLog(msg) print("LOG: " .. msg) end

function abort(code, err_name, message)
  print(string.format("ABORT: %d %s: %s", code, err_name, message))
  error("aborted")
end

-- Load and run the script under test
dofile("my_script.lua")
print("Test passed!")
```

```bash
# Run the test locally
lua test_harness.lua
```

## Step 4 - Incremental Deployment Strategy

```bash
# Step 1: Test with a minimal no-op script
cat > noop.lua << 'EOF'
RGWDebugLog("Lua: Script loaded and running")
EOF

radosgw-admin script put --infile=noop.lua --context=preRequest

# Make a test request and verify the log line appears
aws --endpoint-url http://rgw.example.com:7480 s3 ls
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=10 | grep "Script loaded"

# Step 2: Add one feature at a time
# Step 3: Test after each addition
# Step 4: Roll back immediately if issues occur
radosgw-admin script rm --context=preRequest
```

## Step 5 - Common Debugging Issues

```lua
-- nil_safety.lua
-- Common bug: accessing nil fields causes script errors

-- WRONG - will crash if Request.Object.Name is nil:
-- local key = string.upper(Request.Object.Name)

-- CORRECT - nil-safe patterns:
local object_name = Request.Object.Name or ""
local upper_name = (object_name ~= "") and string.upper(object_name) or ""

-- WRONG - pairs() on nil crashes:
-- for k, v in pairs(Request.HTTP.Metadata) do

-- CORRECT - check before iterating:
local meta = Request.HTTP.Metadata
if meta then
  for k, v in pairs(meta) do
    RGWDebugLog("Meta: " .. k .. "=" .. v)
  end
end

-- Type checking before arithmetic:
local size = tonumber(Request.HTTP.ContentLength) or 0
local size_mb = size / 1048576.0
RGWDebugLog("Size: " .. string.format("%.2f MB", size_mb))
```

## Step 6 - Remove Debug Instrumentation for Production

```bash
# Compare script performance with and without debug logging
# Run a quick benchmark
time for i in $(seq 1 100); do
  aws --endpoint-url http://rgw.example.com:7480 s3 ls > /dev/null 2>&1
done

# Remove debug lines from production scripts
# Replace RGWDebugLog calls with empty functions or remove them:
# function RGWDebugLog(msg) end  -- no-op in production

# Lower the debug log level after debugging
ceph config set client.rgw.my-store debug_rgw 1
kubectl -n rook-ceph rollout restart deployment/rook-ceph-rgw-my-store-a

# Verify final script
radosgw-admin script get --context=preRequest
```

## Summary

Debugging Lua scripts in Ceph RGW relies on the `RGWDebugLog` function, systematic request field logging, and `pcall` for safe error capture. Local unit testing with a mock RGW API allows development and debugging without a live cluster. Incremental deployment - starting with a no-op script and adding logic gradually - minimizes the risk of introducing bugs into a production RGW instance.
