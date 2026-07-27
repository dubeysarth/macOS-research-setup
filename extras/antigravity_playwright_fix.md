# Antigravity IDE — Playwright Driver Fix

> **Status**: ✅ CONFIRMED WORKING — browser subagent tested, Google loaded successfully.

---

## The Error

When the Antigravity IDE browser subagent tries to open any URL, it fails with:

```
failed to create browser context: failed to run playwright manager:
failed to install playwright: could not install driver:
  error: got non 200 status code: 404 (404 Not Found)
  from https://playwright.azureedge.net/builds/driver/playwright-1.57.0-mac-arm64.zip
  from https://playwright-akamai.azureedge.net/builds/driver/playwright-1.57.0-mac-arm64.zip
  from https://playwright-verizon.azureedge.net/builds/driver/playwright-1.57.0-mac-arm64.zip
```

**Root cause**: The IDE's internal playwright manager needs driver version `1.57.0`, but
that specific version's zip is no longer hosted on the Azure CDN (all three mirrors return 404).

---

## Architecture: How the IDE Uses Playwright

The IDE is an Electron app. The browser subagent runs inside a **native Go binary**:

```
/Applications/Antigravity IDE.app/Contents/Resources/app/extensions/antigravity/bin/language_server_macos_arm
```

This binary embeds the **`playwright-community/playwright-go`** Go library (not the npm playwright).
playwright-go has its own driver management that is **independent of npm's playwright CLI**.

Key strings found in the binary:
```
.cache/ms-playwright-go/1.57.0
cli.js
playwright.sh
https://playwright-akamai.azureedge.net
```

playwright-go manages its driver cache at:
```
~/Library/Caches/ms-playwright-go/<version>/
```

On first use it downloads `playwright-1.57.0-mac-arm64.zip` from the Azure CDN,
extracts it, and expects to find `playwright.sh` (launcher) and `package/cli.js` (the
playwright Node.js CLI) in the cache directory.

---

## What Does NOT Work

### ❌ Attempt 1 — npm install + PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD env var

Standard community advice:

```bash
npm install -g playwright
npx playwright install
echo 'export PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1' >> ~/.zshrc
echo 'export PLAYWRIGHT_BROWSERS_PATH="$(npm root -g)/playwright"' >> ~/.zshrc
source ~/.zshrc
open -a "Antigravity IDE"
```

**Why it fails**: `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1` is an npm playwright CLI env var.
It tells the **npm playwright package** not to download browsers. It has **no effect** on
the Go binary's internal playwright manager, which ignores this env var and still tries to
download its own `1.57.0` driver from the CDN.

### ❌ Attempt 2 — open -a "Antigravity IDE" on a running instance

Running `open -a "Antigravity IDE"` when the IDE is already running only **refocuses** the
existing process. The process does not re-read `~/.zshrc`. The env vars are never picked up.
The IDE must be **fully quit** (`Cmd + Q`) and **cold-launched** from a terminal session that
already has the env vars loaded.

Even after a proper cold restart with env vars set, the npm env vars still don't fix the Go
binary's driver fetching.

### ❌ Attempt 3 — GitHub releases download

```bash
curl -fsSL https://github.com/microsoft/playwright/releases/download/v1.57.0/playwright-1.57.0-mac-arm64.zip
```

Returns 404. The `1.57.0` driver zip is not published as a GitHub release asset.

---

## Key Findings

### 1. The cache directory exists but is empty

```
~/Library/Caches/ms-playwright-go/1.57.0/    ← directory exists, EMPTY
~/Library/Caches/ms-playwright/              ← npm playwright browsers (separate, irrelevant)
```

The IDE creates the versioned directory on first startup but cannot fill it because the CDN
is dead. Every subsequent launch re-attempts the download and fails again.

### 2. npm playwright@1.57.0 contains the correct CLI

The playwright Node.js CLI at version `1.57.0` is still available on the npm registry.

```bash
mkdir -p /tmp/pw157
cd /tmp/pw157
npm install playwright@1.57.0
# playwright-core at /tmp/pw157/node_modules/playwright-core/
# cli.js is at: /tmp/pw157/node_modules/playwright-core/cli.js  ✅
```

### 3. Expected driver directory structure

playwright-go extracts the CDN zip into its cache and expects:

```
~/Library/Caches/ms-playwright-go/1.57.0/
├── playwright.sh        ← bash launcher (entry point playwright-go calls)
├── node                 ← symlink to the real node binary
└── package/             ← playwright-core npm package contents
    ├── cli.js           ← playwright Node.js CLI
    ├── package.json
    ├── lib/
    └── ...
```

---

## The Fix

**Prerequisite** — Node must be active (fnm + lts installed as in Phase 4, Step 3):

```bash
node --version   # must print a version
```

### Step 1 — Install playwright@1.57.0 in a temp dir (avoids dead CDN)

```bash
mkdir -p /tmp/pw157
cd /tmp/pw157
npm install playwright@1.57.0
```

### Step 2 — Populate the driver cache

```bash
DRIVER_DIR="$HOME/Library/Caches/ms-playwright-go/1.57.0"
mkdir -p "$DRIVER_DIR/package"
cp -r /tmp/pw157/node_modules/playwright-core/* "$DRIVER_DIR/package/"
```

### Step 3 — Create the `playwright.sh` launcher

playwright-go calls `playwright.sh` as the entry point to the driver.

```bash
cat > "$DRIVER_DIR/playwright.sh" << 'EOF'
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
node "$SCRIPT_DIR/package/cli.js" "$@"
EOF
chmod +x "$DRIVER_DIR/playwright.sh"
```

### Step 4 — Symlink the `node` binary into the driver directory

playwright-go does not use `playwright.sh` to find Node — it `fork/exec`s a binary named
`node` **directly inside the driver cache directory**. Without this the error is:

```
could not run driver: fork/exec /Users/<you>/Library/Caches/ms-playwright-go/1.57.0/node:
no such file or directory
```

Fix: symlink the real `node` binary (use the permanent fnm path, not the session path):

```bash
ln -sf "$(readlink -f "$(which node)")" "$DRIVER_DIR/node"

# Verify:
ls -la "$DRIVER_DIR/node"
"$DRIVER_DIR/playwright.sh" --version   # → Version 1.57.0
```

> **Important**: Use `readlink -f` to resolve the real binary path.  
> `which node` under fnm returns a session-specific path like
> `~/.local/state/fnm_multishells/<pid>_<ts>/bin/node` that changes every session.  
> The permanent path looks like: `~/.local/share/fnm/node-versions/vX.Y.Z/installation/bin/node`

### Step 5 — Ensure browsers are available

The playwright-go driver needs browsers to control. Run the driver's own install to get the
browser revisions that match playwright 1.57.0:

```bash
"$DRIVER_DIR/playwright.sh" install chromium
```

Or install all browsers:

```bash
"$DRIVER_DIR/playwright.sh" install
```

### Step 6 — Cold-restart the IDE from terminal

```bash
# Cmd+Q the IDE first, then:
open -a "Antigravity IDE"
```

**Must be from terminal** — the IDE must inherit your shell's PATH (including fnm's node)
so the Go binary can find `node` when needed.

> **Cleanup**: `/tmp/pw157` was used only during the fix steps and can be removed:
> `rm -rf /tmp/pw157`

---

## Confirming It Works

After restarting, open the IDE and ask it to run a browser subagent task (e.g., "open google.com and take a screenshot"). If it succeeds, the fix is complete.

---

## If the Subagent Still Fails After This Fix

Check if the 1.57.0 driver's expected browser revisions differ from what was installed:

```bash
# See what browser revisions playwright@1.57.0 expects:
cat ~/Library/Caches/ms-playwright-go/1.57.0/package/browsers.json

# Compare with what's in the cache:
ls ~/Library/Caches/ms-playwright/

# If revision numbers don't match, install 1.57.0-specific browsers:
~/Library/Caches/ms-playwright-go/1.57.0/playwright.sh install chromium
```

Also verify Node is on PATH when IDE launches:

```bash
# In the terminal you use to launch the IDE:
which node   # must resolve (not "not found")
node --version
```

If `node` is not found, `fnm` is not initialized. Add to `~/.zshrc`:

```bash
eval "$(fnm env --use-on-cd)"
fnm use lts-latest
```

Then `source ~/.zshrc` and relaunch.

---

## Summary of Files Created by This Fix

| Path | Purpose |
|------|---------|
| `~/Library/Caches/ms-playwright-go/1.57.0/playwright.sh` | Driver launcher called by the Go binary |
| `~/Library/Caches/ms-playwright-go/1.57.0/node` | Symlink → permanent fnm node binary |
| `~/Library/Caches/ms-playwright-go/1.57.0/package/` | playwright-core 1.57.0 npm package |
