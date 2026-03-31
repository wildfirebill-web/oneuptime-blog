# How to Change the Theme (Light/Dark/High-Contrast) in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, UI, Themes, Accessibility, Dark Mode, Configuration, User Setting

Description: Learn how to switch between Portainer's light, dark, and high-contrast themes in the user settings for improved readability and accessibility.

---

Portainer supports three visual themes: Light (default), Dark, and High-Contrast. Each user can set their own theme preference independently, stored in their profile. The setting applies to the current logged-in user only.

## Changing Your Theme

1. Log in to Portainer.
2. Click your username in the top-right corner.
3. Select **My Account**.
4. Scroll to the **Appearance** section.
5. Choose **Light**, **Dark**, or **High Contrast**.
6. The theme applies immediately - no save button required.

## Theme Options

| Theme | Best For |
|---|---|
| **Light** | Default - well-lit environments, printing |
| **Dark** | Low-light environments, reduces eye strain |
| **High Contrast** | Accessibility requirements, visibility impairment |

## Persisting Theme Across Sessions

The theme selection is saved in Portainer's database against your user account. It persists across:

- Browser refreshes
- Logouts and logins
- Different devices (since it is server-side, not browser-side)

## Setting a Default Theme for New Users (Admin)

Admins can set a default theme for all new user accounts via the API:

```bash
# Get an API token

TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Update your own user theme
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/users/1 \
  -d '{"ThemeSettings":{"color":"dark"}}'
```

Valid values for `color` are: `"light"`, `"dark"`, `"highcontrast"`, `"auto"`.

## Browser Auto Theme

Setting the theme to **Auto** in Portainer makes the UI follow your operating system's light/dark mode preference:

```javascript
// Portainer reads this media query to determine auto theme
window.matchMedia('(prefers-color-scheme: dark)').matches
// true → dark mode applied
// false → light mode applied
```

## High Contrast Mode

High Contrast mode increases color differentiation and font weight, meeting WCAG AA accessibility guidelines. It is recommended for users with:

- Visual impairments
- Bright screen environments
- Color blindness (the high-contrast palette avoids red/green reliance)
