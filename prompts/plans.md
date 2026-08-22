# Plans

## Current State

Basic Memory is an established, actively maintained software project. Agent OS project context was initialized on
2026-08-22 by conservatively extending the existing `AGENTS.md` and adding the standard `prompts/` files. No application
code or runtime behavior was changed.

## Active Plan

There is no active implementation plan. The current maintenance baseline is:

- Preserve the existing repository-specific rules in `AGENTS.md`.
- Keep `prompts/architecture.md` focused on current top-level architecture; use the existing detailed architecture and
  domain documentation for module-level work.
- Record future implementation progress and validation here so another agent can resume safely.
- Treat metadata filterability and physical database indexing as distinct: arbitrary fields are queryable, while only
  selected fields have dedicated indexes and vector adapters do not receive metadata predicates.

## Validation

- Repository root and immediate directory organization surveyed.
- Existing `AGENTS.md`, root project metadata, README headings, and the architecture document's top-level framing
  reviewed.
- Agent OS prompt index and links checked after creation.
- Metadata-filter execution, structured-index migrations, vector candidate filtering, and `embed` opt-out behavior were
  traced through the SQLite and PostgreSQL search repositories and search service on 2026-08-22.
- No code tests are required for this documentation-only setup.

## Safe Resume

1. Read the repository `AGENTS.md` and follow its project-specific development rules.
2. Read `prompts/architecture.md` for the current system map.
3. Check `git status` and preserve unrelated work.
4. Replace this empty active-plan section with the next concrete task, success criteria, and validation commands.
5. Update this file after each implementation and verification chunk.
