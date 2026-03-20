# How to Customize the Rancher Login Page

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, UI, Customization

Description: Learn how to customize the Rancher login page with custom logos, banners, and branding to match your organization's identity.

## Introduction

The Rancher login page is often the first impression users get of your Kubernetes management platform. Rancher provides built-in settings to replace the default SUSE/Rancher branding with your own logo, background, and messaging. This guide covers all available customization options and how to apply them.

## Prerequisites

- Rancher v2.6.0 or later
- Admin access to the Rancher UI
- Your logo image in PNG or SVG format (recommended: 200×50 px for the banner logo)

## Customization Options Overview

Rancher exposes branding controls under **Global Settings** → **Branding**. The login page specifically supports:

| Setting | Description |
|---|---|
| `ui-logo-light` | Logo shown on dark backgrounds (login page) |
| `ui-logo-dark` | Logo shown on light backgrounds |
| `ui-primary-color` | Primary accent color |
| `ui-banner-color` | Banner background color |
| `ui-pl` | Custom product name string |

## Step 1: Access Branding Settings via the UI

1. Log in to Rancher as **admin**.
2. Click the hamburger menu → **Global Settings**.
3. Select the **Branding** tab.
4. Upload your logo under **Logo (Light Background)** and **Logo (Dark Background)**.
5. Adjust **Primary Color** using the color picker.
6. Click **Apply**.

## Step 2: Set Branding via the Rancher API

For automated or GitOps-driven workflows, use the Rancher API:

```bash
# Set the light-mode logo (base64-encoded PNG)
LOGO_B64=$(base64 -w 0 /path/to/logo-light.png)

curl -sk -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"value\": \"data:image/png;base64,${LOGO_B64}\"}" \
  "https://<rancher-url>/v3/settings/ui-logo-light"

# Set the primary color
curl -sk -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "#1a73e8"}' \
  "https://<rancher-url>/v3/settings/ui-primary-color"

# Set the product name
curl -sk -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "Acme Kubernetes Platform"}' \
  "https://<rancher-url>/v3/settings/ui-pl"
```

## Step 3: Add a Login Banner Message

Compliance-heavy environments often need a legal notice on the login screen.

```bash
# Enable the login banner
curl -sk -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "true"}' \
  "https://<rancher-url>/v3/settings/ui-banners"

# Set banner text (HTML is supported)
curl -sk -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "<b>Authorized users only.</b> All activity is monitored."}' \
  "https://<rancher-url>/v3/settings/ui-banner-login"
```

## Step 4: Apply Branding via Helm Values (for fresh installs)

When deploying Rancher with Helm, you can pre-seed branding settings:

```yaml
# values-branding.yaml
extraEnv:
  - name: CATTLE_UI_PL
    value: "Acme Kubernetes Platform"
  - name: CATTLE_UI_PRIMARY_COLOR
    value: "#1a73e8"
```

```bash
helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  -f values-branding.yaml
```

## Step 5: Verify the Changes

Open an incognito browser window and navigate to your Rancher URL. You should see:

- Your custom logo in place of the Rancher logo.
- The custom primary color on buttons and links.
- The product name in the page title and header.
- The login banner (if configured).

## Resetting to Defaults

```bash
# Reset a setting to its default value
curl -sk -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://<rancher-url>/v3/settings/ui-logo-light?action=resetToDefault"
```

## Conclusion

Customizing the Rancher login page is straightforward through the Branding settings panel or the Rancher API. By replacing logos, adjusting colors, setting the product name, and adding compliance banners, you can create a professional, on-brand experience for your users. Automating these settings via Helm values or API calls ensures consistent branding across environments.
