---
name: github-upload-image-to-pr
description: >-
  Upload local images or videos to a GitHub PR and embed them in the description or comments.
  Use when asked to "attach screenshots to PR", "add images to PR", "upload test results to PR",
  "embed screenshots in PR description", "add before/after images to PR", "attach UI screenshots",
  "show test results in PR", "add visual evidence to PR", or any request involving images and PRs.
  Always use this skill when the user wants to visually document changes in a pull request,
  even if they don't use the word "upload" — phrases like "put the screenshot in the PR" or
  "show the image in the PR" should trigger this skill.
  Prefers GitHub CLI's native `--attach` flag (gh 2.99.0+), which needs no browser at all, and
  falls back to browser automation (Chrome DevTools MCP, then Playwright MCP, then agent-browser)
  only when `gh --attach` is unavailable.
allowed-tools: Bash(agent-browser:*), Bash(gh:*), Bash(brew:*), Bash(npx:*), Bash(cp:*), Bash(rm:*), Bash(sleep:*), mcp__chrome-devtools, mcp__playwright, ToolSearch, Read, Glob, Write
license: MIT
---

# Upload Image to PR

There are two ways to get a local image into a PR, and they differ enormously in cost and reliability:

- **Path A — `gh --attach`** (GitHub CLI 2.99.0+, released 2026-09): one shell command. No browser, no MCP server, no snapshots, no polling. Use this whenever it's available.
- **Path B — browser upload** (fallback): drive a real browser to GitHub's comment box, which is the only public way to reach GitHub's image host on older `gh`. Many steps, many failure modes, lots of tokens.

Path B exists because GitHub's REST API still has no image-upload endpoint. Path A works because `gh` itself now talks to the internal upload endpoint for you — so treat Path B as legacy, not as an equal alternative.

## Step 1: Pick the path

```bash
GH_VER=$(gh --version 2>/dev/null | head -1 | awk '{print $3}')
echo "gh: ${GH_VER:-not installed}"
# Path A requires >= 2.99.0
[ -n "$GH_VER" ] && [ "$(printf '%s\n2.99.0\n' "$GH_VER" | sort -V | head -1)" = "2.99.0" ] \
  && echo "=> Path A (gh --attach)" || echo "=> Path B (browser fallback)"
```

Use **Path B** when any of these hold:

| Condition | Why Path A can't work |
|-----------|-----------------------|
| `gh` older than 2.99.0, or not installed | The `--attach` flag doesn't exist yet |
| Host is GitHub Enterprise **Server** | Attachments ship on GitHub.com and Enterprise **Cloud** only |
| No push access to the repo (e.g. someone else's fork) | `--attach` uploads require push access |
| `gh --attach` fails for an unclear reason | Don't sink time into it — note the error and fall back |

`gh` is needed in **both** paths (Path B still embeds via `gh pr edit`), so if it isn't installed at all, ask the user to install it first: `brew install gh` (macOS) or see <https://cli.github.com/>.

### If you land on Path B because `gh` is old, say so loudly

The user is about to pay for a slow, flaky browser session that a one-liner would have handled. That's worth interrupting for — but not worth blocking on, so surface the notice, then get on with Path B and deliver the images. Lead your reply with something like this (match the user's language):

> ⚠️ **`gh` を更新すると、この作業はコマンド1発になります**
> 現在: `gh 2.86.0` / 必要: **2.99.0 以上**
> 更新後は `gh pr comment 123 --attach ./shot.png` だけで完了します（ブラウザ操作なし・高速・安定）。
> 更新コマンド: `brew upgrade gh`
> 今回はブラウザ自動化で続行します。

If the user asks you to upgrade, run the upgrade, re-check the version, and switch to Path A for the current request too.

---

## Path A — `gh --attach` (preferred)

`--attach` takes `<file>` or `<file>#<alt text>`, is repeatable (up to 50 files per command, and the same file can't be attached twice), and is available on `gh pr create` / `pr edit` / `pr comment` and the three matching `gh issue` commands.

### Append to the PR description

```bash
gh pr edit {PR_NUMBER} --attach './screenshot.png#Login error state'
```

Leaving `--body` off keeps the existing description and appends the attachment — no need to read the body and stitch it back together.

### Attach while creating the PR

If the PR doesn't exist yet, don't create it and then edit it — attach in the same call:

```bash
gh pr create --title '...' --body '...' --attach './screenshot.png#Login error state'
```

### Control where the images land

If the body references a local path, `gh` rewrites **that reference in place** instead of appending, so you can lay images out however you like (alt text then comes from the markdown, not from the flag):

```bash
gh pr edit {PR_NUMBER} --body "$(gh pr view {PR_NUMBER} --json body -q .body)

## Screenshots

| Before | After |
| --- | --- |
| ![before](./before.png) | ![after](./after.png) |" \
  --attach ./before.png --attach ./after.png
```

Two things to keep straight here: `--body` **replaces** the description, which is why the existing body is read back in first; and the path string in the body must match the one passed to `--attach` for the rewrite to fire.

### Post as a comment instead

```bash
gh pr comment {PR_NUMBER} --attach './result.png#Test run summary'
```

Same rewrite rule applies with `--body`, so a comment can carry a layout too.

### Videos

`mp4` / `mov` attach the same way and render as a player. Video has no alt text, so don't add a `#...` suffix for one.

### Verify

No browser needed — read the body back and count the attachments:

```bash
gh pr view {PR_NUMBER} --json body -q .body | grep -c 'user-attachments/assets'
```

### Path A gotchas

- **Attaching only appends; it never replaces.** Re-running the same command duplicates the images. To swap an image out, rewrite the body without the stale URL (`gh pr edit {PR} --body "$(...)"`) in the same command that attaches the new one.
- **A `#` in the filename breaks the `<file>#<alt>` split.** Stage a copy with a plain ASCII name first (`cp './weird #name.png' ./shot.png`) and `rm` it afterwards. The same trick handles the Unicode narrow spaces CleanShot X puts in filenames.
- **Partial failure still succeeds partially.** If some files upload and others don't, the PR is updated with the ones that worked, the PR URL is still printed, and the exit status is non-zero. Read the output and report which files failed rather than assuming all-or-nothing.
- Only images and media files are accepted — not arbitrary attachments like `.zip` or `.log`.

---

## Path B — Browser upload (fallback for `gh` < 2.99.0)

Everything below is the legacy route. Reach for it only after Step 1 sent you here.

Since older `gh` can't reach GitHub's image host, this path uses the **PR comment textarea as a staging area for that host** — uploading files there to obtain persistent `user-attachments/assets/` URLs, then updating the PR description or posting a comment via the `gh` CLI.

### Step B0: Resolve PR context and stage the files

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

### Tool Detection and Selection

#### Priority Order

1. **Chrome DevTools MCP** (MCP connection, `mcp__chrome-devtools__*`) — **preferred**: connects to existing browser, login state preserved, most stable of the three backends
2. **Playwright MCP** (MCP connection, `mcp__playwright__*`) — use only if Chrome DevTools MCP is unavailable; connects to existing browser, login state preserved
3. **agent-browser** (CLI via Bash — last-resort fallback, login state preserved with `--profile`)

MCP-based tools connect to an already-running browser instance, so **GitHub login state is automatically preserved**. agent-browser can persist login state using `--profile ~/.agent-browser-github`.

#### Detection

```
# 1. Search specifically for Chrome DevTools MCP first (preferred, most stable)
ToolSearch: "select:mcp__chrome-devtools__navigate_page,mcp__chrome-devtools__take_snapshot,mcp__chrome-devtools__upload_file,mcp__chrome-devtools__evaluate_script,mcp__chrome-devtools__click"

# 2. Only if Chrome DevTools MCP tools aren't found, search for Playwright MCP
ToolSearch: "browser navigate upload"

# 3. Fall back to agent-browser only if no MCP tools found at all
Bash: agent-browser --version
```

### Tool Compatibility Matrix

| Operation | Chrome DevTools MCP (preferred) | Playwright MCP (fallback) | agent-browser (CLI/Bash) |
|-----------|----------------------------------|----------------------------|--------------------------|
| **Navigate** | `navigate_page` | `browser_navigate` | `agent-browser --headed open {url}` |
| **Snapshot** | `take_snapshot` | `browser_snapshot` | `agent-browser snapshot` |
| **Screenshot** | `take_screenshot` | `browser_take_screenshot` | `agent-browser screenshot {path}` |
| **Click** | `click` (uid) | `browser_click` (ref) | `agent-browser click {ref}` |
| **File Upload** | `upload_file` (uid, filePath) | `browser_file_upload` (paths) | `agent-browser upload {ref} {path}` |
| **JS Eval** | `evaluate_script` (function) | `browser_evaluate` (function) | `agent-browser eval '{js}'` |
| **Login State** | Preserved | Preserved | Preserved with `--profile` |

### Step B1: Navigate to PR page and check login state

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

### Step B2: Locate the upload target

Scroll to the comment form at the bottom of the PR and take a snapshot.

**Save the snapshot to a file instead of returning it inline.** A PR page snapshot is easily several hundred lines (description + every review comment), and you only need one uid from it. Chrome DevTools MCP's `take_snapshot` accepts `filePath`, so write it out and `grep` for the dropzone:

```javascript
// Chrome DevTools MCP — write the snapshot out, then grep for the uid
take_snapshot({ filePath: "./.upload-snapshot.txt" })
```

```bash
grep -n "Paste, drop, or click to add files" ./.upload-snapshot.txt
#   549:      uid=3_529 button "Paste, drop, or click to add files"
```

Delete the snapshot file when you're done. Without this, the snapshot cost can look prohibitive enough to talk you out of `upload_file` altogether and into a far worse workaround (inlining the image as base64 into an `evaluate_script` body — thousands of characters that must be transcribed byte-perfect).

**Key gotcha:** GitHub's real `<input type="file">` (id `fc-new_comment_field`) is `display:none`, so it does **not** appear in the accessibility snapshot — you can't get a uid/ref for it that way, and clicking it isn't possible. Instead, target the **visible affordance that opens the file picker**: the dropzone labeled *"Paste, drop, or click to add files"*, or the toolbar *"Attach files"* button. The browser tool clicks it, intercepts the native file chooser, and hands over your file.

In a snapshot these show up as:

```
button "Attach files"
button "Paste, drop, or click to add files"   ← pass this uid/ref to the upload tool
```

The comment box itself is a `<file-attachment>` web component wrapping a `<textarea id="new_comment_field">` — that textarea is where the resulting image reference lands (Step B4). GitHub's UI shifts over time, so if `new_comment_field` isn't there, fall back to `textarea[name="comment[body]"]` or `textarea[id*="comment"]`; the PR-description editor instead uses `textarea[name="pull_request[body]"]` / `textarea[id$="-body"]`.

### Step B3: Upload the image(s)

Upload each file with the detected tool, passing the **dropzone / attach-button uid** from Step B2 (never the hidden input):

```javascript
// Chrome DevTools MCP (preferred): upload_file({ uid: <dropzone uid>, filePath: <path inside the workspace root> })
// Playwright MCP (fallback):       browser_file_upload({ paths: [<path inside the workspace root>] }) once the file chooser is open
// agent-browser (last resort):     agent-browser upload {ref} {absolute_path}    (any path, /tmp/ ok)
```

Use the in-repo staged path from Step B0 for the MCP backends (see the workspace-root note there). Wait **2–3 seconds between uploads**. For multiple images, upload them all into the same comment box before extracting URLs — more efficient than navigating between uploads.

### Step B4: Retrieve the uploaded image URL(s)

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

### Step B5: Clear the textarea (do not submit the comment)

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

### Step B6: Embed images in the PR

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

### Step B7: Verify the result

Reload the page and take a screenshot to confirm the images are displayed correctly.

Also assert programmatically that the images actually loaded. **Match both URL forms**: when rendering, GitHub rewrites `user-attachments/assets/...` to a signed `private-user-images.githubusercontent.com` URL, so searching only for the embedded form reports zero images and makes it look like nothing was attached.

```javascript
// MCP-based tools — every embedded image plus whether the browser decoded it
() => {
  const imgs = [...document.querySelectorAll('img')]
    .filter((el) => /user-attachments|private-user-images/.test(el.src));
  return imgs.map((el) => ({ alt: el.alt, loaded: el.naturalWidth > 0 }));
}
```

`loaded: true` on every entry means the embed worked. An empty array means the `<img>` tags never made it into the body — re-check Step B6.

## Tips

- **Try Path A first, every time**: one `gh pr edit --attach` / `gh pr comment --attach` call replaces the entire browser session below. Only the conditions in Step 1 justify Path B
- **Image sizing**: `--attach` lets GitHub pick the size; on Path B (or when you hand-write the markup) control it with `<img width="800" alt="description" src="..." />`
- **Multiple images**: Path A repeats `--attach` in one command; Path B uploads all images into the same textarea, then extracts all URLs before clearing
- **Prefer Chrome DevTools MCP within Path B**: It's the most stable backend — always try it first via `ToolSearch`. Fall back to Playwright MCP only if Chrome DevTools MCP tools aren't found, and to agent-browser only if no MCP tools are available at all
- **agent-browser login persistence**: Use `--profile ~/.agent-browser-github` to persist GitHub login across sessions

## Troubleshooting

### Path A (`gh --attach`)

| Issue | Solution |
|-------|----------|
| `unknown flag: --attach` | `gh` is older than 2.99.0 — nudge the user to upgrade (Step 1) and continue on Path B |
| Images duplicated after a re-run | `--attach` only appends. Rewrite the body without the stale URLs in the same `gh pr edit --body ... --attach ...` call |
| Upload rejected / permission error | `--attach` needs push access, and works on GitHub.com and Enterprise **Cloud** only. Fall back to Path B |
| Alt text swallowed the filename, or the file isn't found | A `#` in the path is parsed as the alt-text separator — stage a copy with a plain ASCII name |
| Some files uploaded, command exited non-zero | Expected behavior for partial failure: the PR keeps the successful ones. Report which files failed and retry just those |
| Body reference wasn't rewritten, image appended at the end instead | The path in the body must match the `--attach` path string exactly |

### Path B (browser upload)

| Issue | Solution |
|-------|----------|
| Not logged in (MCP tools) | SSO screen may appear — take snapshot, find "Continue" button, click it |
| Not logged in (agent-browser) | Use `--headed` mode, navigate to login page, ask user to log in manually |
| Browser window not visible | For agent-browser, ensure `--headed` flag is used |
| File path with special characters (e.g., Unicode narrow spaces from CleanShot) | Stage a copy with a simple name inside the repo: `cp /path/'CleanShot ... .png' ./.upload-staging.png` (in-repo so MCP tools can read it — see Step B0) |
| `Access denied: path ... not within any of the configured workspace roots` | MCP browser tools only read files inside their workspace root. Stage the image **inside the repo** (Step B0), not `/tmp/` |
| Can't find a uid/ref for the file input | The `<input type="file">` is `display:none` and absent from the snapshot — target the visible *"Paste, drop, or click to add files"* dropzone or *"Attach files"* button instead (Step B2) |
| PR page snapshot is huge / feels too expensive to take | Pass `filePath` to `take_snapshot` and `grep` the file for the dropzone uid (Step B2). Don't let snapshot size push you into inlining base64 into `evaluate_script` |
| Verification finds 0 images even though they render | GitHub rewrites the asset URL to `private-user-images.githubusercontent.com` on render — match both that and `user-attachments`, then check `naturalWidth > 0` (Step B7) |
| Textarea has the file but no `![](url)` markdown | GitHub inserts an `<img …>` HTML tag for images, not Markdown. Extract the `user-attachments/assets/` URL with the regex in Step B4, which matches both forms |
| Textarea doesn't contain URLs yet | GitHub shows an `![Uploading…]()` placeholder first — poll the textarea until a `user-attachments/assets/` URL appears (1–5s) instead of a fixed wait |
| Textarea selector not found | GitHub UI changes occasionally — fall back through `new_comment_field` → `textarea[name="comment[body]"]` → `textarea[id*="comment"]` (Step B2) |
| Chrome DevTools MCP disconnected | Reconnect via `/mcp` command |
| agent-browser not found | `npm install -g agent-browser && agent-browser install` |
| No browser tools found | Use `ToolSearch` to search for available browser tools |
| PR not found / 404 | Private repos return 404 for unauthenticated users — check login state |

## Notes

- `gh --attach` landed in GitHub CLI **2.99.0** (2026-09) on `gh pr create` / `pr edit` / `pr comment` and the matching `gh issue create` / `issue edit` / `issue comment`. Up to 50 files per command; the same file can't be attached twice
- `gh pr edit --attach` without `--body` keeps the existing description and appends — the read-modify-write dance is only needed when you want to control placement
- GitHub `user-attachments/assets/` URLs are **persistent** — images remain accessible even without submitting the comment. When rendered, GitHub rewrites them to `private-user-images.githubusercontent.com` CDN URLs; the `user-attachments/assets/...` form is the stable one to embed
- On Path B upload, GitHub injects an `<img width=… height=… alt=… src=… />` tag (alt is derived from the filename) — not `![](…)` markdown. Extract by the asset URL, not the wrapper
- MCP browser tools can only read upload files inside their workspace root, so stage images in the repo, not `/tmp/`
- Editing the description directly in the browser UI is fragile due to GitHub UI structure changes — updating via `gh pr edit` is strongly preferred
- MCP-based tools connect to existing browser instances, preserving cookies and login sessions
