# IMPLEMENTATION PLAN: Local RAG Assistant

## 1. Architecture Overview

### 1.1 Recommended Stack

| Layer | Choice | Version | Justification |
|-------|--------|---------|---------------|
| Vector DB | Chroma | >= 1.0.0 | Embedded, persistent to local filesystem, zero-ops, Python-native. Ideal for small document volumes (dozens to low hundreds). |
| Embeddings (batch & query) | Ollama + `nomic-embed-text` | Ollama >= 0.1.x | 768-dim, Matryoshka representation, Metal-native on Apple Silicon. Single model for both ingestion and query. Use task prefixes (`search_document:` for indexed chunks, `search_query:` for questions) unless local retrieval tests prove the Ollama wrapper performs better without them. |
| LLM Runtime | Ollama | >= 0.1.x | Metal-native, OpenAI-compatible REST API, local-only. No internet required after model download. |
| LLM Model | Fast profile: Llama 3.2 3B (Q4). Quality profile: Mistral 7B / equivalent 7B-8B local model (Q4) | — | 3B is fast and cheap for smoke tests; 7B-8B is the realistic target for better Spanish answers on a MacBook M2 Pro with 32GB RAM. Keep model selection configurable. |
| RAG Framework | Internal RAG interfaces + optional LangChain adapters | >= 0.3.x if used | The domain code should depend on local interfaces (`Embedder`, `VectorStore`, `Retriever`, `Generator`), not directly on LangChain. LangChain can still be used inside adapters for Chroma/Ollama and future agents. |
| Doc Source | Local document directory (`docs/`) + document loaders + optional BookStack importer | — | The canonical MVP source is local files under `docs/`, converted by loaders into a common internal `Document` model. Start with Markdown, plain text, JSON, YAML, and recipe-style structured files. BookStack is an import adapter that mirrors pages into `docs/`. |
| CLI Framework | Typer | >= 0.9.x | Type-hinted, auto-generated `--help`, minimal boilerplate. |
| Web UI | Streamlit | >= 1.30.x | Fastest path to interactive UI from Python, built-in state management, no frontend build step. (Stretch goal in Phase 1; core MVP is CLI-only.) |
| Dependency Manager | pip + `pyproject.toml` | — | Standard Python tooling, no extra toolchain to install. |

### 1.2 System Diagram

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ BookStack or │────▶│ Importers    │────▶│ docs/* files │
│ local files  │     │ (optional)   │     │ (canonical)  │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Chroma     │◀────│  Ingestion   │◀────│  Chunking +  │
│  (vector DB) │     │  Pipeline    │     │  Embedding   │
└──────┬───────┘     └──────────────┘     │  (Ollama     │
       │                                  │  nomic-embed) │
       │ retrieve                         └──────────────┘
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    User      │◀────│   Answer +   │◀────│    Ollama    │
│    (CLI)     │     │  Citations   │     │  (LLM gen)   │
└──────────────┘     └──────────────┘     └──────────────┘
```

### 1.3 Data Flow

1. **Prepare source**: The MVP can start from local files under `docs/`. Supported Phase 1 formats are `.md`, `.txt`, `.json`, `.yaml`, and `.yml`. If BookStack is used, the user runs `rag import bookstack`; the importer authenticates via API token, paginates through `GET /api/pages`, exports each page as Markdown, and writes the result into `docs/`.
2. **Load + normalize**: A `DocumentLoader` registry selects a loader by file extension and converts each source file into a common internal `Document` model: `id`, `title`, `text`, `format`, `source_file`, and metadata. Structured recipe files are normalized into readable text sections such as title, tags, ingredients, steps, notes, servings, and time.
3. **Ingest**: The ingestion pipeline scans `docs/`, computes SHA256 of each source file, compares against a manifest of previously indexed files, and processes only new or changed files. Each loaded `Document` is split into chunks using a format-aware strategy: Markdown by headings, structured files by fields/sections, and plain text by paragraphs. Chunks are embedded using Ollama `nomic-embed-text` (768-dim), normally prefixed with `search_document:`, and stored in Chroma with metadata (source file, format, optional upstream source ID, chunk index, SHA256).
4. **Query**: User asks a question via CLI (`rag ask "..."`). The question is embedded using the **same** Ollama `nomic-embed-text` model, normally prefixed with `search_query:`. Chroma performs similarity search (top-k=5) to retrieve relevant chunks.
5. **Generate**: Retrieved chunks are assembled into a prompt with the user's question. Ollama (fast or quality profile) generates an answer. Source citations (file path and optional upstream title/source) are appended.
6. **Respond**: Answer + citations are displayed in the CLI.

### 1.4 Configuration Schema

All settings are loaded from environment variables or a `.env` file (via `pydantic-settings`). The canonical `.env` location is `~/.config/rag-assistant/.env`.

| Config Key | Environment Variable | Default | Required | Description |
|------------|---------------------|---------|----------|-------------|
| `source_type` | `RAG_SOURCE_TYPE` | `local` | No | Source mode. MVP default is `local`; `bookstack` is used only by the optional importer. |
| `bookstack_url` | `BOOKSTACK_URL` | — | No | BookStack base URL (required only for `rag import bookstack`). |
| `bookstack_token_id` | `BOOKSTACK_API_TOKEN_ID` | — | No | BookStack API Token ID (required only for `rag import bookstack`). |
| `bookstack_token_secret` | `BOOKSTACK_API_TOKEN_SECRET` | — | No | BookStack API Token Secret (required only for `rag import bookstack`). |
| `ollama_base_url` | `OLLAMA_BASE_URL` | `http://localhost:11434` | No | Ollama server address |
| `llm_provider` | `RAG_LLM_PROVIDER` | `ollama` | No | Generation backend. MVP implements `ollama`; future providers can use OpenAI-compatible HTTP endpoints. |
| `llm_profile` | `RAG_LLM_PROFILE` | `fast` | No | `fast` for 3B local model, `quality` for 7B-8B local model. |
| `llm_model` | `RAG_LLM_MODEL` | `llama3.2:3b-instruct-q4_K_M` | No | LLM model tag (configurable per Quality/Performance trade-off) |
| `embedding_model` | `RAG_EMBED_MODEL` | `nomic-embed-text:v1.5` | No | Embedding model tag |
| `embedding_document_prefix` | `RAG_EMBED_DOCUMENT_PREFIX` | `search_document: ` | No | Prefix applied before embedding indexed chunks. Can be set to empty after retrieval testing. |
| `embedding_query_prefix` | `RAG_EMBED_QUERY_PREFIX` | `search_query: ` | No | Prefix applied before embedding user questions. Can be set to empty after retrieval testing. |
| `chroma_persist_dir` | `CHROMA_PERSIST_DIR` | `data/chroma/` | No | Chroma persistent storage directory |
| `docs_dir` | `RAG_DOCS_DIR` | `docs/` | No | Local document directory scanned by loaders. |
| `supported_extensions` | `RAG_SUPPORTED_EXTENSIONS` | `.md,.txt,.json,.yaml,.yml` | No | Comma-separated list of file extensions enabled for ingestion. |
| `log_level` | `RAG_LOG_LEVEL` | `INFO` | No | Python logging level (DEBUG, INFO, WARNING, ERROR) |
| `log_file` | `RAG_LOG_FILE` | `data/rag.log` | No | Log file path |

### 1.5 Error Handling & Logging Strategy

- **Logging**: Use Python `logging` module. Log to both stderr (console) and a rotating file at `data/rag.log`. Default level: `INFO`; `DEBUG` via `RAG_LOG_LEVEL=DEBUG`.
- **Exception hierarchy**: Define in `src/rag_assistant/exceptions.py`:
  - `RAGError` (base) — all user-facing errors inherit from this.
  - `ConfigError` — missing or invalid configuration.
  - `SourceImportError` — optional importer failures, including BookStack API/auth errors.
  - `IngestionError` — chunking/embedding/Chroma failures.
  - `QueryError` — retrieval or generation failures.
- **User-facing errors**: All `RAGError` subclasses carry a `user_message` attribute. The CLI prints `user_message` to stderr and exits with code 1. Internal details go to the log file.
- **Retry strategy**: Use `tenacity` library for all network-bound operations (BookStack import, Ollama embeddings/generation). Pattern: `@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=1, max=10))`. Only retry on `httpx.HTTPStatusError` (5xx, 429).
- **Graceful degradation**: If Ollama is unreachable, show a clear setup message. If Chroma is empty, instruct user to add supported files under `docs/` or run `rag import bookstack`, then run `rag ingest`.

### 1.6 Quick Start (Happy Path)

```bash
# 1. Install the tool
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 2. Download models (~2.3 GB, requires internet)
ollama pull llama3.2:3b-instruct-q4_K_M   # ~2.0 GB
ollama pull nomic-embed-text:v1.5          # ~274 MB

# 3. Add documents
mkdir -p docs
# Copy or create .md, .txt, .json, .yaml, or .yml files under docs/

# Optional: import from BookStack into docs/
export BOOKSTACK_URL="https://wiki.example.com"
export BOOKSTACK_API_TOKEN_ID="your_token_id"
export BOOKSTACK_API_TOKEN_SECRET="your_token_secret"
rag import bookstack

# 4. Ingest, ask
rag ingest
rag ask "¿Cuál es el propósito del proyecto X?"
```

---

## 2. Phase 1: MVP — Read-Only Q&A (Estimated: 2-3 weeks)

### 2.1 Scope

**Included (Core MVP — CLI only):**
- [ ] Python project scaffold with `pyproject.toml` and virtual environment
- [ ] Ollama installed with `llama3.2` (or `mistral`) and `nomic-embed-text` models
- [ ] Local document source support: index supported files under `docs/`
- [ ] Phase 1 document loaders for `.md`, `.txt`, `.json`, `.yaml`, `.yml`
- [ ] Structured recipe loader convention for JSON/YAML recipe files
- [ ] Optional BookStack importer that pulls pages via paginated REST API into `docs/`
- [ ] Document ingestion pipeline: loader-aware parsing, format-aware chunking, Ollama batch embedding, Chroma storage, SHA256-based incremental indexing
- [ ] CLI command `rag ask "pregunta"` that retrieves chunks and generates sourced answers
- [ ] All components run locally with no internet dependency after initial model download

**Stretch Goal (if time permits after core MVP):**
- [ ] Streamlit web UI with Q&A input, answer display with source citations, and a "Re-index" button

**Explicitly Excluded:**
- [ ] Web UI — stretch goal only; core MVP is CLI-only
- [ ] Hybrid search (keyword + vector) — Phase 2
- [ ] Re-ranking — Phase 2
- [ ] Multi-document query decomposition — Phase 2
- [ ] Document creation or editing — Phase 3
- [ ] Autonomous agents — Phase 4
- [ ] Scheduled import/indexing (cron/launchd) — can be added later; MVP is on-demand only
- [ ] Authentication or multi-user support — single-user local tool
- [ ] FastEmbed or alternative embedding runtimes — Ollama is the single implemented embedding backend, but the code should hide it behind an `Embedder` interface
- [ ] PDF, DOCX, EPUB, OCR, and scanned documents — Phase 2+ because extraction quality and dependencies need separate handling

### 2.2 Technical Decisions Per Milestone

#### Milestone 1.1: Project Setup & Dependencies

- **Complexity**: S
- **Deliverable**: `pyproject.toml` with pinned dependencies, virtual environment, Ollama installed with models, project directory structure
- **Technical decisions**:
  - Python >= 3.11, <= 3.13 (tested range; Python 3.14 may break chromadb and other ML package deps — use pyenv if needed)
  - Project structure:
    ```
    local-rag-assistant/
    ├── pyproject.toml
    ├── src/
    │   └── rag_assistant/
    │       ├── __init__.py
    │       ├── cli.py              # Typer CLI entry point
    │       ├── sources.py          # Source discovery and supported file scanning
    │       ├── documents.py        # Internal Document model and normalized text helpers
    │       ├── loaders/
    │       │   ├── __init__.py
    │       │   ├── markdown.py
    │       │   ├── text.py
    │       │   └── structured.py   # JSON/YAML and recipe-style structured documents
    │       ├── importers/
    │       │   ├── __init__.py
    │       │   └── bookstack.py    # Optional BookStack -> docs/ importer
    │       ├── ingest.py           # Ingestion pipeline
    │       ├── query.py            # RAG query chain
    │       ├── interfaces.py       # Embedder, VectorStore, Retriever, Generator protocols
    │       ├── adapters/
    │       │   ├── __init__.py
    │       │   ├── ollama.py       # Ollama embedding/generation adapter
    │       │   └── chroma.py       # Chroma vector store adapter
    │       ├── config.py           # Settings (env vars via pydantic-settings)
    │       ├── models.py           # Pydantic models for config/data
    │       └── exceptions.py       # Custom exception hierarchy
    ├── scripts/
    │   └── import_bookstack.py     # Thin CLI wrapper calling src.rag_assistant.importers.bookstack
    ├── docs/                       # Local document directory (gitignored)
    ├── data/
    │   ├── chroma/                 # Chroma persistent store (gitignored)
    │   └── rag.log                 # Log file (gitignored)
    ├── index_manifest.json         # SHA256 manifest for incremental indexing
    ├── web/
    │   └── app.py                  # Streamlit web UI (stretch goal)
    └── tests/
        ├── fixtures/
        │   ├── sample_pages.json          # Mocked BookStack API response (3 pages)
        │   ├── sample_doc_es.md            # Spanish Markdown fixture
        │   ├── sample_doc_en.md            # English Markdown fixture
        │   ├── sample_note.txt             # Plain text fixture
        │   └── sample_recipe.yaml          # Structured recipe fixture
        ├── test_sources.py
        ├── test_loaders.py
        ├── test_bookstack_importer.py
        ├── test_ingest.py
        ├── test_query.py
        ├── test_diagnostics.py
        ├── eval/
        │   └── questions.yaml
        └── conftest.py                     # Shared fixtures (mock Chroma, mock Ollama)
    ```
  - Dependencies pinned in `pyproject.toml`:
    **Core**: `chromadb>=1.0.0`, `typer>=0.9.0`, `pydantic>=2.0.0`, `pydantic-settings>=2.0.0`, `httpx>=0.27.0`, `tenacity>=8.0.0`, `pyyaml>=6.0.0`
    **Adapters (acceptable in Phase 1, isolated from domain logic)**: `langchain>=0.3.0`, `langchain-ollama>=0.2.0`, `langchain-chroma>=0.2.0`
    **Web (stretch goal)**: `streamlit>=1.30.0`
    **Test**: `pytest>=8.0.0`, `pytest-mock>=3.0.0`, `pytest-asyncio>=0.21.0`
  - **`.env.example`**: Create a `.env.example` in the project root containing all 18 config keys with default values and documentation comments. Users copy it to `~/.config/rag-assistant/.env` and customize.
  - Ollama models pinned with explicit tags:
    - `llama3.2:3b-instruct-q4_K_M` (LLM, ~2.0 GB download)
    - `nomic-embed-text:v1.5` (embeddings, ~274 MB download)
    - **Total model download**: ~2.3 GB
- **Acceptance criteria**:
  - `pip install -e .` succeeds in a fresh `python3.11` virtual environment
  - `ollama list` shows `llama3.2:3b-instruct-q4_K_M` and `nomic-embed-text:v1.5`
  - `ollama ps` confirms at least one model is loaded and serving on `http://localhost:11434`
  - `rag --help` displays CLI commands: `import`, `ingest`, `inspect`, `search`, `ask`, `status`, `check`, `reset-index`, `eval`
  - `rag --version` prints the installed version

#### Progress Tracking — Milestone 1.1

| Unit | Title | Status |
|------|-------|--------|
| 1.1.a | Create `pyproject.toml` scaffold with Python version gate | 🔲 Pending |
| 1.1.b | Pin all Phase 1 dependencies with exact versions | 🔲 Pending |
| 1.1.c | Create directory tree (`src/rag_assistant/`, `tests/`, etc.) | 🔲 Pending |
| 1.1.d | Implement `config.py` with pydantic-settings (18 env vars) | 🔲 Pending |
| 1.1.e | Implement `exceptions.py` with full hierarchy | 🔲 Pending |
| 1.1.f | Implement `cli.py` skeleton with Typer (8 subcommands) | 🔲 Pending |
| 1.1.g | Pull Ollama models: `nomic-embed-text:v1.5` + `llama3.2:3b` | 🔲 Pending |
| 1.1.h | Create `.env.example` template | 🔲 Pending |

#### Milestone 1.2: Document Sources, Loaders + Optional BookStack Importer

- **Complexity**: S
- **Deliverable**: `src/rag_assistant/sources.py` discovers supported files under `docs/`; `src/rag_assistant/loaders/` converts each supported format into an internal `Document`; `src/rag_assistant/importers/bookstack.py` optionally pulls BookStack pages via paginated REST API and writes `.md` files to `docs/`.
- **Technical decisions**:
  - **Loader-first source**: `docs/` is the canonical source for Phase 1. Users can manually copy supported files into it without configuring BookStack.
  - **Internal `Document` model**: Every loader returns `Document(id, title, text, format, source_file, metadata)`. The rest of the ingestion/query pipeline consumes only this model, never raw file-format specifics.
  - **Supported Phase 1 loaders**:
    - Markdown (`.md`): preserve heading hierarchy for chunking and citations.
    - Plain text (`.txt`): infer title from first non-empty line or filename; chunk by paragraphs.
    - Structured data (`.json`, `.yaml`, `.yml`): parse with standard JSON/PyYAML, flatten predictable fields into readable text, and preserve original keys in metadata.
    - Recipe convention (`.json`, `.yaml`, `.yml`): if fields such as `ingredients`, `steps`, `servings`, `prep_time`, `cook_time`, `tags`, or `notes` exist, render them as recipe sections for better retrieval.
  - **Recipe example**:
    ```yaml
    title: Tortilla de patatas
    tags: [receta, cena, española]
    servings: 4
    prep_time: 15 min
    cook_time: 25 min
    ingredients:
      - 4 patatas
      - 5 huevos
      - 1 cebolla
    steps:
      - Pelar y cortar las patatas.
      - Freír a fuego medio.
      - Mezclar con huevo batido.
    notes: Queda mejor reposando 5 minutos.
    ```
    The loader normalizes this into searchable text with sections for title, tags, servings, times, ingredients, steps, and notes.
  - **Source metadata sidecar**: For imported documents, write a small metadata sidecar or frontmatter fields containing upstream source (`bookstack`), page ID, page URL/title, book slug, optional chapter ID, and import timestamp. For manually added files, source metadata is optional.
  - **Authentication**: BookStack API token (Token ID + Secret) passed via environment variables or `~/.config/rag-assistant/.env`. Header format: `Authorization: Token <token_id>:<token_secret>`.
  - **API base URL** via `BOOKSTACK_URL` environment variable (e.g., `https://wiki.example.com`).
  - **Pagination**: BookStack uses offset-based pagination (`count` + `offset`). The importer MUST:
    1. Call `GET /api/pages?count=500&offset=0&sort=+id` to get the first page of results.
    2. Read `"total"` from the JSON response to know the total page count.
    3. Loop, incrementing `offset` by 500, until `offset >= total`.
    4. Accumulate all page entries from all responses before processing.
  - **Markdown export**: For each page, call `GET /api/pages/{id}/export/markdown` to get the Markdown content. Do NOT rely on the `markdown` field from the list endpoint (it is empty for WYSIWYG-edited pages).
  - **Image stripping**: BookStack Markdown exports may contain image references (`![...](https://wiki.example.com/uploads/...)`). Strip `<img>` HTML tags and replace Markdown image links with `[Image: filename]` placeholders to avoid broken references in the local document directory.
  - **Directory structure**: `docs/{book_slug}/{chapter_or_uncategorized}/{page_slug}.md`. The pages endpoint provides `chapter_id` but not always a chapter slug, and some pages are not in a chapter. The importer must either cache chapter details from `/api/chapters` to resolve slugs or use `_pages`/`uncategorized` as a fallback folder.
  - **Error handling**: Use `tenacity` for retry on `httpx.HTTPStatusError` (5xx, 429) with exponential backoff (max 3 retries). Log per-page failures to stderr and the log file; continue processing remaining pages on individual errors. Raise `SourceImportError` on auth failure or total API unavailability.
  - **Idempotent**: Re-running overwrites existing files; no duplicate creation. Writes are atomic (write to temp file, then `os.replace`).
- **Acceptance criteria**:
  - Placing supported files (`.md`, `.txt`, `.json`, `.yaml`, `.yml`) manually under `docs/` requires no BookStack credentials
  - `rag ingest --dry-run` lists discovered files grouped by loader and reports unsupported extensions without failing
  - A YAML recipe fixture is loaded into a `Document` whose text includes title, ingredients, steps, tags/notes when present
  - A JSON structured fixture is loaded into a `Document` with stable metadata and readable flattened text
  - Running `rag import bookstack` populates `docs/` with `.md` files whose count matches the `"total"` field from `GET /api/pages`
  - Each imported `.md` file contains non-empty Markdown content (at least 10 characters after stripping whitespace)
  - Each imported file's path follows the pattern `docs/<book_slug>/<chapter_or_uncategorized>/<page_slug>.md`
  - Re-running `rag import bookstack` produces identical files (same SHA256 per path) when upstream content has not changed
  - Missing or invalid credentials produce a clear error message: "BookStack credentials not configured. Set BOOKSTACK_URL, BOOKSTACK_API_TOKEN_ID, and BOOKSTACK_API_TOKEN_SECRET."
  - A BookStack instance with >500 pages is fully imported (all pages present, no duplicates, no gaps)
  - Import logs progress per page to stderr (e.g., "Imported 42/150 pages...")

#### Progress Tracking — Milestone 1.2

| Unit | Title | Status |
|------|-------|--------|
| 1.2.a | Define `Document` pydantic model in `documents.py` | 🔲 Pending |
| 1.2.b | Implement source discovery + file scanning in `sources.py` | 🔲 Pending |
| 1.2.c | Implement text loader (`.txt`) with fixture | 🔲 Pending |
| 1.2.d | Implement markdown loader (`.md`) with fixtures | 🔲 Pending |
| 1.2.e | Implement structured loader (`.json`, `.yaml`, `.yml`) + recipe | 🔲 Pending |
| 1.2.f | Implement loader registry + `__init__.py` dispatch | 🔲 Pending |
| 1.2.g | Wire `rag ingest --dry-run` to use sources + loader registry | 🔲 Pending |
| 1.2.h | Implement `rag inspect <path>` diagnostic | 🔲 Pending |
| 1.2.i | Implement BookStack API client (auth, pagination, export) | 🔲 Pending |
| 1.2.j | Implement BookStack → `docs/` writer (dirs, image stripping, atomic) | 🔲 Pending |
| 1.2.k | Implement chapter slug resolution fallback | 🔲 Pending |
| 1.2.l | Wire `rag import bookstack` + test with mocked responses | 🔲 Pending |
| 1.2.m | Loader unit tests: success, malformed, metadata | 🔲 Pending |
| 1.2.n | BookStack importer tests: auth, pagination, idempotency | 🔲 Pending |

#### Milestone 1.3: Document Ingestion Pipeline

- **Complexity**: M
- **Deliverable**: Ingestion pipeline that reads supported files under `docs/`, loads them through the `DocumentLoader` registry, chunks them with format-aware strategies, embeds with the configured `Embedder` adapter (Phase 1: Ollama `nomic-embed-text`), stores in Chroma via the configured `VectorStore` adapter, with SHA256-based incremental indexing
- **Technical decisions**:
  - **Embedding model**: Ollama `nomic-embed-text` (768-dim). Used for BOTH batch ingestion and queries, ensuring consistent 768-dim vectors in a single Chroma collection.
  - **Embedding prefixes**: By default, prefix indexed chunks with `search_document: ` and user queries with `search_query: `. Add a small retrieval-quality test set so prefixes can be disabled only if local results are better without them.
  - **Adapter boundary**: Domain logic calls `Embedder.embed_documents()`, `Embedder.embed_query()`, and `VectorStore.upsert/delete/search()`. If LangChain is used, imports such as `from langchain_ollama import OllamaEmbeddings` and `from langchain_chroma import Chroma` stay inside adapter modules only.
  - **Loader registry**: Select loaders by extension. Unsupported files are skipped with a warning and listed in `rag ingest --dry-run`; they are not fatal unless the user explicitly targets one file.
  - **Chunking strategy**:
    - Markdown: split at H1/H2/H3 boundaries (i.e., `#`, `##`, `###`). Each chunk includes parent headers as context prefix.
    - Plain text: split by paragraphs, then combine adjacent paragraphs up to the soft size limit.
    - JSON/YAML structured documents: split by top-level sections/fields. Recipe files should produce natural chunks for overview, ingredients, steps, and notes.
    - Common limits: minimum chunk size 100 characters where possible; maximum chunk size 1500 characters as a soft limit; skip chunks that are entirely whitespace or shorter than 50 characters after stripping.
  - **Metadata schema** per chunk stored in Chroma:
    ```python
    {
      "source_file": "docs/my-book/my-chapter/my-page.md",
      "source_type": "local",
      "document_format": "markdown",
      "upstream_source": "bookstack",
      "bookstack_page_id": "123",
      "bookstack_book_slug": "my-book",
      "bookstack_chapter_id": "456",
      "document_title": "My Page",
      "chunk_index": 0,
      "section_path": "My Book > My Chapter > My Page",
      "sha256": "abc123...",
      "indexed_at": "2025-01-15T10:30:00Z"
    }
    ```
  - **Incremental indexing**: Maintain `index_manifest.json` mapping `source_file` → `{sha256, loader, document_format}`. On each run:
    1. Compute SHA256 of every supported file under `docs/`.
    2. For new files (not in manifest): load, chunk, embed, store in Chroma.
    3. For changed files (SHA256 differs): delete old Chroma entries by `source_file` filter, then chunk, embed, and store new entries.
    4. For deleted files (in manifest but not on disk): delete all Chroma entries matching that `source_file` and remove from manifest.
    5. For unchanged files: skip (report as "skipped").
  - **Chroma connection**: Collection named `local_docs`, persistent directory `data/chroma/`. If using LangChain, use `Chroma(persist_directory=..., collection_name="local_docs", embedding_function=ollama_embeddings)` inside `adapters/chroma.py`.
  - **Batch embedding**: Send all new/changed chunks in a single `OllamaEmbeddings.embed_documents()` call for efficiency. Ollama handles batching internally.
  - **Partial batch failure** (M1.4 handles query-time errors): If embedding a file fails mid-batch (e.g., Ollama timeout on one chunk), the entire failed file is logged with `IngestionError` and skipped; successfully processed files from the same run are committed. No rollback — the manifest only updates files that completed. The user sees a per-file summary at the end of `rag ingest` showing `N succeeded, M failed, K skipped`.
- **Acceptance criteria**:
  - Running `rag ingest` processes all supported files under `docs/` and stores chunks in Chroma
  - `index_manifest.json` is created/updated with correct SHA256 hashes and loader metadata; each key is a relative path
  - Re-running `rag ingest` with no file changes reports "0 files changed, N files unchanged (skipped)"
  - Modifying a single supported file and re-running processes ONLY that file (verified by log output showing 1 changed, N-1 skipped)
  - Deleting a supported file and re-running removes its chunks from Chroma and its entry from the manifest
  - Unsupported files are reported and skipped; they do not block ingestion of supported files
  - Recipe fixtures can be queried by ingredients, tags, and preparation steps
  - Chroma collection contains the expected number of chunks (verifiable via `collection.count()`)
  - Running `rag ask` immediately after `rag ingest` (without re-syncing) works — no "empty Chroma" error

#### Progress Tracking — Milestone 1.3

| Unit | Title | Status |
|------|-------|--------|
| 1.3.a | Define `Embedder` and `VectorStore` interfaces in `interfaces.py` | 🔲 Pending |
| 1.3.b | Implement Ollama adapter (embeddings) | 🔲 Pending |
| 1.3.c | Implement Chroma adapter (CRUD + metadata) | 🔲 Pending |
| 1.3.d | Implement format-aware chunking (markdown/text/structured) | 🔲 Pending |
| 1.3.e | Implement SHA256 manifest + incremental indexing | 🔲 Pending |
| 1.3.f | Wire `rag ingest` CLI command | 🔲 Pending |
| 1.3.g | Implement `rag status` diagnostic | 🔲 Pending |
| 1.3.h | Implement `rag reset-index` with confirmation prompt | 🔲 Pending |
| 1.3.i | Ingestion tests: incremental, delete, unchanged, recipe chunks | 🔲 Pending |

#### Milestone 1.4: RAG Query Chain (CLI)

- **Complexity**: M
- **Deliverable**: CLI command `rag ask "pregunta"` that retrieves relevant chunks and generates an answer via Ollama
- **Technical decisions**:
  - **Retrieval**: Embed query with the configured `Embedder` adapter using the same model and query prefix used by ingestion. Chroma similarity search with `k=5`. Use the `VectorStore.search()` interface and retrieve full metadata + document text.
  - **Prompt template**: System prompt in Spanish (since user documents are Spanish-heavy):
    ```
    Eres un asistente experto que responde preguntas basándote exclusivamente en los documentos proporcionados.
    Si la información no está en los documentos, responde EXACTAMENTE: "No encuentro información suficiente en los documentos para responder esta pregunta."
    Incluye siempre las fuentes de donde obtuviste la información.

    Contexto:
    {context}

    Pregunta: {question}

    Respuesta:
    ```
  - **Citation format**: After the answer, list sources as:
    ```
    Fuentes:
    - {source_title_or_path} (docs/path/to/file.md)
    ```
  - **LLM**: Use the configured `Generator` adapter. Phase 1 default is Ollama with model `llama3.2:3b-instruct-q4_K_M`, temperature 0.1 (factual/low-creativity), max_tokens 1024. Add a `quality` profile for a 7B-8B local model and run the same Spanish evaluation set before changing the default.
  - **Streaming**: Stream tokens to CLI output for responsiveness. If using LangChain, `ChatOllama.stream()` stays inside `adapters/ollama.py`; CLI code consumes a generator of text chunks. Print citations after the answer.
  - **Error handling**:
    - If Chroma collection is empty (0 documents), print: "No documents indexed. Add supported files under docs/ or run 'rag import bookstack', then run 'rag ingest'." Exit code 1.
    - If Ollama is unreachable (connection refused), print: "Ollama is not running. Start it with 'ollama serve' or open the Ollama desktop app."
    - If Ollama model is not pulled, print: "Model 'llama3.2:3b-instruct-q4_K_M' not found. Run 'ollama pull llama3.2:3b-instruct-q4_K_M' first."
    - If retrieval returns 0 chunks (low similarity), print: "No relevant documents found for this question."
- **Acceptance criteria**:
  - `rag ask "¿Qué es el proyecto X?"` returns an answer containing at least 20 characters of generated text, with at least one source citation referencing a file in `docs/`
  - `rag ask "pregunta sin respuesta en los documentos"` returns the exact fallback message verbatim: "No encuentro información suficiente en los documentos para responder esta pregunta."
  - Answer includes at least one "Fuentes:" citation line when chunks were retrieved (chunk count > 0)
  - Streaming output shows tokens as they arrive (visible incremental text, not batched at the end)
  - Clear error messages when Chroma is empty, Ollama is not running, or the model is missing
  - `rag ask` with no arguments displays usage: "Usage: rag ask '<your question>'"

#### Progress Tracking — Milestone 1.4

| Unit | Title | Status |
|------|-------|--------|
| 1.4.a | Define `Retriever` and `Generator` interfaces | 🔲 Pending |
| 1.4.b | Implement Ollama generator adapter (streaming, temp control) | 🔲 Pending |
| 1.4.c | Implement RAG query chain (embed → retrieve → prompt → generate) | 🔲 Pending |
| 1.4.d | Implement `rag search "query"` diagnostic | 🔲 Pending |
| 1.4.e | Wire `rag ask "pregunta"` with streaming | 🔲 Pending |
| 1.4.f | Implement `rag check` diagnostic | 🔲 Pending |
| 1.4.g | Query tests: success, empty index, unreachable Ollama, fallback | 🔲 Pending |

#### Milestone 1.5: Web UI (Stretch Goal)

> **Note**: This milestone is a **stretch goal** — only begin after Milestones 1.1–1.4 are complete and passing all acceptance criteria. The core MVP is CLI-only.

- **Complexity**: M
- **Deliverable**: Streamlit web interface for Q&A with source citations and a "Re-index" button
- **Technical decisions**:
  - **Framework**: Streamlit (fastest path from Python, no JS build step, built-in session state)
  - **Layout**:
    - Sidebar: "Import from BookStack" button (optional, triggers `import_bookstack()`), "Re-index" button (triggers `ingest()`), status indicator, document count
    - Main area: Chat-style interface with question input, answer display, source citations
  - **State management**: `st.session_state` for conversation history (last 10 turns). Each turn stores question, answer, and sources.
  - **Import/index triggers**: Buttons call the same Python functions as the CLI. Show a progress bar during long operations.
  - **Reusability**: Import logic from `src/rag_assistant/` modules — do not duplicate code.
  - **Launch command**: `rag web` (Typer subcommand that runs `streamlit run web/app.py`)
- **Acceptance criteria**:
  - `rag web` launches a Streamlit server (default `http://localhost:8501`)
  - User can type a question, receive an answer with source citations within 30 seconds
  - Conversation history persists across turns within the session (last 10 Q&A pairs visible)
  - "Re-index" button triggers ingestion of current supported files and displays a progress indicator
  - Optional "Import from BookStack" button imports pages into `docs/` before re-indexing
  - Sidebar shows current document count and last index timestamp

#### Progress Tracking — Milestone 1.5

| Unit | Title | Status |
|------|-------|--------|
| 1.5.a | Streamlit chat UI with question input, answer, citations | 🔲 Pending |
| 1.5.b | Sidebar: Re-index button + progress bar | 🔲 Pending |
| 1.5.c | Sidebar: Import from BookStack button | 🔲 Pending |
| 1.5.d | Conversation history (last 10 turns) in session state | 🔲 Pending |
| 1.5.e | `rag web` Typer subcommand | 🔲 Pending |

#### Progress Tracking — Phase 1 Cross-Cutting

| Unit | Title | Status |
|------|-------|--------|
| 1.E.a | Create evaluation set `tests/eval/questions.yaml` (10-20 Spanish questions) | 🔲 Pending |
| 1.E.b | Implement `rag eval` CLI command | 🔲 Pending |
| 1.E.c | Implement index version metadata (`data/index_metadata.json`) | 🔲 Pending |
| 1.E.d | Implement `RAG_EXCLUDE_PATTERNS` support | 🔲 Pending |

> **Status legend**: 🔲 Pending → 🔄 In Progress → ✅ Done → ⏭️ Skipped. Mark a unit as `✅` immediately after its commit. The same legend applies to all Phase 2–4 tracking tables below.

### 2.3 Phase 1 Acceptance Criteria (End-to-End)

- [ ] User can place local supported files under `docs/` without configuring BookStack
- [ ] User can index `.md`, `.txt`, `.json`, `.yaml`, and `.yml` files
- [ ] User can query a structured recipe file by ingredient, tag, or step
- [ ] Optional: user can run `rag import bookstack` and BookStack pages appear as `.md` files in `docs/` (count matches API `"total"`)
- [ ] User can run `rag ingest` and all documents are chunked, embedded, and stored in Chroma (verified via `collection.count()`)
- [ ] User can run `rag ask "pregunta"` in CLI and receive an answer with at least one source citation
- [ ] All components run locally with no internet dependency after initial `ollama pull`
- [ ] Re-indexing is incremental — unchanged files are skipped, changed files are updated, deleted files are removed
- [ ] `pip install -e .` + `ollama pull llama3.2:3b-instruct-q4_K_M` + `ollama pull nomic-embed-text:v1.5` is the full setup

---

## 3. Phase 2: Enhanced Retrieval (Estimated: 2-3 weeks)

### 3.1 Scope

**Included:**
- [ ] Streamlit Web UI (promoted from Phase 1 stretch goal, if not already complete)
- [ ] Additional document loaders: PDF, DOCX, EPUB, HTML
- [ ] OCR path for scanned PDFs/images only if needed, gated behind explicit dependencies
- [ ] Hybrid search (vector similarity + BM25 keyword search with score fusion)
- [ ] Improved chunking (semantic window awareness, parent-child retrieval for context)
- [ ] Re-ranking step with `BAAI/bge-reranker-base` (or equivalent local reranker)
- [ ] Multi-document query support (query decomposition for synthesizing across 2-3 files)
- [ ] Spanish-language retrieval quality improvements (query expansion, synonym handling)
- [ ] Evaluate alternative embedding backends (e.g., FastEmbed with `jinaai/jina-embeddings-v2-base-es` 768-dim for faster batch ingestion)

**Explicitly Excluded:**
- [ ] External search APIs (all retrieval remains local)
- [ ] Full semantic chunking with LLM-based boundaries (too slow for batch)
- [ ] Cross-lingual retrieval beyond Spanish/English
- [ ] Lossless layout preservation for PDFs/DOCX — extraction is text-first, not visual-fidelity-first

### 3.2 Milestones

#### Milestone 2.0: Rich Document Loaders
- **Complexity**: M
- **Deliverable**: Additional loaders for PDF, DOCX, EPUB, and HTML, all returning the same internal `Document` model used in Phase 1
- **Technical decisions**:
  - PDF text extraction via a lightweight parser first; OCR is optional and disabled by default because it adds heavier dependencies and slower processing.
  - DOCX extraction via a dedicated parser that preserves headings/lists where possible.
  - EPUB/HTML extraction strips navigation, scripts, and boilerplate when possible, then preserves title/headings.
  - Every rich loader must emit `document_format`, extraction warnings, and enough metadata to debug low-quality extractions.
- **Acceptance criteria**:
  - A text-based PDF fixture is ingested and queryable.
  - A DOCX fixture with headings and lists is ingested and queryable.
  - A malformed or image-only PDF produces a clear warning and does not block ingestion of other files.
  - Extracted text quality can be inspected via `rag inspect <path>`.

#### Progress Tracking — Milestone 2.0

| Unit | Title | Status |
|------|-------|--------|
| 2.0.a | Implement PDF loader (lightweight text extraction) | 🔲 Pending |
| 2.0.b | Implement DOCX loader (headings/lists preservation) | 🔲 Pending |
| 2.0.c | Implement EPUB loader (strip navigation, preserve headings) | 🔲 Pending |
| 2.0.d | Implement HTML loader (strip boilerplate, preserve title/headings) | 🔲 Pending |
| 2.0.e | Integration test: PDF + DOCX + EPUB + HTML fixtures | 🔲 Pending |
| 2.0.f | Wire rich loaders into loader registry and `rag ingest` | 🔲 Pending |

#### Milestone 2.1: Hybrid Search
- **Complexity**: M
- **Deliverable**: Retrieval pipeline that combines Chroma vector search with BM25 keyword search, fused via Reciprocal Rank Fusion (RRF)
- **Technical decisions**:
  - BM25 via `rank_bm25` library (pure Python, no dependencies)
  - Build BM25 index over chunk text during ingestion (store alongside Chroma at `data/bm25_index.pkl`)
  - RRF formula: `score = sum(1 / (k + rank))` with `k=60` (standard)
  - Return top-5 results after fusion
  - BM25 index rebuilt incrementally (only changed chunks) on re-ingest
- **Acceptance criteria**:
  - Hybrid search returns a result list of 5 chunks; vector-only and BM25-only each contribute at least 1 chunk in the top-5 for a manual test of 5 queries
  - BM25 index is rebuilt incrementally (only changed chunks processed)
  - RRF fusion produces a single ranked list with no duplicate chunks

#### Progress Tracking — Milestone 2.1

| Unit | Title | Status |
|------|-------|--------|
| 2.1.a | Implement BM25 index + incremental rebuild | 🔲 Pending |
| 2.1.b | Implement RRF fusion (vector + BM25, k=60) | 🔲 Pending |
| 2.1.c | Wire hybrid search into retrieval pipeline | 🔲 Pending |
| 2.1.d | Hybrid search tests: vector contribution, BM25 contribution, no duplicates | 🔲 Pending |

#### Milestone 2.2: Advanced Chunking
- **Complexity**: M
- **Deliverable**: Parent-child chunking strategy where child chunks (smaller, for retrieval) reference parent chunks (larger, for context)
- **Technical decisions**:
  - Child chunks: 300-500 characters (optimized for embedding precision)
  - Parent chunks: full section under an H2/H3 header (up to 2000 characters)
  - Store parent text in Chroma metadata; retrieve parent when child matches
  - Maintain backward compatibility — existing single-level chunks still work
- **Acceptance criteria**:
  - Ingestion creates both child and parent chunks for each document
  - Retrieval returns parent context when a child chunk matches; parent text is included in the LLM prompt
  - Existing single-level chunks from Phase 1 remain queryable after upgrade

#### Progress Tracking — Milestone 2.2

| Unit | Title | Status |
|------|-------|--------|
| 2.2.a | Implement parent-child chunking strategy (child 300-500c, parent ≤2000c) | 🔲 Pending |
| 2.2.b | Store parent text in Chroma metadata | 🔲 Pending |
| 2.2.c | Modify retrieval to return parent context on child match | 🔲 Pending |
| 2.2.d | Backward compatibility test: Phase 1 single-level chunks remain queryable | 🔲 Pending |

#### Milestone 2.3: Re-ranking
- **Complexity**: S
- **Deliverable**: Re-ranking step that takes top-10 retrieved chunks and re-scores them with a cross-encoder reranker
- **Technical decisions**:
  - Model: `BAAI/bge-reranker-base` via `sentence-transformers` (runs locally on CPU)
  - Retrieve top-10 from hybrid search, re-rank to top-5
  - Fallback: if reranker model fails to load, skip re-ranking and use hybrid results
  - Add `rerank_score` to chunk metadata for debugging
- **Acceptance criteria**:
  - Re-ranked top-5 list differs from hybrid top-5 list for at least 3 out of 10 test queries (indicating the reranker is making decisions, not just passing through)
  - Graceful fallback when reranker is unavailable — query succeeds with hybrid results
  - Re-ranking adds < 2 seconds latency per query

#### Progress Tracking — Milestone 2.3

| Unit | Title | Status |
|------|-------|--------|
| 2.3.a | Integrate `BAAI/bge-reranker-base` cross-encoder | 🔲 Pending |
| 2.3.b | Implement re-rank step: top-10 → score → top-5 | 🔲 Pending |
| 2.3.c | Graceful fallback when reranker unavailable | 🔲 Pending |
| 2.3.d | Reranker tests: order diff, latency, fallback | 🔲 Pending |

#### Milestone 2.4: Multi-document Synthesis
- **Complexity**: M
- **Deliverable**: Query decomposition pipeline that handles questions spanning multiple documents
- **Technical decisions**:
  - Detect multi-document intent: if query contains "compara", "diferencia", "resumen de X y Y", or returns chunks from 3+ distinct source files
  - Decompose into sub-queries via LLM (one-shot prompt)
  - Execute each sub-query independently, merge results, synthesize final answer
  - Limit to 3 sub-queries maximum to control latency
- **Acceptance criteria**:
  - Query "resume el proyecto X y sus dependencias" (chunks from 2+ source files) returns a single answer with citations from each source file
  - Each sub-query result is cited separately in the final answer under "Fuentes:"
  - Total latency (ingestion to response) remains under 15 seconds for multi-document queries

#### Progress Tracking — Milestone 2.4

| Unit | Title | Status |
|------|-------|--------|
| 2.4.a | Implement multi-document intent detection (LLM-based) | 🔲 Pending |
| 2.4.b | Implement query decomposition into sub-queries (max 3) | 🔲 Pending |
| 2.4.c | Implement merge + synthesize final answer with per-source citations | 🔲 Pending |
| 2.4.d | Multi-document query tests: comparison, summary, latency | 🔲 Pending |

### 3.3 Phase 2 Acceptance Criteria

- [ ] Hybrid search returns chunks from both vector and keyword paths for queries with distinctive keywords
- [ ] Re-ranking modifies the result order for most queries (non-pass-through)
- [ ] Multi-document query returns citations from multiple source files in a single answer
- [ ] All Phase 1 CLI commands and acceptance criteria still pass (no regressions)

---

## 4. Phase 3: Document Creation (Estimated: 3-4 weeks)

### 4.1 Scope

**Included:**
- [ ] Writing interface in both CLI and Web UI
- [ ] Draft generation from retrieved context + user topic
- [ ] Human-in-the-loop review workflow (edit before saving)
- [ ] Save drafts back to `docs/` (local) or push to BookStack via API

**Excluded:**
- [ ] Autonomous writing without human review — always requires approval (Phase 4)
- [ ] Full WYSIWYG editor parity with BookStack — Markdown editor only
- [ ] Real-time collaboration

### 4.2 Milestones

#### Milestone 3.1: Draft Generation
- **Complexity**: M
- **Deliverable**: CLI command `rag draft "tema"` and corresponding Web UI flow that generates a document draft from retrieved context
- **Technical decisions**:
  - Retrieve relevant chunks for the topic (reuse Phase 1/2 retrieval)
  - Prompt LLM to generate a structured Markdown draft with headers, based on retrieved context
  - Draft is written to `drafts/` directory with timestamped filename (`drafts/2025-01-15_configurar-entorno.md`)
  - Draft includes a "Sources used" section at the bottom listing referenced documents
- **Acceptance criteria**:
  - `rag draft "Cómo configurar el entorno de desarrollo"` produces a Markdown file in `drafts/` with at least 200 characters of content
  - Generated draft contains at least one Markdown header (`##`)
  - Draft includes a "Fuentes utilizadas" section referencing files from `docs/`
  - Running `rag draft` with no arguments displays usage information

#### Progress Tracking — Milestone 3.1

| Unit | Title | Status |
|------|-------|--------|
| 3.1.a | Implement `rag draft "tema"` CLI command | 🔲 Pending |
| 3.1.b | Implement draft generation prompt + LLM call | 🔲 Pending |
| 3.1.c | Write draft to `drafts/` with timestamped filename + "Fuentes utilizadas" section | 🔲 Pending |
| 3.1.d | Draft generation tests: content length, headers, sources | 🔲 Pending |

#### Milestone 3.2: Writing Interface
- **Complexity**: L
- **Deliverable**: Interactive editor in CLI (via `rich` + `prompt_toolkit`) and Web UI (Streamlit text area with preview)
- **Technical decisions**:
  - CLI: `prompt_toolkit` for multi-line editing with syntax highlighting (Markdown)
  - Web: Streamlit `st.text_area` with Markdown preview via `st.markdown`
  - Context panel: show retrieved source chunks alongside the editor for reference
  - Auto-save drafts to `drafts/` every 30 seconds (Web) or on explicit `Ctrl+S` (CLI)
- **Acceptance criteria**:
  - User can open a draft, edit it, save changes, and verify the file was updated on disk
  - Context panel in the editor displays at least one source chunk from the original retrieval
  - Auto-save recovers content if the browser tab is closed and reopened within the session

#### Progress Tracking — Milestone 3.2

| Unit | Title | Status |
|------|-------|--------|
| 3.2.a | Implement CLI interactive editor (`rich` + `prompt_toolkit`) | 🔲 Pending |
| 3.2.b | Implement Streamlit text editor with Markdown preview | 🔲 Pending |
| 3.2.c | Implement context panel (show source chunks alongside editor) | 🔲 Pending |
| 3.2.d | Implement auto-save (30s Web / Ctrl+S CLI) | 🔲 Pending |

#### Milestone 3.3: Review Workflow
- **Complexity**: M
- **Deliverable**: Approval/revision/commit flow for drafts
- **Technical decisions**:
  - `rag review <draft>` opens the draft for final review
  - User can: approve (moves to `docs/`), revise (re-opens editor), or discard (moves to `drafts/archive/`)
  - On approve: file is moved to `docs/` using the local document directory convention, `index_manifest.json` is updated
  - Optional: push to BookStack via API (create new page or update existing)
  - All actions are logged to `drafts/history.jsonl` with ISO 8601 timestamps and action type
- **Acceptance criteria**:
  - Approved draft appears in `docs/` and is picked up by the next `rag ingest`
  - Revision cycle works: draft → edit → save → review → approve without data loss
  - Discarded drafts are moved to `drafts/archive/` with a timestamped filename
  - `drafts/history.jsonl` contains one valid JSON line per action with fields: `timestamp`, `action`, `draft_path`, `destination`

#### Progress Tracking — Milestone 3.3

| Unit | Title | Status |
|------|-------|--------|
| 3.3.a | Implement `rag review <draft>` CLI | 🔲 Pending |
| 3.3.b | Implement approve → move to `docs/` | 🔲 Pending |
| 3.3.c | Implement discard → move to `drafts/archive/` | 🔲 Pending |
| 3.3.d | Implement optional BookStack push on approve | 🔲 Pending |
| 3.3.e | Implement `drafts/history.jsonl` audit log | 🔲 Pending |
| 3.3.f | Review workflow tests: approve, revise, discard cycles | 🔲 Pending |

### 4.3 Phase 3 Acceptance Criteria

- [ ] User can generate a draft on a topic backed by their own documents
- [ ] User can review and edit drafts in both CLI and Web UI before saving
- [ ] Approved documents are saved to `docs/` and indexed on next re-index
- [ ] Draft history is preserved and auditable via `drafts/history.jsonl`

---

## 5. Phase 4: Autonomous Writing Agents (Estimated: 4-6 weeks)

### 5.1 Scope

**Included:**
- [ ] Agent architecture using the existing internal interfaces; LangChain/LangGraph ReAct-style agents are implementation options, not domain dependencies
- [ ] Task delegation: research → outline → draft → revise pipeline
- [ ] Tool use: document search, file I/O, optional web search (if configured)
- [ ] Safety and approval gates at each stage

**Excluded:**
- [ ] Fully unsupervised writing — human approval always required before saving
- [ ] External tool integration beyond configured sources (BookStack, local docs)
- [ ] Multi-agent orchestration (single agent with tool selection)

### 5.2 Milestones

#### Milestone 4.1: Agent Framework
- **Complexity**: L
- **Deliverable**: Agent workflow with a ReAct-style loop, equipped with tools for document search, file read/write, and draft management
- **Technical decisions**:
  - Agent type: implementation can use LangChain/LangGraph `create_react_agent`, but it must consume the existing `Retriever` and `Generator` interfaces rather than importing low-level retrieval/generation code directly
  - Tools:
    - `search_docs(query)`: retrieves relevant chunks from Chroma
    - `read_draft(path)`: reads an existing draft
    - `write_draft(path, content)`: writes a draft file
    - `save_to_docs(path, content)`: moves draft to `docs/` (requires approval gate)
  - Agent prompt includes role definition, tool descriptions, and safety constraints
  - All tool calls are logged to `drafts/agent_history.jsonl` for auditability
- **Acceptance criteria**:
  - Agent can execute: "find information about X and summarize it" — produces a summary from retrieved chunks
  - Agent uses `search_docs` tool correctly and includes citations in its output
  - Agent cannot write to `docs/` without explicit human confirmation (the `save_to_docs` tool is gated)

#### Progress Tracking — Milestone 4.1

| Unit | Title | Status |
|------|-------|--------|
| 4.1.a | Implement ReAct agent loop with tool definitions | 🔲 Pending |
| 4.1.b | Implement `search_docs`, `read_draft`, `write_draft`, `save_to_docs` tools | 🔲 Pending |
| 4.1.c | Implement `drafts/agent_history.jsonl` audit log | 🔲 Pending |
| 4.1.d | Agent framework tests: tool usage, summarization, citations | 🔲 Pending |

#### Milestone 4.2: Task Decomposition
- **Complexity**: L
- **Deliverable**: Pipeline that breaks a writing task into subtasks: research → outline → draft → revise
- **Technical decisions**:
  - Task planner: LLM generates a structured plan with subtasks
  - Each subtask is executed by the agent with appropriate tools
  - Intermediate results (outline, draft v1, draft v2) are saved to `drafts/` with version suffixes
  - User can intervene at any stage to modify the plan or output
- **Acceptance criteria**:
  - Given task "write a guide on deploying the project", agent produces: research notes → outline → draft → revised draft — each as a separate file in `drafts/`
  - Each stage output is at least 100 characters and is saved before proceeding to the next stage
  - User can edit the outline file, and the agent continues from the modified outline

#### Progress Tracking — Milestone 4.2

| Unit | Title | Status |
|------|-------|--------|
| 4.2.a | Implement task planner (LLM generates subtask plan) | 🔲 Pending |
| 4.2.b | Implement research → outline → draft → revise pipeline | 🔲 Pending |
| 4.2.c | Implement user intervention at any pipeline stage | 🔲 Pending |
| 4.2.d | Task decomposition tests: multi-stage output, user edit resume | 🔲 Pending |

#### Milestone 4.3: Safety Gates
- **Complexity**: M
- **Deliverable**: Content approval workflow, scope boundaries, and rate limits
- **Technical decisions**:
  - Approval gate: before any write to `docs/` or BookStack API, present the content to the user for approval via CLI prompt or Web UI dialog
  - Scope boundaries: agent can only access `docs/`, `drafts/`, and configured BookStack instance — no arbitrary file system access
  - Rate limits: max 10 tool calls per task, max 3 revision cycles
  - Audit log: all agent actions recorded in `drafts/agent_history.jsonl` with timestamps, action types, and outcomes
  - Kill switch: `rag agent stop` terminates any running agent task by PID
- **Acceptance criteria**:
  - Agent cannot overwrite files in `docs/` without user confirmation (confirmation prompt is displayed and requires explicit "yes" input)
  - Agent fails with a clear error when attempting to access paths outside `docs/` and `drafts/`
  - Rate limits prevent runaway tool call loops (agent stops after 10 tool calls per task)
  - Audit log captures at least: task start, each tool call, and task completion, each with ISO 8601 timestamps
  - `rag agent stop` terminates the agent process within 5 seconds

#### Progress Tracking — Milestone 4.3

| Unit | Title | Status |
|------|-------|--------|
| 4.3.a | Implement approval gate before any `docs/` write | 🔲 Pending |
| 4.3.b | Implement scope boundaries (restrict to `docs/`, `drafts/`, BookStack) | 🔲 Pending |
| 4.3.c | Implement rate limits (max 10 tool calls, max 3 revision cycles) | 🔲 Pending |
| 4.3.d | Implement `rag agent stop` kill switch | 🔲 Pending |
| 4.3.e | Safety gate tests: overwrite prevention, path restriction, rate limits, audit log, kill switch | 🔲 Pending |

### 5.3 Phase 4 Acceptance Criteria

- [ ] Agent can receive a writing task, research from local docs, produce a draft, and wait for approval
- [ ] Safety gates prevent file overwrites without confirmation
- [ ] Task decomposition produces structured intermediate outputs (outline, draft, revision)
- [ ] All agent actions are logged and auditable
- [ ] Kill switch terminates running agent tasks within 5 seconds

---

## 6. MVP Validation, Diagnostics & Safety

### 6.1 Evaluation Set

- **Goal**: Maintain a small, repeatable evaluation set so changes to models, embeddings, chunking, loaders, or prompts can be compared objectively.
- **Location**: `tests/eval/questions.yaml`
- **Minimum contents**: 10-20 Spanish-heavy questions based on real documents.
- **Per-question fields**:
  ```yaml
  - id: receta_tortilla_ingredientes
    question: "Que recetas tengo con huevos y patatas?"
    expected_sources:
      - docs/recetas/tortilla.yaml
    expected_terms:
      - huevos
      - patatas
    should_answer: true
  - id: pregunta_sin_contexto
    question: "Cual es la politica de vacaciones de una empresa que no aparece en mis documentos?"
    expected_sources: []
    should_answer: false
  ```
- **Acceptance criteria**:
  - `rag eval` runs the evaluation set against the current index and model configuration.
  - The report includes retrieval hit rate, source match rate, unsupported/empty answer count, and average latency.
  - Changes to embedding prefixes, chunk sizes, loaders, or default model must run `rag eval` before being accepted.

### 6.2 Diagnostic CLI Commands

- `rag status`: Shows configured docs directory, supported extensions, active LLM/embedding models, Chroma collection name, indexed document count, chunk count, last ingestion timestamp, and skipped unsupported files.
- `rag inspect <path>`: Loads one source file through its loader and prints normalized `Document` metadata + text preview without embedding or writing to Chroma.
- `rag search "query"`: Runs retrieval only, without LLM generation. Prints top-k chunks with scores, source files, document format, and section path.
- `rag check`: Verifies that the environment is ready for RAG operations. Checks: Ollama server is reachable, required models are pulled (or suggests `ollama pull`), Chroma persistence directory exists and is writable, `docs/` directory exists, Python and key dependency versions meet minimums. Exits with code 0 if all checks pass, code 1 otherwise, printing a per-check report.
- `rag reset-index`: Deletes the local Chroma collection and manifest after an explicit confirmation prompt.
- `rag eval`: Runs the evaluation set in `tests/eval/questions.yaml`.

### 6.3 Index Versioning

The index must include enough version metadata to know when a full re-ingestion is required.

- Store in `data/index_metadata.json`:
  - `embedding_model`
  - `embedding_dimension`
  - `embedding_document_prefix`
  - `embedding_query_prefix`
  - `chunking_version`
  - `loader_versions`
  - `collection_name`
- If any of these values change, either:
  - force a full re-ingestion, or
  - create a new collection name derived from the embedding/chunking configuration.
- Never silently reuse an index built with incompatible embedding dimensions or prefixes.

### 6.4 Unsupported Files Policy

- Phase 1 supports `.md`, `.txt`, `.json`, `.yaml`, and `.yml`.
- Unsupported files are listed in `rag ingest --dry-run` and `rag status`.
- Unsupported files do not fail ingestion unless the user explicitly targets one file.
- Phase 2 adds PDF, DOCX, EPUB, HTML, and optional OCR.

### 6.5 Local Privacy & Anti-Secret Rules

The assistant must avoid indexing secrets or unrelated project internals by default.

- Do not index hidden files or hidden directories unless explicitly enabled.
- Always exclude:
  - `.git/`
  - `.venv/`
  - `node_modules/`
  - `__pycache__/`
  - `.env`
  - `.env.*`
  - private keys (`*.pem`, `*.key`, `id_rsa`, `id_ed25519`)
  - binary files
  - common backup files (`*.bak`, `*.tmp`, `*.swp`)
- Add `RAG_EXCLUDE_PATTERNS` for user-configurable exclusions.
- `rag ingest --dry-run` must show excluded/skipped counts.

### 6.6 MVP Use Cases

Use these as smoke tests and evaluation seeds:

- "Que puedo cocinar con huevos y patatas?"
- "Dame recetas rapidas para cenar."
- "Que notas tengo sobre el proyecto X?"
- "Resume la documentacion de X con fuentes."
- "Que documentos hablan de configurar el entorno de desarrollo?"
- "Que informacion tengo sobre compras, ingredientes o despensa?"

---

## 7. AI Implementation Rules & Useful Skills

### 7.1 Project-Specific Rules For AI Agents

Any AI agent implementing this project should follow these rules before editing code:

- Read this implementation plan first and keep changes scoped to the current phase.
- Do not add Phase 2 dependencies to the MVP unless the plan is explicitly updated.
- Keep domain logic independent from LangChain. LangChain imports are allowed only inside adapters or future agent modules.
- All source formats must pass through the `DocumentLoader -> Document -> Chunker -> Embedder -> VectorStore` pipeline.
- Do not add format-specific logic to query/generation code.
- Prefer typed Python, `pathlib`, small modules, and explicit Pydantic models for config and data contracts.
- All file-system writes must stay under project-managed directories (`docs/`, `data/`, `drafts/`) unless explicitly configured.
- Network calls must use timeouts, clear errors, and retries only for idempotent operations.
- Every loader needs fixture tests for success, malformed input, and metadata.
- Every CLI command needs a basic success test and at least one user-facing error test.
- Any change to model, embeddings, chunking, or loader normalization must update or run the evaluation set.
- Never index secrets, hidden files, dependency directories, or binary files by default.

### 7.2 Project Skill (Already Created)

The project-specific skill `local-rag-assistant-implementation` already exists at `.agents/skills/local-rag-assistant-implementation/SKILL.md`. It contains:

- Architecture summary and the `DocumentLoader -> Document -> Chunker -> Embedder -> VectorStore` pipeline.
- Current phase (Phase 1 MVP) and explicitly excluded Phase 2+ features.
- Loader contract, privacy rules, diagnostics guidance, and testing rules.
- An implementation checklist at `references/implementation-checklist.md` for finalizing work.

Use `skill local-rag-assistant-implementation` at the start of every implementation session to load these rules automatically.

### 7.3 Useful Existing Skills

- `skill-creator`: Useful to create the project-specific implementation skill above.
- `openai-docs`: Use only when adding external OpenAI or OpenAI-compatible provider support and current API/model details are needed.
- `web-design-guidelines`: Useful when reviewing the Streamlit or future web UI for accessibility, clarity, and interaction quality.
- `vercel-react-best-practices`: Useful only if the UI later changes from Streamlit to React/Next.js.
- `vercel-composition-patterns`: Useful only if a React component architecture is introduced later.
- `find-skills`: Useful if a new implementation area appears, such as Python packaging, testing, document parsing, deployment, or local ML operations, and you want to search for a focused skill.

### 7.4 Suggested Repository Rules File

Add an `AGENTS.md` or equivalent repo-level rules file when code implementation begins. It should include:

- Current phase: Phase 1 MVP only.
- Required commands before finalizing implementation: unit tests, loader tests, and `rag eval` when relevant.
- Forbidden shortcuts: direct LangChain usage in domain modules, indexing hidden files, hardcoding user paths, adding heavy document parsers to MVP.
- Preferred commands: `rag ingest --dry-run`, `rag inspect <path>`, `rag search "query"`, and `rag status` for debugging.

---

## 8. Risks & Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Ollama model quality for Spanish is insufficient (`llama3.2` 3B may struggle with Spanish nuance) | High | Medium | Treat 3B as the `fast` profile. Test 10-20 Spanish questions against a 7B-8B `quality` profile before committing to the default model for real use. Keep model selection configurable via `RAG_LLM_MODEL`. |
| `nomic-embed-text` Spanish embedding quality or prefix behavior is insufficient | Medium | Medium | Phase 1 uses `nomic-embed-text` for consistency and starts with `search_document:`/`search_query:` prefixes. Add a small retrieval-quality test set and compare prefixes on/off. If retrieval quality is poor, Phase 2 can introduce FastEmbed with a Spanish/multilingual embedding model. |
| BookStack API rate limits or instability | Medium | Low | Keep BookStack as an optional importer, not the canonical source. Implement exponential backoff with `tenacity` (max 3 retries). BookStack default rate limit is 180 req/min — import at ~2 pages/sec stays well under this. |
| Chroma performance degrades with document growth | Low | Low (current volume is small) | Chroma 1.x handles ~100K vectors comfortably on a single machine. If growth exceeds this, plan migration to Qdrant or Weaviate. |
| Streamlit Web UI becomes slow with large conversations | Low | Low | Limit conversation history to last 10 turns. Paginate older turns if needed. |
| Loader output changes after parser upgrades | Medium | Medium | Store loader name/version in `index_manifest.json`. If a loader version changes, force re-ingestion for files handled by that loader. |
| Structured documents flatten poorly and lose useful context | Medium | Medium | Keep the original keys in metadata and add fixture tests for recipes and generic JSON/YAML. Provide `rag inspect <path>` to preview normalized text before embedding. |
| SHA256 incremental indexing misses upstream metadata changes | Low | Low | SHA256 covers full source file content. Store importer metadata in frontmatter or sidecar files so title/source changes can be captured and re-indexed when relevant. |
| LLM hallucinates answers not grounded in retrieved context | High | Medium | Use strict system prompt instructing LLM to answer only from context. Include explicit fallback message for unanswerable questions. Add "confidence" scoring in Phase 2. |
| Apple Metal support in Ollama breaks on macOS update | Medium | Low | Pin Ollama version. Test after macOS updates. Fallback to CPU mode (slower but functional). |
| User documents contain sensitive data | High | Medium | All processing is local. No data leaves the machine. Document this prominently in README. Add startup warning in CLI and Web UI. |
| BookStack image references in Markdown exports produce broken local links | Low | Medium | Strip `<img>` HTML tags and replace Markdown image links with `[Image: filename]` placeholders during import (Phase 1). Optional image download in Phase 3. |
| Ollama model download (~2.3 GB) fails on slow/unreliable internet | Medium | Low | Document model sizes upfront. Provide `ollama pull` commands in Quick Start. Models are downloaded once and cached. |
| Chroma SQLite backend has file-locking issues on macOS | Low | Low | Chroma uses WAL mode by default. Single-user, single-process access pattern avoids concurrency issues. |
| LangChain partner package version skew (langchain-ollama, langchain-chroma vs core langchain) | Medium | Medium | Keep LangChain isolated inside adapters/agent modules. Pin compatible versions in `pyproject.toml` only if those adapters are used, and test `pip install -e .` before merging changes. |
| Python 3.14 incompatibility with chromadb, pydantic-settings, or other ML deps | High | Medium | The system has Python 3.14.2; most ML packages target 3.11–3.13. Pin range to `>=3.11, <=3.13` and install a compatible version via pyenv before starting. Add `rag check` to verify Python version at startup. |

---

## 9. Open Questions & Future Decisions

1. **Additional document sources**: Local files under `docs/` are canonical in Phase 1 and BookStack is an optional importer. Future importers can target Obsidian vaults, Git repositories, Notion exports, or other systems that export files into `docs/`. Recommendation: keep the importer interface small in Phase 1 (`import -> docs/`) and avoid a broad plugin system until a second real source is needed.

2. **Default LLM model**: Llama 3.2 3B (Q4) vs a 7B-8B quality model — which should be the default for day-to-day use? 3B is faster and uses less RAM; 7B-8B is more likely to produce useful Spanish answers on the available MacBook M2 Pro 32GB. Recommendation: ship `fast` as the setup default, run a 10-20 question Spanish evaluation, then choose whether the documented recommended profile should be `quality`.

3. **Alternative embedding backends (Phase 2)**: If `nomic-embed-text` proves insufficient for Spanish retrieval quality, evaluate FastEmbed or another local embedding backend through the existing `Embedder` interface. A dimension change requires a new Chroma collection and full re-ingestion, so the collection name should include embedding model/version metadata.

4. **Scheduled import/indexing**: User mentioned "nightly/on-startup" as acceptable. Should a `launchd` plist or `cron` entry be set up in Phase 1, or defer to a future phase? Recommendation: defer. MVP is on-demand. Add a `rag schedule --print-launchd` helper later if repeated imports become useful.

5. **Conversation memory**: Should the system maintain long-term conversation history across sessions (stored in a file), or keep it session-only? Recommendation: session-only for Phase 1 (CLI has no persistent memory; Streamlit uses session state). Add persistent conversation history as a Phase 2 feature.

6. **Document format normalization**: BookStack exports to Markdown, but WYSIWYG pages may include HTML artifacts (nested `<div>`, `<span style="...">`, etc.). JSON/YAML recipes and notes may also vary by schema. Recommendation: preserve source files as-is, normalize only inside loaders, and add `rag inspect <path>` so the user can see exactly what will be indexed.

7. **Testing strategy**: Should tests use mocked BookStack API responses and synthetic local documents, or require a live BookStack instance? Recommendation: mocked + synthetic for all unit/integration tests, including Markdown, plain text, JSON/YAML, and recipe fixtures. Test fixtures live in `tests/fixtures/` (see Milestone 1.1 project structure). Manual end-to-end testing against a live BookStack instance before each release.

8. **Image handling**: Phase 1 strips images from imported BookStack Markdown and does not OCR image-only files. Should Phase 2/3 support downloading images locally or OCR scanned recipes/PDFs? Recommendation: defer until there is a real corpus that needs it. Add `rag import bookstack --include-images` and/or `rag ingest --ocr` only when the dependency cost is justified.
