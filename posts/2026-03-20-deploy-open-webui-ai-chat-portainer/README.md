# How to Deploy Open WebUI for AI Chat via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Open WebUI, Ollama, AI, Portainer, Docker, LLM, Self-Hosted

Description: Deploy Open WebUI alongside Ollama using Portainer to provide a ChatGPT-like web interface for your team to interact with locally-running large language models.

---

Open WebUI (formerly Ollama WebUI) is a feature-rich, self-hosted web interface for LLMs that supports Ollama backends as well as OpenAI-compatible APIs. Pairing it with Ollama via a Portainer stack gives your team a private ChatGPT experience that runs entirely on your infrastructure.

## Step 1: Deploy the Stack

```yaml
# open-webui-stack.yml

version: "3.8"

services:
  ollama:
    image: ollama/ollama:0.1.27
    volumes:
      - ollama-data:/root/.ollama
    restart: unless-stopped
    networks:
      - ai-net
    # Add GPU resources below if NVIDIA GPU is available:
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    environment:
      # Point Open WebUI at the Ollama service in this stack
      - OLLAMA_BASE_URL=http://ollama:11434
      # Optional: also connect to OpenAI API
      # - OPENAI_API_KEY=your-openai-key
    volumes:
      - open-webui-data:/app/backend/data
    ports:
      - "3000:8080"    # Open WebUI accessible at http://host:3000
    depends_on:
      - ollama
    restart: unless-stopped
    networks:
      - ai-net

volumes:
  ollama-data:
  open-webui-data:

networks:
  ai-net:
    driver: bridge
```

## Step 2: Pull Models into Ollama

After deploying the stack, use the Portainer console to pull models into the Ollama container:

```bash
# Enter the Ollama container via Portainer's console tab
ollama pull llama3
ollama pull mistral
ollama pull codellama

# List available models
ollama list
```

Or pull models directly from the Open WebUI admin interface at `http://<host>:3000` under **Settings > Models**.

## Step 3: Configure Open WebUI

Access the UI at `http://<host>:3000`:

1. **Create admin account** on first login
2. **Configure model access** under Settings > Models
3. **Set up user accounts** for team members
4. **Configure RAG** (Retrieval-Augmented Generation) by uploading documents

## Step 4: Enable Document RAG

Open WebUI supports uploading documents as context for conversations:

1. Create a collection in **Documents** section
2. Upload PDFs, markdown files, or text documents
3. Reference the collection in conversations: `#collection-name`
4. The model will use the documents as context for answers

Configure the vector store for better RAG performance:

```yaml
# Add to the open-webui service environment
environment:
  - OLLAMA_BASE_URL=http://ollama:11434
  # Use ChromaDB for vector storage (embedded in Open WebUI)
  - CHROMA_HTTP_HOST=chromadb
  - CHROMA_HTTP_PORT=8000
```

Add ChromaDB to the stack:

```yaml
  chromadb:
    image: chromadb/chroma:0.4.22
    volumes:
      - chromadb-data:/chroma/chroma
    networks:
      - ai-net
```

## Step 5: HTTPS with Nginx Reverse Proxy

For production use, put Open WebUI behind Nginx with TLS:

```yaml
  nginx:
    image: nginx:1.25-alpine
    volumes:
      - /opt/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /opt/certs:/etc/nginx/certs:ro
    ports:
      - "443:443"
    depends_on:
      - open-webui
    networks:
      - ai-net
```

```nginx
# /opt/nginx/nginx.conf
server {
    listen 443 ssl;
    server_name ai.example.com;

    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    location / {
        # Proxy to Open WebUI with WebSocket support
        proxy_pass http://open-webui:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Summary

Open WebUI with Ollama via Portainer gives your team a fully private, self-hosted AI chat platform. Data never leaves your infrastructure, models are customizable, and Portainer handles deployment and updates. It's a practical replacement for ChatGPT for teams with data privacy requirements.
