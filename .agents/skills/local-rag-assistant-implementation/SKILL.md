---
name: local-rag-assistant-implementation
description: Project-specific implementation rules for the local RAG assistant. Use when Codex or OpenCode works on this repository's Python CLI MVP, document loaders, recipe JSON/YAML support, BookStack importer, ingestion/chunking, Ollama/Chroma adapters, retrieval, diagnostics, tests, evaluation set, privacy rules, or implementation planning/review.
---

# Local RAG Assistant Implementation

## Required First Reads

1. Read `docs/rules/local-rag-assistant.md` — canonical project rules.
2. Read `docs/plans/local-rag-assistant/IMPLEMENTATION_PLAN.md` — full plan.
3. Read only the plan sections relevant to the current task.

## Current Scope

- Assume Phase 1 MVP unless the user explicitly changes the plan.
- Build a CLI-first local RAG assistant.
- Support local files in `docs/` through document loaders.
- Implement `.md`, `.txt`, `.json`, `.yaml`, and `.yml` loaders first.
- Support structured recipes in JSON/YAML.
- Keep BookStack as an optional importer into `docs/`.

Do not add Phase 2 features unless explicitly requested:

- PDF, DOCX, EPUB, HTML, OCR.
- Hybrid search, reranking, query decomposition.
- Autonomous writing agents.
- External model providers beyond interface-ready design.

## Architecture Contract

Use this pipeline for all ingested content:

```text
DocumentLoader -> Document -> Chunker -> Embedder -> VectorStore
```

Rules:

- Normalize every source format into the internal `Document` model before chunking.
- Keep query/generation code format-agnostic.
- Keep domain logic independent from LangChain.
- Put LangChain-specific imports only inside adapters or future agent modules.
- Use internal interfaces: `Embedder`, `VectorStore`, `Retriever`, `Generator`.
- Keep Ollama and Chroma behind adapters.

## Loader Contract

Every loader must:

- Accept a source path.
- Return `Document(id, title, text, format, source_file, metadata)`.
- Preserve useful metadata for citations and debugging.
- Raise clear, user-facing errors for malformed targeted files.
- Be skipped with a warning for unsupported files discovered during normal ingestion.

Required Phase 1 loaders:

- Markdown: preserve headings.
- Plain text: infer title from first non-empty line or filename.
- JSON/YAML: flatten predictable fields into readable text and preserve original keys in metadata.
- Recipes: render `title`, `tags`, `servings`, `prep_time`, `cook_time`, `ingredients`, `steps`, and `notes` as searchable sections when present.

## Privacy Rules

Never index these by default:

- hidden files or hidden directories
- `.git/`
- `.venv/`
- `node_modules/`
- `__pycache__/`
- `.env`
- `.env.*`
- private keys
- binary files
- backup/temp files

Use `RAG_EXCLUDE_PATTERNS` for user-configurable exclusions.

## Diagnostics

Prefer diagnostic commands before changing generation behavior:

- `rag ingest --dry-run`
- `rag inspect <path>`
- `rag search "query"`
- `rag status`
- `rag eval`

Use `rag search` to determine whether a problem is retrieval or generation.

## Testing Rules

- Use the project-installed `python-testing-patterns` skill when designing or fixing Python tests.
- Add fixture tests for every loader.
- Test malformed structured input.
- Test metadata preservation.
- Test unsupported file skipping.
- Test CLI success and user-facing errors.
- Run `rag eval` or update `tests/eval/questions.yaml` when changing embeddings, chunking, loader normalization, prompts, or model defaults.

## Implementation Style

- Prefer typed Python and small modules.
- Use `pathlib`.
- Use Pydantic for config and data contracts.
- Use deterministic parsers (`json`, `pyyaml`) rather than ad hoc parsing.
- Use timeouts for network calls.
- Retry only idempotent network operations.
- Keep dependencies minimal.
- Do not stage, commit, push, rebase, or alter Git history unless explicitly requested.

## Useful References

- For a compact implementation checklist, read `references/implementation-checklist.md`.
