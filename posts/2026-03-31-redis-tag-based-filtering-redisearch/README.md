# How to Implement Tag-Based Filtering with RediSearch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Tag, Filter

Description: Use RediSearch TAG fields to build fast multi-faceted filtering for e-commerce, content platforms, and product catalogs.

---

Tag-based filtering lets users narrow down search results by categories, attributes, and labels. RediSearch's TAG field type is optimized for exact-match filtering and supports multi-value fields, making it ideal for faceted search.

## Defining a Schema with TAG Fields

```bash
FT.CREATE product_idx ON HASH PREFIX 1 product:
  SCHEMA
    name TEXT WEIGHT 5.0
    brand TAG
    color TAG SEPARATOR ","
    size TAG SEPARATOR ","
    category TAG
    price NUMERIC SORTABLE
    in_stock TAG
```

## Adding Products

Store products with multiple tag values using comma separation:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def add_product(pid: str, name: str, brand: str, colors: list,
                sizes: list, category: str, price: float, in_stock: bool):
    r.hset(f"product:{pid}", mapping={
        "name": name,
        "brand": brand,
        "color": ",".join(colors),
        "size": ",".join(sizes),
        "category": category,
        "price": price,
        "in_stock": "true" if in_stock else "false"
    })

add_product("p001", "Classic Sneaker", "Nike",
            ["white", "black", "red"], ["S", "M", "L", "XL"],
            "footwear", 89.99, True)
add_product("p002", "Running Shoe", "Adidas",
            ["blue", "black"], ["M", "L"],
            "footwear", 120.00, True)
```

## Single Tag Filter

Filter by a single tag value using curly braces:

```python
def filter_by_brand(brand: str):
    query = f"@brand:{{{brand}}}"
    return r.execute_command('FT.SEARCH', 'product_idx', query,
                             'LIMIT', 0, 20)

nike_products = filter_by_brand("Nike")
```

## Multi-Tag OR Filter

Match any of several tag values using the pipe operator:

```python
def filter_by_colors(colors: list):
    color_filter = "|".join(colors)
    query = f"@color:{{{color_filter}}}"
    return r.execute_command('FT.SEARCH', 'product_idx', query,
                             'LIMIT', 0, 20)

# Products available in white OR black
results = filter_by_colors(["white", "black"])
```

## Multi-Tag AND Filter

Require multiple tags to be present simultaneously:

```python
def filter_by_multiple_tags(brand: str, color: str, size: str):
    query = (
        f"@brand:{{{brand}}} "
        f"@color:{{{color}}} "
        f"@size:{{{size}}}"
    )
    return r.execute_command('FT.SEARCH', 'product_idx', query,
                             'LIMIT', 0, 20)

# Nike products in white AND size M
results = filter_by_multiple_tags("Nike", "white", "M")
```

## Combining Tags with Full-Text and Numeric Filters

Build a faceted search function accepting multiple optional filters:

```python
def faceted_search(text: str = None, brand: str = None,
                   color: str = None, max_price: float = None,
                   in_stock: bool = True):
    parts = []

    if text:
        parts.append(text)
    if brand:
        parts.append(f"@brand:{{{brand}}}")
    if color:
        parts.append(f"@color:{{{color}}}")
    if max_price is not None:
        parts.append(f"@price:[0 {max_price}]")
    if in_stock:
        parts.append("@in_stock:{true}")

    query = " ".join(parts) if parts else "*"

    return r.execute_command(
        'FT.SEARCH', 'product_idx', query,
        'SORTBY', 'price', 'ASC',
        'LIMIT', 0, 20
    )

results = faceted_search(text="sneaker", brand="Nike",
                         color="white", max_price=100)
```

## Counting Facets with FT.AGGREGATE

Get facet counts to display filter options in the UI:

```bash
FT.AGGREGATE product_idx "*"
  GROUPBY 1 @brand
  REDUCE COUNT 0 AS count
  SORTBY 2 @count DESC
```

## Summary

RediSearch TAG fields provide efficient exact-match filtering that is essential for e-commerce and content platforms. By combining TAG, TEXT, and NUMERIC fields in a single query, you can implement full faceted search with sub-millisecond latency. Use FT.AGGREGATE to compute facet counts and keep your filter UI in sync with actual inventory.
