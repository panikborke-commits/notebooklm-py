# CLAUDE.md

**Status:** Active
**Last Updated:** 2026-05-24

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**IMPORTANT:** Follow the documentation and file-creation rules in [CONTRIBUTING.md](CONTRIBUTING.md) — especially the naming conventions and protected-section rules.

## Project Overview

`notebooklm-py` is an unofficial Python client for Google NotebookLM that uses undocumented RPC APIs. The library enables programmatic automation of NotebookLM features including notebook management, source integration, AI querying, and studio artifact generation (podcasts, videos, quizzes, etc.).

**Critical constraint**: This uses Google's internal `batchexecute` RPC protocol with obfuscated method IDs that Google can change at any time. All RPC method IDs in `src/notebooklm/rpc/types.py` are undocumented and subject to breakage.

## Development Commands

Use `uv` for all local development:

```bash
# Install all dependencies (dev + browser extras)
uv sync --extra dev --extra browser
playwright install chromium

# Run all tests (e2e excluded by default)
uv run pytest

# Run with coverage (must meet 90% threshold)
uv run pytest --cov

# Run linter
uv run ruff check src/ tests/

# Run formatter
uv run ruff format src/ tests/

# Type checking
uv run mypy src/notebooklm

# Run all pre-commit checks at once
uv run pre-commit run --all-files

# CLI testing
uv run notebooklm --help
```

## Pre-Commit Checks (REQUIRED before committing)

Always run these before committing to avoid CI failures:

```bash
uv run ruff format src/ tests/ && uv run ruff check src/ tests/ && uv run mypy src/notebooklm && uv run pytest
```

## Architecture

### Layered Design

```
CLI Layer (cli/)
    ↓
Client Layer (client.py, _*.py APIs)
    ↓
Core Layer (_core.py)
    ↓
RPC Layer (rpc/)
```

1. **RPC Layer** (`src/notebooklm/rpc/`):
   - `types.py`: All RPC method IDs and enums (source of truth — see protected sections below)
   - `encoder.py`: Request encoding
   - `decoder.py`: Response parsing

2. **Core Layer** (`src/notebooklm/_core.py`):
   - HTTP client management
   - RPC call abstraction
   - Request counter handling

3. **Client Layer** (`src/notebooklm/client.py`, `_*.py`):
   - `NotebookLMClient`: Main async client with namespaced APIs
   - `_notebooks.py`, `_sources.py`, `_artifacts.py`, etc.: Domain APIs

4. **CLI Layer** (`src/notebooklm/cli/`):
   - Modular Click commands
   - `session.py`, `notebook.py`, `source.py`, `generate.py`, etc.

### Key Files

| File | Purpose |
|------|---------|
| `client.py` | Main `NotebookLMClient` class |
| `_core.py` | HTTP and RPC infrastructure |
| `_notebooks.py` | `client.notebooks` API |
| `_sources.py` | `client.sources` API |
| `_artifacts.py` | `client.artifacts` API |
| `_chat.py` | `client.chat` API |
| `_research.py` | `client.research` API |
| `_notes.py` | `client.notes` API |
| `_sharing.py` | `client.sharing` API |
| `_settings.py` | Settings and configuration management |
| `_logging.py` | Internal logging setup |
| `_url_utils.py` | URL validation and normalization helpers |
| `_version_check.py` | Startup version-check against PyPI |
| `auth.py` | Authentication handling (cookies, CSRF tokens) |
| `exceptions.py` | Exception hierarchy |
| `paths.py` | Storage path resolution (`NOTEBOOKLM_HOME`) |
| `types.py` | Public dataclasses and type definitions |
| `notebooklm_cli.py` | CLI entry point (registered as `notebooklm` script) |
| `rpc/types.py` | RPC method IDs and enums (source of truth) |
| `cli/` | Click command modules |

### Repository Structure

```
src/notebooklm/
├── __init__.py          # Public exports
├── __main__.py          # python -m notebooklm entry point
├── client.py            # NotebookLMClient
├── auth.py              # Authentication
├── types.py             # Public dataclasses
├── exceptions.py        # Exception hierarchy
├── paths.py             # Storage path resolution
├── notebooklm_cli.py    # CLI entry point
├── _core.py             # Core HTTP/RPC infrastructure
├── _notebooks.py        # NotebooksAPI
├── _sources.py          # SourcesAPI
├── _artifacts.py        # ArtifactsAPI
├── _chat.py             # ChatAPI
├── _research.py         # ResearchAPI
├── _notes.py            # NotesAPI
├── _sharing.py          # SharingAPI
├── _settings.py         # Settings management
├── _logging.py          # Logging setup
├── _url_utils.py        # URL helpers
├── _version_check.py    # Version update checks
├── rpc/                 # RPC protocol layer
│   ├── types.py         # Method IDs and enums
│   ├── encoder.py       # Request encoding
│   └── decoder.py       # Response parsing
└── cli/                 # CLI implementation
    ├── __init__.py
    ├── helpers.py       # Shared utilities
    ├── session.py       # login, use, status, clear
    ├── notebook.py      # list, create, delete, rename
    ├── source.py        # source add, list, delete
    ├── artifact.py      # artifact commands
    ├── generate.py      # generate audio, video, etc.
    ├── download.py      # download commands
    ├── chat.py          # ask, configure, history
    └── note.py          # note commands

tests/
├── conftest.py          # Shared fixtures
├── vcr_config.py        # VCR.py configuration
├── cassettes/           # Recorded HTTP fixtures (checked in)
├── unit/                # Pure logic tests, no network
├── integration/         # VCR-backed HTTP flow tests
└── e2e/                 # Authenticated live tests (opt-in)
```

## API Patterns

### Client Usage

```python
# Correct pattern — uses namespaced APIs
async with await NotebookLMClient.from_storage() as client:
    notebooks = await client.notebooks.list()
    await client.sources.add_url(nb_id, url)
    result = await client.chat.ask(nb_id, question)
    status = await client.artifacts.generate_audio(nb_id)
```

### CLI Structure

Commands are organized as:
- **Top-level**: `login`, `use`, `status`, `clear`, `list`, `create`, `ask`
- **Grouped**: `source add`, `artifact list`, `generate audio`, `download video`, `note create`

### Skill Installation

Install the bundled SKILL.md for use with Claude Code and other compatible agents:

```bash
# Via the notebooklm CLI
notebooklm skill install

# Via npx (cross-agent discovery)
npx skills add notebooklm-py
```

## Testing Strategy

- **Unit tests** (`tests/unit/`): Pure logic tests — no network calls
- **Integration tests** (`tests/integration/`): VCR-backed HTTP flows using recorded cassettes
- **E2E tests** (`tests/e2e/`): Live API calls requiring authentication; excluded from default `pytest` run

### Pytest Markers

| Marker | Description |
|--------|-------------|
| `e2e` | Requires `notebooklm login` and live network; run with `pytest tests/e2e -m e2e` |
| `readonly` | Safe to run against a shared test notebook; `pytest tests/e2e -m readonly` |
| `vcr` | Uses VCR.py recorded cassettes |
| `variants` | Parameter-variant tests that consume API quota; skipped by default |

### Coverage

The minimum coverage threshold is **90%**. `pytest --cov` will fail if coverage drops below this.

### Recording VCR Cassettes

Integration tests use pre-recorded HTTP cassettes stored in `tests/cassettes/`. To re-record a cassette:

```bash
NOTEBOOKLM_VCR_RECORD=1 uv run pytest tests/integration/test_vcr_<name>.py -v
```

Commit the updated cassette alongside the test change.

### E2E Tests

```bash
# Authenticate first
notebooklm login

# Run read-only e2e tests against your test notebook
pytest tests/e2e -m readonly

# Run full e2e suite (may create/delete notebooks)
pytest tests/e2e -m e2e
```

## Coding Style & Conventions

- Target Python 3.10+
- 4-space indentation, double quotes
- Line length: 100 characters (ruff enforces)
- Module and test file names in `snake_case`
- Internal modules use `_` prefix (`_sources.py`); public exports go in `src/notebooklm/__init__.py`
- Prefer descriptive Click command names that match existing groups (`source`, `artifact`, `research`)
- All Python code should include type hints for public functions

## Commit Style

Follow Conventional Commits matching existing history:

```txt
feat(cli): add bulk source deletion command
fix(rpc): update artifact generation method ID
refactor(test): extract shared VCR fixture to conftest
style: run ruff formatter
docs(readme): update agent setup section
chore(deps): bump httpx to 0.28.0
```

## Common Pitfalls

1. **RPC method IDs change**: Check network traffic and update `rpc/types.py`
2. **Nested list structures**: Params are position-sensitive. Check existing implementations.
3. **Source ID nesting**: Different methods need `[id]`, `[[id]]`, `[[[id]]]`, or `[[[[id]]]]`
4. **CSRF tokens expire**: Use `client.refresh_auth()` or re-run `notebooklm login`
5. **Rate limiting**: Add delays between bulk operations
6. **Coverage threshold**: `pytest --cov` fails below 90% — ensure new code has tests

## AI Agent Rules (from CONTRIBUTING.md)

### File Creation Rules

1. **No Root Rule**: Never create `.md` files in the repository root unless explicitly instructed by the user.
2. **Modify, Don't Fork**: Edit existing files; never create `FILE_v2.md`, `FILE_REFERENCE.md`, or `FILE_updated.md` duplicates.
3. **Scratchpad Protocol**: All analysis, investigation logs, and intermediate work go in `docs/scratch/` with date prefix: `YYYY-MM-DD-<context>.md`
4. **Consolidation First**: Before creating new docs, search for existing related docs and update them instead.

### Protected Sections

Never modify content between `PROTECTED` and `END PROTECTED` markers without explicit user approval. In-source markers look like:

```python
# PROTECTED: Do not modify without approval
class RPCMethod(Enum):
    ...
# END PROTECTED
```

### Naming Conventions

| Type | Format | Example |
|------|--------|---------|
| Root GitHub files | `UPPERCASE.md` | `README.md`, `CONTRIBUTING.md` |
| Agent files | `UPPERCASE.md` | `CLAUDE.md`, `AGENTS.md` |
| Subfolder README | `README.md` | `docs/examples/README.md` |
| All other docs/ files | `lowercase-kebab.md` | `cli-reference.md` |
| Scratch files | `YYYY-MM-DD-context.md` | `2026-01-06-debug-auth.md` |

### Status Headers

Documentation files should include status metadata:

```markdown
**Status:** Active | Deprecated
**Last Updated:** YYYY-MM-DD
```

Ignore files marked `Deprecated`.

### Agent Isolation

When running multiple parallel agents on the same machine, isolate each agent's storage with:

```bash
NOTEBOOKLM_HOME=/tmp/<agent-id> uv run notebooklm ...
```

Pass explicit notebook IDs rather than relying on `notebooklm use` to avoid cross-agent state conflicts. Prefer `--json` output flags for programmatic parsing.

## When to Suggest CLI vs API

- **CLI**: Quick tasks, shell scripts, LLM agent automation
- **Python API**: Application integration, complex workflows, async operations

## Documentation

All docs use lowercase-kebab naming in `docs/`:

| File | Purpose |
|------|---------|
| `docs/cli-reference.md` | CLI commands reference |
| `docs/python-api.md` | Python API reference |
| `docs/configuration.md` | Storage and settings |
| `docs/troubleshooting.md` | Known issues |
| `docs/stability.md` | API versioning and stability policy |
| `docs/development.md` | Architecture, testing, releasing |
| `docs/releasing.md` | Release checklist |
| `docs/rpc-development.md` | RPC capture and debugging |
| `docs/rpc-reference.md` | RPC payload structures |
| `docs/examples/` | Runnable example scripts |
| `docs/scratch/` | Temporary agent investigation logs (date-prefixed, periodic cleanup) |
