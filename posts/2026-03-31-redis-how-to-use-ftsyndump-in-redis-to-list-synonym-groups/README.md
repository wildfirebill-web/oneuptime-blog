# How to Use FT.SYNDUMP in Redis to List Synonym Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Synonym, Full-Text Search, Query Expansion

Description: Learn how to use FT.SYNDUMP in Redis to inspect all synonym groups defined for a RediSearch index, auditing query expansion settings.

---

## What Is FT.SYNDUMP?

`FT.SYNDUMP` returns all synonym groups defined for a specific RediSearch index. It is the complement to `FT.SYNUPDATE` - while `SYNUPDATE` creates and modifies synonym groups, `SYNDUMP` lets you inspect what synonym relationships are currently active.

The output shows which terms belong to which synonym groups, giving you a complete picture of query expansion behavior for the index.

## Basic Syntax

```text
FT.SYNDUMP index
```

Parameter:
- `index` - the name of the RediSearch index

Returns a map of terms to their synonym group IDs.

## Basic Usage

```bash
# First create some synonym groups
FT.SYNUPDATE products vehicles car automobile vehicle
FT.SYNUPDATE products computers laptop notebook computer

# Dump all synonyms for the index
FT.SYNDUMP products
```

Sample output:

```text
1) "car"
2) 1) "vehicles"
3) "automobile"
4) 1) "vehicles"
5) "vehicle"
6) 1) "vehicles"
7) "laptop"
8) 1) "computers"
9) "notebook"
10) 1) "computers"
11) "computer"
12) 1) "computers"
```

The output is a flat array of alternating term and group-array pairs. Each term maps to a list of synonym group IDs it belongs to.

## Understanding the Output Format

The response pairs each term with an array of group IDs it participates in. A term can belong to multiple synonym groups:

```bash
# A term belonging to multiple groups
FT.SYNUPDATE products group1 apple fruit
FT.SYNUPDATE products group2 apple iphone device

FT.SYNDUMP products
# "apple" -> ["group1", "group2"]  (belongs to both groups)
```

## Python Example - Parsing SYNDUMP

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Define synonym groups
r.execute_command('FT.SYNUPDATE', 'articles', 'transport', 'car', 'bus', 'train', 'vehicle')
r.execute_command('FT.SYNUPDATE', 'articles', 'tech', 'laptop', 'computer', 'pc', 'notebook')

# Get all synonyms
raw = r.execute_command('FT.SYNDUMP', 'articles')

# Parse the flat array into a dict
synonyms = {}
for i in range(0, len(raw), 2):
    term = raw[i]
    groups = raw[i + 1]
    synonyms[term] = groups

print("Synonym mapping:")
for term, groups in sorted(synonyms.items()):
    print(f"  {term!r} -> groups: {groups}")
```

## Organizing into Group-to-Terms View

```python
import redis
from collections import defaultdict

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_synonym_groups(r, index_name):
    """Return a dict of group_id -> list of terms."""
    raw = r.execute_command('FT.SYNDUMP', index_name)
    if not raw:
        return {}

    group_to_terms = defaultdict(list)

    for i in range(0, len(raw), 2):
        term = raw[i]
        groups = raw[i + 1]
        for group in groups:
            group_to_terms[group].append(term)

    return dict(group_to_terms)

groups = get_synonym_groups(r, 'articles')
for group_id, terms in sorted(groups.items()):
    print(f"{group_id}: {sorted(terms)}")
```

## Auditing Synonym Groups Before Search

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def audit_synonyms(r, index_name):
    """Print a summary of all synonym groups for an index."""
    raw = r.execute_command('FT.SYNDUMP', index_name)

    if not raw:
        print(f"No synonym groups defined for index: {index_name}")
        return

    total_terms = len(raw) // 2
    print(f"Index: {index_name}")
    print(f"Total terms with synonyms: {total_terms}")
    print()

    from collections import defaultdict
    group_terms = defaultdict(list)
    for i in range(0, len(raw), 2):
        term = raw[i]
        for group in raw[i + 1]:
            group_terms[group].append(term)

    for group, terms in sorted(group_terms.items()):
        print(f"Group '{group}': {', '.join(sorted(terms))}")

audit_synonyms(r, 'products')
```

## Empty Index - No Synonyms

```bash
FT.CREATE empty_index ON HASH PREFIX 1 doc: SCHEMA content TEXT

FT.SYNDUMP empty_index
# Returns: (empty array)
```

## Exporting Synonyms for Migration

```python
import redis
import json
from collections import defaultdict

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def export_synonyms(r, index_name, output_file):
    raw = r.execute_command('FT.SYNDUMP', index_name)
    group_terms = defaultdict(set)

    for i in range(0, len(raw or []), 2):
        term = raw[i]
        for group in raw[i + 1]:
            group_terms[group].add(term)

    export = {
        group: sorted(terms)
        for group, terms in group_terms.items()
    }

    with open(output_file, 'w') as f:
        json.dump(export, f, indent=2)

    print(f"Exported {len(export)} synonym groups to {output_file}")

export_synonyms(r, 'products', '/tmp/synonyms_export.json')
```

## Summary

`FT.SYNDUMP` provides a complete view of all synonym groups defined for a RediSearch index. Since synonyms directly affect search recall by expanding query terms, auditing them regularly ensures your search behavior is intentional and up to date. The flat array output requires parsing to reconstruct the group-to-terms mapping, but the Python examples above demonstrate how to organize it into a more readable format. Use it alongside `FT.SYNUPDATE` to manage the full synonym lifecycle.
