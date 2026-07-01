---
name: github-upload-image-to-pr
description: >-
  Upload local images to a GitHub PR and embed them in the description or comments.
  Use when asked to "attach screenshots to PR", "add images to PR", "upload test results to PR",
  "embed screenshots in PR description", "add before/after images to PR", "attach UI screenshots",
  "show test results in PR", "add visual evidence to PR", or any request involving images and PRs.
  Always use this skill when the user wants to visually document changes in a pull request,
  even if they don't use the word "upload" — phrases like "put the screenshot in the PR" or
  "show the image in the PR" should trigger this skill.
  Prefers Chrome DevTools MCP (most stable) as the browser automation backend, falling back to
  Playwright MCP or agent-browser only when Chrome DevTools MCP is unavailable.
allowed-tools: Bash(agent-browser:*), Bash(gh:*), Bash(npx:*), Bash(cp:*), Bash(rm:*), Bash(sleep:*), mcp__chrome-devtools, mcp__playwright, ToolSearch, Read, Glob, Write
license: MIT
---

# Upload Image to PR

Upload local images to a GitHub PR and embed them in the description or comments using browser automation tools.

## How It Works

Since the GitHub API does not support direct image uploads, this skill uses the **PR comment textarea as a staging area for GitHub's image hosting** — uploading files there to obtain persistent `user-attachments/assets/` URLs, then updating the PR description or posting a comment via the `gh` CLI.

## Step 0: Resolve PR context

If the user didn't specify a PR number or URL, auto-detect it:

```bash
# Get PR number from the current branch
gh pr view --json number,url -q '"\(.number) \(.url)"'
```

If multiple repos or branches are involved, confirm with the user which PR to target.

Also, normalize the image paths to absolute paths and **stage a clean copy inside the current working directory** (the repo root) — not `/tmp/`:

```bash
# Stage inside the repo. The staged filename becomes the image's alt text on GitHub,
# so give it a meaningful name. Delete it after the upload completes.
cp /path/to/'CleanShot 2026-... .png' ./.upload-staging.png
```

Why stage inside the repo and not `/tmp/`? MCP browser tools (Playwright / Chrome DevTools) can only read files **within their configured workspace root**, which is normally the project directory the session launched from. A path under `/tmp/` is outside that root, so the upload call fails with `Access denied: path ... is not within any of the configured workspace roots`. Staging the file in the repo works for every backend (agent-browser, a plain CLI, can read `/tmp/` too — but in-repo is universally safe).

Staging also sidesteps paths with special characters (e.g. the Unicode narrow spaces CleanShot X puts in filenames), which otherwise break shell globbing and tool arguments. Remember to `rm` the staged file once you're done so it isn't accidentally committed.

## Tool Detection and Selection

### Priority Order

1. **Chrome DevTools MCP** (MCP connection, `mcp__chrome-devtools__*`) — **preferred**: connects to existing browser, login state preserved, most stable of the three backends
2. **Playwright MCP** (MCP connection, `mcp__playwright__*`) — use only if Chrome DevTools MCP is unavailable; connects to existing browser, login state preserved
3. **agent-browser** (CLI via Bash — last-resort fallback, login state preserved with `--profile`)

MCP-based tools connect to an already-running browser instance, so **GitHub login state is automatically preserved**. agent-browser can persist login state using `--profile ~/.agent-browser-github`.

### Detection

```
# 1. Search specifically for Chrome DevTools MCP first (preferred, most stable)
ToolSearch: "select:mcp__chrome-devtools__navigate_page,mcp__chrome-devtools__take_snapshot,mcp__chrome-devtools__upload_file,mcp__chrome-devtools__evaluate_script,mcp__chrome-devtools__click"

# 2. Only if Chrome DevTools MCP tools aren't found, search for Playwright MCP
ToolSearch: "browser navigate upload"

# 3. Fall back to agent-browser only if no MCP tools found at all
Bash: agent-browser --version
```

## Tool Compatibility Matrix

| Operation | Chrome DevTools MCP (preferred) | Playwright MCP (fallback) | agent-browser (CLI/Bash) |
|-----------|----------------------------------|----------------------------|--------------------------|
| **Navigate** | `navigate_page` | `browser_navigate` | `agent-browser --headed open {url}` |
| **Snapshot** | `take_snapshot` | `browser_snapshot` | `agent-browser snapshot` |
| **Screenshot** | `take_screenshot` | `browser_take_screenshot` | `agent-browser screenshot {path}` |
| **Click** | `click` (uid) | `browser_click` (ref) | `agent-browser click {ref}` |
| **File Upload** | `upload_file` (uid, filePath) | `browser_file_upload` (paths) | `agent-browser upload {ref} {path}` |
| **JS Eval** | `evaluate_script` (function) | `browser_evaluate` (function) | `agent-browser eval '{js}'` |
| **Login State** | Preserved | Preserved | Preserved with `--profile` |

## Steps

### Step 1: Navigate to PR page and check login state

Navigate to the PR page and immediately take a snapshot to verify login state.

```javascript
// Chrome DevTools MCP (preferred)
navigate_page({ url: "https://github.com/{owner}/{repo}/pull/{number}", type: "url" })

// Playwright MCP (fallback if Chrome DevTools MCP is unavailable)
browser_navigate({ url: "https://github.com/{owner}/{repo}/pull/{number}" })

// agent-browser (last-resort fallback; use --profile to persist login state)
agent-browser --headed --profile ~/.agent-browser-github open "https://github.com/{owner}/{repo}/pull/{number}"
```

**If SSO authentication screen appears:** Take a snapshot, locate the "Continue" button, and click it.

**If NOT logged in (agent-browser only):**
1. Navigate to `https://github.com/login`
2. Ask the user to log in manually in the headed browser window.
3. Wait for user confirmation, then navigate back to the PR page.

### Step 2: Locate the upload target

Scroll to the comment form at the bottom of the PR and take a snapshot.

**Key gotcha:** GitHub's real `<input type="file">` (id `fc-new_comment_field`) is `display:none`, so it does **not** appear in the accessibility snapshot — you can't get a uid/ref for it that way, and clicking it isn't possible. Instead, target the **visible affordance that opens the file picker**: the dropzone labeled *"Paste, drop, or click to add files"*, or the toolbar *"Attach files"* button. The browser tool clicks it, intercepts the native file chooser, and hands over your file.

In a snapshot these show up as:

```
button "Attach files"
button "Paste, drop, or click to add files"   ← pass this uid/ref to the upload tool
```

The comment box itself is a `<file-attachment>` web component wrapping a `<textarea id="new_comment_field">` — that textarea is where the resulting image reference lands (Step 4). GitHub's UI shifts over time, so if `new_comment_field` isn't there, fall back to `textarea[name="comment[body]"]` or `textarea[id*="comment"]`; the PR-description editor instead uses `textarea[name="pull_request[body]"]` / `textarea[id$="-body"]`.

### Step 3: Upload the image(s)

Upload each file with the detected tool, passing the **dropzone / attach-button uid** from Step 2 (never the hidden input):

```javascript
// Chrome DevTools MCP (preferred): upload_file({ uid: <dropzone uid>, filePath: <path inside the workspace root> })
// Playwright MCP (fallback):       browser_file_upload({ paths: [<path inside the workspace root>] }) once the file chooser is open
// agent-browser (last resort):     agent-browser upload {ref} {absolute_path}    (any path, /tmp/ ok)
```

Use the in-repo staged path from Step 0 for the MCP backends (see the workspace-root note there). Wait **2–3 seconds between uploads**. For multiple images, upload them all into the same comment box before extracting URLs — more efficient than navigating between uploads.

### Step 4: Retrieve the uploaded image URL(s)

While uploading, GitHub shows a placeholder (`![Uploading file.png…]()`) in the textarea, then swaps it for the final reference once the file lands. So **poll the textarea until a `user-attachments/assets/` URL appears** instead of trusting a fixed sleep — it usually takes 1–5 seconds.

**GitHub inserts the reference as an HTML `<img>` tag** (with auto-detected `width`/`height`), e.g.:

```
<img width="342" height="354" alt="filename" src="https://github.com/user-attachments/assets/8fc1b84a-..." />
```

Don't assume the `![alt](url)` markdown form — match the asset URL itself so extraction stays robust to either form:

```javascript
// MCP-based tools — returns every asset URL plus the raw value for sanity-checking
() => {
  const ta = document.getElementById('new_comment_field')
          || document.querySelector('textarea[name="comment[body]"], textarea[id*="comment"]');
  if (!ta) return { error: 'textarea not found' };
  const urls = [...ta.value.matchAll(/https:\/\/github\.com\/user-attachments\/assets\/[0-9a-fA-F-]+/g)].map(m => m[0]);
  return { raw: ta.value, urls };
}
```

```bash
# agent-browser
agent-browser eval 'const ta=document.getElementById("new_comment_field")||document.querySelector("textarea[id*=comment]");JSON.stringify([...(ta?.value||"").matchAll(/https:\/\/github\.com\/user-attachments\/assets\/[0-9a-fA-F-]+/g)].map(m=>m[0]))'
```

Keep the whole `<img …>` tag if you want to preserve GitHub's auto-detected `width`/`height`; otherwise take just the URL and wrap it yourself (`![alt](url)` or your own `<img>`).

### Step 5: Clear the textarea (do not submit the comment)

Dispatch an `input` event after clearing so GitHub's comment-draft autosave also clears — otherwise the staged content can reappear on the next page load.

```javascript
// MCP-based tools
() => {
  const ta = document.getElementById('new_comment_field')
           || document.querySelector('textarea[name="comment[body]"], textarea[id*="comment"]');
  if (!ta) return "textarea not found";
  ta.value = "";
  ta.dispatchEvent(new Event('input', { bubbles: true }));
  return "cleared";
}
```

```bash
# agent-browser
agent-browser eval 'const ta=document.getElementById("new_comment_field")||document.querySelector("textarea[id*=comment]"); if(ta){ta.value="";ta.dispatchEvent(new Event("input",{bubbles:true}))} "cleared"'
```

### Step 6: Embed images in the PR

Embed using either the full `<img …>` tag GitHub gave you (keeps the auto-detected size) or plain markdown `![alt](url)`.

**Option A — Update PR description** (append images to existing body):
```bash
EXISTING_BODY=$(gh pr view {PR_NUMBER} --json body -q .body)

gh pr edit {PR_NUMBER} --body "$(printf '%s\n\n## Screenshots\n\n%s' "$EXISTING_BODY" '<img width="800" alt="screenshot" src="https://github.com/user-attachments/assets/..." />')"
```

**Option B — Post as a new comment**:
```bash
gh pr comment {PR_NUMBER} --body '## Screenshots

<img width="800" alt="screenshot" src="https://github.com/user-attachments/assets/..." />'
```

Use Option A by default unless the user explicitly asks for a comment, or if the PR description is already long and a comment would be cleaner.

### Step 7: Verify the result

Reload the page and take a screenshot to confirm the images are displayed correctly.

## Tips

- **Image sizing**: Control display size via HTML `<img>` tags: `<img width="800" alt="description" src="..." />`
- **Multiple images**: Upload all images in one session to the same textarea; extract all URLs before clearing
- **Prefer Chrome DevTools MCP**: It's the most stable backend — always try it first via `ToolSearch`. Fall back to Playwright MCP only if Chrome DevTools MCP tools aren't found, and to agent-browser only if no MCP tools are available at all
- **agent-browser login persistence**: Use `--profile ~/.agent-browser-github` to persist GitHub login across sessions

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Not logged in (MCP tools) | SSO screen may appear — take snapshot, find "Continue" button, click it |
| Not logged in (agent-browser) | Use `--headed` mode, navigate to login page, ask user to log in manually |
| Browser window not visible | For agent-browser, ensure `--headed` flag is used |
| File path with special characters (e.g., Unicode narrow spaces from CleanShot) | Stage a copy with a simple name inside the repo: `cp /path/'CleanShot ... .png' ./.upload-staging.png` (in-repo so MCP tools can read it — see Step 0) |
| `Access denied: path ... not within any of the configured workspace roots` | MCP browser tools only read files inside their workspace root. Stage the image **inside the repo** (Step 0), not `/tmp/` |
| Can't find a uid/ref for the file input | The `<input type="file">` is `display:none` and absent from the snapshot — target the visible *"Paste, drop, or click to add files"* dropzone or *"Attach files"* button instead (Step 2) |
| Textarea has the file but no `![](url)` markdown | GitHub inserts an `<img …>` HTML tag for images, not Markdown. Extract the `user-attachments/assets/` URL with the regex in Step 4, which matches both forms |
| Textarea doesn't contain URLs yet | GitHub shows an `![Uploading…]()` placeholder first — poll the textarea until a `user-attachments/assets/` URL appears (1–5s) instead of a fixed wait |
| Textarea selector not found | GitHub UI changes occasionally — fall back through `new_comment_field` → `textarea[name="comment[body]"]` → `textarea[id*="comment"]` (Step 2) |
| Chrome DevTools MCP disconnected | Reconnect via `/mcp` command |
| agent-browser not found | `npm install -g agent-browser && agent-browser install` |
| No browser tools found | Use `ToolSearch` to search for available browser tools |
| PR not found / 404 | Private repos return 404 for unauthenticated users — check login state |

## Notes

- GitHub `user-attachments/assets/` URLs are **persistent** — images remain accessible even without submitting the comment. When rendered, GitHub rewrites them to `private-user-images.githubusercontent.com` CDN URLs; the `user-attachments/assets/...` form is the stable one to embed
- On upload, GitHub injects an `<img width=… height=… alt=… src=… />` tag (alt is derived from the filename) — not `![](…)` markdown. Extract by the asset URL, not the wrapper
- MCP browser tools can only read upload files inside their workspace root, so stage images in the repo, not `/tmp/`
- Editing the description directly in the browser UI is fragile due to GitHub UI structure changes — updating via `gh pr edit` is strongly preferred
- Multiple images can be uploaded in a single session before extracting URLs
- MCP-based tools connect to existing browser instances, preserving cookies and login sessions
