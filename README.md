# 📚 Naive RAG

A **Retrieval-Augmented Generation** pipeline built with [LangChain](https://www.langchain.com/), [Ollama](https://ollama.com/), and [ChromaDB](https://www.trychroma.com/) — powered by a reactive [Marimo](https://marimo.io/) notebook UI.

Ingest text documents, embed them into a vector store, and query them with a local LLM — all running **100 % offline** on your own hardware.

---

## ✨ Features

- **Document Ingestion** — Recursively load `.txt` files, split them into chunks, and deduplicate before storing.
- **Vector Search** — Similarity search over document embeddings via ChromaDB.
- **Chain-of-Thought QA** — Ask questions through a LangChain prompt → Ollama LLM chain with step-by-step reasoning.
- **Interactive UI** — Full-width Marimo notebook with dropdowns, text inputs, and live Markdown output.
- **Deterministic Doc IDs** — SHA-256 hashes of content + source + offset guarantee idempotent ingestion.
- **Configurable** — All knobs (embedder model, chunk size, Chroma auth, doc paths) live in a single `config.toml`.
- **Docker-ready** — Dockerfile and Compose file included.

---

## 🏗️ Architecture

```
┌──────────────┐    load & split    ┌──────────────┐   embed    ┌──────────────┐
│  Text Files  │ ─────────────────► │  LangChain   │ ────────►  │   ChromaDB   │
│  (.txt)      │                    │  Splitter     │            │  (HTTP)      │
└──────────────┘                    └──────────────┘            └──────┬───────┘
                                                                      │
                                         similarity search            │
                                    ┌─────────────────────────────────┘
                                    ▼
                             ┌──────────────┐   prompt + context   ┌──────────────┐
                             │   Retriever  │ ───────────────────► │  Ollama LLM  │
                             └──────────────┘                      │ (DeepSeek)   │
                                                                   └──────────────┘
```

---

## 📋 Prerequisites

| Requirement | Version |
|-------------|---------|
| **Python**  | ≥ 3.13  |
| **uv**      | latest  |
| **Ollama**  | latest  |
| **ChromaDB server** | ≥ 0.6.x |

### Models to pull in Ollama

```bash
ollama pull nomic-embed-text
ollama pull deepseek-r1:1.5b    # or deepseek-r1:7b
```

---

## 🚀 Getting Started

### 1. Clone the repo

```bash
git clone https://github.com/<your-username>/naive-rag.git
cd naive-rag
```

### 2. Install dependencies

```bash
uv sync
```

### 3. Start ChromaDB

Make sure a ChromaDB server is running on the host/port specified in `config.toml` (default `localhost:8765`):

```bash
# Example: run ChromaDB via Docker
docker run -d -p 8765:8000 chromadb/chroma
```

### 4. Configure

Edit **`config.toml`** to match your environment:

```toml
[chroma-client]
    host = "localhost"
    port = 8765
    chroma_client_auth_provider = "chromadb.auth.basic_authn.BasicAuthClientProvider"
    chroma_client_auth_credentials = "admin:admin"

[chroma-collection]
    name = "witcher-novels"

[embedder]
    name = "nomic-embed-text:latest"

[docs]
    glob_pattern = "path/to/your/text/files/*.txt"

[docs.splitter]
    chunk_size = 2000
    chunk_overlap = 100
```

### 5. Run the app

```bash
uv run marimo run src/app.py
```

The Marimo notebook will open in your browser. From there you can:

1. **Ingest documents** — The data-ingestion cell loads, splits, and stores new documents automatically.
2. **Ask questions** — Type a question in the text input to get a chain-of-thought answer from the LLM.
3. **Search documents** — Use the retrieval cell to run similarity searches against your vector store.

---

## 🐳 Docker

```bash
# Build & run with Compose
docker compose up --build
```

See [`README.Docker.md`](README.Docker.md) for cloud deployment instructions.

---

## 📁 Project Structure

```
naive-rag/
├── config.toml                  # App configuration (models, DB, paths)
├── pyproject.toml               # Project metadata & dependencies
├── requirements.txt             # Pinned pip dependencies (for Docker)
├── Dockerfile                   # Container image definition
├── compose.yaml                 # Docker Compose services
├── README.Docker.md             # Docker-specific docs
│
└── src/
    ├── app.py                   # Marimo notebook — main entry point
    └── modules/
        ├── __init__.py          # Re-exports: get_config, get_ollama_embedder, get_chroma_store
        ├── config.py            # TOML config loader
        ├── embedder.py          # Ollama embedding wrapper
        ├── vector_store.py      # ChromaDB client + Chroma store factory
        └── doc_actions/
            ├── __init__.py      # Re-exports: load_docs, add_new_docs_to_db, etc.
            ├── doc_actions.py   # Document loading, splitting, and ID generation
            └── add_docs_to_db.py # Dedup-aware document ingestion into ChromaDB
```

---

## ⚙️ Configuration Reference

| Section | Key | Description | Default |
|---------|-----|-------------|---------|
| `chroma-client` | `host` | ChromaDB server hostname | `localhost` |
| `chroma-client` | `port` | ChromaDB server port | `8765` |
| `chroma-client` | `chroma_client_auth_credentials` | `user:password` for Basic auth | `admin:admin` |
| `chroma-collection` | `name` | Name of the Chroma collection | `witcher-novels` |
| `embedder` | `name` | Ollama embedding model name | `nomic-embed-text:latest` |
| `docs` | `glob_pattern` | Glob path to your `.txt` source files | — |
| `docs.splitter` | `chunk_size` | Max characters per chunk | `2000` |
| `docs.splitter` | `chunk_overlap` | Overlap between adjacent chunks | `100` |

---

## 🛠️ Tech Stack

- **[LangChain](https://www.langchain.com/)** — Document loading, text splitting, prompt templates, and chain orchestration.
- **[LangChain-Ollama](https://python.langchain.com/docs/integrations/llms/ollama/)** — LLM and embedding integrations via Ollama.
- **[ChromaDB](https://www.trychroma.com/)** — Vector database for storing and retrieving document embeddings.
- **[Ollama](https://ollama.com/)** — Local LLM inference (DeepSeek-R1) and embedding generation (Nomic Embed Text).
- **[Marimo](https://marimo.io/)** — Reactive Python notebook used as the interactive UI.
- **[uv](https://docs.astral.sh/uv/)** — Fast Python package manager and project tool.

---
