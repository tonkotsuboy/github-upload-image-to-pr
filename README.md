# GitHub Upload Image to PR

An AI agent skill that uploads local images to a GitHub PR and embeds them in the description or comments — automatically, just by asking.

[日本語版はこちら](./README.ja.md)

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
