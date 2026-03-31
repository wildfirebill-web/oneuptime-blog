# How to Use FT.DICTDEL in Redis to Remove Dictionary Terms

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Dictionary, Spell Check, Full-Text Search

Description: Learn how to use FT.DICTDEL in Redis to remove specific terms from a RediSearch custom dictionary, keeping your spell check data accurate.

---

## What Is FT.DICTDEL?

`FT.DICTDEL` removes one or more terms from a named RediSearch dictionary. It is the counterpart to `FT.DICTADD` and is used to maintain dictionary accuracy by removing outdated, incorrect, or no-longer-relevant terms.

Dictionaries in RediSearch are used by `FT.SPELLCHECK` to recognize valid domain-specific terms that should not be flagged as misspelled.

## Basic Syntax

```text
FT.DICTDEL dict term [term ...]
```

Parameters:
- `dict` - the name of the dictionary
- `term` - one or more terms to remove

Returns the number of terms successfully deleted. Terms that do not exist in the dictionary are not counted.

## Removing a Single Term

```bash
# First, populate a dictionary
FT.DICTADD products_dict "redis" "nosql" "oldterm" "deprecated"

# Remove a single outdated term
FT.DICTDEL products_dict "oldterm"
# Returns: 1

# Verify it was removed
FT.DICTDUMP products_dict
# 1) "redis"
# 2) "nosql"
# 3) "deprecated"
```

## Removing Multiple Terms at Once

```bash
FT.DICTDEL products_dict "deprecated" "nosql"
# Returns: 2

FT.DICTDUMP products_dict
# 1) "redis"
```

## Handling Non-Existent Terms

Attempting to delete a term that is not in the dictionary returns 0 for that term:

```bash
# Term "nonexistent" is not in the dictionary
FT.DICTDEL products_dict "nonexistent"
# Returns: 0

# Mix of existing and non-existing
FT.DICTDEL products_dict "redis" "nonexistent"
# Returns: 1 (only "redis" was deleted)
```

## Python Example

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Populate dictionary
r.execute_command('FT.DICTADD', 'tech_dict', 'redis', 'mysql', 'deprecated_api', 'old_feature')

# Remove deprecated terms
removed = r.execute_command('FT.DICTDEL', 'tech_dict', 'deprecated_api', 'old_feature')
print(f"Removed {removed} terms")

# Verify
remaining = r.execute_command('FT.DICTDUMP', 'tech_dict')
print(f"Remaining terms: {remaining}")
```

## Bulk Removal by Loading from a List

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Terms to remove (e.g., loaded from config or database)
terms_to_remove = [
    'v1_endpoint',
    'legacy_schema',
    'deprecated_field',
    'old_product_name',
]

# Remove all at once (variadic call)
if terms_to_remove:
    removed = r.execute_command('FT.DICTDEL', 'my_dict', *terms_to_remove)
    print(f"Deleted {removed} of {len(terms_to_remove)} terms")
```

## Updating a Dictionary (Add and Remove Together)

```python
import redis

def update_dictionary(r, dict_name, add_terms=None, remove_terms=None):
    """Update a RediSearch dictionary - add new terms and remove old ones."""
    results = {}

    if add_terms:
        added = r.execute_command('FT.DICTADD', dict_name, *add_terms)
        results['added'] = added

    if remove_terms:
        removed = r.execute_command('FT.DICTDEL', dict_name, *remove_terms)
        results['removed'] = removed

    all_terms = r.execute_command('FT.DICTDUMP', dict_name)
    results['total'] = len(all_terms) if all_terms else 0

    return results

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

result = update_dictionary(
    r,
    'product_names',
    add_terms=['new_product_v2', 'feature_x'],
    remove_terms=['old_product_v1', 'removed_feature']
)
print(result)
```

## Clearing an Entire Dictionary

There is no single command to delete a dictionary. To clear all terms, dump the dictionary and delete everything:

```bash
# Get all terms
FT.DICTDUMP my_dict

# Delete all (if you know all terms)
FT.DICTDEL my_dict term1 term2 term3 ...
```

Or in Python:

```python
import redis

def clear_dictionary(r, dict_name):
    """Remove all terms from a dictionary."""
    terms = r.execute_command('FT.DICTDUMP', dict_name)
    if terms:
        removed = r.execute_command('FT.DICTDEL', dict_name, *terms)
        print(f"Cleared {removed} terms from {dict_name}")
    else:
        print(f"Dictionary {dict_name} is already empty")

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
clear_dictionary(r, 'old_dict')
```

## Summary

`FT.DICTDEL` is the complement to `FT.DICTADD`, allowing you to keep RediSearch custom dictionaries accurate and current by removing obsolete or incorrect terms. It accepts multiple terms in a single call and returns the count of successfully deleted entries. Regular dictionary maintenance ensures that `FT.SPELLCHECK` gives accurate, relevant suggestions without being confused by stale domain-specific terms.
