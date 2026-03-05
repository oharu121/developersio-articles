---
title: "Claude Codeのカスタムコマンドで技術ブログの執筆からContentfulへの投稿まで自動化してみた"
slug: "custom-commands-for-blogging-with-contentful"
tags:
  - Claude Code
  - Contentful
  - Automation
---

## はじめに

技術ブログを書くとき、記事の執筆・フォルダ整理・CMS（Contentful）への投稿と、意外と手作業が多いです。

そこで、Claude Codeのカスタムコマンド機能を使い、スラッシュコマンドだけで記事の下書き作成からContentfulへのドラフト投稿までを完結させるツールを作ってみました。

本記事ではこのツールの仕組みと構築手順を紹介します。

## 作ったもの

2つのカスタムコマンドを作成しました。

| コマンド | 機能 |
|---------|------|
| `/article <トピック>` | トピックを元に、ガイドラインに準拠した日本語の技術ブログ記事の下書きを生成 |
| `/publish <ファイルパス>` | 指定した記事ファイルをContentfulにドラフトとして投稿 |

## 前提・環境

- Claude Code（CLI）がインストール済みであること
- Contentfulアカウントがあり、CMA（Content Management API）トークンを発行できること
- ブログのContent TypeがContentful上に定義済みであること

## プロジェクト構成

```
developersio-articles/
├── .claude/
│   ├── commands/
│   │   ├── article.md      # /article コマンド定義
│   │   └── publish.md      # /publish コマンド定義
│   └── contentful-config.json  # Contentful接続設定（自動生成可）
├── .env                     # CMAトークン（gitignore対象）
├── .env.example             # テンプレート
├── .gitignore
└── <カテゴリフォルダ>/
    └── <slug>.md            # 記事ファイル
```

## /article コマンドの仕組み

### カスタムコマンドとは

Claude Codeでは、`.claude/commands/` ディレクトリにMarkdownファイルを配置すると、そのファイル名がスラッシュコマンドとして使えるようになります。`$ARGUMENTS` というプレースホルダーでユーザーの入力を受け取れます。

### /article の動作フロー

`/article AWS Lambdaでpython3.12ランタイムを試す` のようにトピックを渡すと、以下のフローが実行されます。

1. **フォルダ選択** — プロジェクト内の既存フォルダを一覧表示し、トピックに関連するフォルダを提案。なければ新しいフォルダ名の候補を提示
2. **タグ提案** — トピックに基づいて関連タグの候補を提案し、ユーザーが選択・追加・修正
3. **記事生成** — ガイドラインに準拠した日本語の技術ブログ記事をMarkdownで生成

### 記事のフォーマット

生成される記事にはYAML frontmatterが含まれます。

```yaml
---
title: "日本語のタイトル"
slug: "english-kebab-case-slug"
tags:
  - tag1
  - tag2
---
```

- `title` — 日本語の記事タイトル
- `slug` — URL用の英語キー（ファイル名にも使用）
- `tags` — 記事に関連するタグのリスト

### メディアガイドラインの組み込み

`article.md` のコマンド定義に、メディアガイドラインを直接記載しています。これにより、Claude Codeが記事を生成する際に自動的にガイドラインを考慮してくれます。

含めているルール例：

- 「やってみた」視点で実体験ベースの記事にする
- ディス（他製品・サービスの批判）を避ける
- 憶測や伝聞ではなくソースに基づいた内容にする
- 生成AIへの丸投げはせず、あくまでアシスタントとして使う

## /publish コマンドの仕組み

### 初回セットアップの自動化

`/publish` コマンドには、初回実行時の自動セットアップ機能を組み込んでいます。

**Step 1: CMAトークンの確認**

`.env` ファイルに `CONTENTFUL_CMA_TOKEN` が設定されているか確認します。未設定の場合は、トークンの取得方法をユーザーに案内して処理を中断します。

**Step 2: Contentful設定の自動生成**

`.claude/contentful-config.json` が存在しない場合、CMAトークンを使ってContentful APIから自動的に設定を検出します。

1. `GET /spaces` でスペースIDを取得
2. `GET /content_types` でブログ記事のContent Typeを特定
3. `GET /entries?content_type=authorProfile` で著者一覧を取得し、ユーザーに選択してもらう
4. 結果を `contentful-config.json` に保存

この仕組みにより、2回目以降の実行では設定ファイルを読み込むだけでスキップされます。

### 投稿フロー

```
/publish macos/open-with-vscode-from-finder-right-click-menu.md
```

のように記事ファイルのパスを渡すと、以下が実行されます。

1. 記事ファイルを読み込み、frontmatterからメタ情報を抽出
2. 投稿内容をユーザーに表示して確認を取る
3. Contentful Content Management APIにドラフトエントリを作成
4. 成功時はContentfulのエントリURLを表示

### API呼び出し

Contentful CMAへのリクエストは以下の形式です。

```
POST https://api.contentful.com/spaces/{space_id}/environments/{environment_id}/entries
```

必要なヘッダー：

```
Authorization: Bearer {CMA_TOKEN}
Content-Type: application/vnd.contentful.management.v1+json
X-Contentful-Content-Type: blogPost
```

Contentful CMAでは、エントリを作成するとデフォルトでドラフト状態になります。公開（Publish）するには別途APIを呼ぶ必要があるため、誤って公開してしまう心配はありません。

### CMAトークンに関する注意点

Contentful CMAトークンはスペースレベルのアクセスキーであり、特定のユーザーに紐付いていません。つまり、トークンがあれば誰の名前でも記事を投稿できてしまいます。Author Profileはエントリ内のリンクフィールドに過ぎず、認証とは関係ありません。

そのため：

- トークンは `.env` に保管し、`.gitignore` で除外する
- `contentful-config.json` に自分のAuthor IDを設定する
- トークンの共有・コミットは絶対に避ける

## セットアップ手順

このツールを自分の環境で使う手順です。

### 1. リポジトリをクローン

```bash
git clone <リポジトリURL>
cd developersio-articles
```

### 2. .envファイルを作成

```bash
cp .env.example .env
```

`.env` を編集して `CONTENTFUL_CMA_TOKEN` にCMAトークンを設定します。CMAトークンは [Contentfulの設定画面](https://app.contentful.com/account/profile/cma_tokens) から生成できます。

### 3. 初回の/publishで設定を自動生成

```
/publish <任意の記事ファイル>
```

初回実行時に `contentful-config.json` が自動生成されます。スペースの選択と著者プロフィールの選択を求められるので、指示に従ってください。

## 使い方の例

### 記事を書く

```
/article AWS LambdaでPython 3.12ランタイムを試してみた
```

フォルダ選択 → タグ選択 → 記事生成 の対話フローが始まります。

### 記事を投稿する

```
/publish aws-lambda/try-aws-lambda-python312.md
```

内容確認 → Contentfulにドラフト投稿 → エントリURLが表示されます。

## まとめ

Claude Codeのカスタムコマンド機能を使うことで、技術ブログの執筆ワークフローをかなりシンプルにできました。

ポイントをまとめると：

- **カスタムコマンドはMarkdownファイル1つで定義できる** — 外部スクリプトやパッケージ不要
- **ガイドラインをコマンドに直接組み込める** — 記事品質を自動的に担保
- **Contentful APIとの連携も可能** — CMAトークンさえあれば、ドラフト投稿まで自動化できる
- **初回セットアップの自動化** — 設定が存在しなければAPIから検出・生成する仕組みにより、他のメンバーも簡単に使い始められる

カスタムコマンドの定義ファイルは単なるMarkdownなので、ブログ以外の用途（ドキュメント生成、コードレビュー、デプロイ手順など）にも応用できます。
