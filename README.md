# Naive RAG

A **Retrieval-Augmented Generation (RAG)** pipeline built entirely with local, open-source tooling — no cloud APIs, no API keys, no data leaving your machine.

Ingest `.txt` documents, embed them into a ChromaDB vector store, and query them through a DeepSeek-R1 LLM with chain-of-thought reasoning — all driven by a reactive [Marimo](https://marimo.io/) notebook UI.

> **Stack:** Python · LangChain · ChromaDB · Ollama (DeepSeek-R1) · Marimo · Docker · uv

---

## What this project demonstrates

| Skill | Implementation |
|---|---|
| RAG pipeline design | End-to-end: ingestion → embedding → retrieval → generation |
| Vector database usage | ChromaDB HTTP client with Basic Auth, collection management |
| LLM integration | Ollama-backed LangChain chain with prompt templating |
| Idempotent data ingestion | SHA-256 content hashing to skip already-stored documents |
| Reactive UI | Marimo notebook — live updates on user input without re-runs |
| Containerisation | Multi-service Docker Compose (app + ChromaDB) |
| Config-driven design | Single `config.toml` controls all runtime knobs |

---

## Architecture

```
┌──────────────┐   load & split   ┌───────────────┐   embed   ┌──────────────┐
│  .txt Files  │ ───────────────► │   LangChain   │ ────────► │   ChromaDB   │
└──────────────┘                  │  TextSplitter │           │  (HTTP)      │
                                  └───────────────┘           └──────┬───────┘
                                                                      │ similarity search
                                                                      ▼
                                                             ┌────────────────┐
                                                             │    Retriever   │
                                                             └───────┬────────┘
                                                                     │ context + question
                                                                     ▼
                                                             ┌────────────────┐
                                                             │   Ollama LLM   │
                                                             │  (DeepSeek-R1) │
                                                             └────────────────┘
```

**Key design decisions:**
- Documents are assigned deterministic IDs (SHA-256 of content + source + chunk offset), so re-ingesting the same files is safe and idempotent.
- The LLM chain uses a "Let's think step by step" prompt to elicit chain-of-thought reasoning from the small 1.5B model.
- Marimo's reactive execution model means the UI updates live as you type — no button clicks needed.

---

## Project Structure

```
naive-rag/
├── config.toml                       # All runtime config (models, DB, paths)
├── pyproject.toml                    # Dependencies managed by uv
├── Dockerfile / compose.yaml         # Container setup
└── src/
    ├── app.py                        # Marimo notebook — entry point
    └── modules/
        ├── config.py                 # TOML loader
        ├── embedder.py               # Ollama embedding wrapper
        ├── vector_store.py           # ChromaDB client factory
        └── doc_actions/
            ├── doc_actions.py        # Load, split, hash documents
            └── add_docs_to_db.py     # Dedup-aware ingestion
```

---

## Getting Started

### Prerequisites

| Tool | Version |
|---|---|
| Python | ≥ 3.13 |
| [uv](https://docs.astral.sh/uv/) | latest |
| [Ollama](https://ollama.com/) | latest |
| ChromaDB server | ≥ 0.6.x |

Pull the required models:

```bash
ollama pull nomic-embed-text
ollama pull deepseek-r1:1.5b
```

### 1. Clone & install

```bash
git clone https://github.com/ranji-t/naive-rag.git
cd naive-rag
uv sync
```

### 2. Start ChromaDB

```bash
docker run -d -p 8765:8000 chromadb/chroma
```

### 3. Configure

Edit `config.toml` to point at your documents and ChromaDB instance:

```toml
[chroma-client]
host = "localhost"
port = 8765
chroma_client_auth_credentials = "admin:admin"

[chroma-collection]
name = "my-collection"

[embedder]
name = "nomic-embed-text:latest"

[docs]
glob_pattern = "path/to/your/files/*.txt"

[docs.splitter]
chunk_size = 2000
chunk_overlap = 100
```

### 4. Run

```bash
uv run marimo run src/app.py
```

The notebook opens in your browser. From there:
1. **Data Ingestion** — loads, splits, and stores your documents (skips duplicates automatically).
2. **Chain of Thought** — ask a free-form question; the LLM answers with step-by-step reasoning.
3. **Retrieval** — run a raw similarity search to see which document chunks match your query.

---

## Docker (full stack)

```bash
docker compose up --build
```

See [`README.Docker.md`](README.Docker.md) for cloud deployment notes.

---

## Configuration Reference

| Section | Key | Description | Default |
|---|---|---|---|
| `chroma-client` | `host` | ChromaDB hostname | `localhost` |
| `chroma-client` | `port` | ChromaDB port | `8765` |
| `chroma-client` | `chroma_client_auth_credentials` | `user:password` | `admin:admin` |
| `chroma-collection` | `name` | Collection name | `witcher-novels` |
| `embedder` | `name` | Ollama embedding model | `nomic-embed-text:latest` |
| `docs` | `glob_pattern` | Glob path to `.txt` files | — |
| `docs.splitter` | `chunk_size` | Max chars per chunk | `2000` |
| `docs.splitter` | `chunk_overlap` | Overlap between chunks | `100` |

---

## Tech Stack

- **[LangChain](https://www.langchain.com/)** — document loading, text splitting, prompt templates, chain orchestration
- **[LangChain-Ollama](https://python.langchain.com/docs/integrations/llms/ollama/)** — LLM and embedding integrations
- **[ChromaDB](https://www.trychroma.com/)** — vector database (HTTP mode with auth)
- **[Ollama](https://ollama.com/)** — local inference for DeepSeek-R1 and Nomic Embed Text
- **[Marimo](https://marimo.io/)** — reactive Python notebook as the interactive UI
- **[uv](https://docs.astral.sh/uv/)** — fast dependency management
