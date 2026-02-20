# Ollama Docker Setup

Local LLM server with NVIDIA GPU support, exposed for use with agentic tools (Claude Code, Copilot, etc).

## Project Structure

```
ollama/
├── docker-compose.yml
├── common-settings.yaml    # Shared restart/logging/network config
├── .env                    # COMPOSE_HTTP_TIMEOUT=300
├── ollama/
│   └── Dockerfile          # FROM ollama/ollama
└── open-webui/
    └── Dockerfile          # FROM ghcr.io/open-webui/open-webui:main
```

## Quick Start

```bash
docker-compose up -d
```

API: `http://localhost:11434`
Web UI: `http://localhost:3000`

First time opening the Web UI you will be prompted to create an admin account. Your account, chat history, and settings are all persistent across restarts.

need to redirect claude code to talk to ollama running in the container

```powershell
# set permanently
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "ollama", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "http://localhost:11434", "User")
# set for this session
$env:ANTHROPIC_API_KEY = "ollama"
$env:ANTHROPIC_BASE_URL = "http://localhost:11434"
# set your model
claude --model minimax-m2.5:cloud
```

---

## Models

### Pull a model

use the webui or use the following commands

```bash
docker exec -it ollama ollama pull llama3.2
docker exec -it ollama ollama pull mistral
docker exec -it ollama ollama pull codellama
docker exec -it ollama ollama pull deepseek-r1
```
Browse all models at https://ollama.com/library

### List downloaded models
```bash
docker exec -it ollama ollama list
```

### Remove a model
```bash
docker exec -it ollama ollama rm llama3.2
```

### Show model info
```bash
docker exec -it ollama ollama show llama3.2
```

---

## Running Models

### Interactive chat in terminal
```bash
docker exec -it ollama ollama run llama3.2
```
Type `/bye` to exit the chat.

### One-shot prompt
```bash
docker exec -it ollama ollama run llama3.2 "Explain what a transformer model is"
```

---

## API Usage

Ollama exposes an OpenAI-compatible API on port `11434`.

### Generate (raw)
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Hello, who are you?",
  "stream": false
}'
```

### Chat (OpenAI-compatible)
```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### List models via API
```bash
curl http://localhost:11434/api/tags
```

---

## Connecting Agentic Tools

| Tool | Setting | Value |
|---|---|---|
| Claude Code (custom) | Base URL | `http://localhost:11434/v1` |
| Copilot (Ollama ext) | Endpoint | `http://localhost:11434` |
| Continue.dev | `apiBase` | `http://localhost:11434` |
| LangChain | `base_url` | `http://localhost:11434` |

API key can be anything (e.g. `ollama`) — it is not validated.

---

## Container Management

### Start
```bash
docker-compose up -d
```

### Stop (keeps data)
```bash
docker-compose down
```

### Force kill immediately
```bash
docker-compose kill
```

### Restart
```bash
docker-compose restart
```

### View logs
```bash
docker-compose logs -f ollama
docker-compose logs -f open-webui
```

### Rebuild images (after Dockerfile changes)
```bash
docker-compose up -d --build
```

### Check GPU is being used
```bash
# OS level
nvidia-smi

# Inside ollama container
docker logs ollama 2>&1 | grep -i gpu
```

---

## Data & Storage

All data persists across restarts via named Docker volumes:

| Volume | Contents |
|---|---|
| `ollama_data` | Downloaded models |
| `open_webui_data` | User accounts, chat history, settings, uploaded docs |

### See volume location
```bash
docker volume inspect ollama_ollama_data
docker volume inspect ollama_open_webui_data
```

### Backup
```bash
# Models
docker run --rm -v ollama_ollama_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/ollama_backup.tar.gz -C /data .

# Web UI data (accounts, chats, settings)
docker run --rm -v ollama_open_webui_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/open_webui_backup.tar.gz -C /data .
```

### Restore from backup
```bash
docker run --rm -v ollama_ollama_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/ollama_backup.tar.gz -C /data

docker run --rm -v ollama_open_webui_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/open_webui_backup.tar.gz -C /data
```

---

## Updating

```bash
docker-compose build --pull
docker-compose up -d
```

This pulls the latest base images and rebuilds without losing your downloaded models or Web UI data.
