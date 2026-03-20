# How to Configure Custom Branding in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Branding, UI, Customization

Description: A comprehensive guide to configuring custom branding in Rancher, including logos, colors, favicons, and product names across the entire UI.

## Introduction

Rancher's white-labeling capabilities allow enterprises and MSPs to fully replace the default Rancher branding with their own corporate identity. This guide goes beyond the login page and covers branding the entire Rancher dashboard — from favicons to navigation colors.

## Prerequisites

- Rancher v2.6+
- Admin access
- Prepared assets: logo (SVG/PNG), favicon (ICO/PNG 32×32), and brand color hex codes

## Branding Settings Reference

All branding settings live under `Global Settings → Branding` and are also available through the Rancher v3 API at `/v3/settings/`.

| Setting Key | Default | Description |
|---|---|---|
| `ui-logo-light` | Rancher logo | Logo on dark backgrounds |
| `ui-logo-dark` | Rancher logo | Logo on light backgrounds |
| `ui-favicon` | Rancher favicon | Browser tab icon |
| `ui-primary-color` | `#3D98D3` | Buttons, links, highlights |
| `ui-pl` | `Rancher` | Product name in title bar |
| `ui-community-links` | `true` | Show community/forums links |
| `ui-issues` | `true` | Show "File an Issue" links |

## Step 1: Prepare Your Assets

```bash
# Recommended image specifications:
# Logo: 200x40 px, transparent background, PNG or SVG
# Favicon: 32x32 px, ICO or PNG

# Convert your logo to base64 for API usage
BASE64_LOGO=$(base64 -w 0 logo.png)
BASE64_FAVICON=$(base64 -w 0 favicon.png)
```

## Step 2: Apply Branding via the UI

1. Go to **☰ → Global Settings → Branding**.
2. Upload logos for both light and dark modes.
3. Upload the favicon.
4. Use the color picker to set the primary color.
5. Enter your product name in the **Company Name / Product** field.
6. Toggle off **Community Links** and **Issue Reporting** for private environments.
7. Click **Apply**.

## Step 3: Apply Branding via API Script

Create a reusable script to apply branding across multiple Rancher instances:

```bash
#!/usr/bin/env bash
# apply-branding.sh — Apply custom branding to a Rancher instance

RANCHER_URL="${1:?Usage: apply-branding.sh <rancher-url> <token>}"
RANCHER_TOKEN="${2:?}"

apply_setting() {
  local key="$1"
  local value="$2"
  curl -sk -X PUT \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"value\": \"${value}\"}" \
    "${RANCHER_URL}/v3/settings/${key}"
  echo "  Set ${key}"
}

# Upload logos
LOGO_LIGHT=$(base64 -w 0 logo-light.png)
LOGO_DARK=$(base64 -w 0 logo-dark.png)
FAVICON=$(base64 -w 0 favicon.png)

apply_setting "ui-logo-light"    "data:image/png;base64,${LOGO_LIGHT}"
apply_setting "ui-logo-dark"     "data:image/png;base64,${LOGO_DARK}"
apply_setting "ui-favicon"       "data:image/png;base64,${FAVICON}"
apply_setting "ui-primary-color" "#1a73e8"
apply_setting "ui-pl"            "Acme Kubernetes Platform"
apply_setting "ui-community-links" "false"
apply_setting "ui-issues"        "false"

echo "Branding applied successfully."
```

```bash
chmod +x apply-branding.sh
./apply-branding.sh https://rancher.example.com token-xxxxx:yyyyy
```

## Step 4: Persist Branding Through Upgrades

Branding settings are stored in the Rancher database and survive upgrades. However, if you re-initialize the database, settings will be lost. To ensure they are re-applied after disaster recovery:

```yaml
# Store branding as a Kubernetes Secret for GitOps pipelines
apiVersion: v1
kind: Secret
metadata:
  name: rancher-branding-config
  namespace: cattle-system
type: Opaque
stringData:
  apply-branding.sh: |
    # (paste script contents here)
```

## Step 5: Hide Community and Documentation Links

For air-gapped or compliance environments where external links must be removed:

```bash
# Disable "Community" menu item
curl -sk -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "false"}' \
  "https://<rancher-url>/v3/settings/ui-community-links"

# Disable "File an Issue" link
curl -sk -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "false"}' \
  "https://<rancher-url>/v3/settings/ui-issues"
```

## Verifying Branding Changes

After applying, perform a hard refresh (`Ctrl+Shift+R`) in your browser and check:

- Logo appears in the top-left corner.
- Favicon appears in the browser tab.
- Buttons and links use the custom primary color.
- The page title shows your product name.
- Community links are hidden (if disabled).

## Conclusion

Custom branding in Rancher lets you deliver a seamless, white-labeled Kubernetes management experience. By configuring logos, colors, favicons, and product names — and automating the process with scripts — you can ensure consistent branding across all environments while maintaining a professional, enterprise-grade interface for your users.
