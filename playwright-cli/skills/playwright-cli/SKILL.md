---
name: playwright-cli
description: Give an AI agent browser access to navigate, inspect, and interact with websites. Use when asked to check a page, verify changes look right, click through a flow, take screenshots, inspect console errors, or explore what an app is doing.
allowed-tools: Bash(playwright-cli:*) Bash(npx:*)
---

## When to use this skill

Use this skill when asked to:
- "check if the page looks right after my changes"
- "open the app and see if X works"
- "take a screenshot of the current state"
- "click through this flow and tell me what happens"
- "look for any console errors on this page"
- "verify the layout hasn't broken"
- "what does the page look like now?"

## Core loop

```bash
# 1. Open a browser
playwright-cli open https://example.com

# 2. Take a snapshot to see what's on the page (always do this before interacting)
playwright-cli snapshot

# 3. Interact using element refs from the snapshot
playwright-cli click e5
playwright-cli fill e2 "some text" --submit

# 4. Snapshot again to see what changed
playwright-cli snapshot

# 5. Screenshot to capture evidence
playwright-cli screenshot --filename=result.png

# 6. Close when done
playwright-cli close
```

## Installation

```bash
brew install playwright-cli
playwright-cli --version
```

## Reading the page (snapshot)

Always take a snapshot before interacting — it maps the page into named element refs (`e1`, `e2`, …):

```bash
playwright-cli snapshot                  # full page
playwright-cli snapshot "#main"          # scope to a selector
playwright-cli snapshot e15              # scope to a ref
playwright-cli snapshot --depth=4        # shallower — faster on large pages
playwright-cli snapshot --boxes          # include bounding boxes
```

## Navigating

```bash
playwright-cli goto https://example.com/settings
playwright-cli go-back
playwright-cli go-forward
playwright-cli reload
```

## Interacting

```bash
playwright-cli click e5
playwright-cli click "#submit-button"                          # CSS selector also works
playwright-cli click "getByRole('button', { name: 'Save' })"  # Playwright locator
playwright-cli dblclick e7
playwright-cli fill e2 "user@example.com"
playwright-cli fill e2 "user@example.com" --submit            # presses Enter after filling
playwright-cli type "text to type into focused element"
playwright-cli hover e4
playwright-cli select e9 "option-value"                       # dropdown
playwright-cli check e12
playwright-cli uncheck e12
playwright-cli press Enter
playwright-cli press ArrowDown
playwright-cli dialog-accept
playwright-cli dialog-dismiss
```

## Inspecting the page

```bash
# Read element attributes not visible in the snapshot
playwright-cli eval "document.title"
playwright-cli eval "el => el.textContent" e5
playwright-cli eval "el => el.getAttribute('data-testid')" e5
playwright-cli eval "el => el.id" e5
playwright-cli eval "el => getComputedStyle(el).display" e5

# Console messages (check for JS errors)
playwright-cli console
playwright-cli console error    # filter by level: log, info, warning, error

# Network requests (check what the page is calling)
playwright-cli requests
playwright-cli request 5        # detail for request #5
```

## Screenshots

```bash
playwright-cli screenshot                         # saves with timestamp
playwright-cli screenshot --filename=before.png
playwright-cli screenshot e5                      # just one element
playwright-cli pdf --filename=page.pdf
```

## Advanced: run arbitrary code

When the CLI commands aren't enough, use `run-code` to execute full Playwright code:

```bash
# Wait for something to appear
playwright-cli run-code "async page => {
  await page.locator('.loading').waitFor({ state: 'hidden' });
}"

# Wait for app state
playwright-cli run-code "async page => {
  await page.waitForFunction(() => window.appReady === true);
}"

# Work with iframes
playwright-cli run-code "async page => {
  const frame = page.locator('iframe#embed').contentFrame();
  return await frame.locator('h1').textContent();
}"

# Get full page content
playwright-cli run-code "async page => {
  return await page.content();
}"
```

See [references/running-code.md](references/running-code.md) for the full reference.

## Reference docs

- **Running custom code** — [references/running-code.md](references/running-code.md)
- **Auth state, cookies, localStorage** — [references/storage-state.md](references/storage-state.md)
- **Session management / attaching to browsers / named sessions** — [references/session-management.md](references/session-management.md)
- **Request mocking** — [references/request-mocking.md](references/request-mocking.md)
- **Element attributes** — [references/element-attributes.md](references/element-attributes.md)
- **Tracing** — [references/tracing.md](references/tracing.md)
- **Video recording** — [references/video-recording.md](references/video-recording.md)

Session cleanup: `playwright-cli close` / `playwright-cli close-all` / `playwright-cli kill-all`. Raw output: prefix any command with `--raw`. Resize: `playwright-cli resize 1920 1080`. Browser choice: `playwright-cli open --browser=firefox|webkit|chrome`.
