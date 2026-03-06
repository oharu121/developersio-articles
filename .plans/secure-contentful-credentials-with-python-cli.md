# Plan: Secure Contentful credentials with Python CLI

**Status:** Completed
**Date:** 2026-03-06

## Context

The `/publish` command was working well for creating and updating Contentful draft entries, but it had a fundamental security flaw: the LLM was reading the `.env` file directly to extract the `CONTENTFUL_CMA_TOKEN`, then constructing curl commands with `Authorization: Bearer {token}` headers. This meant the CMA token — a credential with full write access to the Contentful space — was being sent to Anthropic's API as part of the conversation context on every publish operation.

The user raised the concern that credentials should never need to leave the local machine. The LLM's value in the publish flow is excerpt generation and user interaction, not HTTP request construction — a deterministic script handles API calls more reliably anyway.

## Approach

Extract all Contentful API interactions into a standalone Python CLI script (`scripts/contentful.py`) that reads `.env` internally. The LLM calls the script via Bash and receives only structured JSON results — entry IDs, URLs, field values — never the token itself.

Python was chosen over Node.js to avoid adding package management to a markdown-only repo. The script uses only stdlib modules (`urllib.request`, `json`, `re`, `pathlib`) so no `pip install` is needed. A simple regex-based frontmatter parser replaces a full YAML library dependency, since the project's frontmatter format is straightforward (strings, arrays, dates).

The `from __future__ import annotations` import ensures compatibility with macOS's system Python 3.9 while allowing modern type hint syntax.

## Changes

### 1. Python CLI script (`scripts/contentful.py`)

A single-file CLI with four subcommands:

- **`setup --check`** — Verifies `.env` has a valid token and `.claude/contentful-config.json` exists, returns `{"ok": true/false}` without exposing the token
- **`setup --list-spaces/--list-authors/--list-content-types/--save`** — Interactive config discovery flow, returning JSON lists for the LLM to present as choices
- **`get <article-file>`** — Reads `articleId` from frontmatter, fetches the entry from Contentful, returns localized field values and version number for diff comparison
- **`create <article-file> --excerpt "..."`** — Parses frontmatter and body, POSTs a new draft entry, returns `entry_id` and Contentful URL
- **`update <article-file> --excerpt "..."`** — Fetch-merge-put pattern: fetches existing entry, merges local fields (title, slug, content, tags, excerpt) while preserving Contentful-only fields (thumbnail, categories), PUTs with `X-Contentful-Version` header for optimistic locking

All output is JSON to stdout; errors are JSON to stderr with non-zero exit codes.

### 2. Rewritten publish command (`.claude/commands/publish.md`)

The command now explicitly states **"Do not read `.env` directly"** at the top. The flow changed from:

- Before: `grep CONTENTFUL_CMA_TOKEN .env` → construct curl → execute → parse response
- After: `python3 scripts/contentful.py <subcommand>` → parse JSON response

The LLM's responsibilities are now limited to:
- Generating excerpts (creative task where LLM adds value)
- User interaction via `AskUserQuestion` (confirmations, diff display, excerpt choices)
- Writing back `articleId` and `publishedAt` to frontmatter after successful creation

## Files Modified

| File | Change |
|------|--------|
| [scripts/contentful.py](scripts/contentful.py) | **New** — stdlib-only Python CLI handling all Contentful CMA API calls |
| [.claude/commands/publish.md](.claude/commands/publish.md) | **Rewritten** — delegates API calls to Python script, removes all direct token/curl usage |

## Guard Rails

| Scenario | Behavior |
|----------|----------|
| `.env` missing or no token | Script returns `{"error": "..."}` to stderr, exit 1. LLM shows setup instructions |
| Config file missing | `setup --check` returns `{"ok": false, "has_config": false}`, triggers auto-discovery flow |
| `create` on article with existing `articleId` | Script rejects with error: "use 'update' instead" |
| `update` on article without `articleId` | Script rejects with error: "use 'create' instead" |
| 409 Conflict on update | Script reports HTTP 409 error; LLM advises re-fetch and retry |
| Python 3.9 (macOS system) | `from __future__ import annotations` ensures compatibility |

## Verification

1. `python3 scripts/contentful.py setup --check` — returns `{"ok": true}` without exposing token
2. `python3 scripts/contentful.py get gemini/gemini-workspace-search-instead-of-rag.md` — returns entry JSON with fields and version
3. `python3 scripts/contentful.py create <unpublished-article> --excerpt "test"` — creates draft in Contentful
4. `python3 scripts/contentful.py update <published-article> --excerpt "test"` — updates entry preserving Contentful-only fields
5. Confirm CMA token never appears in Claude conversation context during `/publish` flow

## Breaking Changes

The `/publish` command now requires Python 3.9+ to be available. This is present by default on macOS. The command syntax and user-facing behavior are unchanged — users still run `/publish <file>` and interact with the same prompts.
