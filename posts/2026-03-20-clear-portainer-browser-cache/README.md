# How to Clear the Portainer Browser Cache to Fix UI Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Browser Cache, UI, JavaScript, Upgrade Issues

Description: Learn how to properly clear Portainer's browser cache to fix UI glitches, blank screens, and outdated JavaScript after upgrades using browser DevTools and cache-busting techniques.

---

Portainer ships as a single-page application (SPA). After a Portainer upgrade, old JavaScript files cached by the browser can conflict with the new backend API, causing blank screens, missing UI elements, or cryptic JavaScript errors.

## The Problem with Browser Cache After Upgrades

When you upgrade Portainer, the JavaScript bundle filenames change. If your browser still has the old bundle cached, it will:

1. Load old JS files that reference API endpoints that no longer exist
2. Fail silently or show a blank screen
3. Display outdated UI components mixed with new API responses

## Step 1: Hard Reload (Quick Fix)

```
# Windows/Linux: Ctrl + Shift + R
# Mac:           Cmd + Shift + R
# Firefox:       Ctrl + F5
```

This bypasses the browser cache for the current page. Try this first.

## Step 2: Clear Site Data via DevTools

Open DevTools (`F12`) and go to the **Application** tab:

1. Expand **Storage** in the left sidebar.
2. Click **Clear site data** at the bottom.
3. Check all boxes (Local Storage, Session Storage, Cache Storage, Service Workers).
4. Click **Clear site data**.

Reload the page.

## Step 3: Clear Browser Cache Manually

**Chrome / Edge:**
1. Press `Ctrl+Shift+Delete`
2. Set time range to **All time**
3. Check **Cached images and files** and **Cookies and other site data**
4. Click **Clear data**

**Firefox:**
1. Press `Ctrl+Shift+Delete`
2. Select **Cache** and **Cookies**
3. Click **Clear Now**

## Step 4: Use Incognito/Private Mode

Test in a fresh incognito window which has no cached data:

```
Chrome:  Ctrl+Shift+N
Firefox: Ctrl+Shift+P
Edge:    Ctrl+Shift+N
```

If Portainer works in incognito, the issue is definitely the browser cache.

## Step 5: Add Cache-Busting Headers to Portainer

For teams where browser cache is a recurring problem after updates, add cache-control headers via a reverse proxy:

```nginx
location / {
    proxy_pass http://portainer:9000;
    # Force browsers to revalidate the main HTML page on every request
    add_header Cache-Control "no-cache" always;
}

# But cache static assets normally (JS/CSS have content hashes in filenames)
location ~* \.(js|css|png|woff2)$ {
    proxy_pass http://portainer:9000;
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

## Step 6: Check Service Worker Cache

Portainer may register a service worker that caches assets independently:

In Chrome DevTools: **Application > Service Workers** → click **Unregister** if one is listed for the Portainer origin.
