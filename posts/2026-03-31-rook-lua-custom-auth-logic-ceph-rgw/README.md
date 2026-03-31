# How to Implement Custom Authentication Logic with Lua in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Lua, Authentication

Description: Learn how to implement custom authentication and authorization checks in Ceph RGW using Lua scripts to enforce policies beyond standard S3 IAM.

---

## Overview

While Ceph RGW has a full S3-compatible IAM system, Lua scripts allow you to add authentication and authorization logic on top of the standard checks. This is useful for enforcing custom policies such as IP allowlisting, time-of-day access restrictions, API key validation via an external service, or multi-factor authorization requirements.

## Step 1 - Basic Authentication Extension

Lua scripts run after standard S3 authentication succeeds. You can add additional checks:

```lua
-- custom_auth.lua
-- Additional authentication layer on top of S3 auth

-- Block access from unauthorized IP ranges
local function check_ip_allowlist(request_ip)
  local allowed_ranges = {
    "10.0.0.",     -- Internal network
    "192.168.1.",  -- Office network
  }

  for _, prefix in ipairs(allowed_ranges) do
    if string.find(request_ip, prefix, 1, true) then
      return true
    end
  end
  return false
end

-- Get the client IP from the request
local client_ip = Request.HTTP.Environment["REMOTE_ADDR"] or ""

if not check_ip_allowlist(client_ip) then
  RGWDebugLog("AUTH: Blocked unauthorized IP " .. client_ip ..
              " for user=" .. (Request.User.Id or "anonymous"))
  abort(403, "Forbidden", "Access from your IP address is not permitted.")
end
```

## Step 2 - Time-of-Day Access Restriction

```lua
-- time_based_access.lua
-- Restrict write operations to business hours (UTC)

local function get_hour_utc()
  -- os.time() returns UTC epoch on Linux
  local t = os.date("!*t", os.time())
  return t.hour
end

local function get_day_of_week()
  local t = os.date("!*t", os.time())
  return t.wday  -- 1=Sunday, 7=Saturday
end

local method = Request.HTTP.Method
local hour = get_hour_utc()
local dow = get_day_of_week()

-- Allow reads any time, but restrict writes to Mon-Fri 08:00-18:00 UTC
local write_methods = {PUT=true, DELETE=true, POST=true}

if write_methods[method] then
  local is_weekend = (dow == 1 or dow == 7)
  local outside_hours = (hour < 8 or hour >= 18)

  if is_weekend or outside_hours then
    local user = Request.User.Id or "anonymous"
    -- Allow admin bypass
    if user ~= "admin" then
      RGWDebugLog("AUTH: Write blocked outside business hours user=" .. user)
      abort(403, "WriteAccessRestricted",
            "Write operations are only permitted during business hours.")
    end
  end
end
```

## Step 3 - Enforce Bucket-Level Custom Tags

```lua
-- bucket_tag_auth.lua
-- Check if a user is authorized to access a bucket by matching user groups to bucket tags

-- In a real implementation, you would look up user groups from
-- the request context or an external cache
local function user_has_group(user_id, group)
  -- Simplified: prefix-based group assignment
  -- In practice, store group memberships in RGW user attributes
  local group_prefix_map = {
    finance = "fin-",
    engineering = "eng-",
    ops = "ops-",
  }
  local prefix = group_prefix_map[group]
  if prefix then
    return string.find(user_id, prefix, 1, true) ~= nil
  end
  return false
end

local bucket = Request.Bucket.Name or ""
local user = Request.User.Id or "anonymous"

-- Buckets with "dept:" prefix require group membership
local dept = string.match(bucket, "^dept%-(%a+)%-")
if dept then
  if not user_has_group(user, dept) then
    RGWDebugLog("AUTH: User " .. user ..
                " denied access to dept bucket " .. bucket)
    abort(403, "DepartmentAccessDenied",
          "You are not a member of the department that owns this bucket.")
  end
end
```

## Step 4 - API Key Header Validation

```lua
-- api_key_auth.lua
-- Validate a custom API key header in addition to S3 credentials

local REQUIRED_API_KEY = "x-custom-api-key"
local VALID_KEYS = {
  ["abc123-production"] = true,
  ["def456-staging"] = true,
}

-- Check for API key on write operations
if Request.HTTP.Method == "PUT" or Request.HTTP.Method == "POST" then
  local api_key = Request.HTTP.Header[REQUIRED_API_KEY]

  if not api_key then
    RGWDebugLog("AUTH: Missing required API key header")
    abort(400, "MissingAPIKey",
          "The header " .. REQUIRED_API_KEY .. " is required for write operations.")
  end

  if not VALID_KEYS[api_key] then
    RGWDebugLog("AUTH: Invalid API key: " .. api_key)
    abort(403, "InvalidAPIKey", "The provided API key is not valid.")
  end
end
```

## Step 5 - Deploy and Test the Auth Scripts

```bash
# Upload the script
radosgw-admin script put \
  --infile=custom_auth.lua \
  --context=preRequest

# Test with a blocked IP (if testing from outside allowed range)
curl -v http://rgw.example.com:7480/mybucket/ \
  -H "Authorization: AWS ${ACCESS_KEY}:${SIGNATURE}"

# Check the authentication log output
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=30 | grep "AUTH:"

# Remove the script if needed
radosgw-admin script rm --context=preRequest
```

## Step 6 - Handle Script Errors Safely

```lua
-- Always use pcall to prevent script bugs from blocking all requests
local ok, err = pcall(function()
  -- Your authentication logic here
  local user = Request.User.Id
  if user == "suspended-user" then
    abort(403, "AccountSuspended", "Your account has been suspended.")
  end
end)

if not ok then
  -- Log the error but allow the request to proceed
  -- This prevents a Lua bug from causing an outage
  RGWDebugLog("Auth script error: " .. tostring(err))
end
```

## Summary

Lua custom authentication scripts in Ceph RGW extend the standard S3 IAM system with additional checks such as IP allowlisting, time-of-day restrictions, and custom header validation. Scripts run after standard S3 authentication, so they serve as an additional authorization layer rather than a replacement. Using `abort()` for rejections and `pcall` for error safety ensures custom auth logic is both effective and resilient.
