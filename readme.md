
# ADTHE — Dev Environment Setup

## Prerequisites

Install the following before proceeding:

| Tool | Version | Download |
|---|---|---|
| Docker Desktop | Latest | https://www.docker.com/products/docker-desktop |
| VS Code | Latest | https://code.visualstudio.com |
| VS Code Dev Containers extension | Latest | [Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) |
| Git | Latest | https://git-scm.com |

> **Hardware minimum:** 16GB RAM recommended. OpenCTI + Elasticsearch are memory-intensive. Allocate at least 10GB to Docker in Docker Desktop → Settings → Resources.

---

## 1. Clone the Repository

```bash
git clone https://github.com/your-org/adthe.git
cd adthe
```

---

## 2. Configure Environment Variables

Copy the example env file and fill in your values:

```bash
cp .env.example .env
```

Edit `.env`:

```env
# PostgreSQL
POSTGRES_USER=adthe
POSTGRES_PASSWORD=changeme
POSTGRES_DB=adthe

# OpenCTI
OPENCTI_BASE_URL=http://localhost:8080
OPENCTI_ADMIN_EMAIL=admin@adthe.local
OPENCTI_ADMIN_PASSWORD=changeme
OPENCTI_ADMIN_TOKEN=changeme-token

# MinIO
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# RabbitMQ
RABBITMQ_USER=guest
RABBITMQ_PASS=guest

# GitHub
GITHUB_TOKEN=your-github-pat

# Wazuh
WAZUH_API_URL=https://your-wazuh-manager:55000
WAZUH_API_KEY=your-wazuh-api-key
```

> **Never commit `.env` to GitHub.** It is already listed in `.gitignore`.

---

## 3. Start the Dev Container

### Option A — VS Code (recommended)

1. Open the `adthe` folder in VS Code
2. When prompted **"Reopen in Container"** click it — or manually:
   - `Ctrl+Shift+P` (Windows/Linux) / `Cmd+Shift+P` (macOS)
   - Select **Dev Containers: Reopen in Container**
3. VS Code will:
   - Build the `Dockerfile.dev` image
   - Start all services via `docker-compose.yml`
   - Install Python dependencies from `requirements.txt`
   - Attach the editor to the `adthe-pipeline` container

### Option B — Terminal only

```bash
# Build and start all services
docker compose up -d --build

# Attach a shell to the pipeline container
docker exec -it adthe-pipeline bash
```

---

## 4. Verify Services Are Running

```bash
docker compose ps
```

All services should show `running`. Then verify each:

| Service | URL | Default Credentials |
|---|---|---|
| OpenCTI | http://localhost:8080 | admin@adthe.local / changeme |
| Prefect UI | http://localhost:4200 | — |
| MinIO Console | http://localhost:9001 | minioadmin / minioadmin |
| Ollama API | http://localhost:11434 | — |
| PostgreSQL | localhost:5432 | adthe / changeme |

### Check OpenCTI is ready

OpenCTI and Elasticsearch take 3–5 minutes on first boot:

```bash
docker compose logs -f opencti
```

Wait for:
```
OpenCTI platform is ready
```

---

## 5. Pull an Ollama Model

Once the stack is running, pull a model into Ollama:

```bash
# From inside the pipeline container (or your host terminal)
docker exec -it adthe-ollama ollama pull llama3
```

Update `settings.yaml` to match whichever model you pull:

```yaml
ollama:
  model: "llama3"
```

**Recommended models by hardware:**

| RAM Available | Model |
|---|---|
| 8GB | `mistral` |
| 16GB | `llama3` |
| 32GB+ | `mixtral` or `llama3:70b` |

---

## 6. Repository File Placement

Ensure your `.devcontainer/` folder is structured as follows:

```
adthe/
├── .devcontainer/
│   └── devcontainer.json
├── Dockerfile.dev
├── docker-compose.yml
├── requirements.txt
├── settings.yaml
└── .env
```

---

## Stopping the Environment

```bash
# Stop all containers (preserves data volumes)
docker compose stop

# Stop and remove containers (preserves data volumes)
docker compose down

# Stop and remove everything including volumes (destructive)
docker compose down -v
```

---

## Troubleshooting

**Elasticsearch fails to start**
```bash
# Increase vm.max_map_count on Linux hosts
sudo sysctl -w vm.max_map_count=262144
```
On Docker Desktop (Mac/Windows) this is handled automatically.

**OpenCTI stuck on startup**
```bash
docker compose restart opencti
```
If it persists, check Elasticsearch is healthy first:
```bash
docker compose logs elasticsearch
```

**Ollama running slowly / out of memory**
- Switch to a smaller model (`mistral` instead of `llama3`)
- Ensure Docker has enough RAM allocated in Docker Desktop settings

**Port conflicts**
- Check nothing else is running on ports `8080`, `4200`, `11434`, `5432`, or `9001`
- Edit port mappings in `docker-compose.yml` if needed