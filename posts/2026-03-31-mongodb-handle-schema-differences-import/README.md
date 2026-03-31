# How to Handle Schema Differences During MongoDB Data Import

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema, Import, Migration, Data Engineering

Description: Learn strategies for handling schema differences, missing fields, and type mismatches when importing data into MongoDB from heterogeneous sources.

---

Real-world data import tasks rarely involve perfectly structured source data. Fields may be missing, named differently, stored as wrong types, or organized in incompatible structures. Building schema-aware import pipelines that handle these differences gracefully is essential for reliable data loading.

## Common Schema Difference Patterns

Before writing transformation code, identify the categories of differences:

```text
1. Field renaming: "first_name" -> "firstName"
2. Missing fields: some documents lack optional fields
3. Type mismatches: "123" (string) should be 123 (integer)
4. Null vs missing: null value vs field absent entirely
5. Nested vs flat: "address_city" vs "address.city"
6. Date format differences: "2024-01-15" vs Unix timestamps
```

## Field Mapping and Renaming

Define a mapping dictionary to rename fields during transformation:

```python
FIELD_MAP = {
    "first_name": "firstName",
    "last_name": "lastName",
    "email_address": "email",
    "phone_number": "phone",
    "date_of_birth": "dob",
    "created_date": "createdAt",
}

def apply_field_map(doc, field_map):
    transformed = {}
    for source_key, value in doc.items():
        target_key = field_map.get(source_key, source_key)
        transformed[target_key] = value
    return transformed
```

## Type Coercion

Handle type mismatches with safe conversion functions:

```python
from datetime import datetime

def safe_int(value, default=None):
    try:
        return int(float(str(value).strip()))
    except (ValueError, TypeError):
        return default

def safe_float(value, default=None):
    try:
        return float(str(value).strip())
    except (ValueError, TypeError):
        return default

def safe_date(value, formats=None, default=None):
    if isinstance(value, datetime):
        return value
    formats = formats or ["%Y-%m-%d", "%m/%d/%Y", "%d-%m-%Y", "%Y-%m-%dT%H:%M:%S"]
    for fmt in formats:
        try:
            return datetime.strptime(str(value).strip(), fmt)
        except ValueError:
            continue
    return default
```

## Default Values for Missing Fields

Apply defaults for fields that may be absent in some source records:

```python
FIELD_DEFAULTS = {
    "isActive": True,
    "emailVerified": False,
    "role": "user",
    "loginCount": 0,
    "preferences": {},
}

def apply_defaults(doc, defaults):
    for field, default_value in defaults.items():
        if field not in doc or doc[field] is None:
            doc[field] = default_value
    return doc
```

## Flattening Nested Source Data

Convert dot-notation keys to nested documents:

```python
def unflatten(doc, separator="_"):
    result = {}
    for key, value in doc.items():
        parts = key.split(separator, 1)
        if len(parts) == 2 and isinstance(result.get(parts[0]), dict):
            result[parts[0]][parts[1]] = value
        elif len(parts) == 2:
            result[parts[0]] = {parts[1]: value}
        else:
            result[key] = value
    return result

# "address_city" -> "address": {"city": ...}
```

## Validation Before Insert

Validate required fields and reject invalid documents:

```python
REQUIRED_FIELDS = ["email", "createdAt"]

def validate_document(doc):
    errors = []
    for field in REQUIRED_FIELDS:
        if not doc.get(field):
            errors.append(f"Missing required field: {field}")
    if doc.get("email") and "@" not in doc["email"]:
        errors.append(f"Invalid email: {doc['email']}")
    return errors

def transform_and_validate(raw_doc):
    doc = apply_field_map(raw_doc, FIELD_MAP)
    doc["age"] = safe_int(doc.get("age"))
    doc["salary"] = safe_float(doc.get("salary"))
    doc["createdAt"] = safe_date(doc.get("createdAt"))
    doc = apply_defaults(doc, FIELD_DEFAULTS)

    errors = validate_document(doc)
    return doc, errors
```

## Import with Error Logging

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["myapp"]

valid_docs = []
invalid_docs = []

for raw in source_records:
    doc, errors = transform_and_validate(raw)
    if errors:
        invalid_docs.append({"original": raw, "errors": errors})
    else:
        valid_docs.append(doc)

if valid_docs:
    db["users"].insert_many(valid_docs, ordered=False)
    print(f"Inserted {len(valid_docs)} valid documents")

if invalid_docs:
    db["import_errors"].insert_many(invalid_docs)
    print(f"Logged {len(invalid_docs)} invalid documents to import_errors")
```

## Summary

Handling schema differences during MongoDB import requires a systematic approach: define field mappings for renames, write type coercion functions for mismatched types, apply defaults for missing fields, and validate required fields before insertion. Storing invalid documents in a separate error collection rather than failing the entire import lets you process the clean records while reviewing and fixing problematic ones separately. This pattern makes import pipelines resilient to the inevitable imperfections in real-world data sources.
