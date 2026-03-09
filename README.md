# claude-tools

Developer tooling setup for Claude Code. Installs and configures all the recommended tools from the [claude.md-boilerplate](https://github.com/leighstillard/claude.md-boilerplate) template in one command.

## What It Sets Up

| Tool | What It Does | Install Method |
|---|---|---|
| **RTK** (Rust Token Killer) | CLI proxy that compresses tool output, saving 60-90% of tokens | `cargo install rust_token_killer` |
| **LSP servers** | Semantic code understanding — go-to-definition, find-references, type info in ~50ms | pyright (npm), gopls (go install), typescript-language-server (npm) |
| **Claude Code plugins** | superpowers (planning, TDD, code review), code-review, security-guidance | `claude plugin install` |
| **Settings patching** | Enables `ENABLE_LSP_TOOL` in `~/.claude/settings.json` | jq |

## Quick Start

```bash
git clone https://github.com/leighstillard/claude-tools.git
cd claude-tools
./claude-tools setup
```

The script auto-detects your project language from source files. If no code exists yet, it prompts you to choose.

### Options

```bash
./claude-tools setup --dry-run              # Preview what would be done
./claude-tools setup --lang python          # Force language (python|go|typescript|all)
./claude-tools setup --lang python,go       # Multiple languages
./claude-tools setup --skip-rtk             # Skip RTK installation
./claude-tools setup --skip-lsp             # Skip LSP server installation
./claude-tools setup --skip-plugins         # Skip Claude Code plugin installation
```

## Verify It Works

After setup completes, start a Claude Code session and ask:

> **"Verify my developer tools are working"**

Claude will exercise each tool (RTK, LSP, plugins) and present a status table confirming everything is accessible from within its context. This catches issues like plugins that are installed but disabled, or LSP servers that aren't on PATH.

## Prerequisites

- **jq** — required for settings patching
- **npm** — required for Python and TypeScript LSP servers
- **claude** CLI — required for plugin installation
- **cargo** — required for RTK (optional, skipped gracefully if missing)
- **go** — required for gopls (optional, skipped gracefully if missing)

## Relationship to claude.md-boilerplate

This repo is a companion to [claude.md-boilerplate](https://github.com/leighstillard/claude.md-boilerplate), which provides an opinionated `CLAUDE.md` template for production-grade engineering standards (security, testing, observability, etc.).

The boilerplate defines *what standards to follow*. This repo sets up *the tools that help you follow them* — token-efficient CLI output, semantic code navigation, and automated review workflows.

You don't need the boilerplate to use claude-tools, but they work best together.

## License

Unlicense
