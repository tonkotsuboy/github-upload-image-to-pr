# GitHub Upload Image to PR

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

Copyright 2026 tonkotsuboy
