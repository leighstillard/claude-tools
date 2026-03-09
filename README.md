# claude-tools

One command to set up all the developer tools that make Claude Code faster, smarter, and cheaper to run. Without these tools, Claude burns tokens reading files to find definitions, grepping across directories, and processing verbose CLI output. With them, RTK cuts 60-90% of tokens from command output, and LSP replaces multi-file grep searches with pinpoint lookups in ~50ms — meaning Claude spends tokens writing code, not searching for it.

```bash
git clone https://github.com/leighstillard/claude-tools.git
cd claude-tools
./claude-tools setup
```

## FAQ

**What does this install?**
RTK (60-90% token savings by compressing verbose CLI output before it hits Claude's context), LSP servers (replaces expensive grep-and-read-file cycles with instant go-to-definition and find-references — one LSP call instead of reading 10 files to find where a function is defined), and Claude Code plugins (superpowers, code-review, security-guidance). It also patches `~/.claude/settings.json` to enable LSP.

**How does it know what to install?**
It scans your project for `.py`, `.go`, `.ts` files and config files like `go.mod`, `pyproject.toml`, `package.json`. If nothing is found, it asks you.

**Will it break my existing setup?**
No. Every step checks before acting — if a tool is already installed, it updates it. If a setting is already configured, it skips it. Run `--dry-run` first if you want to see what would happen.

**Does it need sudo?**
No. Everything installs to user-level locations (cargo, npm global, go install, `~/.claude/`).

**What's the relationship to claude.md-boilerplate?**
[claude.md-boilerplate](https://github.com/leighstillard/claude.md-boilerplate) defines *what standards to follow*. This repo sets up *the tools that help you follow them*. You don't need the boilerplate to use claude-tools, but they work best together.

**How do I know it worked?**
After setup, start Claude Code and ask: "Verify my developer tools are working". Claude exercises each tool and presents a status table. This catches things like plugins that are installed but disabled.

---

## What It Sets Up

| Tool | What It Does | Install Method |
|---|---|---|
| **RTK** (Rust Token Killer) | CLI proxy that compresses tool output, saving 60-90% of tokens | `cargo install rust_token_killer` |
| **LSP servers** | Semantic code understanding — go-to-definition, find-references, type info in ~50ms | pyright (npm), gopls (go install), typescript-language-server (npm) |
| **Claude Code plugins** | superpowers (planning, TDD, code review), code-review, security-guidance | `claude plugin install` |
| **Settings patching** | Enables `ENABLE_LSP_TOOL` in `~/.claude/settings.json` | jq |

## Options

```bash
./claude-tools setup --dry-run              # Preview what would be done
./claude-tools setup --lang python          # Force language (python|go|typescript|all)
./claude-tools setup --lang python,go       # Multiple languages
./claude-tools setup --skip-rtk             # Skip RTK installation
./claude-tools setup --skip-lsp             # Skip LSP server installation
./claude-tools setup --skip-plugins         # Skip Claude Code plugin installation
```

## Prerequisites

- **jq** — required for settings patching
- **npm** — required for Python and TypeScript LSP servers
- **claude** CLI — required for plugin installation
- **cargo** — required for RTK (optional, skipped gracefully if missing)
- **go** — required for gopls (optional, skipped gracefully if missing)

## License

Unlicense
