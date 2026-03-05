# Plan: Improve /article UX with AskUserQuestion

**Status:** Completed
**Date:** 2026-03-05

## Context

When running the `/article` command, Claude would display folder and tag choices as a plain text numbered menu in the chat output. Users had to type a number or folder name to respond, which felt clunky compared to the native interactive UI that Claude Code provides through its `AskUserQuestion` tool — clickable buttons that users can simply tap to select.

## Approach

Replace the text-based numbered menu instructions in `article.md` with instructions to use the `AskUserQuestion` tool. This tool renders native clickable options in the Claude Code UI, supports both single-select (folder) and multi-select (tags), and automatically provides an "Other" free-input option. By combining both folder and tag selection into a single `AskUserQuestion` call with two questions, the flow becomes a one-step interactive prompt instead of two separate text exchanges.

## Changes

### 1. Folder selection — text menu to AskUserQuestion

The "最初のステップ" section previously described a numbered text menu format with a display example. This was replaced with instructions to use `AskUserQuestion` with single-select mode. Existing folders get `(既存)` in their label, new folder suggestions get `[新規作成]`. The static menu example block was removed since `AskUserQuestion` handles rendering.

### 2. Tag selection — merged into the same tool call

Tag selection was previously a separate step ("トピックに関連するタグの候補も合わせて提案する") with no interactive UI. It is now a second question in the same `AskUserQuestion` call using `multiSelect: true`, giving users clickable tag chips with 3-4 suggested options.

### 3. Simplified flow

The 5-step flow (list dirs → show menu → parse response → propose tags → create file) was consolidated to 4 steps (list dirs → AskUserQuestion with both questions → create folder if needed → create file).

## Files Modified

| File | Change |
|------|--------|
| [.claude/commands/article.md](.claude/commands/article.md) | Replaced text menu instructions with AskUserQuestion tool usage for folder and tag selection |

## Guard Rails

| Scenario | Behavior |
|----------|----------|
| No existing folders match the topic | Only new folder suggestions are shown as options |
| User selects "Other" for folder | AskUserQuestion's built-in free-input lets them type a custom folder name |
| User selects "Other" for tags | Free-input allows comma-separated custom tags |
| New folder selected | Folder is created before writing the article file |

## Verification

1. Run `/article some topic about AWS Lambda` in Claude Code
2. Confirm an interactive clickable menu appears (not a text-based numbered list)
3. Confirm both folder selection and tag selection appear in the same prompt
4. Select a folder and tags by clicking, verify the article file is created correctly

## Breaking Changes

None
