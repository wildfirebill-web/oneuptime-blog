# How to Use FT.DICTDUMP in Redis to List Dictionary Terms

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Dictionary, Spell Check, Full-Text Search

Description: Learn how to use FT.DICTDUMP in Redis to list all terms in a RediSearch custom dictionary for auditing and management purposes.

---

## What Is FT.DICTDUMP?

`FT.DICTDUMP` returns all terms stored in a named RediSearch dictionary. It is primarily used for inspection and auditing of custom dictionaries that were built using `FT.DICTADD`. These dictionaries are used by `FT.SPELLCHECK` to recognize valid domain-specific terms.

## Basic Syntax

```text
FT.DICTDUMP dict
```

Parameter:
- `dict` - the name of the dictionary to list

Returns an array of all terms in the dictionary. Returns an empty array if the dictionary has no terms or does not exist.

## Basic Usage

```bash
# First add some terms
FT.DICTADD tech_terms "redis" "nosql" "kubernetes" "docker" "terraform"

# List all terms
FT.DICTDUMP tech_terms
# Returns:
# 1) "redis"
# 2) "nosql"
# 3) "kubernetes"
# 4) "docker"
# 5) "terraform"
```

## Empty Dictionary

```bash
# Non-existent dictionary returns empty array
FT.DICTDUMP unknown_dict
# Returns: (empty array)

# After all terms are deleted
FT.DICTDEL tech_terms "redis" "nosql" "kubernetes" "docker" "terraform"
FT.DICTDUMP tech_terms
# Returns: (empty array)
```

## Python Example

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Add terms
r.execute_command('FT.DICTADD', 'my_dict', 'alpha', 'beta', 'gamma', 'delta')

# List all terms
terms = r.execute_command('FT.DICTDUMP', 'my_dict')
print(f"Dictionary contains {len(terms)} terms:")
for term in sorted(terms):
    print(f"  - {term}")
```

## Auditing Multiple Dictionaries

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Assume these dictionaries exist
dict_names = ['brands_dict', 'tech_dict', 'product_dict']

for dict_name in dict_names:
    terms = r.execute_command('FT.DICTDUMP', dict_name)
    count = len(terms) if terms else 0
    print(f"{dict_name}: {count} terms")
    if count > 0:
        print(f"  Sample: {terms[:3]}")
```

## Exporting a Dictionary

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def export_dictionary(r, dict_name, output_file):
    """Export a dictionary to a JSON file."""
    terms = r.execute_command('FT.DICTDUMP', dict_name)
    if not terms:
        print(f"Dictionary {dict_name} is empty or does not exist")
        return

    export_data = {
        'dictionary': dict_name,
        'terms': sorted(terms),
        'count': len(terms),
    }

    with open(output_file, 'w') as f:
        json.dump(export_data, f, indent=2)

    print(f"Exported {len(terms)} terms to {output_file}")

export_dictionary(r, 'tech_dict', '/tmp/tech_dict_backup.json')
```

## Importing a Dictionary from a File

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def import_dictionary(r, dict_name, input_file):
    """Import terms from a JSON file into a dictionary."""
    with open(input_file, 'r') as f:
        data = json.load(f)

    terms = data.get('terms', [])
    if not terms:
        print("No terms to import")
        return

    added = r.execute_command('FT.DICTADD', dict_name, *terms)
    print(f"Imported {added} terms into {dict_name}")

    # Verify
    current = r.execute_command('FT.DICTDUMP', dict_name)
    print(f"Dictionary now has {len(current)} terms total")

import_dictionary(r, 'tech_dict', '/tmp/tech_dict_backup.json')
```

## Comparing Two Dictionaries

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def compare_dicts(r, dict1_name, dict2_name):
    """Find terms unique to each dictionary and shared terms."""
    terms1 = set(r.execute_command('FT.DICTDUMP', dict1_name) or [])
    terms2 = set(r.execute_command('FT.DICTDUMP', dict2_name) or [])

    only_in_1 = terms1 - terms2
    only_in_2 = terms2 - terms1
    shared = terms1 & terms2

    print(f"Only in {dict1_name}: {sorted(only_in_1)}")
    print(f"Only in {dict2_name}: {sorted(only_in_2)}")
    print(f"Shared: {sorted(shared)}")
    return only_in_1, only_in_2, shared

compare_dicts(r, 'dict_v1', 'dict_v2')
```

## Summary

`FT.DICTDUMP` is the inspection command for RediSearch custom dictionaries, returning all terms currently stored in a named dictionary. It is useful for auditing dictionary contents before spell checking operations, exporting dictionaries for backup or migration, and comparing dictionaries between environments. Combine it with `FT.DICTADD` and `FT.DICTDEL` to build a complete dictionary management workflow.
