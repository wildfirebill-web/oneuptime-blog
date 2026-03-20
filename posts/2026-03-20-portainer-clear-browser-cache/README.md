# How to Clear the Portainer Browser Cache to Fix UI Issues - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Browser, UI, Cache

Description: Fix Portainer UI issues by properly clearing browser cache, local storage, and cookies - the first troubleshooting step for most Portainer display and JavaScript errors.

## Introduction

Many Portainer UI issues - blank screens, outdated interfaces after upgrades, JavaScript errors, authentication loops, and stale data displays - are caused by cached browser data from a previous version. Clearing the browser cache is always the first troubleshooting step before investigating backend issues.

## What Browser Data Affects Portainer

| Data Type | What It Stores | Effect When Stale |
|-----------|---------------|------------------|
| HTTP Cache | JS/CSS files, images | Old UI code running |
| Local Storage | Session tokens, UI state | Auth loops, wrong settings |
| Session Storage | Temporary app state | Stale page data |
| Cookies | Session ID, auth tokens | Login failures |
| IndexedDB | Application data | Corrupted state |
| Service Workers | Offline functionality | Outdated cached responses |

## Method 1: Hard Reload (Fastest)

Forces reload of all assets without clearing the full cache:

```sql
Windows/Linux: Ctrl + Shift + R
Mac: Cmd + Shift + R

Or:
1. Open DevTools (F12)
2. Right-click the refresh button
3. Select "Empty Cache and Hard Reload"
```

## Method 2: Chrome - Clear Site Data

The most thorough method:

```text
1. Open DevTools (F12)
2. Click the "Application" tab
3. In left panel: Storage
4. Click "Clear site data"
5. Check all boxes:
   ✓ Unregister service workers
   ✓ Local and session storage
   ✓ IndexedDB
   ✓ Web SQL
   ✓ Cookies
   ✓ Cache storage
   ✓ Application cache
6. Click "Clear site data"
7. Reload the page
```

## Method 3: Clear Cache via Browser Settings

### Chrome

```text
1. Settings (⋮) → Settings
2. Privacy and security → Clear browsing data
3. Select: All time
4. Check:
   ✓ Cookies and other site data
   ✓ Cached images and files
5. Click "Clear data"
```

Or use keyboard shortcut: `Ctrl+Shift+Delete`

### Firefox

```text
1. Menu (☰) → History → Clear Recent History
2. Time range: Everything
3. Check:
   ✓ Cookies
   ✓ Cache
   ✓ Active Logins
   ✓ Site Preferences
4. Click "OK"
```

Or: `Ctrl+Shift+Delete`

### Safari

```text
1. Develop menu → Empty Caches
2. And: Safari → Clear History
```

Enable Develop menu: Safari → Settings → Advanced → Show Develop menu.

## Method 4: Clear Portainer-Specific Storage via Console

Target only Portainer's data without clearing other sites:

```javascript
// Open browser console on Portainer page (F12 → Console)

// Clear all local storage for this origin
localStorage.clear();
console.log('localStorage cleared');

// Clear session storage
sessionStorage.clear();
console.log('sessionStorage cleared');

// Clear cookies for this domain
document.cookie.split(";").forEach(function(c) {
  document.cookie = c.replace(/^ +/, "").replace(/=.*/, "=;expires=" + new Date().toUTCString() + ";path=/");
});
console.log('Cookies cleared');

// Reload
window.location.reload(true);
```

## Method 5: Incognito/Private Mode Test

Before clearing, test if the issue is cache-related:

```text
Chrome: Ctrl+Shift+N → new incognito window
Firefox: Ctrl+Shift+P → new private window
Safari: File → New Private Window
Edge: Ctrl+Shift+N → new InPrivate window
```

Navigate to Portainer in the private window:
- **Issue gone** in private mode = browser cache is the problem
- **Issue persists** in private mode = server-side issue

## Method 6: Clear Service Workers

After Portainer upgrades, old service workers may serve cached responses:

```javascript
// In browser console:
// List all registered service workers
navigator.serviceWorker.getRegistrations().then(registrations => {
  registrations.forEach(reg => {
    reg.unregister();
    console.log('Unregistered:', reg.scope);
  });
});

// Clear all caches
caches.keys().then(cacheNames => {
  return Promise.all(
    cacheNames.map(cacheName => {
      console.log('Clearing cache:', cacheName);
      return caches.delete(cacheName);
    })
  );
});

// Reload
window.location.reload(true);
```

## Method 7: Use a Different Browser

Test in a completely different browser to isolate the issue:

```bash
# If issue is specific to Chrome, try Firefox

# If specific to Firefox, try Chrome or Edge
# This quickly confirms if it's browser-specific vs server-side
```

## Common UI Issues Resolved by Cache Clearing

| Symptom | Root Cause |
|---------|-----------|
| UI stuck on old Portainer version | Cached JS files |
| Login form doesn't appear | Corrupted localStorage |
| "Session expired" on every refresh | Stale session token |
| Container list doesn't update | Cached API responses |
| Settings form doesn't save | Stale application state |
| Theme changes don't apply | Cached CSS |
| 2FA screen appears after disabling 2FA | Cached auth state |

## Method 8: Fix After Portainer URL/Domain Change

If you changed Portainer's domain, old cached data from the old URL may cause issues:

```bash
# Old URL cached data won't affect the new URL (different origin)
# But if you're accessing via IP AND hostname, they're different origins

# Access consistently via either:
# https://portainer.yourdomain.com (always)
# OR
# http://192.168.1.100:9000 (always)
# NOT a mix of both
```

## Conclusion

Browser cache clearing is the fastest and most effective fix for the majority of Portainer UI issues. Always test in incognito mode first - if that works, clearing the full browser cache for the Portainer origin will resolve it. For post-upgrade issues, use the DevTools Application tab to clear all site data at once, including service workers and IndexedDB, to ensure no old code is cached.
