# Local RAG Assistant — Project Rules

Canonical rules. Every AI agent (OpenCode, Codex, Claude Code, Cursor, etc.)
should be configured to reference this file as the single source of truth.

---

## Source Of Truth

- Read `docs/plans/local-rag-assistant/IMPLEMENTATION_PLAN.md` before planning or implementing.
- Treat the current approved scope as Phase 1 MVP unless the user explicitly updates the plan.
- Keep technical artifacts, code comments, commit messages, plan headings, and acceptance criteria in English.
- User-facing conversation can be in Spanish when the user writes in Spanish.

## Architecture Contract

- The ingestion pipeline is loader-first: `DocumentLoader -> Document -> Chunker -> Embedder -> VectorStore`.
- All supported source formats must be normalized into the internal `Document` model before chunking.
- Do not add format-specific parsing logic to query, generation, or vector-store code.
- Keep domain logic independent from LangChain. LangChain imports are allowed only inside adapters or future agent modules.
- Use internal interfaces for model and storage boundaries: `Embedder`, `VectorStore`, `Retriever`, `Generator`.
- BookStack is an optional importer into `docs/`, not the canonical source of truth.

## Phase 1 Scope

Phase 1 includes:

- CLI-first local RAG MVP.
- Local files under `docs/`.
- Loaders for `.md`, `.txt`, `.json`, `.yaml`, `.yml`.
- Structured recipe support for JSON/YAML.
- Optional BookStack import into Markdown files.
- Ollama embeddings/generation through adapters.
- Chroma local vector store.
- Diagnostic commands: `status`, `inspect`, `search`, `reset-index`, `eval`.

Phase 1 explicitly excludes unless the plan is changed:

- PDF, DOCX, EPUB, HTML, OCR, and scanned documents.
- Web UI except as a stretch goal.
- Hybrid search, reranking, and query decomposition.
- Document writing agents and autonomous editing.
- External model providers beyond adapter-ready interfaces.

## Privacy & Safety

- Never index hidden files or hidden directories by default.
- Never index `.git/`, `.venv/`, `node_modules/`, `__pycache__/`, `.env`, `.env.*`, private keys, binary files, or backup/temp files.
- Use `RAG_EXCLUDE_PATTERNS` for user-configurable exclusions.
- `rag ingest --dry-run` must show skipped and excluded counts.
- File-system writes must stay under project-managed directories (`docs/`, `data/`, `drafts/`) unless explicitly configured.
- Do not stage, commit, push, rebase, or alter Git history unless the user explicitly asks.

## Implementation Standards

- Prefer typed Python, `pathlib`, small modules, and explicit Pydantic models for config/data contracts.
- Use deterministic parsers for structured data; avoid ad hoc string parsing when Python libraries are available.
- Network calls must use timeouts and clear user-facing errors.
- Retry only idempotent network operations.
- Keep dependencies minimal. Do not add Phase 2 parser dependencies to the MVP.
- Every loader needs fixture tests for success, malformed input, and metadata.
- Every CLI command needs a basic success test and at least one user-facing error test.
- Any change to model defaults, embedding prefixes, chunking, loaders, or prompt behavior must update or run the evaluation set.

## Required Validation

- Run the narrowest relevant tests for changed code.
- Run loader tests when touching `sources.py`, `documents.py`, or `loaders/`.
- Run diagnostic CLI tests when touching CLI commands.
- Run `rag eval` when changing retrieval, embeddings, chunking, prompts, or model defaults.
- Use `rag inspect <path>` and `rag search "query"` to debug ingestion/retrieval before changing generation.

## Skills Reference

Project-installed skills available to agents:

| Name                                 | Location                                | When to use                                                            |
| ------------------------------------ | --------------------------------------- | ---------------------------------------------------------------------- |
| `local-rag-assistant-implementation` | `.opencode/skills/` + `.agents/skills/` | Implementation, review, or planning work for the RAG assistant project |
| `python-testing-patterns`            | `.agents/skills/`                       | Designing, writing, reviewing, or fixing Python tests                  |
| `find-skills`                        | `.agents/skills/`                       | Discovering new agent skills for unplanned tasks                       |
| `web-design-guidelines`              | `.agents/skills/`                       | Reviewing web UI (Streamlit) for accessibility and UX                  |

Do not use React, Next.js, or web UI skills unless the project explicitly moves from Streamlit to a React-based UI.
