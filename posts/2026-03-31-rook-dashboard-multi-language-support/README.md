# How to Configure Dashboard Multi-Language Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Dashboard, Localization, Configuration

Description: Enable and configure multi-language support in the Ceph Dashboard, allowing users to switch the interface language for international team environments.

---

## Overview

The Ceph Dashboard supports multiple languages through built-in internationalization (i18n). Users can select their preferred language from the Dashboard settings, and administrators can configure the default language for new sessions.

## Supported Languages

As of Ceph Reef and Squid, the Dashboard ships with translations for:
- English (default)
- Chinese (Simplified) - zh-Hans
- German - de-DE
- Korean - ko-KR
- Portuguese (Brazil) - pt-BR
- Spanish - es-ES
- French - fr-FR

Check available languages in the current build:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-mgr -- \
  ls /usr/share/ceph/mgr/dashboard/frontend/dist/assets/i18n/
```

## Setting User Language Preference

Individual users can set their language preference in the Dashboard UI:

1. Click the user icon in the top-right corner
2. Select "User profile"
3. Set the "Language" dropdown to the preferred locale
4. Click "Save"

This setting is stored per user account and persists across sessions.

## Setting the Default Language

Administrators can set a cluster-wide default language:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard set-locale zh-Hans

# Verify
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard get-locale
```

Available locale codes:

```bash
# List configured locales
kubectl -n rook-ceph exec deploy/rook-ceph-mgr -- \
  python3 -c "
import os, json
i18n_path = '/usr/share/ceph/mgr/dashboard/frontend/dist/assets/i18n/'
if os.path.exists(i18n_path):
    files = os.listdir(i18n_path)
    print('\n'.join(f.replace('.json','') for f in files if f.endswith('.json')))
"
```

## Language via Browser Accept-Language Header

The Dashboard respects the browser's `Accept-Language` header if no user preference is set. Users on configured browsers will automatically see the Dashboard in their OS language:

```bash
# Example: Chrome on a Chinese locale system will request zh-Hans automatically
# No server configuration needed for browser-based language detection
```

## Verify Translation Quality

Not all strings may be translated in every language. Check translation completeness for your language:

```bash
# Count translated strings vs total
kubectl -n rook-ceph exec deploy/rook-ceph-mgr -- \
  python3 -c "
import json
with open('/usr/share/ceph/mgr/dashboard/frontend/dist/assets/i18n/de-DE.json') as f:
    data = json.load(f)
total = len(data)
translated = sum(1 for v in data.values() if v)
print(f'German: {translated}/{total} strings translated ({100*translated//total}%)')
"
```

## Contributing Translations

To improve translations or add a new language, contribute to the Ceph project:

```bash
# Ceph uses Transifex for translation management
# Project: https://www.transifex.com/ceph/ceph-dashboard/

# Extract strings for translation
cd src/pybind/mgr/dashboard/frontend
npm run i18n:extract
```

## Summary

Ceph Dashboard multi-language support allows individual users to select their preferred interface language or administrators to set a cluster-wide default. The Dashboard respects browser Accept-Language headers for automatic language detection. Use `ceph dashboard set-locale` to set the default language for your team's primary locale.
