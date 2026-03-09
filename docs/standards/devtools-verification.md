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
