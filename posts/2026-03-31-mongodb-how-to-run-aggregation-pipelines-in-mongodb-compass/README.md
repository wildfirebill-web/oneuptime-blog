# How to Run Aggregation Pipelines in MongoDB Compass

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, MongoDB Compass, Aggregation, Developer Tool, Query Optimization

Description: Learn how to use MongoDB Compass's visual aggregation pipeline builder to construct, test, and export aggregation pipelines with stage-by-stage output preview.

---

## The Aggregation Tab in Compass

MongoDB Compass provides a visual aggregation pipeline builder in the **Aggregations** tab of each collection. It lets you:
- Add pipeline stages from a dropdown menu
- Preview output after each stage
- See the document count at each step
- Export the pipeline as JavaScript, Python, Java, or C# code

## Opening the Aggregation Builder

1. Connect to your MongoDB instance in Compass
2. Select your database and collection
3. Click the **Aggregations** tab
4. Click **Add Stage** to add your first pipeline stage

## Building a Pipeline Stage by Stage

Each stage has:
- A **Stage Type** dropdown (`$match`, `$group`, `$project`, `$sort`, `$lookup`, etc.)
- An **editor panel** for the stage configuration (JSON syntax)
- A **preview panel** showing sample output documents from that stage

Example: Build a pipeline that calculates revenue by product category.

**Stage 1: $match**
```javascript
{ status: "completed" }
```

**Stage 2: $group**
```javascript
{
  _id: "$category",
  totalRevenue: { $sum: "$amount" },
  orderCount: { $sum: 1 },
  avgOrderValue: { $avg: "$amount" }
}
```

**Stage 3: $sort**
```javascript
{ totalRevenue: -1 }
```

**Stage 4: $project**
```javascript
{
  _id: 0,
  category: "$_id",
  totalRevenue: { $round: ["$totalRevenue", 2] },
  orderCount: 1,
  avgOrderValue: { $round: ["$avgOrderValue", 2] }
}
```

## Using the Preview Panel

After entering each stage, Compass shows a preview of the first 20 documents output by the pipeline up to that stage. This lets you verify each transformation before adding the next stage.

If the preview shows an empty result, check:
- That filter conditions in `$match` match actual field names and values
- That `$group` `_id` field references exist in the input documents

## Adding a $lookup Stage

```javascript
// Join orders with users collection
{
  from: "users",
  localField: "userId",
  foreignField: "_id",
  as: "user"
}
```

Compass preview shows the joined `user` array embedded in each order document.

## Saving Pipelines

Compass allows you to save frequently used pipelines:
1. Build your pipeline
2. Click **Save Pipeline** in the top right
3. Give it a name (e.g., "Revenue by Category")
4. Access saved pipelines from the **Saved Pipelines** dropdown

## Exporting the Pipeline to Code

Once your pipeline is working correctly, export it to use in your application:

1. Click the **Export to Language** button
2. Select your target language (JavaScript, Python, Java, C#)
3. Copy the generated code

Example exported JavaScript:

```javascript
const pipeline = [
  { $match: { status: "completed" } },
  {
    $group: {
      _id: "$category",
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$amount" },
    },
  },
  { $sort: { totalRevenue: -1 } },
  {
    $project: {
      _id: 0,
      category: "$_id",
      totalRevenue: { $round: ["$totalRevenue", 2] },
      orderCount: 1,
      avgOrderValue: { $round: ["$avgOrderValue", 2] },
    },
  },
];

const result = await db.collection("orders").aggregate(pipeline).toArray();
```

## Using the Collation and Comment Options

In the settings panel at the top of the Aggregations tab you can set:
- **Collation** - for locale-aware string comparisons
- **Max Time MS** - timeout for slow pipelines
- **Comment** - appears in MongoDB logs to identify the query

## Viewing Explain Output

Click the **Explain** button to see the execution plan for your aggregation pipeline, showing which stages use indexes and which perform collection scans. This helps optimize pipeline performance before deploying to production.

## Summary

MongoDB Compass's aggregation builder lets you construct and test pipelines interactively with stage-by-stage output previews. Building complex aggregations in Compass is faster than writing and running shell commands because you can immediately see the result of each stage and fix issues before moving forward. Use the export feature to convert your tested pipeline directly into application code.
