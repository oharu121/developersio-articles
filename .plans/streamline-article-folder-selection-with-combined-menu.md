# Plan: Streamline /article folder selection with combined menu

**Status:** Completed
**Date:** 2026-03-05

## Context

When running `/article`, the assistant would list existing folders, then ask an open-ended question — "which folder do you want?" — and wait for a response. This interrupted the article creation flow with an unnecessary round-trip. The user had to read the list, type a folder name or confirm the suggestion, and then wait again for the article to be generated. For a command that should feel fast and fluid, this pause was friction.

## Approach

Replace the multi-step folder selection with a single combined numbered menu that shows all options at once: existing relevant folders, suggested new folder names, and a free text option. The user picks a number and the flow continues immediately. This eliminates the need for a second interaction when creating a new folder (previously: pick "create new" → then pick a name), while still giving full control.

The key design decision was merging existing and new folder options into one menu rather than having a two-step flow (select existing or new → then pick name). This keeps it to exactly one user interaction regardless of the choice.

## Changes

### 1. Rewritten folder selection instructions

The "最初のステップ" section in `article.md` was rewritten from 7 sequential steps (list → filter → suggest → ask → create → ask tags → create file) to 5 steps with a single menu interaction. The new instructions tell the assistant to:

- Scan existing subdirectories
- Build a combined numbered menu with `(既存)` and `[新規作成]` labels
- Include a "その他" free text option as the last item
- Only show new folder suggestions (no existing) when no existing folders are relevant
- Proceed immediately after user picks a number

### 2. Added menu display example

A concrete example block was added to the command definition so the assistant produces consistent formatting:

```
記事のフォルダを選択してください:
1. macos/ (既存)
2. [新規作成] developer-tools/
3. その他（フォルダ名を入力）
```

## Files Modified

| File | Change |
|------|--------|
| [.claude/commands/article.md](.claude/commands/article.md) | Replaced 7-step folder selection with single combined menu flow + display example |

## Guard Rails

| Scenario | Behavior |
|----------|----------|
| No existing folders in project | Menu shows only new folder suggestions + free text option |
| No existing folders match the topic | Same as above — irrelevant existing folders are omitted |
| User types a custom folder name instead of a number | Accepted via the "その他" option |
| Only one existing folder is relevant | Still shows menu (with that folder + new suggestions + free text) for consistency |

## Verification

1. Run `/article AWS Lambdaをやってみた` in a project with `aws-lambda/` and `macos/` folders
2. Confirm a numbered menu appears with `aws-lambda/ (既存)` as an option, plus new folder suggestions and free text
3. Pick a number and confirm the article is created without a second prompt for folder name
4. Run `/article` on a topic with no matching existing folder — confirm only new suggestions and free text appear

## Breaking Changes

None
