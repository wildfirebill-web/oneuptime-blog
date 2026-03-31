# How to Use MongoDB Atlas Data Explorer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Data Explorer, Query, GUI

Description: Use the MongoDB Atlas Data Explorer to browse collections, run queries, build aggregation pipelines, and manage indexes without leaving your browser.

---

The Atlas Data Explorer is a browser-based UI that lets you interact with your MongoDB data without installing any client tools. It supports CRUD operations, aggregation pipeline building, index management, and schema analysis.

## Opening Data Explorer

1. Log in to [cloud.mongodb.com](https://cloud.mongodb.com).
2. Select your project, then click **Browse Collections** on your cluster tile.
3. The Data Explorer opens showing a list of databases on the left.

## Browsing and Filtering Documents

Click a collection name to see its documents in card view. Use the filter bar at the top to add query predicates:

```text
{ "status": "active", "age": { "$gt": 25 } }
```

Use the **Options** dropdown to set a projection, sort order, or limit:

```text
Project:  { "name": 1, "email": 1, "_id": 0 }
Sort:     { "createdAt": -1 }
Limit:    50
```

Click **Find** to execute. Results appear in a paginated card view that you can switch to Table or JSON format.

## Inserting and Editing Documents

- Click **Insert Document** to open an in-browser JSON editor. You can type raw JSON or use the field-level editor.
- Click the pencil icon on any document to edit it inline.
- Click the trash icon to delete a single document.

## Running Aggregation Pipelines

Switch to the **Aggregation** tab. Each stage is a separate card:

```text
Stage 1: $match
{ "category": "electronics", "inStock": true }

Stage 2: $group
{ "_id": "$brand", "totalRevenue": { "$sum": "$price" } }

Stage 3: $sort
{ "totalRevenue": -1 }
```

Atlas shows a preview of documents after each stage, making it easy to debug intermediate results.

## Managing Indexes

Click the **Indexes** tab to see all existing indexes with their size and usage statistics. To create a new index:

1. Click **Create Index**.
2. Add field names and sort directions.
3. Toggle options: Unique, Sparse, TTL, Partial filter expression.
4. Click **Create**.

Atlas builds the index using a rolling build so production reads and writes continue uninterrupted.

## Schema Analysis

The **Schema** tab samples up to 1,000 documents and shows field-level statistics: data types, value distributions, and null counts. This is useful for catching schema inconsistencies introduced by application bugs.

## Exporting Query Results

From any find result, click **Export Collection** to download:
- JSON (full documents)
- CSV (flat projection)

For large exports, use `mongoexport` from the CLI instead, as the UI export caps at a few thousand documents.

```bash
mongoexport \
  --uri "mongodb+srv://user:pass@cluster.mongodb.net/mydb" \
  --collection orders \
  --query '{"status":"pending"}' \
  --out pending_orders.json
```

## Keyboard Shortcuts

| Action | Shortcut |
|--------|----------|
| Execute query | Ctrl + Enter |
| Add pipeline stage | Alt + Enter |
| Format JSON | Shift + Alt + F |

## Summary

The Atlas Data Explorer gives developers and DBAs a fast way to browse, query, and manage MongoDB data through a browser without configuring a local client. The aggregation pipeline builder and inline schema analysis make it particularly useful for exploratory data work and onboarding new team members.
