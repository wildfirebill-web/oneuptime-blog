# How to Use MongoDB with Jupyter Notebooks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Jupyter, Python, Data Analysis, PyMongo

Description: Learn how to connect MongoDB to Jupyter Notebooks using PyMongo and Pandas to query, analyze, and visualize data from your MongoDB collections interactively.

---

Jupyter Notebooks provide an interactive environment for exploring MongoDB data with Python. Combining PyMongo, Pandas, and Matplotlib lets you run queries, transform results, and visualize data without leaving the notebook.

## Install Dependencies

```bash
pip install jupyter pymongo pandas matplotlib seaborn
```

Launch Jupyter:

```bash
jupyter notebook
```

## Connect to MongoDB

```python
from pymongo import MongoClient
import pandas as pd
import matplotlib.pyplot as plt

# Connect to MongoDB
client = MongoClient("mongodb://localhost:27017/")
db = client["salesdb"]
orders = db["orders"]

# Verify connection
print(f"Collections: {db.list_collection_names()}")
```

For MongoDB Atlas, use your connection string:

```python
client = MongoClient("mongodb+srv://<user>:<password>@cluster0.example.mongodb.net/")
```

## Query Data and Load into Pandas

```python
# Run an aggregation and load results into a DataFrame
pipeline = [
    { "$match": { "status": "completed" } },
    { "$group": {
        "_id": "$category",
        "total_revenue": { "$sum": "$amount" },
        "order_count": { "$sum": 1 }
    }},
    { "$sort": { "total_revenue": -1 } }
]

results = list(orders.aggregate(pipeline))
df = pd.DataFrame(results)
df.rename(columns={"_id": "category"}, inplace=True)
df.head(10)
```

## Analyze the Data

```python
# Basic statistics
print(df.describe())

# Top 5 categories by revenue
top5 = df.nlargest(5, "total_revenue")
print(top5)

# Average order value per category
df["avg_order_value"] = df["total_revenue"] / df["order_count"]
df.sort_values("avg_order_value", ascending=False).head(10)
```

## Visualize Results

```python
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Bar chart - revenue by category
top5.plot.bar(x="category", y="total_revenue", ax=axes[0], color="steelblue")
axes[0].set_title("Top 5 Categories by Revenue")
axes[0].set_ylabel("Total Revenue ($)")
axes[0].tick_params(axis="x", rotation=45)

# Scatter plot - order count vs revenue
axes[1].scatter(df["order_count"], df["total_revenue"], alpha=0.6)
axes[1].set_xlabel("Order Count")
axes[1].set_ylabel("Total Revenue ($)")
axes[1].set_title("Order Count vs Revenue")

plt.tight_layout()
plt.show()
```

## Use a Connection Helper Function

For notebooks shared across a team, create a reusable connection helper:

```python
import os
from pymongo import MongoClient

def get_db(db_name: str) -> object:
    uri = os.environ.get("MONGODB_URI", "mongodb://localhost:27017/")
    client = MongoClient(uri)
    return client[db_name]

db = get_db("salesdb")
```

Store the URI in a `.env` file or environment variable rather than hardcoding credentials in the notebook.

## Export Analysis Results Back to MongoDB

```python
# Write enriched analysis results back to a collection
summary_collection = db["revenue_summary"]
summary_collection.drop()
summary_collection.insert_many(df.to_dict("records"))
print(f"Inserted {len(df)} summary documents")
```

## Summary

MongoDB integrates cleanly with Jupyter Notebooks through PyMongo and Pandas. Use aggregation pipelines to pre-process data on the MongoDB server, load results into DataFrames for analysis, and Matplotlib or Seaborn for visualization. Store credentials in environment variables rather than in notebook cells to keep them out of version control.
