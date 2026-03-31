# How to Implement Search-As-You-Type with Atlas Search in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mongodb, Atlas Search, Search-As-You-Type, Autocomplete

Description: Learn how to build a complete search-as-you-type feature in MongoDB Atlas using autocomplete indexes, debouncing, relevance scoring, and frontend integration.

---

## Overview

Search-as-you-type (also called type-ahead or incremental search) updates results with every keystroke. MongoDB Atlas Search's `autocomplete` operator combined with appropriate index configuration makes this possible directly from MongoDB. This guide covers the full stack - from index creation to frontend integration.

## Step 1 - Create the Autocomplete Index

Create an Atlas Search index with autocomplete fields:

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": [
        {
          "type": "autocomplete",
          "analyzer": "lucene.standard",
          "tokenization": "edgeGram",
          "minGrams": 2,
          "maxGrams": 15,
          "foldDiacritics": true
        },
        {
          "type": "string"
        }
      ],
      "category": {
        "type": "string"
      },
      "price": {
        "type": "number"
      },
      "inStock": {
        "type": "boolean"
      }
    }
  }
}
```

The `edgeGram` tokenization creates prefix indexes - for "laptop", it indexes "la", "lap", "lapt", "lapto", "laptop" so any prefix returns the full word.

## Step 2 - Build the Search Query

```javascript
// search-as-you-type query function
async function searchProducts(queryStr, options = {}) {
  const {
    limit = 10,
    category = null,
    maxPrice = null,
    inStockOnly = false
  } = options;
  
  // Build filter array
  let filters = [];
  if (inStockOnly) {
    filters.push({ equals: { path: "inStock", value: true } });
  }
  if (maxPrice) {
    filters.push({ range: { path: "price", lte: maxPrice } });
  }
  if (category) {
    filters.push({ text: { query: category, path: "category" } });
  }
  
  let searchStage = {
    compound: {
      should: [
        // Autocomplete for prefix matching as user types
        {
          autocomplete: {
            query: queryStr,
            path: "name",
            fuzzy: { maxEdits: 1, prefixLength: 2 },
            score: { boost: { value: 3 } }  // boost autocomplete matches
          }
        },
        // Full text search for complete words
        {
          text: {
            query: queryStr,
            path: "name",
            score: { boost: { value: 1 } }
          }
        }
      ],
      minimumShouldMatch: 1
    }
  };
  
  // Add filters if any
  if (filters.length > 0) {
    searchStage.compound.filter = filters;
  }
  
  const db = client.db("mydb");
  return await db.collection("products").aggregate([
    { $search: { index: "autocomplete_idx", ...searchStage } },
    { $limit: limit },
    {
      $project: {
        _id: 1,
        name: 1,
        price: 1,
        category: 1,
        inStock: 1,
        score: { $meta: "searchScore" }
      }
    }
  ]).toArray();
}
```

## Step 3 - Create the Backend API

```javascript
// Express.js API endpoint
const express = require('express');
const { MongoClient } = require('mongodb');

const app = express();
const client = new MongoClient(process.env.MONGODB_URI);

app.get('/api/search', async (req, res) => {
  const { q, category, maxPrice, inStock } = req.query;
  
  // Require at least 2 characters
  if (!q || q.trim().length < 2) {
    return res.json({ results: [], query: q });
  }
  
  try {
    const results = await searchProducts(q.trim(), {
      limit: 10,
      category: category || null,
      maxPrice: maxPrice ? parseFloat(maxPrice) : null,
      inStockOnly: inStock === 'true'
    });
    
    res.json({ results, query: q });
  } catch (err) {
    console.error('Search error:', err);
    res.status(500).json({ error: 'Search failed' });
  }
});

app.listen(3000, async () => {
  await client.connect();
  console.log('Server running on port 3000');
});
```

## Step 4 - Frontend Integration with Debouncing

Debouncing prevents a query on every single keypress:

```javascript
// search-ui.js

let searchTimeout = null;
const DEBOUNCE_MS = 200;  // wait 200ms after last keystroke

const searchInput = document.getElementById('search-input');
const resultsContainer = document.getElementById('search-results');

searchInput.addEventListener('input', function(event) {
  const query = event.target.value.trim();
  
  // Clear pending search
  clearTimeout(searchTimeout);
  
  // Hide results if query is too short
  if (query.length < 2) {
    resultsContainer.innerHTML = '';
    return;
  }
  
  // Show loading state
  resultsContainer.innerHTML = '<div class="loading">Searching...</div>';
  
  // Debounce the actual search
  searchTimeout = setTimeout(async function() {
    try {
      const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
      const data = await response.json();
      
      if (data.query !== searchInput.value.trim()) {
        // Stale response - ignore
        return;
      }
      
      renderResults(data.results);
    } catch (err) {
      resultsContainer.innerHTML = '<div class="error">Search unavailable</div>';
    }
  }, DEBOUNCE_MS);
});

function renderResults(results) {
  if (results.length === 0) {
    resultsContainer.innerHTML = '<div class="no-results">No results found</div>';
    return;
  }
  
  const html = results.map(function(result) {
    return `
      <div class="result-item" data-id="${result._id}">
        <span class="result-name">${result.name}</span>
        <span class="result-price">$${result.price.toFixed(2)}</span>
        <span class="result-stock ${result.inStock ? 'in-stock' : 'out-of-stock'}">
          ${result.inStock ? 'In Stock' : 'Out of Stock'}
        </span>
      </div>
    `;
  }).join('');
  
  resultsContainer.innerHTML = html;
}
```

## Handling Keyboard Navigation

```javascript
// Add keyboard navigation to results
let selectedIndex = -1;

searchInput.addEventListener('keydown', function(event) {
  const items = resultsContainer.querySelectorAll('.result-item');
  
  if (event.key === 'ArrowDown') {
    selectedIndex = Math.min(selectedIndex + 1, items.length - 1);
    highlightResult(items, selectedIndex);
    event.preventDefault();
  } else if (event.key === 'ArrowUp') {
    selectedIndex = Math.max(selectedIndex - 1, -1);
    highlightResult(items, selectedIndex);
    event.preventDefault();
  } else if (event.key === 'Enter' && selectedIndex >= 0) {
    items[selectedIndex].click();
    event.preventDefault();
  } else if (event.key === 'Escape') {
    resultsContainer.innerHTML = '';
    selectedIndex = -1;
  }
});

function highlightResult(items, index) {
  items.forEach((item, i) => {
    item.classList.toggle('selected', i === index);
  });
}
```

## Performance Considerations

```text
minGrams: 2    - Minimum query length before autocomplete kicks in
maxGrams: 15   - Limits index size; longer prefixes fall back to full text

Index size grows with maxGrams. For a field with long text, start with:
  minGrams: 2
  maxGrams: 10

Debounce at 150-250ms to balance responsiveness and server load
Set minimum query length to 2 characters to avoid very broad matches
```

## Summary

Building search-as-you-type with MongoDB Atlas Search requires an autocomplete index with edge n-gram tokenization, a backend API that queries the `autocomplete` operator, and a frontend that debounces keystrokes before firing requests. Combining autocomplete with text search using the `compound` operator with `should` clauses ensures you catch both prefix matches and full-word matches. Debouncing at 150-250ms reduces server load while keeping the experience responsive, and handling stale responses prevents race conditions when multiple queries are in flight.
