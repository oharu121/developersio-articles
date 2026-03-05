# Plan: Add excerpt generation to publish flow

**Status:** Completed
**Date:** 2026-03-05

## Context

When publishing articles to Contentful via the `/publish` command, the excerpt field had to be set manually in Contentful's web UI after the draft was created. This meant an extra round-trip to the Contentful editor for every new article, and it was easy to forget — leaving published articles without a proper summary displayed on listing pages.

The excerpt was deliberately excluded from the original publish command design because it was grouped with other "Contentful-only" fields like thumbnail and categories. However, unlike those visual/organizational fields, the excerpt can be reliably generated from the article content itself, making it a natural fit for automation.

## Approach

Rather than adding excerpt to the article frontmatter (which would require the AI assistant to generate it during drafting, before the content is finalized), we generate it at publish time — when the article is complete and ready for review. The excerpt is generated from the final article body, presented to the user for approval or editing via `AskUserQuestion`, then included in the Contentful API call.

For updates, since the excerpt may have been manually refined in Contentful, we show the current value and let the user choose: keep it, regenerate from updated content, or rewrite it themselves.

## Changes

### 1. New article publish flow (新規作成フロー)
Added a new step between content confirmation and the API call. The command generates a 1-2 sentence excerpt from the article body, presents it via `AskUserQuestion` for user approval/editing, and includes the approved text in the `POST /entries` request.

### 2. Update flow (更新フロー)
Added excerpt handling after field diff display. The command fetches the existing excerpt from Contentful and presents three options via `AskUserQuestion`: keep the current excerpt, regenerate from updated content, or write a custom one. The warning message was updated to list excerpt as a locally-managed field (removed from the "preserved" list).

### 3. API request body
Added `"excerpt": { "{locale}": "記事の概要文" }` to the new entry creation JSON example, so the command includes the excerpt field in the Contentful CMA request.

### 4. Contentful config fields mapping
Added `"excerpt": "excerpt"` to the `fields` object in the `contentful-config.json` template, ensuring the field name is discoverable during auto-configuration.

### 5. Merge rules
Added `fields.excerpt` to the list of locally-managed fields that get overwritten during updates. Removed `excerpt` from the comment listing preserved Contentful-only fields.

## Files Modified

| File | Change |
|------|--------|
| [.claude/commands/publish.md](.claude/commands/publish.md) | Added excerpt generation/approval steps to both new and update flows, updated API spec, config template, merge rules, and warning messages |

## Guard Rails

| Scenario | Behavior |
|----------|----------|
| User rejects generated excerpt | Can edit via "Other" free input in AskUserQuestion |
| Update with no existing excerpt on Contentful | Generates a new one (same as new article flow) |
| Update with existing excerpt user wants to keep | "Keep current" option preserves the Contentful value |
| Excerpt field doesn't exist in content type | Contentful API ignores unknown fields in the request — no error |

## Verification

1. Run `/publish` on a new article (no `articleId`) — confirm excerpt is generated, shown for approval, and included in the Contentful draft
2. Run `/publish` on an existing article (with `articleId`) — confirm the current excerpt is displayed and user can choose to keep, regenerate, or rewrite
3. Check that the warning message in the update flow lists `excerpt` in the locally-managed fields

## Breaking Changes

None
