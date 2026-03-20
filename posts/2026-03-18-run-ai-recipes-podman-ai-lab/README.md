# How to Run AI Recipes with Podman AI Lab

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, AI, Machine Learning, Recipes, RAG, Chatbot

Description: Learn how to use the pre-built AI recipes in Podman AI Lab to quickly deploy chatbots, summarizers, and RAG applications.

---

> AI Recipes are ready-to-run application templates that pair a model with a purpose-built frontend.

Podman AI Lab includes a collection of AI Recipes, which are complete application stacks that combine a model server with a frontend application. These recipes let you deploy functional AI applications like chatbots, code generators, and retrieval-augmented generation (RAG) systems with a single click. This guide walks through discovering, running, and customizing recipes.

---

## Prerequisites

```bash
# Ensure Podman machine has enough resources for recipe containers

podman machine inspect --format 'CPUs: {{.Resources.CPUs}}, RAM: {{.Resources.Memory}}MB'

# Recipes run multiple containers, so allocate enough resources
podman machine stop
podman machine set --cpus 6 --memory 12288
podman machine start

# Verify at least one model is downloaded
podman machine ssh ls /var/lib/containers/ai-lab/models/
```

## Browsing Available Recipes

### Accessing the Recipe Catalog

1. Open Podman Desktop and navigate to **AI Lab > Recipes**.
2. Browse the catalog of available recipes.

Common recipes include:

- **Chatbot**: A conversational AI interface backed by a local LLM.
- **Summarizer**: Paste text and get AI-generated summaries.
- **Code Generation**: Generate code from natural language prompts.
- **RAG Chatbot**: Chat with your own documents using retrieval-augmented generation.
- **Object Detection**: Analyze images with a local vision model.

## Running a Recipe

### Starting the Chatbot Recipe

1. In the Recipes catalog, select **Chatbot**.
2. Choose a model to use (e.g., mistral-7b-instruct).
3. Click **Start** to launch the recipe.
4. Wait for all containers to pull and start.

```bash
# Monitor the recipe containers as they start
podman ps --filter "label=ai-lab-recipe" \
  --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

# A typical chatbot recipe runs two containers:
# 1. Model server (inference backend)
# 2. Web frontend (chat UI)

# Check logs of the model server
podman logs $(podman ps --filter "label=ai-lab-recipe-server" -q) 2>&1 | tail -10

# Check logs of the frontend
podman logs $(podman ps --filter "label=ai-lab-recipe-frontend" -q) 2>&1 | tail -10
```

### Accessing the Recipe Frontend

```bash
# Find the frontend port
podman ps --filter "label=ai-lab-recipe-frontend" --format "{{.Ports}}"

# The recipe frontend is typically available at:
# http://localhost:8501 (for Streamlit-based recipes)
# http://localhost:3000 (for React-based recipes)

# Open it in your default browser
# On macOS
open http://localhost:8501

# On Linux
xdg-open http://localhost:8501
```

## Running the RAG Recipe

The RAG (Retrieval-Augmented Generation) recipe lets you chat with your own documents.

```bash
# The RAG recipe typically includes three containers:
# 1. Model server (inference backend)
# 2. Vector database (ChromaDB or similar)
# 3. Web frontend (chat UI with document upload)

# After starting the RAG recipe from the UI, verify all containers are running
podman ps --filter "label=ai-lab-recipe" \
  --format "table {{.Names}}\t{{.Status}}"

# Upload documents through the web frontend:
# 1. Open the RAG frontend in your browser
# 2. Click "Upload Documents"
# 3. Select PDF, TXT, or Markdown files
# 4. Wait for document indexing to complete
# 5. Ask questions about your uploaded documents
```

## Managing Recipe Containers

```bash
# List all running recipe containers
podman ps --filter "label=ai-lab-recipe" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check resource usage of recipe containers
podman stats --filter "label=ai-lab-recipe" --no-stream \
  --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Stop a running recipe (stops all containers in the recipe)
podman stop $(podman ps --filter "label=ai-lab-recipe=chatbot" -q)

# Remove stopped recipe containers
podman rm $(podman ps -a --filter "label=ai-lab-recipe=chatbot" -q)
```

## Customizing Recipes

### Modifying Recipe Configuration

```bash
# Recipe source code is available on GitHub
# Clone the recipes repository to make modifications
git clone https://github.com/containers/ai-lab-recipes.git
cd ai-lab-recipes

# View the available recipe directories
ls recipes/

# Each recipe contains:
# - Containerfile (for building the frontend)
# - app.py or similar (the application code)
# - requirements.txt (Python dependencies)
# - README.md (documentation)
```

### Building a Modified Recipe

```bash
# Navigate to a recipe directory
cd ai-lab-recipes/recipes/chatbot

# Modify the application code
# For example, change the system prompt in app.py

# Build the custom frontend container
podman build -t my-custom-chatbot:latest .

# Run your custom recipe manually
# First, start the model server
podman run -d --name recipe-model-server \
  -p 8080:8080 \
  -v /var/lib/containers/ai-lab/models:/models:ro \
  ghcr.io/containers/ai-lab-model-service:latest \
  --model /models/mistral-7b-instruct-q4_0.gguf

# Then start your custom frontend, connecting it to the model server
podman run -d --name recipe-frontend \
  -p 8501:8501 \
  -e MODEL_ENDPOINT=http://host.containers.internal:8080 \
  my-custom-chatbot:latest
```

## Troubleshooting Recipes

```bash
# If a recipe fails to start, check for port conflicts
podman ps --format "{{.Ports}}" | sort

# Check if containers in the recipe are crashing
podman ps -a --filter "label=ai-lab-recipe" \
  --format "table {{.Names}}\t{{.Status}}\t{{.ExitCode}}"

# View detailed logs from a failing container
podman logs --tail 50 $(podman ps -a --filter "label=ai-lab-recipe" --filter "status=exited" -q --latest)

# If the model server runs out of memory, use a smaller model
# Stop the recipe and restart with a smaller quantized model

# Clean up all recipe containers and start fresh
podman stop $(podman ps --filter "label=ai-lab-recipe" -q) 2>/dev/null
podman rm $(podman ps -a --filter "label=ai-lab-recipe" -q) 2>/dev/null
```

## Summary

AI Recipes in Podman AI Lab provide turnkey AI applications that you can deploy locally with minimal effort. From basic chatbots to document-aware RAG systems, recipes bundle a model server with a purpose-built frontend. You can run them as-is for quick experimentation or clone the recipe source code to build custom variations. Recipes are an excellent way to demonstrate AI capabilities without writing application code from scratch.
