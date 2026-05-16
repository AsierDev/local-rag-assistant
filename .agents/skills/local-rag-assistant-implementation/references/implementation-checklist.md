# Implementation Checklist

Use this checklist before finalizing implementation work.

## Scope

- Work matches the current approved phase in `docs/plans/local-rag-assistant/IMPLEMENTATION_PLAN.md`.
- No Phase 2 dependency or feature was added to Phase 1.
- No unrelated refactor was introduced.

## Architecture

- Source files pass through `DocumentLoader -> Document -> Chunker -> Embedder -> VectorStore`.
- Domain modules do not import LangChain.
- Ollama and Chroma usage stays inside adapters.
- Query/generation code is format-agnostic.

## Loaders

- Loader returns a normalized `Document`.
- Loader preserves metadata needed for citations and debugging.
- Malformed targeted input has a clear error.
- Unsupported discovered files are skipped with a warning.
- Recipe fields are searchable when present.

## Privacy

- Hidden files are excluded by default.
- Secrets, private keys, dependency directories, binary files, and backups are excluded.
- `rag ingest --dry-run` reports skipped/excluded counts.

## Diagnostics

- `rag inspect <path>` works for changed loaders.
- `rag search "query"` can show retrieval results without generation.
- `rag status` reports index/model/source state.

## Tests

- Loader fixtures cover success, malformed input, and metadata.
- CLI tests cover success and user-facing errors.
- Retrieval/chunking/model changes update or run `rag eval`.
