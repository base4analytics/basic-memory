# Architecture

## System Shape

Basic Memory is a local-first knowledge-management system and Python package built around the Model Context Protocol.
Markdown note files are canonical user content; database, graph, search, and materialization records are derived state.
The product is exposed through three primary entry points: a FastAPI HTTP API, an MCP server for AI clients, and a Typer
command-line interface. Local and cloud project routing share the same product surface while selecting different runtime
adapters.

At the highest level, requests flow from an entry point through explicitly wired application services and persistence,
with MCP operations using typed HTTP clients to reach the API:

`CLI / MCP / API -> application services -> repositories and indexing -> files and databases`

## Top-Level Repository Organization

- `src/basic_memory/` — the primary Python application and package. It contains the API, MCP, CLI, domain behavior,
  persistence, indexing, synchronization, and runtime composition. This document intentionally does not enumerate its
  internal modules.
- `tests/` — fast unit and component tests, organized broadly around the application concerns.
- `test-int/` — integration, smoke, and realistic end-to-end behavior tests.
- `benchmarks/` — performance-oriented fixtures, tests, scripts, and supporting documentation.
- `plugins/` — host-native plugin distributions, currently including Claude Code and Codex packaging.
- `skills/` — canonical, framework-independent Basic Memory agent skills.
- `integrations/` — separately packaged host integrations, currently Hermes and OpenClaw.
- `docs/` — engineering architecture, domain model, specifications, and release documentation.
- `scripts/` — repository maintenance, release, validation, and automation utilities.
- `.agents/`, `.claude/`, `.claude-plugin/`, `.codex/` — agent-harness instructions, skills, commands, and package
  metadata.
- `.github/` — CI, release, contribution, and repository automation.
- Root manifests and tooling — `pyproject.toml` and `uv.lock` define the Python package and dependencies; `justfile`
  provides the canonical development commands; Docker Compose and container files support reproducible local and
  PostgreSQL workflows; service manifests describe external distribution surfaces.
- Root project documentation — `README.md`, `CONTRIBUTING.md`, `SECURITY.md`, `NOTE-FORMAT.md`, and related policy and
  release files describe product use, contribution, and governance.
- `prompts/` — active Agent OS architecture and implementation handoff context.

## Architectural Boundaries

- The Python package is the core product implementation; top-level plugins, skills, and integrations adapt or package
  that capability for specific AI hosts rather than forming a second application core.
- API, MCP, and CLI are separate composition roots. Global configuration is read at those boundaries and passed inward
  explicitly.
- Markdown bytes are canonical. Database metadata, graph relations, search indexes, and materialization status are
  derived and eventually consistent by design.
- The repository supports SQLite for the default local path and PostgreSQL for parity and hosted-oriented validation.
- Root `just` targets are the stable orchestration layer for checks across the Python core and the separately packaged
  agent integrations.

## Metadata-Filtered Search

Basic Memory makes every valid custom frontmatter path *filterable*, but it does not create a physical database index
for every user-defined key. These are separate concepts:

- Users may write arbitrary top-level or nested metadata and query it through `metadata_filters`; `tags` and `status`
  are convenience parameters that become metadata filters.
- Users may choose ordinary note tags and custom metadata values, but there is no user-facing declaration for creating
  additional database indexes. Adding an indexed metadata field currently requires a schema migration and matching
  query implementation.
- SQLite has generated, indexed columns for the common `status`, `type`, and `tags` fields. Equality and range predicates
  on `status` and `type` can use those columns. Tag containment is implemented with `json_each` plus compatibility
  `LIKE` predicates, so the B-tree index on the complete `tags_json` value does not make general tag-membership queries
  index-efficient. Arbitrary and nested keys use `json_extract` and normally require row-by-row JSON evaluation.
- PostgreSQL defines a general GIN index over `entity.entity_metadata` and dedicated expression indexes for `tags`,
  `type`, and `status`. Current structured-filter SQL uses `jsonb_extract_path[_text]` expressions, however, rather than
  predicates that directly match those indexed expressions or apply containment to the whole JSONB column. The indexes
  exist, but the current query shape should not be assumed to use them; confirm important workloads with `EXPLAIN`.

### Field Ownership and Lifecycle Use

The three optimized fields are user-authored frontmatter, but they are not interchangeable flags:

- `status` is an unconstrained scalar. Basic Memory does not reserve an enum or assign lifecycle semantics to its
  values; `search_notes(status=...)` is a convenience form of an exact metadata filter. A consumer may therefore define
  values such as `active` and `deleted`, making `status` the best fit of the three for an application-level soft delete.
- `type` is also user-selected, but it is the note's semantic class. `write_note` exposes it as `note_type`, defaults it
  to `note`, and prevents extra `metadata` from overriding the standard field. Overloading it as deletion state would
  discard useful note classification and interfere with note-type search and schema operations.
- `tags` are user-selected labels. They can represent lifecycle state, but tag membership has the weaker SQLite query
  behavior described above and mixes workflow state with topical categorization. They are a poorer soft-delete key.

The structured-filter grammar has no negation or existence operator. A consumer that uses soft delete must stamp every
live note positively (for example, `status: active`) and make every ordinary retrieval positively require that value;
filtering for "not deleted" is not available, and notes with a missing `status` do not match.

For a GBrain-style consumer, the strongest single-project soft-delete representation supported by this code is the
combination `status: deleted` plus `embed: false`. The positive `status: active` filter excludes deleted notes from
ordinary full-text and metadata-filtered results, while `embed: false` removes their chunks from the vector index so
they do not consume semantic candidate slots. The Markdown note and full-text rows remain available for an explicit
`status: deleted` cleanup search and later `delete_note`. This is implementation evidence from this fork, not proof that
every deployed Basic Memory Cloud version exposes identical behavior; confirm the cloud version before depending on it.

The sibling GBrain project selected a stronger two-project boundary instead:
ordinary recall is pinned to `bpb-brain/Brain`, and forgotten chains live in
`bpb-brain/Brain Trash`. This avoids making deleted-note exclusion depend on
metadata-filter or vector-filter behavior. Basic Memory's `move_note` has no
cross-project destination, so that consumer must implement forget as
tombstone, deterministic copy, exact verification, and only then note-level
source deletion. `status` and `embed` are therefore evidence and possible
defense-in-depth for that consumer, not its authoritative deletion boundary.

Full-text searches apply metadata predicates in the SQL query by joining search rows to their owning entity. Vector
and hybrid searches use a less selective two-stage path:

1. Embed the query and retrieve a bounded nearest-neighbor chunk set from the configured vector adapter. The normal
   candidate window is at least `semantic_vector_k` (default 100) and grows with pagination or reranking.
2. Run a separate filter-only relational search, capped at 50,000 search rows, and intersect its allowed `(type, id)`
   keys with the vector candidates.

Metadata is therefore not pushed into SQLite `sqlite-vec`, PostgreSQL `pgvector`, or the external vector adapter. A
selective filter still pays for unfiltered nearest-neighbor retrieval plus the relational filter query, and matching
items outside the bounded vector candidate window cannot be recovered by the later intersection. Hybrid search applies
the filter to both its full-text and vector branches, but its vector branch retains this candidate-window limitation.

### The `embed` Frontmatter Field

`embed` is a special metadata key, not an ordinary tag and not a request to create an index. It controls semantic
indexing for the owning note:

- Missing `embed` defaults to enabled.
- Boolean false, false-like strings (`false`, `0`, `no`, `off`), and numeric zero opt the note out.
- Boolean true, true-like strings, and nonzero numbers enable it; unrecognized values default to enabled so malformed
  metadata does not silently remove a note from semantic search.
- Opting out clears or skips that note's derived vector chunks. It does not remove the Markdown note or its full-text
  search rows, so text search remains available.

Users can set `embed` directly in YAML frontmatter or pass it as custom metadata through note-writing interfaces.

## Filename and Permalink Normalization

`Entity.safe_title` and `Entity.permalink` are related but distinct provider projections. With the default
`kebab_filenames=false`, a generated Markdown filename may preserve an underscore in its sanitized title, while
`generate_permalink()` always converts spaces and underscores to hyphens across every path segment. Consumers that
precompute deterministic paths should therefore make both directories and generated storage titles permalink-safe
rather than assuming filename text and permalink text are interchangeable.

The sibling GBrain Phase 5 implementation applies this to the Basic Memory provider schema for application type
`system_schema`: its schema entity and protected application note type remain `system_schema`, but its deterministic
provider-schema storage title is `gbrain-v0-system-schema-provider-schema`. It also hyphenates deterministic protected
record directories such as `canonical/access-policy/` and `system/gbrain-basic-memory-schema-v0/`. This avoids a
possible divergence between generated file paths and provider permalinks. This is source-level behavior from the fork;
the deployed Cloud response still remains the final contract evidence when that separately approved artifact is first
written.

## Authoritative Detail

This file is a top-level map, not a replacement for detailed design documentation. Use `docs/ARCHITECTURE.md` for layer
and dependency detail, `docs/DOMAIN_MODEL.md` for domain invariants and ownership, and `AGENTS.md` for development and
consistency rules.
