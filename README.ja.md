# GitHub Upload Image to PR

> [!IMPORTANT]
> **このスキルは GitHub CLI ネイティブの `--attach`（`gh` 2.99.0 以降）を優先して使います。** `gh` が新しければ `gh pr edit --attach ./screenshot.png`（または `gh pr comment --attach`）の1コマンドで完了し、ブラウザ自動化も MCP サーバーも不要です。`gh` が古い場合はアップグレードを促したうえで、従来のブラウザ経路にフォールバックして作業を完遂します。詳細は [v2.99.0 のリリースノート](https://github.com/cli/cli/releases/tag/v2.99.0) と [GitHub のドキュメント](https://docs.github.com/en/github-cli/github-cli/attaching-files-with-github-cli) を参照してください。

ローカルの画像を GitHub PR に自動でアップロードし、PR の説明欄やコメントに埋め込む AI エージェント向けスキルです。

[English version](./README.md)

## インストール

```bash
gh skill install tonkotsuboy/github-upload-image-to-pr github-upload-image-to-pr
```

または

```bash
npx skills add tonkotsuboy/github-upload-image-to-pr
```

## 使い方

以下のようなフレーズでスキルが起動します：

- 「このスクリーンショットをPRに添付して」
- 「PR説明欄に画像を追加して」
- 「テスト結果をPRにアップロードして」
- 「このスクショをPRに貼って」
- 「ビフォーアフターの画像をPRに埋め込んで」

## 仕組み

`gh` のバージョンによって、2つの経路のどちらかを選びます。

### 経路A — `gh --attach`（推奨・`gh` 2.99.0 以降）

GitHub CLI 2.99.0 で `gh pr create` / `pr edit` / `pr comment`（および対応する `gh issue` 系コマンド）に、繰り返し指定できる `--attach` フラグが追加されました。アップロードは1コマンドで済みます。

```bash
gh pr edit 23 --attach './screenshot.png#ログインエラーの状態'
gh pr comment 23 --attach ./before.png --attach ./after.png
```

`--body` を付けなければ既存の説明文はそのまま保たれ、添付が末尾に追記されます。本文に `![before](./before.png)` のようにローカルパスの参照があれば、その参照が**その場でアップロード後の URL に書き換えられる**ため、Before/After のテーブルのようなレイアウトも1コマンドで作れます。

利用にはリポジトリへの push 権限が必要で、対応しているのは GitHub.com と GitHub Enterprise Cloud です（Enterprise Server は非対応）。

### 経路B — ブラウザアップロード（フォールバック）

GitHub の公式 REST API には、埋め込み用の画像をアップロードするエンドポイントが **存在しません**。そのため `--attach` が登場する前は、人間が Web UI で行うのと同じ手順をブラウザ自動化で再現するしかありませんでした。

1. Chrome DevTools MCP または Playwright MCP でブラウザを操作し、**PR ページを開く**
2. PR の会話欄の下部にある**コメントテキストエリアを見つける**
3. テキストエリアに紐づいたファイル入力に画像ファイルをアップロードする。これにより GitHub 内部のアップロード処理が走り、`https://github.com/user-attachments/assets/...` という**永続的な URL が生成される**
4. コメントを送信する前にテキストエリアの値から **URL を取り出す**
5. テキストエリアを**クリアする**（コメントを投稿しなくても画像 URL は有効なまま残る）
6. `gh pr edit` で **PR の説明文を更新**し、画像をマークダウンとして埋め込む

このアプローチが成立するのは、GitHub の画像ホスティングがコメントの投稿とは独立しているためです。ファイルをアップロードした時点で画像は永続化され、コメントを送信しなくても URL はずっと有効です。スキルがこの経路を使うのは、`gh` が 2.99.0 より古い場合、ホストが Enterprise Server の場合、push 権限がない場合です。

## 必要なもの

- スキルに対応した AI エージェント（例：[Claude Code](https://claude.ai/claude-code)）
- [GitHub CLI (`gh`)](https://cli.github.com/) — **経路A には 2.99.0 以降が必要**。どちらの経路でも必須
- 経路B（`gh` が古い場合）のみ、以下のいずれかのブラウザ自動化ツール：
  - **Chrome DevTools MCP**（推奨 — 既存のブラウザに接続し、ログイン状態を維持）
  - **Playwright MCP**（既存のブラウザインスタンスに接続）

## ライセンス

MIT

Copyright 2026 tonkotsuboy
