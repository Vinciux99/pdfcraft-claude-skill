# PdfCraft skill for Claude

PdfCraft (ArtCraft suite, storytold/pdfcraft) is an open-source Rust PDF editor, a clean-room reimplementation of Acrobat (status: **alpha**). Every engine function is exposed as a **tool** with a `snake_case` name and a JSON-Schema argument description (`doc_open`, `page_rotate`, `text_edit`, ...). Many tools are tied to a **registry command** with a dotted id (`page.rotate`, `file.open`, ...), the same id used by menus, shortcuts and the Cmd/Ctrl+K palette.

This repository packages a **skill** that teaches Claude to drive PdfCraft (part of the [ArtCraft suite](https://github.com/storytold/pdfcraft) by storytold) and that **knows how to create the MCP server and guide you through it**, on **Windows and macOS**. It is unofficial and not affiliated with storytold.

> Part of the [ArtCraft Claude skills hub](https://github.com/Vinciux99/artcraft-claude-skills): all 13 skills, a picker and a one-command installer.

## What it does
- Operating manual for PdfCraft: connection modes, ground rules, every MCP tool, recipes, CLI, control channel, command families and a safe workflow (work on copies, verify, save only with your OK).
- Built-in MCP setup assistant: Claude checks whether the server is connected, then walks you through installing it (automatic script or manual config), the launcher, authentication and verification.
- Troubleshooting table for the usual problems (server "disconnected", ports, macOS quarantine, Windows Store config location...).

## What you can ask Claude
- "Merge these three PDFs, delete page 4 and rotate pages 1 and 3 by 90°."
- "Extract pages 2–5 into a new file and compress it for email."
- "Fill in this form with my details and save a copy."
- "Replace the logo on page 1 with this PNG and show me the result."
- "Find every occurrence of 'amount due' and highlight it with a comment."
- "OCR these scanned PDFs so they become searchable (batch the whole folder)."
- "Redact all email addresses and then verify nothing is left underneath."
- "Export this PDF to Word and to PNG images at 150 dpi."

## Highlights
- Headless MCP with a `--root` sandbox (confines every read/write), compact mode and `tool_search`.
- Index/coordinate conventions (1-based pages, top-left origin, the `text_edit` `dy` exception) spelled out.
- Recipes for text/image edits, page operations, forms, comments, redaction and export.
- Classic CLI, JSON scripts and the folder `batch` (convert / OCR / optimize).
- Built-in guidance for creating and troubleshooting the MCP server on Windows and macOS.

## App-specific notes
- No fixed port: the MCP server is **headless only** (`mcp --root <Documents>/PdfCraft`); the UI control channel writes `{port, token}` to `~/.pdfcraft-control.json`.
- Put the PDFs you want Claude to touch inside `Documents/PdfCraft` (the root), or change the root in the registered args.
- Editing existing text is fragile on real-world files: always verify with a render.

## Get the skill

| Where | How |
|---|---|
| **Claude app** (claude.ai / Desktop) | Download [`pdfcraft-skill.zip](https://github.com/Vinciux99/pdfcraft-claude-skill/releases/latest/download/pdfcraft-skill.zip)`, then Customize > Skills > "+" > Create skill > Upload a skill. Needs Settings > Capabilities > "Code execution and file creation" on. |
| **Claude Code, plugin** | `/plugin marketplace add Vinciux99/artcraft-claude-skills` then `/plugin install pdfcraft@artcraft` |
| **Claude Code, manual** | copy `skills/pdfcraft/` to `~/.claude/skills/pdfcraft/` (the setup script below does it) |

## Create the MCP server

Prerequisite: install PdfCraft from [upstream releases](https://github.com/storytold/pdfcraft/releases) (Windows `.msi`, macOS `.dmg`) and Claude Desktop and/or Claude Code.

**One command (recommended)**

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps pdfcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps pdfcraft
```
Add `-DryRun` (Windows) or `--dry-run` (macOS) to preview, and `-Mode live|headless|both` / `--mode ...` to choose the mode.

**Or from a clone:** double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time right-click > Open).

**Or just ask Claude:** once the skill is installed, say "set up the PdfCraft MCP server" and Claude will guide you step by step.

After setup: fully quit and reopen Claude, open PdfCraft with the **"PdfCraft (Claude)"** launcher, then ask Claude to use it. If the server shows "disconnected", the app was not running when Claude started: open it with the launcher and re-enable the server, or use headless mode.

What the setup changes (all backed up or reversible): Claude Desktop config (`mcpServers.pdfcraft`, backup `<file>.backup-<timestamp>`), the launcher, Claude Code's user-scope MCP list and `~/.claude/skills/pdfcraft/`.

## Repository layout
```
skills/pdfcraft/SKILL.md      the skill
app.json                   machine-readable settings (ports, paths, MCP args) used by the scripts
scripts/                   artcraft-setup.ps1 (Windows), artcraft-setup.sh (macOS), json-tool.js
setup-windows.cmd          double-click wrapper
setup-macos.command        double-click wrapper
.claude-plugin/plugin.json Claude Code plugin manifest
.github/workflows/ci.yml   validation, setup tests on Windows and macOS, release zip
```

## CI and releases
Every push validates `app.json`, `SKILL.md` and the scripts, then runs the setup scripts against a fake install on **Windows** and **macOS** runners. Every push to `main` builds `pdfcraft-skill.zip` and publishes (or updates) the GitHub release `v1.0.0`.

## Status and caveats
- Windows paths and arguments come from a working install. The macOS paths (`/Applications/PdfCraft.app`, CLI location and release asset name) follow the upstream packaging but have **not** been verified on a Mac; the skill marks unverified items and the setup script reports what it could not find.
- PdfCraft is a young open-source project; behaviour can change between releases. `list_commands` / `pdfcraft-cli commands` is always the source of truth.
- Unofficial community packaging, MIT licensed (see `LICENSE`).
