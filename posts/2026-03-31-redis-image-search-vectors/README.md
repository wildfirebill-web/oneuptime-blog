# How to Build an Image Search System with Redis Vectors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Search, Image Search, Embedding

Description: Build a reverse image search system using CLIP image embeddings stored in Redis Vector Search to find visually similar images.

---

Reverse image search finds visually similar images by comparing their embeddings rather than metadata. Redis Vector Search combined with CLIP embeddings provides a fast, accurate image similarity system you can deploy in minutes.

## How CLIP Works

CLIP (Contrastive Language-Image Pre-training) maps both images and text into the same embedding space, allowing cross-modal search. You can search images using either another image or a text description.

## Installation

```bash
pip install redis transformers torch pillow numpy
```

## Creating the Image Vector Index

```python
import redis
import numpy as np

r = redis.Redis(host='localhost', port=6379, decode_responses=False)

# CLIP ViT-B/32 produces 512-dimensional embeddings
r.execute_command(
    'FT.CREATE', 'image_idx', 'ON', 'HASH',
    'PREFIX', '1', 'img:',
    'SCHEMA',
    'filename', 'TEXT',
    'label', 'TAG',
    'embedding', 'VECTOR', 'HNSW', '6',
    'TYPE', 'FLOAT32', 'DIM', '512',
    'DISTANCE_METRIC', 'COSINE'
)
```

## Generating Image Embeddings with CLIP

```python
from transformers import CLIPProcessor, CLIPModel
from PIL import Image
import torch

model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

def embed_image(image_path: str) -> bytes:
    image = Image.open(image_path).convert("RGB")
    inputs = processor(images=image, return_tensors="pt")
    with torch.no_grad():
        features = model.get_image_features(**inputs)
    vec = features[0].numpy()
    vec = vec / np.linalg.norm(vec)  # normalize
    return vec.astype(np.float32).tobytes()

def embed_text(text: str) -> bytes:
    inputs = processor(text=[text], return_tensors="pt", padding=True)
    with torch.no_grad():
        features = model.get_text_features(**inputs)
    vec = features[0].numpy()
    vec = vec / np.linalg.norm(vec)
    return vec.astype(np.float32).tobytes()
```

## Indexing Images

```python
def index_image(image_id: str, image_path: str,
                filename: str, label: str):
    embedding = embed_image(image_path)
    r.hset(f"img:{image_id}", mapping={
        "filename": filename.encode(),
        "label": label.encode(),
        "embedding": embedding
    })

# Index a set of product images
index_image("001", "/images/red_sneaker.jpg", "red_sneaker.jpg", "footwear")
index_image("002", "/images/blue_sneaker.jpg", "blue_sneaker.jpg", "footwear")
index_image("003", "/images/laptop_bag.jpg", "laptop_bag.jpg", "bags")
```

## Searching by Image

Find the most visually similar images to a query image:

```python
def search_by_image(query_image_path: str, top_k: int = 5):
    query_vec = embed_image(query_image_path)

    result = r.execute_command(
        'FT.SEARCH', 'image_idx',
        f'*=>[KNN {top_k} @embedding $vec AS score]',
        'PARAMS', 2, 'vec', query_vec,
        'RETURN', 3, 'filename', 'label', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    results = []
    for i in range(1, len(result), 2):
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        results.append(fields)
    return results

similar = search_by_image("/query/my_shoe.jpg")
```

## Searching by Text Description

CLIP lets you search images using natural language:

```python
def search_by_text(text_query: str, top_k: int = 5,
                   label_filter: str = None):
    query_vec = embed_text(text_query)

    if label_filter:
        search_query = f"(@label:{{{label_filter}}})=>[KNN {top_k} @embedding $vec AS score]"
    else:
        search_query = f"*=>[KNN {top_k} @embedding $vec AS score]"

    result = r.execute_command(
        'FT.SEARCH', 'image_idx', search_query,
        'PARAMS', 2, 'vec', query_vec,
        'RETURN', 3, 'filename', 'label', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    results = []
    for i in range(1, len(result), 2):
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        results.append(fields)
    return results

results = search_by_text("red running shoes for sports")
```

## Summary

Redis Vector Search with CLIP embeddings provides a powerful foundation for both image-to-image and text-to-image search. By storing normalized CLIP embeddings as HNSW vectors in Redis, you get millisecond-latency visual search that scales with your catalog. The cross-modal nature of CLIP means the same index serves both image uploads and text queries.
