# GitHub Upload Image to PR

> [!IMPORTANT]
> **This skill now prefers GitHub CLI's native `--attach` flag (`gh` 2.99.0+).** When your `gh` is new enough, it runs `gh pr edit --attach ./screenshot.png` (or `gh pr comment --attach`) and is done — no browser automation, no MCP server, no flakiness. On older `gh` it prompts you to upgrade and falls back to the browser route so the work still gets done. See the [v2.99.0 release notes](https://github.com/cli/cli/releases/tag/v2.99.0) and [GitHub's docs](https://docs.github.com/en/github-cli/github-cli/attaching-files-with-github-cli).

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

The skill picks one of two paths based on your `gh` version.

### Path A — `gh --attach` (preferred, `gh` 2.99.0+)

GitHub CLI 2.99.0 added a repeatable `--attach` flag to `gh pr create` / `pr edit` / `pr comment` (and the matching `gh issue` commands), so uploading is a single command:

```bash
gh pr edit 23 --attach './screenshot.png#Login error state'
gh pr comment 23 --attach ./before.png --attach ./after.png
```

Without `--body`, the existing description is kept and the attachment appended. If the body references a local path such as `![before](./before.png)`, that reference is rewritten in place — which is how you get Before/After tables and other layouts from one command.

Requires push access to the repository, and works on GitHub.com and GitHub Enterprise Cloud (not Enterprise Server).

### Path B — browser upload (fallback)

GitHub does **not** provide a public REST API endpoint for uploading image attachments, so before `--attach` existed the only route was to drive a browser the way a human would:

1. **Open the PR page** in a browser via Chrome DevTools MCP or Playwright MCP
2. **Locate the comment textarea** at the bottom of the PR conversation
3. **Upload the image file** using the file input attached to the textarea — this triggers GitHub's internal upload pipeline and generates a persistent `https://github.com/user-attachments/assets/...` URL
4. **Extract the URL** from the textarea value before submitting anything
5. **Clear the textarea** (the image URL remains valid even without posting the comment)
6. **Update the PR description** via `gh pr edit`, embedding the image as markdown

This works because GitHub's image hosting is separate from comment submission — images persist the moment they're uploaded, whether or not the comment is ever posted. The skill uses this path when `gh` is older than 2.99.0, when the host is Enterprise Server, or when you lack push access.

## Requirements

- An AI agent that supports skills (e.g., [Claude Code](https://claude.ai/claude-code))
- [GitHub CLI (`gh`)](https://cli.github.com/) — **2.99.0 or newer for Path A**; required in both paths
- Only for Path B (older `gh`), one of the following browser automation tools:
  - **Chrome DevTools MCP** (recommended — connects to your existing browser, login state preserved)
  - **Playwright MCP** (connects to an existing browser instance)

## License

MIT

Copyright 2026 tonkotsuboy
