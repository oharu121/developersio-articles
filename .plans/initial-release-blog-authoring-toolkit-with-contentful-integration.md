# Plan: Initial release: blog authoring toolkit with Contentful integration

**Status:** Completed
**Date:** 2026-03-05

## Context

Writing DevelopersIO blog articles involved multiple manual steps: drafting content while remembering the media guidelines, organizing files into categories, choosing tags, then copy-pasting everything into Contentful's web UI. Each step was disconnected, and the guidelines were easy to forget or overlook.

The goal was to create a streamlined workflow where a single CLI tool handles the entire pipeline — from article drafting to CMS publishing — while enforcing the DevelopersIO media guidelines automatically.

## Approach

Use Claude Code's custom command feature (`.claude/commands/*.md`) to define slash commands that orchestrate the full workflow. This approach requires zero external dependencies — no npm packages, no Python scripts, no build tooling. The commands are plain Markdown files that instruct Claude on what to do, and Claude handles the rest (file I/O, API calls via curl, interactive prompts).

For Contentful integration, the command uses the Content Management API directly. A one-time auto-discovery flow detects the space ID, content type, and author profile from the CMA token, saving the results to a local config file for future runs.

Key design decisions:
- Commands are Markdown, not scripts — portable, readable, version-controllable
- Contentful config is auto-generated, not manually maintained
- CMA token is the only secret (`.env`), everything else is derived from it
- Articles use YAML frontmatter for metadata, keeping content and config in one file
- Update flow uses fetch-merge-put to preserve fields set directly in Contentful (thumbnails, excerpts, etc.)

## Changes

### 1. `/article` command
New file `.claude/commands/article.md` that creates DevelopersIO-compliant blog articles. The command:
- Lists existing category folders and proposes relevant ones (or suggests new ones)
- Proposes tags based on the topic for user selection
- Generates a Markdown article with YAML frontmatter (title, slug, tags)
- Embeds the full DevelopersIO media guidelines (dis policy, citation rules, AWS guidelines, AI usage policy)
- Uses the slug as both the filename and URL key

### 2. `/publish` command
New file `.claude/commands/publish.md` that publishes articles to Contentful as drafts. The command has two prerequisite checks:
- **Token check**: Verifies `CONTENTFUL_CMA_TOKEN` exists in `.env`, guides user through setup if missing
- **Config auto-discovery**: If `.claude/contentful-config.json` doesn't exist, uses the CMA API to detect space ID (`GET /spaces`), content type (`GET /content_types`), and author profile (`GET /entries?content_type=authorProfile`), then saves the config

Two publish flows:
- **Create** (no `articleId` in frontmatter): `POST /entries` to create draft, writes back `articleId` to frontmatter
- **Update** (`articleId` present): `GET /entries/{id}` to fetch current state, merges only locally-managed fields (title, slug, content, tags), `PUT /entries/{id}` with `X-Contentful-Version` to update. This preserves thumbnails, excerpts, and other fields set in Contentful's UI.

### 3. `/release` command
Adapted from another project's release command for this markdown-only repo. Handles GitHub issue creation with plan files and AC tables, git tagging, GitHub releases. No build/lint/test steps since this is a content repo.

### 4. Project infrastructure
- `.env.example` — template with `CONTENTFUL_CMA_TOKEN` placeholder
- `.gitignore` — excludes `.env` and `.claude/contentful-config.json`
- `.claude/contentful-config.json` — auto-generated Contentful connection config (space ID, content type, author ID, locale, field mappings)

## Files Modified

| File | Change |
|------|--------|
| [.claude/commands/article.md](.claude/commands/article.md) | **New** — article writing command with DevelopersIO guidelines |
| [.claude/commands/publish.md](.claude/commands/publish.md) | **New** — Contentful publishing command with create/update flows |
| [.claude/commands/release.md](.claude/commands/release.md) | **New** — release automation adapted for markdown repo |
| [.env.example](.env.example) | **New** — CMA token template |
| [.gitignore](.gitignore) | **New** — excludes secrets and local config |

## Guard Rails

| Scenario | Behavior |
|----------|----------|
| Missing CMA token | `/publish` stops with setup instructions linking to Contentful token page |
| Missing config file | Auto-discovery runs: detects space, content type, asks user to pick author |
| Duplicate slug on Contentful | API returns validation error, command reports it |
| Update would wipe Contentful-only fields | Fetch-merge-put preserves all non-local fields (thumbnail, excerpt, etc.) |
| Concurrent edit on Contentful | 409 Conflict detected, user prompted to retry |
| No existing category folders | `/article` proposes new folder names based on topic |

## Verification

1. Run `/article <topic>` — confirm folder selection, tag proposal, and article generation with frontmatter
2. Run `/publish <article-path>` on a new article — confirm draft created in Contentful with correct fields
3. Edit article locally, run `/publish` again — confirm update (not duplicate) in Contentful
4. Run `/publish` without `.env` — confirm setup instructions are shown
5. Run `/publish` without `contentful-config.json` — confirm auto-discovery flow runs

## Breaking Changes

None (initial release)
