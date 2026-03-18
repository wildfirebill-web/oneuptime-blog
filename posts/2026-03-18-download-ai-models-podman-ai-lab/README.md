# How to Download AI Models with Podman AI Lab

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, AI, Machine Learning, LLM, Model Management

Description: Learn how to browse, download, and manage AI models using the Podman AI Lab extension for local inference.

---

> Downloading AI models locally means your data never leaves your machine.

Podman AI Lab provides a curated catalog of AI models that you can download and run entirely on your local hardware. From large language models to code assistants, the catalog offers a range of models optimized for different use cases and hardware configurations. This guide covers how to find, download, and manage models effectively.

---

## Prerequisites

Before downloading models, ensure the AI Lab extension is installed and your Podman machine has adequate resources.

```bash
# Verify Podman is running
podman info --format '{{.Version.Version}}'

# Check available disk space (models can be 2-10GB each)
podman machine ssh df -h /

# Ensure at least 50GB of free disk space
podman machine stop
podman machine set --disk-size 150
podman machine start
```

## Browsing the Model Catalog

### Using the Podman Desktop UI

Open Podman Desktop and navigate to the AI Lab section. Click on **Models** to see the full catalog. Models are organized by category:

- **Language Models**: General-purpose LLMs for text generation and chat
- **Code Models**: Specialized models for code generation and completion
- **Instruction Models**: Fine-tuned models that follow instructions well

### Using the CLI to List Available Models

```bash
# List all containers related to AI Lab model services
podman ps -a --filter "label=ai-lab-model" --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"

# Check what model files have been downloaded
podman machine ssh ls -lh /var/lib/containers/ai-lab/models/

# Check total disk usage by models
podman machine ssh du -sh /var/lib/containers/ai-lab/models/
```

## Downloading Models

### Popular Models to Start With

```bash
# The AI Lab catalog includes models in GGUF format optimized for CPU inference
# Common models available in the catalog:

# Llama-based models (Meta)
# - llama-2-7b-chat (small, fast, good for testing)
# - llama-3-8b-instruct (newer, better quality)

# Mistral models
# - mistral-7b-instruct (excellent quality-to-size ratio)

# Code-focused models
# - codellama-7b-instruct (code generation)
# - granite-code-3b (IBM, lightweight code model)
```

### Download a Model via the UI

1. Open Podman Desktop and go to **AI Lab > Models**.
2. Browse the catalog or use the search bar to find a model.
3. Click the **Download** button next to your chosen model.
4. Monitor the download progress in the notification area.

### Verify Downloaded Models

```bash
# List downloaded model files and their sizes
podman machine ssh find /var/lib/containers/ai-lab/models -name "*.gguf" -exec ls -lh {} \;

# Check the integrity of a downloaded model file
podman machine ssh sha256sum /var/lib/containers/ai-lab/models/mistral-7b-instruct.gguf
```

## Managing Model Storage

### Check Disk Usage

```bash
# See how much space each model is using
podman machine ssh du -sh /var/lib/containers/ai-lab/models/*

# Example output:
# 4.1G    /var/lib/containers/ai-lab/models/llama-2-7b-chat-q4_0.gguf
# 4.4G    /var/lib/containers/ai-lab/models/mistral-7b-instruct-q4_0.gguf
# 3.8G    /var/lib/containers/ai-lab/models/codellama-7b-instruct-q4_0.gguf
```

### Remove Unused Models

```bash
# Remove a specific model file to free disk space
podman machine ssh rm /var/lib/containers/ai-lab/models/llama-2-7b-chat-q4_0.gguf

# Verify the model was removed
podman machine ssh ls -la /var/lib/containers/ai-lab/models/
```

## Understanding Model Formats and Quantization

```bash
# Models in the catalog use GGUF format with various quantization levels
# Lower quantization = smaller file, faster inference, lower quality
# Higher quantization = larger file, slower inference, better quality

# Common quantization levels:
# Q4_0 - 4-bit quantization, smallest, good for testing (~4GB for 7B params)
# Q4_K_M - 4-bit with k-quant, good balance (~4.5GB for 7B params)
# Q5_K_M - 5-bit with k-quant, better quality (~5GB for 7B params)
# Q8_0 - 8-bit quantization, near full quality (~7GB for 7B params)

# Check your available RAM to choose the right quantization
podman machine inspect --format 'Memory: {{.Resources.Memory}}MB'

# Rule of thumb: model file size should be less than 80% of available RAM
# 8GB RAM  -> Q4_0 models (7B parameter models)
# 16GB RAM -> Q5_K_M or Q8_0 models (7B parameter models)
# 32GB RAM -> Q4_0 models (13B parameter models)
```

## Downloading Custom Models

### Import a Model from Hugging Face

```bash
# Download a GGUF model file from Hugging Face manually
podman machine ssh curl -L -o /var/lib/containers/ai-lab/models/custom-model.gguf \
  "https://huggingface.co/TheBloke/Mistral-7B-Instruct-v0.2-GGUF/resolve/main/mistral-7b-instruct-v0.2.Q4_K_M.gguf"

# Verify the download completed successfully
podman machine ssh ls -lh /var/lib/containers/ai-lab/models/custom-model.gguf

# The model should now appear in the AI Lab interface after a refresh
```

## Troubleshooting Download Issues

```bash
# If a download fails or gets stuck, clear partial downloads
podman machine ssh find /var/lib/containers/ai-lab/models -name "*.part" -delete

# Check network connectivity from the Podman machine
podman machine ssh curl -I https://huggingface.co

# If disk space runs out during download
podman machine stop
podman machine set --disk-size 200
podman machine start

# Restart the AI Lab service if models are not appearing
podman restart $(podman ps --filter "label=ai-lab" -q)
```

## Summary

Podman AI Lab simplifies the process of downloading and managing AI models for local inference. The curated catalog provides pre-optimized models in GGUF format with various quantization levels to match your hardware. By understanding model sizes and quantization trade-offs, you can choose the right model for your available resources. Keep an eye on disk space as models accumulate, and remove unused models to reclaim storage.
