# developersio-articles

Claude Code custom commands for writing and publishing [DevelopersIO](https://dev.classmethod.jp/) blog articles to Contentful.

## Commands

| Command | Description |
|---------|-------------|
| `/article <topic>` | Generate a Japanese blog article following DevelopersIO media guidelines |
| `/publish <file>` | Publish an article to Contentful as a draft (or update an existing one) |
| `/release` | Create a GitHub issue, tag, and release |

## Setup

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed
- A Contentful account with access to the DevelopersIO space

### 1. Clone and configure

```bash
git clone https://github.com/oharu121/developersio-articles.git
cd developersio-articles
cp .env.example .env
```

### 2. Add your Contentful CMA token

Get a token from [Contentful CMA tokens page](https://app.contentful.com/account/profile/cma_tokens), then edit `.env`:

```
CONTENTFUL_CMA_TOKEN=your-token-here
```

### 3. First publish auto-configures everything

The first time you run `/publish`, it will automatically:

1. Detect your Contentful space ID
2. Identify the blog post content type
3. Ask you to select your author profile
4. Save the config to `.claude/contentful-config.json`

## Usage

### Writing an article

```
/article AWS LambdaでPython 3.12ランタイムを試してみた
```

This will:
1. Propose a category folder (or suggest new ones)
2. Propose relevant tags for selection
3. Generate a Markdown article with YAML frontmatter

### Publishing to Contentful

```
/publish aws-lambda/try-aws-lambda-python312.md
```

- **New article** (no `articleId` in frontmatter): creates a draft and writes back the `articleId`
- **Existing article** (`articleId` present): fetches the current entry, merges local changes, and updates. Fields set in Contentful (thumbnail, excerpt, etc.) are preserved.

## Project structure

```
.
├── .claude/
│   ├── commands/
│   │   ├── article.md      # /article command
│   │   ├── publish.md      # /publish command
│   │   └── release.md      # /release command
│   └── contentful-config.json  # Auto-generated (gitignored)
├── .env                     # CMA token (gitignored)
├── .env.example
├── .plans/                  # Release plan files
└── <category>/
    └── <slug>.md            # Articles
```

## Article format

Articles use YAML frontmatter:

```yaml
---
title: "日本語のタイトル"
slug: "english-kebab-case-slug"
tags:
  - Tag1
  - Tag2
articleId: "contentful-entry-id"  # Auto-populated by /publish
---

Article content in Markdown...
```

## License

Private — for internal use at Classmethod, Inc.
