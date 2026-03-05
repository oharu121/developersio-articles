---
title: "macOSのFinderで右クリックから「VS Codeで開く」を追加する方法"
slug: "open-with-vscode-from-finder-right-click-menu"
tags:
  - macOS
  - VSCode
  - Automator
---

## はじめに

Windowsでは、VS Codeをインストールすると右クリックメニューに「Open with Code」が自動で追加されますが、macOSにはこの機能がありません。

macOSでも同じように、Finderの右クリックメニューからファイルやフォルダをVS Codeで開けるようにしたかったので、標準搭載のAutomatorアプリを使って設定してみました。一度設定すれば、以降はずっと使えます。

## 前提・環境

- macOS（本記事ではmacOS Sequoiaで確認）
- Visual Studio Codeがインストール済み（Applicationsフォルダに配置されていること）

## 手順

### 1. Automatorを起動する

`Cmd + Space`でSpotlightを開き、「Automator」と入力してEnterを押します。

### 2. クイックアクションを新規作成する

Automatorが起動したら、「New Document（新規書類）」をクリックし、「Quick Action（クイックアクション）」を選択します。

### 3. ワークフローの設定を行う

ワークフロー画面の上部にある設定を以下のように変更します。

- **「Workflow receives current（ワークフローが受け取る現在の項目）」** を `files or folders（ファイルまたはフォルダ）` に設定
- **「in（検索対象）」** を `Finder` に設定

### 4. アクションを追加する

1. 左上の検索バーに「Open Finder Items」と入力します
2. 表示された「Open Finder Items（Finder項目を開く）」アクションを、右側のグレーのエリアにドラッグ&ドロップします
3. アクション内の **「Open with:（このアプリケーションで開く）」** ドロップダウンを `Visual Studio Code` に変更します

もしリストにVS Codeが表示されない場合は、「Other...（その他）」をクリックしてApplicationsフォルダからVisual Studio Codeを選択してください。

### 5. 保存する

`Cmd + S`を押して、名前を「Open in VS Code」として保存します。

## 動作確認

設定が完了したら、以下の手順で動作を確認します。

1. Finderで任意のファイルまたはフォルダを右クリックする
2. メニューの「Quick Actions（クイックアクション）」にカーソルを合わせる
3. 「Open in VS Code」を選択する
4. VS Codeで対象のファイル/フォルダが開くことを確認する

## まとめ

macOSの標準アプリであるAutomatorを使うことで、Finderの右クリックメニューにVS Codeで開くオプションを追加できました。サードパーティのツールを入れる必要がなく、一度設定すれば永続的に使えるので便利です。

Windowsから移行してきた方や、ターミナルで`code .`を打つのが面倒な方にはおすすめの設定です。
