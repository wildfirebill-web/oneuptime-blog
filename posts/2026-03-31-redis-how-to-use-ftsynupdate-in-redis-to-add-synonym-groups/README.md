# How to Use FT.SYNUPDATE in Redis to Add Synonym Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Synonym, Full-Text Search, Query Expansion

Description: Learn how to use FT.SYNUPDATE in Redis to define synonym groups that expand search queries, improving recall for semantically equivalent terms.

---

## What Is FT.SYNUPDATE?

`FT.SYNUPDATE` creates or updates a synonym group for a RediSearch index. Synonym groups allow you to define sets of terms that should be treated as interchangeable during search. When a query contains a term from a synonym group, results matching any term in the group are returned.

For example, if "car", "automobile", and "vehicle" are synonyms, a search for "car" will also return documents containing "automobile" or "vehicle".

## Basic Syntax

```text
FT.SYNUPDATE index synonym_group_id [SKIPINITIALSCAN] term [term ...]
```

Parameters:
- `index` - the RediSearch index name
- `synonym_group_id` - a string identifier for this synonym group (you assign it)
- `SKIPINITIALSCAN` - do not rescan existing documents (useful for large indices)
- `term` - two or more terms to include in the synonym group

## Setting Up a Sample Index

```bash
# Create an index
FT.CREATE products ON HASH PREFIX 1 prod: SCHEMA description TEXT

# Add some documents
HSET prod:1 description "Buy a new car today at great prices"
HSET prod:2 description "Used automobile for sale"
HSET prod:3 description "Electric vehicle available now"
HSET prod:4 description "Laptop computer on sale"
HSET prod:5 description "Notebook PC great deals"
```

## Creating a Synonym Group

```bash
# Make "car", "automobile", "vehicle" synonymous
FT.SYNUPDATE products transport_group car automobile vehicle
# Returns: OK

# Make "laptop", "notebook", "computer" synonymous (different group)
FT.SYNUPDATE products computer_group laptop notebook computer
# Returns: OK
```

## Querying with Synonyms Active

```bash
# Search for "car" - now returns car, automobile, and vehicle results
FT.SEARCH products "@description:(car)"
# Returns prod:1, prod:2, prod:3 (all three variants matched)

# Search for "laptop"
FT.SEARCH products "@description:(laptop)"
# Returns prod:4, prod:5 (laptop and notebook matched)
```

## Updating an Existing Synonym Group

You can add more terms to an existing group by reusing the group ID:

```bash
# Initially
FT.SYNUPDATE products color_group red crimson

# Add more synonyms to the same group
FT.SYNUPDATE products color_group red crimson scarlet ruby
# Returns: OK (group now has 4 terms)
```

## Using SKIPINITIALSCAN for Large Indices

By default, updating synonyms triggers a rescan of all existing documents to update their synonym mappings. For large indices, this can be expensive:

```bash
# Skip rescan - new synonyms apply only to newly indexed documents
FT.SYNUPDATE products new_group SKIPINITIALSCAN term1 term2
```

Use `SKIPINITIALSCAN` only when you are adding synonyms for future documents and do not need existing documents to match.

## Python Example

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Create index
r.execute_command('FT.CREATE', 'articles',
    'ON', 'HASH',
    'PREFIX', '1', 'article:',
    'SCHEMA', 'title', 'TEXT', 'body', 'TEXT'
)

# Add documents
r.hset('article:1', mapping={'title': 'Redis Caching Strategies', 'body': 'Cache invalidation is hard'})
r.hset('article:2', mapping={'title': 'In-Memory Store Best Practices', 'body': 'Store data in memory'})
r.hset('article:3', mapping={'title': 'Key-Value Database Guide', 'body': 'Key value storage systems'})

# Define synonyms: redis, cache, in-memory, key-value are related
r.execute_command('FT.SYNUPDATE', 'articles', 'redis_synonyms',
    'redis', 'cache', 'in-memory', 'key-value')

# Search for redis - returns all three documents
results = r.execute_command('FT.SEARCH', 'articles', 'redis')
print(f"Results: {results[0]} documents found")
```

## Viewing Defined Synonyms

Use `FT.SYNDUMP` to inspect all synonym groups:

```bash
FT.SYNDUMP products
# Returns synonym groups and which terms map to which groups
```

## Multiple Synonym Groups in Practice

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# E-commerce synonym groups
synonym_groups = {
    'colors_red': ['red', 'crimson', 'scarlet', 'ruby'],
    'colors_blue': ['blue', 'navy', 'azure', 'cobalt'],
    'sizes': ['large', 'big', 'xl', 'extra-large'],
    'condition': ['new', 'brand-new', 'unused', 'sealed'],
    'sale': ['sale', 'discount', 'clearance', 'offer', 'deal'],
}

for group_id, terms in synonym_groups.items():
    r.execute_command('FT.SYNUPDATE', 'ecommerce', group_id, *terms)
    print(f"Added synonym group: {group_id} with {len(terms)} terms")
```

## Summary

`FT.SYNUPDATE` enables query expansion in RediSearch by defining groups of semantically equivalent terms. When a search query contains a term from a synonym group, Redis automatically includes results matching all other terms in that group, improving search recall without requiring users to know all possible phrasings. Use distinct group IDs for different synonym sets, combine with `FT.SYNDUMP` to inspect them, and use `SKIPINITIALSCAN` judiciously when working with large existing indices.
