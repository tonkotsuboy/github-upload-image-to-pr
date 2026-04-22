# GitHub Upload Image to PR

An AI agent skill that uploads local images to a GitHub PR and embeds them in the description or comments — automatically, just by asking.

## Installation

```bash
gh skill install tonkotsuboy/github-upload-image-to-pr github-upload-image-to-pr
```

or

```bash
npx skills add tonkotsuboy/github-upload-image-to-pr
```

## Usage

Trigger the skill with phrases like:

- "Attach this screenshot to the PR"
- "Add images to the PR description"
- "Upload test results to the PR"
- "Put this screenshot in the PR"
- "Embed before/after images in the PR"

## How It Works

### Why not just use the GitHub API?

GitHub does **not** provide a public REST API endpoint for uploading image attachments to embed in PR descriptions or comments. The official GitHub API allows creating/editing PR content as markdown text, but has no endpoint for uploading binary files like images.

As a workaround, this skill uses **browser automation** to upload images the same way a human would through the GitHub web UI.

### The mechanism

1. **Open the PR page** in a browser via Chrome DevTools MCP or Playwright MCP
2. **Locate the comment textarea** at the bottom of the PR conversation
3. **Upload the image file** using the file input attached to the textarea — this triggers GitHub's internal upload pipeline and generates a persistent `https://github.com/user-attachments/assets/...` URL
4. **Extract the URL** from the textarea value before submitting anything
5. **Clear the textarea** (the image URL remains valid even without posting the comment)
6. **Update the PR description** via `gh pr edit`, embedding the image as markdown

This approach works because GitHub's image hosting is separate from comment submission — images are persisted the moment they're uploaded, regardless of whether the comment is ever posted.

## Requirements

- An AI agent that supports skills (e.g., [Claude Code](https://claude.ai/claude-code))
- One of the following browser automation tools:
  - **Chrome DevTools MCP** (recommended — connects to your existing browser, login state preserved)
  - **Playwright MCP** (connects to an existing browser instance)
- [GitHub CLI (`gh`)](https://cli.github.com/) — for updating the PR description via API

## License

MIT

Copyright 2026 tonkotsuboy

---

# GitHub Upload Image to PR（日本語）

ローカルの画像を GitHub PR に自動でアップロードし、PR の説明欄やコメントに埋め込む AI エージェント向けスキルです。

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

### なぜ GitHub API を使わないのか

GitHub の公式 REST API には、PR の説明文やコメントに埋め込む画像をアップロードするエンドポイントが **存在しません**。GitHub API でできるのは、マークダウンテキストとして PR の内容を作成・編集することだけで、画像などのバイナリファイルをアップロードする手段は公式には提供されていません。

そのため、このスキルでは**ブラウザ自動化**を使って、人間が GitHub の Web UI で行うのと同じ手順で画像をアップロードします。

### 処理の流れ

1. Chrome DevTools MCP または Playwright MCP でブラウザを操作し、**PR ページを開く**
2. PR の会話欄の下部にある**コメントテキストエリアを見つける**
3. テキストエリアに紐づいたファイル入力に画像ファイルをアップロードする。これにより GitHub 内部のアップロード処理が走り、`https://github.com/user-attachments/assets/...` という**永続的な URL が生成される**
4. コメントを送信する前にテキストエリアの値から **URL を取り出す**
5. テキストエリアを**クリアする**（コメントを投稿しなくても画像 URL は有効なまま残る）
6. `gh pr edit` で **PR の説明文を更新**し、画像をマークダウンとして埋め込む

このアプローチが成立するのは、GitHub の画像ホスティングがコメントの投稿とは独立しているためです。ファイルをアップロードした時点で画像は永続化され、コメントを送信しなくても URL はずっと有効です。

## 必要なもの

- スキルに対応した AI エージェント（例：[Claude Code](https://claude.ai/claude-code)）
- 以下のいずれかのブラウザ自動化ツール：
  - **Chrome DevTools MCP**（推奨 — 既存のブラウザに接続し、ログイン状態を維持）
  - **Playwright MCP**（既存のブラウザインスタンスに接続）
- [GitHub CLI (`gh`)](https://cli.github.com/) — API 経由で PR 説明文を更新するために使用

## ライセンス

MIT
