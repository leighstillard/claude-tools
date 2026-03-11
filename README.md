# claude-tools

One command to set up all the developer tools that make Claude Code faster, smarter, and cheaper to run. Without these tools, Claude burns tokens reading files to find definitions, grepping across directories, and processing verbose CLI output. With them, RTK cuts 60-90% of tokens from command output, and LSP replaces multi-file grep searches with pinpoint lookups in ~50ms — meaning Claude spends tokens writing code, not searching for it.

```bash
git clone https://github.com/leighstillard/claude-tools.git
cd claude-tools
./claude-tools setup
```

## Example run and impact

```
leigh@codex:~/workspace/model-profiler$ ../claude-tools/claude-tools setup --lang python,go
claude-tools v0.1.0 — setup

Languages (forced): python go

=== Prerequisites ===
  OK: jq
  OK: npm
  OK: curl
  OK: go
  OK: claude

=== RTK (Rust Token Killer) ===
  FOUND: rtk v0.27.2
  Updating...
[INFO] Installing rtk...
[INFO] Detected: linux x86_64
[INFO] Target: x86_64-unknown-linux-musl
[INFO] Version: v0.28.2
[INFO] Downloading from: https://github.com/rtk-ai/rtk/releases/download/v0.28.2/rtk-x86_64-unknown-linux-musl.tar.gz
[INFO] Extracting...
[INFO] Successfully installed rtk to /home/leigh/.local/bin/rtk
[INFO] Verification: rtk 0.28.2

[INFO] Installation complete! Run 'rtk --help' to get started.
  CONFIGURED: hook is set up
  DONE

=== LSP Servers ===
  [Python] pyright
    FOUND: v1.1.408
    Updating...
npm ERR!     /home/leigh/.npm/_logs/2026-03-11T12_37_11_033Z-debug-0.log
  [Go] gopls
    Installing...
go: golang.org/x/tools/gopls@v0.21.1 requires go >= 1.25; switching to go1.25.8
  DONE

=== Claude Code Settings ===
  ENABLE_LSP_TOOL: already enabled
  DONE

=== Claude Code Plugins ===
  Adding superpowers marketplace...
Adding marketplace...
SSH not configured, cloning via HTTPS: https://github.com/pcvelz/superpowers.git
Refreshing marketplace cache (timeout: 120s)…
Cloning repository (timeout: 120s): https://github.com/pcvelz/superpowers.git
Clone complete, validating marketplace…
Cleaning up old marketplace cache…
✔ Successfully added marketplace: superpowers-extended-cc-marketplace (declared in user settings)
  FOUND: pyright-lsp
  FOUND: gopls-lsp
  INSTALLING: superpowers-extended-cc...
    Installing plugin "superpowers-extended-cc@superpowers-extended-cc-marketplace"...
    ✔ Successfully installed plugin: superpowers-extended-cc@superpowers-extended-cc-marketplace (scope: user)
  FOUND: code-review
  INSTALLING: security-guidance...
    Installing plugin "security-guidance@claude-plugins-official"...
    ✔ Successfully installed plugin: security-guidance@claude-plugins-official (scope: user)
  DONE

=== Engineering Standards (claude.md-boilerplate) ===

  claude.md-boilerplate is a companion template that gives Claude Code
  persistent engineering standards — security rules, tenant isolation,
  structured logging, test coverage requirements, and MCP safety guardrails.

  It adds a CLAUDE.md file (~75 lines, always loaded) and detailed pillar
  files that Claude reads on demand. Without it, Claude can write working
  code that skips the non-functional requirements that matter in production.

  Learn more: https://github.com/leighstillard/claude.md-boilerplate

  Install engineering standards into this project? (y/N): y
  Cloning boilerplate...
  INSTALLED: CLAUDE.md + docs/standards/

========================================
  Setup complete!
========================================

Start a Claude Code session and run this prompt to tailor the
standards to your project, then verify your tools:

  Review CLAUDE.md and all files in docs/standards/. For each pillar file:
  1. Assess whether this pillar is relevant to our project: [describe your
     project]. Remove any that are not relevant and update CLAUDE.md.
  2. Review the compliance references (GDPR, PCI-DSS, APRA, SOX, etc.) and
     remove any that do not apply to our use case.
  3. Check all technical recommendations against current best practices.
     Flag anything outdated, deprecated, or superseded. Update as needed.
  4. Summarise what you changed and why.
  5. Then verify my developer tools are working.

leigh@codex:~/workspace/model-profiler$ rtk gain
RTK Token Savings (Global Scope)
════════════════════════════════════════════════════════════

Total commands:    20
Input tokens:      3.1K
Output tokens:     1.5K
Tokens saved:      1.7K (53.9%)
Total exec time:   251ms (avg 12ms)
Efficiency meter: █████████████░░░░░░░░░░░ 53.9%

By Command
───────────────────────────────────────────────────────────────────────
  #  Command                   Count  Saved    Avg%    Time  Impact
───────────────────────────────────────────────────────────────────────
 1.  rtk git diff HEAD~3 -...      1    682   47.9%    13ms  ██████████
 2.  rtk ls                        2    643   66.2%     9ms  █████████░
 3.  rtk git status                3    151   77.0%    18ms  ██░░░░░░░░
 4.  rtk git commit                3    111   94.7%    17ms  ██░░░░░░░░
 5.  rtk git diff HEAD~3 -...      1     63   46.7%    14ms  █░░░░░░░░░
 6.  rtk git log --oneline -5      2      6    3.6%     4ms  ░░░░░░░░░░
 7.  rtk git log --oneline...      1      3    5.3%    15ms  ░░░░░░░░░░
 8.  rtk git rev-parse --s...      3      0    0.0%     4ms  ░░░░░░░░░░
 9.  rtk git add slurpy/tr...      1      0    0.0%    19ms  ░░░░░░░░░░
10.  rtk git add slurpy/tr...      1      0    0.0%    17ms  ░░░░░░░░░░
───────────────────────────────────────────────────────────────────────

leigh@codex:~/workspace/model-profiler$
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
[claude.md-boilerplate](https://github.com/leighstillard/claude.md-boilerplate) defines *what standards to follow*. This repo sets up *the tools that help you follow them*. During setup, you'll be offered the option to install the boilerplate's engineering standards (CLAUDE.md + pillar files) into your project. If you accept, the script gives you a tailoring prompt to run in Claude Code so you can strip out anything that doesn't apply.

**How do I know it worked?**
After setup, start Claude Code and ask: "Verify my developer tools are working". Claude exercises each tool and presents a status table. This catches things like plugins that are installed but disabled.

---

## What It Sets Up

| Tool | What It Does | Install Method |
|---|---|---|
| **RTK** (Rust Token Killer) | CLI proxy that compresses tool output, saving 60-90% of tokens | `curl -fsSL .../rtk/install.sh \| sh` |
| **LSP servers** | Semantic code understanding — go-to-definition, find-references, type info in ~50ms | pyright (npm), gopls (go install), typescript-language-server (npm) |
| **Claude Code plugins** | superpowers (planning, TDD, code review), code-review, security-guidance | `claude plugin install` |
| **Settings patching** | Enables `ENABLE_LSP_TOOL` in `~/.claude/settings.json` | jq |

## Installing Into Another Project

RTK, LSP servers, plugins, and settings are global — they only need to be installed once. To add the engineering standards (CLAUDE.md + pillar files) to a different project, run setup from that project's directory:

```bash
cd /path/to/your-project
~/claude-tools/claude-tools setup --skip-rtk --skip-lsp --skip-plugins
```

This skips the global tools (already installed) and just offers to install the boilerplate into the current directory.

## Options

```bash
./claude-tools setup --dry-run              # Preview what would be done
./claude-tools setup --lang python          # Force language (python|go|typescript|all)
./claude-tools setup --lang python,go       # Multiple languages
./claude-tools setup --skip-rtk             # Skip RTK installation
./claude-tools setup --skip-lsp             # Skip LSP server installation
./claude-tools setup --skip-plugins         # Skip Claude Code plugin installation
./claude-tools setup --skip-boilerplate     # Skip the engineering standards offer
```

## Prerequisites

- **jq** — required for settings patching
- **npm** — required for Python and TypeScript LSP servers
- **claude** CLI — required for plugin installation
- **curl** — required for RTK install (optional, skipped gracefully if missing)
- **go** — required for gopls (optional, skipped gracefully if missing)

## License

Unlicense
