# How to Share Atlas Charts Dashboards with Your Team

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Dashboard, Sharing, Collaboration

Description: Learn the different ways to share MongoDB Atlas Charts dashboards with teammates, external stakeholders, and embedded in web applications.

---

## Sharing Options Overview

MongoDB Atlas Charts provides several ways to share dashboards depending on your audience and security requirements:

1. **Atlas project access** - team members with Atlas project roles see dashboards automatically
2. **Dashboard-specific sharing** - share with specific Atlas users or by email
3. **Unauthenticated public link** - shareable URL for anyone with the link
4. **Embedded charts** - charts embedded in a web application with optional JWT auth

## Sharing with Atlas Project Members

Any user with access to your Atlas project automatically has access to your Charts dashboards, based on their project role:

| Atlas Role | Charts Access |
|---|---|
| Project Owner | Full edit access |
| Project Data Access Admin | Full edit access |
| Project Data Access Read Only | View only |
| Project Read Only | View only |

No additional configuration is required. Users log into [cloud.mongodb.com](https://cloud.mongodb.com) and navigate to Charts.

## Dashboard-Specific Sharing

To share with a specific user or to grant access beyond their project role:

1. Open the dashboard
2. Click the **Share** button in the top right
3. Click **Add People**
4. Enter the Atlas account email address
5. Set the permission level: **Viewer** or **Editor**

Viewers can see the dashboard and interact with filters. Editors can modify charts and add new ones.

## Sharing a Dashboard Link

To share a direct link to a dashboard with project members:

1. Click **Share** then **Get Link**
2. Copy the URL

This link only works for users who already have Atlas project access. Anyone without access will see an authentication prompt.

## Unauthenticated Public Links

For sharing with stakeholders who do not have Atlas accounts - such as clients or executives outside your organization:

1. Click **Share - Embed**
2. Enable **Unauthenticated Access** at the dashboard level
3. Copy the generated URL

This URL can be shared with anyone. The data visible is limited by chart-level filters you configured. Use this only for non-sensitive, aggregate data.

**Security note**: Enabling unauthenticated access exposes the underlying collection metadata. Never use this for dashboards that display PII, financial data, or confidential business information.

## Scheduling Dashboard Email Reports

Atlas Charts supports scheduled email delivery of dashboard snapshots:

1. Click **Share - Schedule Reports**
2. Set the frequency: Daily, Weekly, or Monthly
3. Add recipient email addresses
4. Choose a time zone and delivery time
5. Click **Save**

Recipients receive a PDF or PNG snapshot of the dashboard at the configured schedule. This is useful for weekly executive summaries without requiring recipients to log into Atlas.

## Setting Default Filter Values for Recipients

When sharing a link, you can encode default filter values in the URL. Set your dashboard filters to the desired state, then copy the URL from the browser. The filter state is preserved in the URL parameters and pre-applied when the recipient opens the link.

## Managing Dashboard Permissions

To audit or change who has access:

1. Open the dashboard
2. Click **Share**
3. Review the list of users and their roles
4. Click the role next to any user to change it, or click **Remove** to revoke access

## Transferring Dashboard Ownership

If a dashboard owner leaves the team, a Project Owner can transfer dashboard ownership:

1. Go to the dashboard list
2. Click the three-dot menu next to the dashboard
3. Select **Transfer Ownership**
4. Enter the new owner's Atlas email

## Best Practices for Team Dashboards

- **Naming conventions**: prefix dashboard names with the team or use case (e.g., "Sales - Weekly Revenue", "Ops - Infrastructure Health")
- **Pinning**: star dashboards you use frequently so they appear at the top of the dashboard list
- **Version notes**: use the dashboard description field to note the last major change and who made it
- **Test before scheduling**: preview the dashboard in a different browser (incognito) to verify it renders correctly for viewers before enabling scheduled reports

## Summary

Atlas Charts dashboards can be shared via Atlas project roles for internal teams, direct email invitations for specific collaborators, unauthenticated public links for external stakeholders, and scheduled email reports for async consumption. Dashboard-specific permission settings let you control view versus edit access granularly. For any dashboard containing sensitive data, restrict access to authenticated Atlas users and avoid enabling public links.
