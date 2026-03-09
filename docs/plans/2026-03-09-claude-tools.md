# claude-tools Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build `claude-tools`, a bash setup script that installs and configures all recommended Claude Code developer tooling (RTK, LSP servers, plugins), plus a Claude-side verification doc that Claude reads on first interaction to confirm everything works.

**Architecture:** Single bash script (`claude-tools`) at the repo root with one subcommand (`setup`). It detects the project language from source files (or prompts if none found), installs the appropriate LSP servers, ensures RTK is installed and configured, installs Claude Code plugins, patches `settings.json`, and prints a reminder to verify inside Claude. A separate `docs/standards/devtools-verification.md` file gives Claude the steps to exercise each tool and confirm it works. CLAUDE.md gets a one-line addition linking to the verification doc.

**Tech Stack:** Bash, npm, cargo, Claude Code CLI (`claude plugin`)

---

### Task 1: Create the claude-tools script skeleton

**Files:**
- Create: `claude-tools`

**Step 1: Write the script skeleton with help text and argument parsing**

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_VERSION="0.1.0"
CLAUDE_DIR="${HOME}/.claude"
SETTINGS_FILE="${CLAUDE_DIR}/settings.json"

usage() {
  cat <<EOF
claude-tools v${SCRIPT_VERSION} — Developer tooling setup for Claude Code

Usage: ./claude-tools setup [OPTIONS]

Commands:
  setup    Install and configure all recommended developer tools

Options:
  --lang <language>   Force language (python|go|typescript|all). Auto-detected if omitted.
  --skip-rtk          Skip RTK installation
  --skip-lsp          Skip LSP server installation
  --skip-plugins      Skip Claude Code plugin installation
  --dry-run           Show what would be done without doing it
  -h, --help          Show this help

After setup completes, start a Claude Code session and ask:
  "Verify my developer tools are working"
EOF
}

main() {
  if [[ $# -eq 0 ]]; then
    usage
    exit 0
  fi

  local cmd="${1}"
  shift

  case "${cmd}" in
    setup) cmd_setup "$@" ;;
    -h|--help) usage ;;
    *) echo "Unknown command: ${cmd}. Run './claude-tools --help' for usage." >&2; exit 1 ;;
  esac
}

main "$@"
```

**Step 2: Make it executable and verify it runs**

Run: `chmod +x claude-tools && ./claude-tools --help`
Expected: Help text prints cleanly, exit 0

**Step 3: Commit**

```bash
git add claude-tools
git commit -m "feat: add claude-tools script skeleton with help text"
```

---

### Task 2: Add language detection

**Files:**
- Modify: `claude-tools`

**Step 1: Write the detect_languages function**

Add after the `usage()` function:

```bash
detect_languages() {
  local langs=()

  # Check for Python files
  if compgen -G "*.py" >/dev/null 2>&1 || compgen -G "**/*.py" >/dev/null 2>&1 || [[ -f "pyproject.toml" ]] || [[ -f "requirements.txt" ]] || [[ -f "setup.py" ]]; then
    langs+=("python")
  fi

  # Check for Go files
  if compgen -G "*.go" >/dev/null 2>&1 || compgen -G "**/*.go" >/dev/null 2>&1 || [[ -f "go.mod" ]]; then
    langs+=("go")
  fi

  # Check for TypeScript/JavaScript files
  if compgen -G "*.ts" >/dev/null 2>&1 || compgen -G "**/*.ts" >/dev/null 2>&1 || [[ -f "tsconfig.json" ]] || [[ -f "package.json" ]]; then
    langs+=("typescript")
  fi

  if [[ ${#langs[@]} -eq 0 ]]; then
    echo ""
  else
    printf '%s\n' "${langs[@]}"
  fi
}

prompt_language() {
  echo "No source files detected. Which language(s) will this project use?"
  echo ""
  echo "  1) Python"
  echo "  2) Go"
  echo "  3) TypeScript / JavaScript"
  echo "  4) All of the above"
  echo ""
  read -rp "Enter choice (1-4, or comma-separated e.g. 1,3): " choice

  local langs=()
  IFS=',' read -ra selections <<< "${choice}"
  for sel in "${selections[@]}"; do
    sel=$(echo "${sel}" | tr -d ' ')
    case "${sel}" in
      1) langs+=("python") ;;
      2) langs+=("go") ;;
      3) langs+=("typescript") ;;
      4) langs=("python" "go" "typescript"); break ;;
      *) echo "Invalid selection: ${sel}" >&2 ;;
    esac
  done

  printf '%s\n' "${langs[@]}"
}
```

**Step 2: Write a test — run detection in the current repo**

Run: `bash -c 'source claude-tools; detect_languages'`
Expected: Should not output "python" or "typescript" (this repo has no .py or .ts files, no go.mod). May output nothing since this is a docs-only repo.

**Step 3: Commit**

```bash
git add claude-tools
git commit -m "feat: add language detection for LSP server selection"
```

---

### Task 3: Add RTK installation and configuration check

**Files:**
- Modify: `claude-tools`

**Step 1: Write the install_rtk function**

```bash
install_rtk() {
  local dry_run="${1:-false}"

  echo ""
  echo "=== RTK (Rust Token Killer) ==="

  # Check if cargo is available
  if ! command -v cargo &>/dev/null; then
    echo "  SKIP: cargo not found. Install Rust first: https://rustup.rs"
    echo "  Then re-run: ./claude-tools setup"
    return 1
  fi

  # Check if rtk is installed
  if command -v rtk &>/dev/null; then
    local current_version
    current_version=$(rtk --version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)
    echo "  FOUND: rtk v${current_version}"

    if [[ "${dry_run}" == "true" ]]; then
      echo "  DRY-RUN: would run 'cargo install rust_token_killer' to update"
    else
      echo "  Updating..."
      cargo install rust_token_killer 2>&1 | tail -1
    fi
  else
    echo "  NOT FOUND: installing rtk..."
    if [[ "${dry_run}" == "true" ]]; then
      echo "  DRY-RUN: would run 'cargo install rust_token_killer'"
    else
      cargo install rust_token_killer
    fi
  fi

  # Check if rtk is configured for Claude Code
  if command -v rtk &>/dev/null || [[ "${dry_run}" == "true" ]]; then
    local rtk_status
    rtk_status=$(rtk init --show 2>&1 || true)

    if echo "${rtk_status}" | grep -q "Hook:.*rtk-rewrite.sh"; then
      echo "  CONFIGURED: hook is set up"
    else
      echo "  CONFIGURING: running 'rtk init -g --auto-patch'..."
      if [[ "${dry_run}" == "true" ]]; then
        echo "  DRY-RUN: would run 'rtk init -g --auto-patch'"
      else
        rtk init -g --auto-patch
      fi
    fi
  fi

  echo "  DONE"
}
```

**Step 2: Verify it compiles (syntax check)**

Run: `bash -n claude-tools`
Expected: No output (clean syntax)

**Step 3: Commit**

```bash
git add claude-tools
git commit -m "feat: add RTK installation and configuration check"
```

---

### Task 4: Add LSP server installation

**Files:**
- Modify: `claude-tools`

**Step 1: Write the install_lsp function**

```bash
install_lsp() {
  local dry_run="${1:-false}"
  shift
  local langs=("$@")

  echo ""
  echo "=== LSP Servers ==="

  for lang in "${langs[@]}"; do
    case "${lang}" in
      python)
        echo "  [Python] pyright"
        if command -v pyright &>/dev/null; then
          local pyright_ver
          pyright_ver=$(pyright --version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)
          echo "    FOUND: v${pyright_ver}"
          if [[ "${dry_run}" == "true" ]]; then
            echo "    DRY-RUN: would run 'npm install -g pyright@latest'"
          else
            echo "    Updating..."
            npm install -g pyright@latest 2>&1 | tail -1
          fi
        else
          if [[ "${dry_run}" == "true" ]]; then
            echo "    DRY-RUN: would run 'npm install -g pyright'"
          else
            echo "    Installing..."
            npm install -g pyright
          fi
        fi
        ;;
      go)
        echo "  [Go] gopls"
        if command -v gopls &>/dev/null; then
          local gopls_ver
          gopls_ver=$(gopls version 2>/dev/null | head -1 | grep -oE 'v[0-9]+\.[0-9]+\.[0-9]+' | head -1)
          echo "    FOUND: ${gopls_ver}"
          if [[ "${dry_run}" == "true" ]]; then
            echo "    DRY-RUN: would run 'go install golang.org/x/tools/gopls@latest'"
          else
            echo "    Updating..."
            go install golang.org/x/tools/gopls@latest
          fi
        else
          if ! command -v go &>/dev/null; then
            echo "    SKIP: go not found. Install Go first."
          elif [[ "${dry_run}" == "true" ]]; then
            echo "    DRY-RUN: would run 'go install golang.org/x/tools/gopls@latest'"
          else
            echo "    Installing..."
            go install golang.org/x/tools/gopls@latest
          fi
        fi
        ;;
      typescript)
        echo "  [TypeScript] typescript-language-server"
        if command -v typescript-language-server &>/dev/null; then
          echo "    FOUND"
          if [[ "${dry_run}" == "true" ]]; then
            echo "    DRY-RUN: would run 'npm install -g typescript-language-server@latest typescript@latest'"
          else
            echo "    Updating..."
            npm install -g typescript-language-server@latest typescript@latest 2>&1 | tail -1
          fi
        else
          if [[ "${dry_run}" == "true" ]]; then
            echo "    DRY-RUN: would run 'npm install -g typescript-language-server typescript'"
          else
            echo "    Installing..."
            npm install -g typescript-language-server typescript
          fi
        fi
        ;;
    esac
  done

  echo "  DONE"
}
```

**Step 2: Syntax check**

Run: `bash -n claude-tools`
Expected: No output

**Step 3: Commit**

```bash
git add claude-tools
git commit -m "feat: add LSP server installation for Python, Go, TypeScript"
```

---

### Task 5: Add settings.json patching (ENABLE_LSP_TOOL)

**Files:**
- Modify: `claude-tools`

**Step 1: Write the patch_settings function**

```bash
patch_settings() {
  local dry_run="${1:-false}"

  echo ""
  echo "=== Claude Code Settings ==="

  if ! command -v jq &>/dev/null; then
    echo "  ERROR: jq is required for settings patching. Install jq first."
    return 1
  fi

  mkdir -p "${CLAUDE_DIR}"

  if [[ ! -f "${SETTINGS_FILE}" ]]; then
    echo "  Creating ${SETTINGS_FILE}..."
    if [[ "${dry_run}" == "true" ]]; then
      echo "  DRY-RUN: would create settings.json with ENABLE_LSP_TOOL=1"
    else
      echo '{}' > "${SETTINGS_FILE}"
    fi
  fi

  # Check if ENABLE_LSP_TOOL is already set
  local current_lsp
  current_lsp=$(jq -r '.env.ENABLE_LSP_TOOL // empty' "${SETTINGS_FILE}" 2>/dev/null)

  if [[ "${current_lsp}" == "1" ]]; then
    echo "  ENABLE_LSP_TOOL: already enabled"
  else
    echo "  ENABLE_LSP_TOOL: enabling..."
    if [[ "${dry_run}" == "true" ]]; then
      echo "  DRY-RUN: would set env.ENABLE_LSP_TOOL=1 in settings.json"
    else
      local tmp
      tmp=$(mktemp)
      jq '.env = (.env // {}) + {"ENABLE_LSP_TOOL": "1"}' "${SETTINGS_FILE}" > "${tmp}" && mv "${tmp}" "${SETTINGS_FILE}"
    fi
  fi

  echo "  DONE"
}
```

**Step 2: Syntax check**

Run: `bash -n claude-tools`
Expected: No output

**Step 3: Commit**

```bash
git add claude-tools
git commit -m "feat: add settings.json patching for LSP"
```

---

### Task 6: Add Claude Code plugin installation

**Files:**
- Modify: `claude-tools`

**Step 1: Write the install_plugins function**

```bash
install_plugins() {
  local dry_run="${1:-false}"
  shift
  local langs=("$@")

  echo ""
  echo "=== Claude Code Plugins ==="

  if ! command -v claude &>/dev/null; then
    echo "  ERROR: 'claude' CLI not found. Install Claude Code first."
    return 1
  fi

  # Ensure official marketplace is registered
  local known_marketplaces
  known_marketplaces=$(cat "${CLAUDE_DIR}/plugins/known_marketplaces.json" 2>/dev/null || echo "{}")

  if ! echo "${known_marketplaces}" | jq -e '."claude-plugins-official"' &>/dev/null; then
    echo "  Adding official plugin marketplace..."
    if [[ "${dry_run}" != "true" ]]; then
      claude plugin marketplace add anthropics/claude-plugins-official 2>&1 || true
    fi
  fi

  # Ensure superpowers marketplace is registered
  if ! echo "${known_marketplaces}" | jq -e '."superpowers-extended-cc-marketplace"' &>/dev/null; then
    echo "  Adding superpowers marketplace..."
    if [[ "${dry_run}" != "true" ]]; then
      claude plugin marketplace add pcvelz/superpowers 2>&1 || true
    fi
  fi

  # LSP plugins based on detected languages
  for lang in "${langs[@]}"; do
    case "${lang}" in
      python)
        install_plugin "pyright-lsp@claude-plugins-official" "${dry_run}"
        ;;
      go)
        install_plugin "gopls-lsp@claude-plugins-official" "${dry_run}"
        ;;
      typescript)
        # typescript-language-server plugin — check official marketplace for availability
        install_plugin "typescript-lsp@claude-plugins-official" "${dry_run}"
        ;;
    esac
  done

  # Workflow plugins (always install)
  install_plugin "superpowers-extended-cc@superpowers-extended-cc-marketplace" "${dry_run}"
  install_plugin "code-review@claude-plugins-official" "${dry_run}"
  install_plugin "security-guidance@claude-plugins-official" "${dry_run}"

  echo "  DONE"
}

install_plugin() {
  local plugin_name="${1}"
  local dry_run="${2:-false}"
  local short_name="${plugin_name%%@*}"

  # Check if already installed
  local installed
  installed=$(jq -r ".plugins.\"${plugin_name}\" // empty" "${CLAUDE_DIR}/plugins/installed_plugins.json" 2>/dev/null)

  if [[ -n "${installed}" ]]; then
    echo "  FOUND: ${short_name}"
  else
    echo "  INSTALLING: ${short_name}..."
    if [[ "${dry_run}" == "true" ]]; then
      echo "    DRY-RUN: would run 'claude plugin install ${plugin_name}'"
    else
      claude plugin install "${plugin_name}" 2>&1 | sed 's/^/    /'
    fi
  fi
}
```

**Step 2: Syntax check**

Run: `bash -n claude-tools`
Expected: No output

**Step 3: Commit**

```bash
git add claude-tools
git commit -m "feat: add Claude Code plugin installation"
```

---

### Task 7: Wire up the cmd_setup function

**Files:**
- Modify: `claude-tools`

**Step 1: Write cmd_setup that orchestrates everything**

```bash
cmd_setup() {
  local dry_run="false"
  local forced_lang=""
  local skip_rtk="false"
  local skip_lsp="false"
  local skip_plugins="false"

  while [[ $# -gt 0 ]]; do
    case "${1}" in
      --lang) forced_lang="${2}"; shift 2 ;;
      --skip-rtk) skip_rtk="true"; shift ;;
      --skip-lsp) skip_lsp="true"; shift ;;
      --skip-plugins) skip_plugins="true"; shift ;;
      --dry-run) dry_run="true"; shift ;;
      -h|--help) usage; exit 0 ;;
      *) echo "Unknown option: ${1}" >&2; exit 1 ;;
    esac
  done

  echo "claude-tools v${SCRIPT_VERSION} — setup"
  echo ""

  # Determine languages
  local langs=()
  if [[ -n "${forced_lang}" ]]; then
    if [[ "${forced_lang}" == "all" ]]; then
      langs=("python" "go" "typescript")
    else
      IFS=',' read -ra langs <<< "${forced_lang}"
    fi
    echo "Languages (forced): ${langs[*]}"
  else
    mapfile -t langs < <(detect_languages)
    if [[ ${#langs[@]} -eq 0 ]] || [[ -z "${langs[0]}" ]]; then
      mapfile -t langs < <(prompt_language)
    fi
    echo "Languages (detected): ${langs[*]}"
  fi

  if [[ ${#langs[@]} -eq 0 ]] || [[ -z "${langs[0]}" ]]; then
    echo "ERROR: No languages selected. Use --lang to specify." >&2
    exit 1
  fi

  # Check prerequisites
  echo ""
  echo "=== Prerequisites ==="
  local prereqs_ok="true"

  if ! command -v jq &>/dev/null; then
    echo "  MISSING: jq (required for settings patching)"
    prereqs_ok="false"
  else
    echo "  OK: jq"
  fi

  if ! command -v npm &>/dev/null; then
    echo "  MISSING: npm (required for LSP servers)"
    prereqs_ok="false"
  else
    echo "  OK: npm"
  fi

  if ! command -v claude &>/dev/null; then
    echo "  MISSING: claude CLI (required for plugin installation)"
    prereqs_ok="false"
  else
    echo "  OK: claude"
  fi

  if [[ "${prereqs_ok}" == "false" ]]; then
    echo ""
    echo "Install missing prerequisites and re-run."
    exit 1
  fi

  # Run installation steps
  if [[ "${skip_rtk}" != "true" ]]; then
    install_rtk "${dry_run}"
  fi

  if [[ "${skip_lsp}" != "true" ]]; then
    install_lsp "${dry_run}" "${langs[@]}"
  fi

  patch_settings "${dry_run}"

  if [[ "${skip_plugins}" != "true" ]]; then
    install_plugins "${dry_run}" "${langs[@]}"
  fi

  # Final summary
  echo ""
  echo "========================================"
  echo "  Setup complete!"
  echo "========================================"
  echo ""
  echo "Next step: start a Claude Code session and ask:"
  echo ""
  echo '  "Verify my developer tools are working"'
  echo ""
  echo "Claude will exercise each tool (RTK, LSP, plugins)"
  echo "and confirm they are accessible from within its context."
  echo ""
}
```

**Step 2: Test the full script with --dry-run**

Run: `./claude-tools setup --dry-run --lang python`
Expected: Prints each section with DRY-RUN messages, no actual installations, exits cleanly

**Step 3: Test with --help**

Run: `./claude-tools setup --help`
Expected: Shows usage text

**Step 4: Commit**

```bash
git add claude-tools
git commit -m "feat: wire up setup command with orchestration and dry-run"
```

---

### Task 8: Create the verification doc

**Files:**
- Create: `docs/standards/devtools-verification.md`

**Step 1: Write the verification document**

```markdown
# Developer Tools Verification

When asked to verify developer tools, follow these steps to confirm each tool is accessible and working from within Claude Code's context.

## 1. RTK (Rust Token Killer)

**Check hook is active:**
- Run a simple command like `git status` via Bash
- If RTK is working, the hook rewrites it to `rtk git status` transparently
- Confirm by running `rtk --version` directly — if it returns a version, RTK is installed

**Check configuration:**
- Read `~/.claude/settings.json` and confirm a `PreToolUse` hook entry exists for `rtk-rewrite.sh`
- Read `~/.claude/hooks/rtk-rewrite.sh` and confirm it exists and is executable

**Report:**
- RTK version
- Hook status (configured / not configured)
- Test result (working / not working)

## 2. LSP (Language Server Protocol)

**Check LSP is enabled:**
- Read `~/.claude/settings.json` and confirm `"ENABLE_LSP_TOOL": "1"` is set in `env`

**Check language servers are installed:**
- Python: run `pyright --version`
- Go: run `gopls version`
- TypeScript: run `typescript-language-server --version`

**Check plugins are enabled:**
- Read `~/.claude/plugins/installed_plugins.json`
- For each detected language, confirm the corresponding LSP plugin is installed:
  - Python: `pyright-lsp@claude-plugins-official`
  - Go: `gopls-lsp@claude-plugins-official`
- Confirm the plugin is also enabled in `settings.json` under `enabledPlugins`

**Exercise LSP (if source files exist):**
- Use the LSP tool to perform a go-to-definition or find-references on any function in the project
- If no source files exist, skip this step and note it

**Report:**
- ENABLE_LSP_TOOL setting (enabled / not set)
- Each language server: installed version or "not found"
- Each LSP plugin: installed + enabled / installed but disabled / not installed

## 3. Claude Code Plugins

**Check workflow plugins:**
- Read `~/.claude/plugins/installed_plugins.json` and confirm:
  - `superpowers-extended-cc@superpowers-extended-cc-marketplace` — installed
  - `code-review@claude-plugins-official` — installed
  - `security-guidance@claude-plugins-official` — installed
- Check `settings.json` `enabledPlugins` to confirm each is enabled

**Report:**
- Each plugin: installed + enabled / installed but disabled / not installed

## 4. Summary

Present a table:

| Tool | Status | Details |
|---|---|---|
| RTK | OK / ISSUE | version, hook status |
| LSP (Python) | OK / ISSUE / N/A | pyright version, plugin status |
| LSP (Go) | OK / ISSUE / N/A | gopls version, plugin status |
| LSP (TypeScript) | OK / ISSUE / N/A | ts-language-server version, plugin status |
| superpowers | OK / ISSUE | install + enable status |
| code-review | OK / ISSUE | install + enable status |
| security-guidance | OK / ISSUE | install + enable status |

If any tool shows ISSUE, explain what's wrong and how to fix it.
```

**Step 2: Commit**

```bash
git add docs/standards/devtools-verification.md
git commit -m "docs: add developer tools verification guide for Claude"
```

---

### Task 9: Add verification reference to CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Add one line to the Detailed Standards section**

After the existing list of standard links, add:

```markdown
- [Developer Tools Verification](docs/standards/devtools-verification.md)
```

**Step 2: Verify CLAUDE.md is still clean**

Run: `wc -l CLAUDE.md`
Expected: ~93 lines (was 92, added 1)

**Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: link devtools-verification from CLAUDE.md"
```

---

### Task 10: Update README to reference claude-tools

**Files:**
- Modify: `README.md`

**Step 1: Add claude-tools reference to the Recommended Developer Tools section**

Add at the start of the "Recommended Developer Tools" section, before the RTK subsection:

```markdown
### Quick Setup

Run the setup script to install and configure all recommended tools at once:

\```bash
./claude-tools setup
\```

This detects your project language, installs the appropriate LSP servers, configures RTK, installs Claude Code plugins, and patches your settings. Use `--dry-run` to preview changes first.

After setup, start a Claude Code session and ask: **"Verify my developer tools are working"** — Claude will exercise each tool and confirm everything is accessible.
```

**Step 2: Update the Repository Structure section to include claude-tools**

Add to the tree listing:

```
claude-tools                         # Developer tooling setup script
```

**Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add claude-tools quick setup to README"
```

---

### Task 11: End-to-end dry-run test

**Files:**
- No changes — verification only

**Step 1: Run full dry-run**

Run: `./claude-tools setup --dry-run --lang all`
Expected: All sections print with DRY-RUN messages, clean exit

**Step 2: Run with auto-detection in this repo**

Run: `./claude-tools setup --dry-run`
Expected: No languages detected, prompts for input (or prints error if non-interactive)

**Step 3: Run with --skip flags**

Run: `./claude-tools setup --dry-run --lang python --skip-rtk --skip-plugins`
Expected: Only LSP and settings sections run

**Step 4: Final commit if any fixes needed**

```bash
git add -A
git commit -m "fix: address issues found in end-to-end testing"
```

---

Plan complete and saved to `docs/plans/2026-03-09-claude-tools.md`. Two execution options:

**1. Subagent-Driven (this session)** — I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** — Open new session in a worktree, batch execution with checkpoints

Which approach?