# LocalRAGStack

# Local RAG Stack for Enterprise Architecture Planning

[![Docker](https://img.shields.io/badge/Docker-24.0+-2496ED?style=flat&logo=docker&logoColor=white)](https://www.docker.com/)
[![Ollama](https://img.shields.io/badge/Ollama-llama3.1%20%7C%20nomic--embed--text-black?style=flat&logo=ollama&logoColor=white)](https://ollama.com/)
[![ChromaDB](https://img.shields.io/badge/ChromaDB-v2-orange?style=flat)](https://www.trychroma.com/)
[![AnythingLLM](https://img.shields.io/badge/AnythingLLM-Vector%20Orchestrator-blueviolet?style=flat)](https://useanything.com/)
[![Architecture](https://img.shields.io/badge/Arch-x86__64%20%7C%20KVM%20Hardened-green?style=flat)](https://github.com/)

A production-hardened, self-hosted Retrieval-Augmented Generation (RAG) stack configured to bypass hypervisor instruction-set limitations (`SIGILL` invalid opcode traps) common in virtualized KVM/Proxmox environments.

Engineered specifically to ground large language models on complex enterprise workflows—such as **IBM AIX to AWS Oracle XTTS cross-endian migrations**, cloud infrastructure baselines, and Active Directory hybrid identities.

---

## Architecture Overview

```
               ┌────────────────────────────────────────────────────────┐
               │                      Docker Host                       │
               │                                                        │
               │   ┌────────────────┐           ┌──────────────────┐   │
 Browser ──────┼──►│  AnythingLLM   │◄─────────►│     ChromaDB     │   │
 (Port 3001)   │   │  (Port 3001)   │           │   (Port 8000)    │   │
               │   └───────┬────────┘           └────────┬─────────┘   │
               │           │                             │             │
               │           │ (Embeddings / Chat)         │             │
               │           ▼                             │             │
               │   ┌────────────────┐                    │             │
               │   │     Ollama     │                    │             │
               │   │  (Port 11434)  │                    │             │
               │   └────────────────┘                    │             │
               │                                         │             │
               │   Persistent Volumes:                   │             │
               │   • /app/server/storage                 │             │
               │   • /root/.ollama                       │             │
               │   • /data ◄─────────────────────────────┘             │
               └────────────────────────────────────────────────────────┘
```

### Component Roles

| Service | Image | Role | Host Port |
| :--- | :--- | :--- | :--- |
| **AnythingLLM** | `mintplexlabs/anythingllm:latest` | Orchestrator, document chunker, and workspace UI | `3001` |
| **ChromaDB** | `chromadb/chroma:latest` | Dedicated vector database with disk persistence | `8000` |
| **Ollama** | `ollama/ollama:latest` | Local LLM (`llama3.1:8b`) & embeddings (`nomic-embed-text`) | `11434` |

---

## The LanceDB `SIGILL` Problem & Solution

### Failure Signature
In virtualized systems (Proxmox VE, KVM, QEMU) with default vCPU types (`kvm64`), AnythingLLM encounters silent reboot loops during document ingestion:

```text
[backend] info: [TELEMETRY SENT] {"event":"server_boot", ...}
[backend] info: Primary server in HTTP mode listening on port 3001
```

Inspecting the host kernel ring buffer reveals an invalid opcode signal (`SIGILL`):
```bash
sudo dmesg -T | grep -E -i "segfault|traps|oom|kill|anythingllm|node"
```
```text
traps: node[32968] trap invalid opcode ip:7e069ef299a1 sp:7fffc175d200 error:0 in lancedb.linux-x64-gnu.node
```

### Root Cause
Pre-compiled native LanceDB binaries require vector instruction extensions (AVX2 / AVX-512). When the hypervisor masks these flags, the process terminates immediately.

### Solution
Decouple the vector store from AnythingLLM's Node runtime by hosting an external **ChromaDB** service running under Python, avoiding native binary execution faults.

---

## Quickstart

### 1. Prerequisites
* Docker 24.0+ and Docker Compose v2.
* Host CPU with at least 4 cores and 16 GB RAM recommended.
* Ollama installed and serving on `http://172.17.0.1:11434`.

Pull the required Ollama models:
```bash
ollama pull llama3.1:8b
ollama pull nomic-embed-text
```

### 2. Prepare Storage Volumes
Ensure correct mount points exist and assign necessary write permissions:
```bash
sudo mkdir -p /var/lib/docker/volumes/chroma_data/_data
sudo mkdir -p /var/lib/docker/volumes/anythingllm_storage/_data
sudo mkdir -p /var/lib/docker/volumes/ollama_data/_data

sudo chmod -R 777 /var/lib/docker/volumes/chroma_data/_data
```

> **Note:** Modern ChromaDB images write to internal path `/data`. Do not map to deprecated legacy paths like `/chroma/chroma`.

### 3. Deploy via Docker Compose

Create a `docker-compose.yml`:

```yaml
version: "3.8"

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - /var/lib/docker/volumes/ollama_data/_data:/root/.ollama

  chromadb:
    image: chromadb/chroma:latest
    container_name: chromadb
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - IS_PERSISTENT=TRUE
      - PERSIST_DIRECTORY=/data
      - ANONYMIZED_TELEMETRY=FALSE
    volumes:
      - /var/lib/docker/volumes/chroma_data/_data:/data

  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    restart: unless-stopped
    ports:
      - "3001:3001"
    environment:
      - STORAGE_DIR=/app/server/storage
    volumes:
      - /var/lib/docker/volumes/anythingllm_storage/_data:/app/server/storage
    depends_on:
      - ollama
      - chromadb
```

Start the stack:
```bash
docker compose up -d
```

---

## Configuration & Tuning

### Step 1: Verify Chroma Persistence
Ensure `chroma.sqlite3` initializes properly on the host mount:
```bash
sudo ls -la /var/lib/docker/volumes/chroma_data/_data
```

### Step 2: Configure Vector DB in AnythingLLM UI
1. Navigate to `http://<HOST_IP>:3001`.
2. Go to **Instance Settings (Wrench/Gear)** &rarr; **AI Providers** &rarr; **Vector Database**.
3. Select **Chroma**.
4. Set **Chroma Endpoint**: `http://172.17.0.1:8000` (Docker bridge gateway).
5. Leave **API Header / Token** empty and save changes.

### Step 3: Suppress Agent Tool Hallucinations
Small models like `llama3.1:8b` can mistake domain prompts for tool-calling instructions, returning raw JSON functions (e.g., `{"name": "document-summarizer", ...}`).

To force deterministic retrieval:
1. Open your workspace settings &rarr; **Chat Settings**.
2. Switch **Chat mode** to **`Query`**.
   * *Query mode strictly injects retrieved document chunks and disables the agent tool executor.*
3. Open the **Vector Database** tab:
   * **Max Context Snippets**: `4` (or higher)
   * **Document Similarity Threshold**: `Low (similarity score >= 0.25)` or `0.0`

---

## Operational Guide

### Document Ingestion via CLI
You can queue documents directly into your workspace using the AnythingLLM API:

```bash
docker exec -it anythingllm curl -X POST http://127.0.0.1:3001/api/v1/workspace/<WORKSPACE_SLUG>/update-embeddings \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "adds": ["custom-documents/<DOCUMENT_NAME>.json"],
    "deletes": []
  }'
```

### CPU Utilization During Ingestion
During document embedding, CPU utilization reported by `docker stats` or `top` will spike:
* On a 4-vCPU host, utilization can reach **350% to 400%**.
* This is normal behavior: `nomic-embed-text` parallelizes matrix operations across all available threads.
* Usage drops to near 0% as soon as embeddings are written to `chroma.sqlite3`.

---

## Verification & Sanity Checks

Run an end-to-end RAG query from the CLI:

```bash
docker exec -it anythingllm curl -s -X POST http://127.0.0.1:3001/api/v1/workspace/<WORKSPACE_SLUG>/chat \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What are the storage and instance parameters in our blueprint?",
    "mode": "query"
  }'
```

A healthy response will return synthesized text alongside source citations:
```json
{
  "textResponse": "According to our migration blueprint, the target instance is an EC2 r6i.4xlarge with gp3 storage provisioned at 12,000 IOPS and 500 MB/s throughput...",
  "sources": [
    {
      "title": "aix-migration-blueprint.txt"
    }
  ]
}
```

---

## License
MIT License. See [LICENSE](LICENSE) for details.
