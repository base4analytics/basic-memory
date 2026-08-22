# Project Context

This directory contains the active Agent OS context for Basic Memory. It lets development agents recover the current
architecture and implementation state without relying on chat history.

## Standard Files

- `README.md` — explains this directory and indexes its files.
- `architecture.md` — records current architecture decisions, important defaults, project structure, interfaces, and
  file formats. Read it before making changes in those areas.
- `plans.md` — records current implementation state, active work, validation results, maintenance guidance, and
  safe-resume steps. Read it before continuing implementation or maintenance.

Optional `prompt_dev_*.md` files contain human-authored implementation instructions. Treat them as read-only unless Ben
explicitly asks for an edit.

Optional dated `*_deep_records_*.md` files contain historical chronology, superseded details, or archived validation
notes. Do not read them during routine development unless Ben explicitly requests historical context or an active file
points to a specific record for a current question.

## Update Rules

- Keep `architecture.md` and `plans.md` concise, current, and useful for resuming work.
- Update `architecture.md` when architecture, interfaces, file formats, defaults, or major behavior change.
- Update `plans.md` after implementation or validation work with what changed, what was verified, and what remains.
- Move only strictly historical material into a dated deep-record file; keep uncertain or maintenance-relevant context
  active.
- When adding, renaming, or deleting a prompt file, update this file and the prompt index in the repository `AGENTS.md`.
- Preserve human-authored prompt files and project-specific instructions.

## Deep Records Index

Deep-record files are historical archives. Do not read them during routine development unless Ben explicitly asks for
historical context or an active file points to one for a current question.

- None yet.
