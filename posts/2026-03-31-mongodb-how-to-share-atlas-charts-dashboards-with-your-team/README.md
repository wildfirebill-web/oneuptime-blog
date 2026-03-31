# How to Share Atlas Charts Dashboards with Your Team

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Dashboard, Collaboration, Sharing

Description: Learn how to share MongoDB Atlas Charts dashboards with your team using roles, links, embedding, and scheduled snapshots for collaborative analytics.

---

## Sharing Options in Atlas Charts

Atlas Charts offers four ways to share dashboards:

1. **Within Atlas organization** - share with Atlas project members by role
2. **Public link** - anyone with the link can view (no login required)
3. **Embedded** - embed in a web app with optional authentication
4. **Scheduled snapshots** - email PDF/PNG exports on a schedule

## Option 1 - Share Within Your Atlas Organization

### Share with Specific Users

1. Open your dashboard in Atlas Charts
2. Click the **...** menu in the top right
3. Select **Share Dashboard**
4. In the **Share with people** section, enter email addresses of Atlas organization members
5. Set permission level:
   - **Viewer** - can view and interact with charts (filters, drill-down), cannot edit
   - **Editor** - can edit charts and add new ones
6. Click **Add**

### Share with the Entire Organization

```text
1. Open Share Dashboard
2. Under "General Access", change "Restricted" to "Anyone in organization"
3. Set default role: Viewer or Editor
4. Click Save
```

All Atlas organization members can now access the dashboard without an explicit invitation.

### Managing Permissions

```text
Dashboard owner:
- Can share with others and change permission levels
- Can transfer ownership
- Can delete the dashboard

Editor:
- Can add, edit, and delete charts
- Can change dashboard layout
- Cannot manage sharing permissions

Viewer:
- Can apply dashboard filters and drill-down
- Can export chart data (if allowed)
- Cannot edit charts or layout
```

## Option 2 - Public Link Sharing

For dashboards with non-sensitive, public-facing data:

1. Open Share Dashboard
2. Toggle on **Public link**
3. Copy the generated URL

```text
Example public URL:
https://charts.mongodb.com/charts-project-id/public/dashboards/dashboard-id
```

Anyone with this link can view the dashboard without logging in. Disable the public link at any time by toggling it off - existing link becomes invalid immediately.

## Option 3 - Embed in a Web Application

For embedding in internal tools or customer-facing apps (see the full embedding guide for code examples):

1. Open the individual chart's **...** menu
2. Select **Embed Chart**
3. Choose:
   - **Unauthenticated** for public data
   - **Authenticated** for user-scoped private data
4. Copy the Chart ID and base URL

```javascript
// Share a dashboard's worth of charts with your app team
// Pass the Chart IDs to your engineering team for embedding
const CHART_IDS = {
  revenueByMonth: 'chart-id-001',
  topProducts: 'chart-id-002',
  customerGrowth: 'chart-id-003'
};
```

## Option 4 - Scheduled Email Snapshots

Send dashboard snapshots to stakeholders who don't have Atlas access:

1. Open your dashboard
2. Click **...** menu
3. Select **Schedule Snapshot**
4. Configure the schedule:
   - Frequency: Daily, Weekly, Monthly
   - Time: choose your preferred send time
   - Format: PNG image or PDF
5. Enter recipient email addresses
6. Add a custom message (optional)
7. Click **Save Schedule**

```text
Example schedule configuration:
Frequency: Weekly (every Monday at 8:00 AM)
Format: PDF
Recipients: ceo@company.com, vp-sales@company.com
Message: "Weekly sales dashboard - please review top metrics before Monday standup"
```

## Snapshot Filters

You can schedule snapshots with specific dashboard filter values applied:

1. Apply the filters you want in the dashboard view
2. Click **Schedule Snapshot** - current filter state is captured
3. Recipients receive the dashboard as filtered when the snapshot was scheduled

## Exporting Chart Data

Viewers with export permission can download chart data:

1. Hover over a chart
2. Click the **...** (kebab) menu on the chart
3. Select **Export**
4. Choose format:
   - **PNG** - image of the chart
   - **CSV** - raw data behind the chart

### Controlling Export Permissions

```text
As dashboard owner:
1. Open Share Dashboard
2. Under "Viewer permissions", toggle "Allow data export" on or off
3. This controls whether Viewers can download CSV data
```

## Sharing Best Practices

```text
1. Use descriptive dashboard names: "Q1 2026 Sales KPIs" not "Dashboard 3"
2. Add a description: click the dashboard title area to add context
3. Group related charts on one dashboard to minimize the number of shared links
4. Use scheduled snapshots for executive stakeholders who check email, not tools
5. Keep sensitive data on authenticated-only dashboards
6. Revoke public links when data is no longer meant to be public
7. Give editors the minimum permission needed - viewers cannot accidentally break charts
```

## Checking Who Has Access

```text
1. Open your dashboard
2. Click Share Dashboard
3. The "People with access" section lists all users and their permission levels
4. Click the role dropdown next to any user to change or revoke access
```

## Summary

MongoDB Atlas Charts provides flexible sharing through organization-level access control, public links, web app embedding, and scheduled email snapshots. Share dashboards with Atlas organization members for interactive collaboration, use public links for non-sensitive external audiences, embed in applications for in-product analytics, and schedule PNG/PDF snapshots to keep executive stakeholders informed without requiring them to log into Atlas directly. Always apply the principle of least privilege - default new shares to Viewer access.
