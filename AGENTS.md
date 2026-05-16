# Local RAG Assistant — AGENTS.md

Canonical rules live in `docs/rules/local-rag-assistant.md` — read that first.
This file is a lightweight scaffold; OpenCode also loads it via `instructions`
in `opencode.json`.

## Required First Reads

1. Read `docs/rules/local-rag-assistant.md` — project rules, scope, architecture.
2. Read `docs/plans/local-rag-assistant/IMPLEMENTATION_PLAN.md` — full plan.
3. Read only the plan sections relevant to the current task.

## Project Skills (load by name)

| Skill | Where discovered | When to use |
|-------|-----------------|-------------|
| `local-rag-assistant-implementation` | `.agents/skills/` | Implementation, review, or planning |
| `python-testing-patterns` | `.agents/skills/` | Writing or fixing Python tests |
| `find-skills` | `.agents/skills/` | Discovering skills for new/unknown domains |
| `web-design-guidelines` | `.agents/skills/` | Reviewing Streamlit UI accessibility/UX |

Do not load React/Next.js/web UI skills unless the project moves to React.

## Implementation Constraints

- Keep work inside the approved phase or user-requested scope.
- Do not import LangChain in domain modules.
- Do not add PDF/DOCX/EPUB/OCR dependencies during Phase 1.
- Do not index hidden files, secrets, dependency directories, binary files, or backups.
- Do not stage, commit, push, rebase, or alter Git history unless explicitly requested.
- Run `rag eval` when changing retrieval, embeddings, chunking, prompts, or model defaults.
- Use `rag ingest --dry-run`, `rag inspect <path>`, `rag search "query"`, and `rag status` to debug before changing generation.
