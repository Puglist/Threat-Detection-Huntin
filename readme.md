# ADTHE — Dev Environment Setup

## Stack Overview

| Service | Where | Purpose |
|---|---|---|
| Elasticsearch | Elastic Cloud | SIEM + OpenCTI storage |
| OpenCTI | Local Docker | Threat intel ingestion |
| Ollama | Local Docker | LLM inference |
| Redis | Local Docker | OpenCTI dependency |
| MinIO | Local Docker | OpenCTI dependency |
| RabbitMQ | Local Docker | OpenCTI dependency |

---

## Prerequisites

Install the following before proceeding:

| Tool | Download |
|---|---|
| Docker Desktop | https://www.docker.com/products/docker-desktop |
| VS Code | https://code.visualstudio.com |
| VS Code Dev Containers extension | https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers |
| Git | https://git-scm.com |

> **Hardware minimum:** 8GB RAM. Allocate at least 6GB to Docker in Docker Desktop → Settings → Resources.

---

## 1. Set Up Elastic Cloud

1. Go to https://cloud.elastic.co and create a free account (14-day trial, no credit card)
2. Create a new deployment — select the **Elasticsearch** template
3. Choose any region, default settings are fine for dev
4. Once deployed, copy:
   - **Elasticsearch endpoint URL** (e.g. `https://xxxx.es.us-east-1.aws.found.io`)
   - **Username** (`elastic`)
   - **Password** (shown once on creation — save it)

---

## 2. Clone the Repository

```bash
git clone https://github.com/your-org/adthe.git
cd adthe
```

---

## 3. Configure Environment Variables

```bash
cp .env.example .env
```

Edit `.env`:

```env
# Elastic Cloud
ELASTIC_CLOUD_URL=https://xxxx.es.us-east-1.aws.found.io
ELASTIC_USERNAME=elastic
ELASTIC_PASSWORD=your-elastic-password
ELASTIC_API_KEY=your-elastic-api-key

# OpenCTI
OPENCTI_BASE_URL=http://localhost:8080
OPENCTI_ADMIN_EMAIL=admin@adthe.local
OPENCTI_ADMIN_PASSWORD=changeme
OPENCTI_ADMIN_TOKEN=changeme-token

# MinIO (OpenCTI dependency)
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# RabbitMQ (OpenCTI dependency)
RABBITMQ_USER=guest
RABBITMQ_PASS=guest

# GitHub
GITHUB_TOKEN=your-github-pat
GITHUB_REPO=your-org/adthe
```

> **Never commit `.env` to GitHub.** It is already listed in `.gitignore`.

---

## 4. Start the Dev Container

### Option A — VS Code (recommended)

1. Open the `adthe` folder in VS Code
2. When prompted **"Reopen in Container"** click it — or manually:
   - `Ctrl+Shift+P` (Windows/Linux) / `Cmd+Shift+P` (macOS)
   - Select **Dev Containers: Reopen in Container**
3. VS Code will build the image, start all services, install dependencies, and attach.

### Option B — Terminal only

```bash
docker compose up -d --build
docker exec -it adthe-pipeline bash
```

---

## 5. Verify Services Are Running

```bash
docker compose ps
```

| Service | URL | Credentials |
|---|---|---|
| OpenCTI | http://localhost:8080 | admin@adthe.local / changeme |
| MinIO Console | http://localhost:9001 | minioadmin / minioadmin |
| Ollama API | http://localhost:11434 | — |
| Elastic Cloud | your cloud URL | elastic / your-password |

### Wait for OpenCTI to be ready

```bash
docker compose logs -f opencti
```

Wait for:
```
OpenCTI platform is ready
```

---

## 6. Pull an Ollama Model

```bash
docker exec -it adthe-ollama ollama pull llama3
```

Update `settings.yaml` to match:

```yaml
ollama:
  model: "llama3"
```

**Model selection by available RAM:**

| Docker RAM | Recommended Model |
|---|---|
| 6GB | `mistral` |
| 10GB | `llama3` |
| 16GB+ | `mixtral` |

---

## 7. Verify Elastic Cloud Connection

From inside the pipeline container:

```bash
python -c "
from elasticsearch import Elasticsearch
import os
es = Elasticsearch(os.environ['ELASTIC_CLOUD_URL'],
                   basic_auth=(os.environ['ELASTIC_USERNAME'], os.environ['ELASTIC_PASSWORD']))
print(es.info())
"
```

---

## Stopping the Environment

```bash
# Stop all containers (preserves volumes)
docker compose stop

# Stop and remove containers (preserves volumes)
docker compose down

# Stop and remove everything including volumes (destructive)
docker compose down -v
```

---

## Troubleshooting

**OpenCTI fails to connect to Elasticsearch**
- Verify `ELASTIC_CLOUD_URL` includes `https://` and has no trailing slash
- Check your Elastic Cloud deployment is active (free trials pause after inactivity)

**Ollama running slowly**
- Switch to `mistral` for lower memory usage
- Increase Docker Desktop memory allocation

**Port conflicts**
- Check nothing else is running on `8080`, `11434`, or `9001`
- Edit port mappings in `docker-compose.yml` if needed

**OpenCTI stuck on startup**
```bash
docker compose restart opencti
```