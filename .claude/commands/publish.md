指定された記事ファイルをContentfulにドラフトとして公開、または既存のドラフトを更新します。

ファイルパス: $ARGUMENTS

# 前提チェック（記事の処理前に必ず実行）

## Step 1: CMAトークンの確認

`.env` ファイルに `CONTENTFUL_CMA_TOKEN` が設定されているか確認する。

```bash
grep CONTENTFUL_CMA_TOKEN .env | cut -d'=' -f2
```

- `.env` が存在しない、または `CONTENTFUL_CMA_TOKEN` が空・未設定の場合:
  - ユーザーに以下を案内して処理を中断する:
    1. https://app.contentful.com/account/profile/cma_tokens にアクセス
    2. CMAトークンを生成
    3. プロジェクトルートに `.env` ファイルを作成し `CONTENTFUL_CMA_TOKEN=your-token-here` を記入
- トークンが存在する場合は次のステップへ

## Step 2: Contentful設定の確認・自動生成

`.claude/contentful-config.json` が存在するか確認する。

### 存在する場合
そのまま読み込んで次へ進む。

### 存在しない場合
CMAトークンを使って自動的に設定を検出・生成する:

1. **Space IDの取得**: `GET https://api.contentful.com/spaces` でスペース一覧を取得。複数ある場合はユーザーに選択してもらう。

2. **Content Typeの確認**: `GET https://api.contentful.com/spaces/{space_id}/environments/master/content_types` で一覧を取得し、ブログ記事用のContent Type（通常 `blogPost`）を特定する。

3. **Author Profileの選択**: `GET https://api.contentful.com/spaces/{space_id}/environments/master/entries?content_type=authorProfile` で著者一覧を取得。ユーザーに名前やIDで検索・選択してもらう。

4. **設定ファイルの保存**: 以下の形式で `.claude/contentful-config.json` に保存する:

```json
{
  "space_id": "detected-space-id",
  "environment_id": "master",
  "content_type_id": "blogPost",
  "author_id": "selected-author-id",
  "locale": "en-US",
  "fields": {
    "title": "title",
    "slug": "slug",
    "content": "content",
    "tags": "tags",
    "excerpt": "excerpt",
    "author": "author",
    "language": "language"
  }
}
```

# 記事の公開手順

1. 指定されたファイルを読み込み、YAML frontmatter（title, slug, tags, articleId, publishedAt）と本文を確認する
2. `.claude/contentful-config.json` から Contentful の設定情報を読み込む
3. frontmatter に `articleId` があるかどうかで **新規作成** か **更新** かを判定する

## 新規作成フロー（articleId なし）

1. 内容をユーザーに表示し、公開してよいか確認する
2. 記事本文から1-2文の概要文（excerpt）を生成し、AskUserQuestion でユーザーに提示する。ユーザーは承認、編集、または自分で書き直すことができる
3. `POST` で新規ドラフトエントリを作成する（詳細は「新規作成API仕様」参照）。承認された excerpt もリクエストに含める
4. 成功したら、レスポンスから取得した `entry_id` を記事ファイルの frontmatter に `articleId` として書き戻す
5. frontmatter に `publishedAt` がなければ、現在の日時（ISO 8601形式、例: `2026-03-05T12:00:00+09:00`）を `publishedAt` として追記する。既に `publishedAt` がある場合は変更しない

## 更新フロー（articleId あり）

1. `GET` で既存エントリの全データを取得する（詳細は「更新API仕様」参照）
2. ローカルで変更されたフィールド（title, slug, content, tags）と、Contentful上の現在の値を比較し、差分をユーザーに表示する
3. 既存エントリの excerpt を確認し、AskUserQuestion でユーザーに以下を選択してもらう:
   - 現在の excerpt をそのまま使う（現在の値を表示する）
   - 更新された記事内容から新しい excerpt を再生成する
   - 自分で書き直す
   再生成を選んだ場合は、生成結果を提示して承認を得る
4. 以下の警告を表示する:

   > **注意**: Contentful上で直接編集した内容（thumbnail, categories等）はそのまま保持されます。ローカルで管理しているフィールド（title, slug, content, tags, excerpt）のみが更新されます。

5. ユーザーの確認が取れたら、取得したエントリのフィールドにローカルの変更をマージして `PUT` で更新する

# 新規作成API仕様

エンドポイント: `POST https://api.contentful.com/spaces/{space_id}/environments/{environment_id}/entries`

ヘッダー:
- `Authorization: Bearer {CMA_TOKEN}`
- `Content-Type: application/vnd.contentful.management.v1+json`
- `X-Contentful-Content-Type: {content_type_id}`

リクエストボディ（JSON）:
```json
{
  "fields": {
    "title": { "{locale}": "記事タイトル" },
    "slug": { "{locale}": "article-slug" },
    "content": { "{locale}": "Markdown本文" },
    "tags": { "{locale}": ["tag1", "tag2"] },
    "excerpt": { "{locale}": "記事の概要文" },
    "language": { "{locale}": "ja" },
    "author": {
      "{locale}": {
        "sys": {
          "type": "Link",
          "linkType": "Entry",
          "id": "{author_id}"
        }
      }
    }
  }
}
```

# 更新API仕様

## Step 1: 既存エントリの取得

`GET https://api.contentful.com/spaces/{space_id}/environments/{environment_id}/entries/{articleId}`

レスポンスから以下を取得:
- `sys.version` — 更新時に `X-Contentful-Version` ヘッダーに使用
- `fields` — 全フィールドの現在の値

## Step 2: フィールドのマージ

取得した `fields` オブジェクトに対して、ローカルで管理しているフィールド **のみ** を上書きする:
- `fields.title`
- `fields.slug`
- `fields.content`
- `fields.tags`
- `fields.excerpt`

それ以外のフィールド（thumbnail, author, language, categories, targetLocales 等）は取得したデータをそのまま保持する。

## Step 3: 更新リクエスト

`PUT https://api.contentful.com/spaces/{space_id}/environments/{environment_id}/entries/{articleId}`

ヘッダー:
- `Authorization: Bearer {CMA_TOKEN}`
- `Content-Type: application/vnd.contentful.management.v1+json`
- `X-Contentful-Version: {version}` — Step 1 で取得した `sys.version` の値

リクエストボディ: Step 2 でマージした `fields` オブジェクトを含むJSON

# 成功時

## 新規作成の場合
- ContentfulのエントリURLを表示: `https://app.contentful.com/spaces/{space_id}/entries/{entry_id}`
- 記事ファイルの frontmatter に `articleId: {entry_id}` を追記する
- frontmatter に `publishedAt` がなければ、現在の日時（ISO 8601形式、例: `2026-03-05T12:00:00+09:00`）を追記する

## 更新の場合
- ContentfulのエントリURLを表示: `https://app.contentful.com/spaces/{space_id}/entries/{articleId}`
- 更新完了のメッセージを表示

# エラー時

- HTTPステータスコードとエラー内容を表示し、原因と修正方法を提案する
- 409 Conflict の場合: 他の誰かが同時に編集した可能性がある旨を伝え、再取得してリトライするか確認する
