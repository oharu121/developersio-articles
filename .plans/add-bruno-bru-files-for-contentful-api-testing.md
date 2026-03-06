# Plan: Add Bruno .bru files for Contentful API testing

**Status:** Completed
**Date:** 2026-03-06

## Context

The `scripts/contentful.py` CLI wraps 7 Contentful CMA API calls behind a Python
interface. While this works well for the `/publish` command automation, there was no
way to interactively explore or debug individual API endpoints. Testing required
reading the Python source to reconstruct curl commands or modifying the script itself.

## Approach

Create a Bruno API collection in `.bruno/` with one `.bru` file per API request used
in `contentful.py`. Bruno is an open-source API client (like Postman) that uses
plain-text `.bru` files — git-friendly and no account required. Collection-level
variables in `collection.bru` hold shared values (space ID, token, etc.), keeping
individual request files clean.

The CMA token is kept out of version control by gitignoring `collection.bru` and
providing `collection.example.bru` as a template with empty sensitive fields.

## Changes

### 1. Bruno collection setup
Created `bruno.json` (collection manifest) and `collection.bru` with `vars:pre-request`
block for shared variables: `base_url`, `space_id`, `environment_id`, `cma_token`,
`article_id`, `author_id`, `content_type_id`, `locale`.

### 2. Request files (7 endpoints from contentful.py)
Each file maps 1:1 to an `api_request()` call in the Python CLI:

- `verify-token.bru` — `GET /spaces/{space_id}` (token validation)
- `list-spaces.bru` — `GET /spaces` (space discovery)
- `list-authors.bru` — `GET .../entries?content_type=authorProfile&limit=100`
- `list-content-types.bru` — `GET .../content_types`
- `get-entry.bru` — `GET .../entries/{article_id}`
- `create-entry.bru` — `POST .../entries` with full JSON body and `X-Contentful-Content-Type` header
- `update-entry.bru` — `PUT .../entries/{article_id}` with JSON body and `X-Contentful-Version` header

### 3. Credential protection
Added `.bruno/collection.bru` to `.gitignore` since it contains the CMA token in
plain text. Created `collection.example.bru` as a safe template with empty values
for `space_id`, `cma_token`, `article_id`, and `author_id`.

## Files Modified

| File | Change |
|------|--------|
| [.bruno/bruno.json](.bruno/bruno.json) | **New** — Bruno collection manifest |
| [.bruno/collection.example.bru](.bruno/collection.example.bru) | **New** — Template with empty sensitive values |
| [.bruno/verify-token.bru](.bruno/verify-token.bru) | **New** — GET /spaces/{space_id} |
| [.bruno/list-spaces.bru](.bruno/list-spaces.bru) | **New** — GET /spaces |
| [.bruno/list-authors.bru](.bruno/list-authors.bru) | **New** — GET entries with authorProfile filter |
| [.bruno/list-content-types.bru](.bruno/list-content-types.bru) | **New** — GET content_types |
| [.bruno/get-entry.bru](.bruno/get-entry.bru) | **New** — GET entries/{article_id} |
| [.bruno/create-entry.bru](.bruno/create-entry.bru) | **New** — POST entries with full body |
| [.bruno/update-entry.bru](.bruno/update-entry.bru) | **New** — PUT entries/{article_id} |
| [.gitignore](.gitignore) | Added `.bruno/collection.bru` |

## Guard Rails

| Scenario | Behavior |
|----------|----------|
| Missing collection.bru | User copies collection.example.bru and fills in credentials |
| CMA token committed | Prevented — collection.bru is gitignored |
| Bruno not installed | .bru files are plain text, readable without Bruno |

## Verification

1. Open `.bruno/` folder in Bruno app — all 7 requests should load
2. Select any GET request (e.g., list-spaces), fill in `cma_token` in collection vars, send — should return spaces
3. Confirm `collection.bru` does not appear in `git status` after gitignore

## Breaking Changes

None
