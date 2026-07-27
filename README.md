# macOS Setup — Research + Development Environment

**Audience**: Researchers and academics using MacBook/MacMini (remote servers for heavy jobs).

**Philosophy**:
- MacBook = writing, AI agents, light plotting, SSH orchestration.
- Mac Mini = full dev workstation with build toolchain, Docker, and QGIS.

---

## Repository structure

```
maOS-research-setup/
├── README.md               ← this file
├── lighter/
│   └── Brewfile                ← basictex, no build toolchain
├── heavier/
│   └── Brewfile                ← mactex, gcc/cmake, Docker, QGIS
└── extras/
    └── antigravity_playwright_fix.md   ← fix for browser subagent on fresh install
```

---

## Phase 0 — macOS First-Run Setup *(manual, one-time)*

After erasing and reinstalling macOS:

1. Log in to Apple Account during the setup wizard.
2. **FileVault**: System Settings → Privacy & Security → FileVault → Turn On
3. **Find My**: System Settings → [Your Name] → Find My → Find My Mac → On
4. **macOS updates**: System Settings → General → Software Update → install all
5. Open **Terminal**.

---

## Phase 1 — Command Line Tools and Homebrew

### Step 1 — Xcode Command Line Tools

```bash
xcode-select --install
```

Verify:

```bash
xcode-select -p      # → /Library/Developer/CommandLineTools
git --version
clang --version
```

> Homebrew will install a modern `git` later and place it first on `PATH`.
> The CLT `git` is only needed to bootstrap Homebrew.

### Step 2 — Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Activate immediately (Apple Silicon):
eval "$(/opt/homebrew/bin/brew shellenv)"

# Persist across shell sessions:
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

Verify:

```bash
brew doctor
brew --version
```

---

## Phase 2 — Official AI Agent Apps *(manual install)*

Install from the respective official sources. These are managed manually to avoid
auto-updates breaking workflows.

### Step 1 — Claude (Anthropic)

Download from <https://claude.ai/download>.  
Log in with your Anthropic account after launch.

### Step 2 — ChatGPT / Codex (OpenAI)

Download from <https://openai.com/chatgpt/download/>.  
Log in with your OpenAI account after launch.

### Step 3 — Antigravity IDE (Google)

Download from the official Google distribution channel.  
Log in with your Google account after launch.

---

## Phase 3 — Official AI Agent CLIs *(manual install)*

These CLIs are kept separate from Homebrew to allow pinned versions.

### Step 1 — Claude Code (Anthropic)

Installed automatically as part of the Claude desktop app, or via:

```bash
# If not bundled with the Claude app:
npm install -g @anthropic-ai/claude-code
claude --version
```

### Step 2 — Codex CLI (OpenAI)

The Codex CLI ships with or alongside the ChatGPT app. Verify:

```bash
codex --version
```

If missing, install via the official installer:

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
codex --version
```

### Step 3 — Antigravity CLI (Google)

Install the Antigravity CLI from the official distribution channel. This places
the `agy` binary. Verify:

```bash
agy --version
```

The `agy-ide` shell command (for opening the IDE from terminal) is installed
from within the Antigravity IDE app itself. After installing it from the IDE:

```bash
echo 'export PATH="$HOME/.antigravity-ide/antigravity-ide/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
agy-ide --version
```

> **Two separate Antigravity binaries**:
> - `agy` — CLI agent, installed via the Antigravity CLI installer
> - `agy-ide` — IDE shell command, installed from within the IDE app

---

## Phase 4 — Packages and Applications via Brewfile

### Step 1 — Core CLI

Clone this repository, then pick the Brewfile for your machine:

| Setup | Command |
|---|---|
| lighter | `brew bundle --file lighter/Brewfile` |
| heavier | `brew bundle --file heavier/Brewfile` |

Both Brewfiles install:

- **Core CLI**: `git`, `gh`, `aria2`, `wget`, `curl`, `jq`, `tree`, `htop`,
  `ripgrep`, `fd`, `fzf`, `zoxide`, `rsync`, `tmux`, `mosh`
- **Scientific Python / R**: `miniconda`, `uv`, `r`
- **Node.js**: `fnm`, `pnpm`
- **Writing / publishing**: Quarto, Zotero, Obsidian, Microsoft Office
- **Browsers**: Firefox, Google Chrome
- **Remote access**: Windows App

The `heavier/Brewfile` additionally installs:

- **TeX**: `mactex` (full, ~4 GB) — `lighter/Brewfile` uses `basictex` (~100 MB)
- **Build toolchain**: `gcc`, `cmake`, `pkg-config`, `make`
- **Containerisation**: Docker Desktop
- **Geospatial**: QGIS (see [QGIS Python plugin note](#qgis-python-plugin-note) below)

Verify (after install):

```bash
# lighter:
brew bundle check --no-upgrade --file lighter/Brewfile

# heavier:
brew bundle check --no-upgrade --file heavier/Brewfile
```

### Step 2 — Scientific Python / R (brew-installed)

> R is installed via `brew install r`. Additional R packages are installed
> inside R or RStudio as needed.

#### QGIS Python plugin note

The `heavier/Brewfile` installs QGIS via `cask "qgis"` (the official DMG build). This ships
its own bundled Python interpreter, separate from conda and Homebrew Python:

```
/Applications/QGIS.app/Contents/MacOS/bin/python3
```

Packages installed in conda or via `pip` outside QGIS are **not visible to QGIS plugins**.
To install a package for use inside a QGIS plugin:

```bash
sudo /Applications/QGIS.app/Contents/MacOS/bin/pip3 install <package>
```

To reset all pip-installed packages back to a clean state:

```bash
brew reinstall --cask qgis   # replaces the entire .app bundle
```

> **Why not `brew install qgis` (the formula)?**  
> The formula builds QGIS from source against Homebrew's Python (~45 min compile),
> which makes conda/pip packages directly visible to plugins. The trade-off is a
> long build time and potential breakage after `brew upgrade`. For most plugin
> workflows the cask is the right default; switch to the formula only if you need
> tight conda/pip integration inside QGIS.

### Step 3 — Node.js LTS

`fnm` and `pnpm` are installed via Homebrew in Step 1.

```bash
# Ensure fnm is initialised in this shell (add to ~/.zshrc if not already):
eval "$(fnm env --use-on-cd)"

# Install Node LTS:
fnm install --lts
fnm use lts-latest

# Verify:
node --version
npm --version
pnpm --version
```

### Step 4 — Writing / Publishing

`quarto`, `zotero`, `obsidian`, `microsoft-office` are installed via Homebrew in Step 1.
TeX (`basictex` or `mactex`) is also installed depending on your Brewfile.

- **Zotero browser connectors**: install manually from
  <https://www.zotero.org/download/connectors> for Firefox and Chrome.
- **TeX** adds binaries to `/Library/TeX/texbin`. Add to PATH:

```bash
echo 'export PATH="/Library/TeX/texbin:$PATH"' >> ~/.zprofile
export PATH="/Library/TeX/texbin:$PATH"

# Update tlmgr:
sudo tlmgr update --self
sudo tlmgr update --all

# Verify:
pdflatex --version
```

> With `basictex`, install additional packages as needed:
> `sudo tlmgr install <package>`  
> With `mactex`, all packages are available immediately.

- **Quarto** (`cask "quarto"`) installs via a `.pkg`. If `quarto` is not on PATH
  after `brew bundle`, the silent pkg install was blocked by Gatekeeper. Fix:

```bash
open /opt/homebrew/Caskroom/quarto/$(brew info --cask quarto | awk 'NR==1{print $NF}')/quarto-*-macos.pkg
# Click through the GUI installer, then verify:
quarto --version
```

- **R packages for Quarto** — `knitr` and `rmarkdown` are required to render
  `.qmd` files with R code chunks. Install them after R is on PATH:

```bash
Rscript -e 'install.packages(c("knitr", "rmarkdown"), repos="https://cloud.r-project.org")'
```

Verify everything:

```bash
quarto check
# Expected: LaTeX ✓, Chrome headless ✓, R knitr/rmarkdown ✓
# TinyTeX "not installed" is fine — your TeX install covers it.
# Python Jupyter "None" is fine — Jupyter lives in the local-viz conda env (Phase 7).
```

### Step 5 — Browsers

`firefox` and `google-chrome` are installed via Homebrew in Step 1.

### Step 6 — Remote Access

`windows-app` is installed via Homebrew in Step 1.

---

## Phase 5 — IDE Manual Install

### Step 1 — VS Code 1.98

Install VS Code **version 1.98** from <https://code.visualstudio.com/updates/v1_98>.

> **Why pinned?** The Remote-SSH extension requires VS Code ≥ the remote server's
> supported version. Ubuntu 18.04 LTS and CentOS 7 support only VS Code ≤ 1.98.
> Keep this version until those servers are decommissioned.

### Step 2 — VS Code Insiders

Install VS Code Insiders from <https://code.visualstudio.com/insiders/>. Used
for testing extensions and newer remote server targets.

### Step 3 — VS Code Extensions

```bash
# Confirm VS Code CLI is on PATH:
code --version

# Install extensions:
code --install-extension ms-python.python
code --install-extension ms-toolsai.jupyter
code --install-extension ms-vscode-remote.remote-ssh
code --install-extension quarto.quarto
code --install-extension james-yu.latex-workshop
code --install-extension redhat.vscode-yaml
code --install-extension yzhang.markdown-all-in-one
code --install-extension streetsidesoftware.code-spell-checker
code --install-extension charliermarsh.ruff
```

Enable **Settings Sync**: Code → Settings → Turn on Settings Sync → sign in with GitHub.

---

## Phase 6 — SSH Setup *(remote-first workflow)*

```bash
# Create SSH directory:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/config
chmod 600 ~/.ssh/config

# Generate Ed25519 key (Choose ONE option):

# Option A: With Passphrase (Recommended for manual use)
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519

# Option B: Without Passphrase (Recommended for AI agents/automation)
# -N "" creates a key with no passphrase for seamless automation
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519 -N ""

# Add to macOS Keychain (persists across reboots, useful if you used Option A):
ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# Copy public key to each remote server:
ssh-copy-id -i ~/.ssh/id_ed25519.pub username@your-server.example.com
```

Template `~/.ssh/config` — add one block per server:

```
Host my-server
    HostName your.server.example.com
    User yourusername
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
    UseKeychain yes
    ServerAliveInterval 60   # keepalive every 1 min (under typical 5-min firewall timeout)
    ServerAliveCountMax 5    # tolerate up to 5 min silence (server busy under load)
```

Verify:

```bash
ssh my-server echo "Connection OK"
```

> `~/.ssh/config` is the canonical source of truth for all remote logins.
> GUI SSH clients are optional extras only.

---

## Phase 7 — Miniconda Setup

`miniconda` is installed via Homebrew in Phase 4.

```bash
# Initialise conda for zsh:
conda init zsh
# ↳ Restart Terminal after this.

# Configure channels and behaviour:
conda config --set channel_priority strict
conda config --add channels conda-forge
conda config --set auto_activate false        # do NOT auto-activate base in every shell
# Note: auto_activate_base is a legacy alias for auto_activate; use the latter to avoid warnings
```

> **`auto_activate false`** is essential — without it, conda prepends
> `(base)` to every shell prompt and may interfere with system Python paths.
> (The legacy key `auto_activate_base` is an alias; use `auto_activate` to avoid deprecation warnings.)

### Recommended environment split

Keep local environments lean. Anything compute-heavy or geospatial belongs in
a dedicated conda env rather than the base or a shared env:

| Environment | Purpose |
|---|---|
| `local-viz` | Plotting, dashboards, notebooks |
| `geo` | GDAL, geopandas, rasterio — geospatial workflows |
| *(remote)* | ML, heavy simulation stacks — run on remote servers |

---

## Phase 8 — Antigravity Playwright Driver Fix *(if browser subagent fails)*

The Antigravity IDE browser subagent uses a native Go binary (`language_server_macos_arm`)
with its own Playwright manager. On a fresh Mac it tries to download the pinned driver zip:

```
playwright-1.57.0-mac-arm64.zip  ← from playwright.azureedge.net (CDN is 404 — known issue)
```

This is **not** fixable with `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD` or `PLAYWRIGHT_BROWSERS_PATH`
— those env vars only affect the npm playwright CLI, not the Go binary's driver manager.
See `extras/antigravity_playwright_fix.md` for full investigation details.

**Prerequisite** — Node must be active (Phase 4, Step 3 complete):

```bash
node --version   # must print a version
```

**Step 1** — Install playwright-core 1.57.0 from npm (avoids the dead CDN):

```bash
mkdir -p /tmp/pw157
cd /tmp/pw157
npm install playwright@1.57.0
```

**Step 2** — Populate the driver cache:

```bash
DRIVER_DIR="$HOME/Library/Caches/ms-playwright-go/1.57.0"
mkdir -p "$DRIVER_DIR/package"
cp -r /tmp/pw157/node_modules/playwright-core/* "$DRIVER_DIR/package/"
```

**Step 3** — Create the `playwright.sh` launcher:

```bash
cat > "$DRIVER_DIR/playwright.sh" << 'EOF'
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
node "$SCRIPT_DIR/package/cli.js" "$@"
EOF
chmod +x "$DRIVER_DIR/playwright.sh"
```

**Step 4** — Symlink `node` into the driver directory:

The Go binary `fork/exec`s a binary named `node` **directly inside the driver cache**.
Use `readlink -f` to get the permanent path (not fnm's session-specific shim):

```bash
ln -sf "$(readlink -f "$(which node)")" "$DRIVER_DIR/node"

# Verify the symlink points to a real binary:
ls -la "$DRIVER_DIR/node"
"$DRIVER_DIR/playwright.sh" --version   # should print: Version 1.57.0
```

**Step 5** — Cold-restart the IDE from terminal:

```bash
# Cmd+Q the IDE first, then:
open -a "Antigravity IDE"
```

> **Always launch from terminal** — the IDE must inherit your shell's PATH (including fnm's
> node) so the Go binary can locate `node` when needed.

> **Cleanup**: `/tmp/pw157` was used only during the fix steps and can be removed.

---

## Phase 9 — VPNs

VPN clients require manual installation from institutional IT portals.

---

## Phase 10 — Verify Setup

Run this checklist after completing all phases to confirm everything is working.

```bash
# ── Phase 1: Xcode CLT + Homebrew ─────────────────────────────────────────
xcode-select -p                   # → /Library/Developer/CommandLineTools
brew --version
brew doctor

# ── Phase 4: Core CLI ─────────────────────────────────────────────────────
git --version
gh --version
aria2c --version
tmux -V
mosh --version
rg --version
fzf --version
zoxide --version

# ── Phase 4: Python / R / Node ────────────────────────────────────────────
conda --version
conda activate local-viz
python -c "import numpy, pandas, matplotlib, plotly; print('local-viz OK')"
conda deactivate
uv --version
r --version

node --version
npm --version
pnpm --version

# ── Phase 4: Writing / Publishing ─────────────────────────────────────────
quarto check
pdflatex --version

# ── Phase 2 & 3: AI Apps + CLIs ───────────────────────────────────────────
claude --version
codex --version
agy --version
agy-ide --version

# ── Phase 5: VS Code ──────────────────────────────────────────────────────
code --version

# ── Phase 6: SSH ──────────────────────────────────────────────────────────
ssh my-server echo "SSH OK"
```

---

## Shell Configuration (`~/.zshrc`)

Add the following block to `~/.zshrc` (edit with `nano ~/.zshrc` or `code ~/.zshrc`):

```zsh
# fnm — Node version manager
eval "$(fnm env --use-on-cd)"

# zoxide — smart cd
eval "$(zoxide init zsh)"

# fzf — fuzzy finder shell integration
source "$(brew --prefix)/opt/fzf/shell/completion.zsh"
source "$(brew --prefix)/opt/fzf/shell/key-bindings.zsh"

# PATH: Antigravity CLI `agy`
export PATH="$HOME/.local/bin:$PATH"

# PATH: Antigravity IDE shell command `agy-ide`
export PATH="$HOME/.antigravity-ide/antigravity-ide/bin:$PATH"

# PATH: npm global binaries (Claude Code, Codex, etc.)
export PATH="$(npm config get prefix)/bin:$PATH"

# PATH: TeX binaries
export PATH="/Library/TeX/texbin:$PATH"
```

> `eval "$(/opt/homebrew/bin/brew shellenv)"` lives in `~/.zprofile` (set in Phase 1).  
> `conda init zsh` auto-inserts its block into `~/.zshrc` — do not duplicate it.

Apply changes:

```bash
source ~/.zshrc
```

---

## Update Workflow

Run periodically to keep the machine current:

```bash
# 1. Homebrew (formulae and casks):
brew update && brew upgrade && brew upgrade --cask && brew cleanup

# 2. Dump current Brewfile state (after adding new packages):
brew bundle dump --file lighter/Brewfile --force
# brew bundle dump --file heavier/Brewfile --force

# 3. Conda:
conda update -n base -c conda-forge conda
conda update --all -n local-viz

# 4. Node (via fnm):
fnm install --lts && fnm use lts-latest   # installs + activates new LTS if available

# 5. AI CLIs (manual):
#    Claude Code: npm update -g @anthropic-ai/claude-code
#    Codex: re-run curl installer or update via ChatGPT app
#    Antigravity CLI: reinstall from the official distribution channel

# 6. TeX Live packages:
sudo tlmgr update --self && sudo tlmgr update --all
```

---

## What to Keep Off the Mac

Avoid installing these locally. Use conda environments or remote machines instead:

| Category | Examples |
|---|---|
| Geospatial libs | GDAL, PROJ, GEOS, rasterio, cartopy |
| Scientific I/O | NetCDF, HDF5, netcdf4, h5py |
| Heavy Python stacks | xarray, dask, large domain-specific envs |
| ML frameworks | PyTorch, TensorFlow, JAX |
| GPU toolchains | CUDA, cuDNN |
| Large datasets | rasters, simulation archives |

