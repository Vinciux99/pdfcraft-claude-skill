---
name: pdfcraft
description: Read, edit, fill, redact, sign and export PDFs with PdfCraft (Acrobat-style, ArtCraft suite) through MCP, the CLI or the control channel, and set up, connect or troubleshoot its MCP server on Windows and macOS.
---

# PdfCraft for Claude

PdfCraft (ArtCraft suite, storytold/pdfcraft) is an open-source Rust PDF editor, a clean-room reimplementation of Acrobat (status: **alpha**). Every engine function is exposed as a **tool** with a `snake_case` name and a JSON-Schema argument description (`doc_open`, `page_rotate`, `text_edit`, ...). Many tools are tied to a **registry command** with a dotted id (`page.rotate`, `file.open`, ...), the same id used by menus, shortcuts and the Cmd/Ctrl+K palette.

There are three doors: `pdfcraft-cli run` (single call or JSON script), the opt-in **MCP server** (`pdfcraft-cli mcp`, stdio only) and the Rust API. On top of that, an opt-in **UI control channel** drives the running desktop app. The software changes fast: the source of truth is always `pdfcraft-cli tools` or `command_list`.

**Important for MCP setup:** unlike its sister apps, PdfCraft's MCP server is **headless only** — it works on its own session and does *not* drive the open window. Seeing/touching the GUI needs the control channel (`pdfcraft-cli ui`).

## Quick check: is the MCP server connected?

1. Look at the tools you have. If tools from the MCP server `pdfcraft` are available (for example `list_commands`, `execute` or `inspect_*` tools) the server is connected: skip to "Operating knowledge" below and use them directly.
2. If they are missing, check whether the user installed it under the name `pdfcraft` or `pdfcraft-headless`. In Claude Desktop the server may show as **disconnected**.
3. **Most common cause of "disconnected":** the live server (`mcp --root {documents}/PdfCraft`) exits when PdfCraft is not running at the moment Claude starts. Fix: open PdfCraft with the **"PdfCraft (Claude)" launcher** (it starts the app with its control channel on), then re-enable the server in Claude's settings or fully quit and reopen Claude.
4. If the user does not need to watch the app live, **headless mode** always starts (no window, works on files). See "Live, headless or both".
5. If nothing is configured yet, follow "Set up the MCP server" below. Ask the user which OS they use if you do not know.

## Set up the MCP server (guide the user)

Walk the user through this step by step. Ask for the OS (Windows or macOS) first. If you have a shell on the user's machine you may run the steps yourself, but **ask before editing any config file and always back it up first**. Never print tokens or keys in chat.

> **PdfCraft specifics:** the MCP server is headless only (`mcp --root <Documents>/PdfCraft`); there is no live MCP mode. The launcher still opens the desktop app with its control channel, but the control file (`~/.pdfcraft-control.json`) is used by the CLI and by scripts, not by the MCP server.

### 1. Prerequisites
- Install PdfCraft from its upstream releases page: https://github.com/storytold/pdfcraft/releases (Windows: the `.msi` installer; macOS: the `.dmg`, then drag the app to `/Applications`).
- Install Claude Desktop and/or Claude Code. For Claude Desktop, fully quit and reopen it after any config change.
- The MCP server is the **CLI** that ships with the app: `pdfcraft-cli`. Windows installs it next to the app (`C:\Program Files\PdfCraft\pdfcraft-cli.exe`, not on `PATH`). On macOS the CLI is a separate release download (asset `pdfcraft-cli-<version>-macos-universal.zip`) unless it is bundled in the app (unverified): the setup script below fetches it into `~/.artcraft/bin/` and clears the macOS quarantine flag.
- This app has no fixed control port: the control file holds the random port and token.

### 2. Option A: automatic setup (recommended)
The setup script finds the app and CLI, creates the launcher, backs up and updates Claude Desktop's config, registers the server in Claude Code (if the `claude` command exists) and installs this skill. Preview first with a dry run: nothing is changed.

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps pdfcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps pdfcraft
```
Or clone/download this repository and double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time: right-click, Open). Useful options: `-Mode live|headless|both` / `--mode ...` (default `headless`; `both` registers `pdfcraft` live and `pdfcraft-headless`), `-DryRun` / `--dry-run`, `-NoLauncher`, `-NoClaudeDesktop`, `-NoClaudeCode`, `-NoSkill` (macOS: `--no-launcher`, `--no-claude-desktop`, `--no-claude-code`, `--no-skill`).

What it changes: Claude Desktop config (`%APPDATA%\Claude\claude_desktop_config.json` plus the Microsoft Store/MSIX copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`; macOS `~/Library/Application Support/Claude/claude_desktop_config.json`; backup saved next to it as `<file>.backup-<timestamp>`), the launcher "PdfCraft (Claude)" (Windows: Desktop shortcut; macOS: `~/Applications/PdfCraft (Claude).app` plus a Desktop alias), Claude Code's user-scope MCP list, and `~/.claude/skills/pdfcraft/`.

### 3. Option B: manual setup
**Windows** - edit `%APPDATA%\Claude\claude_desktop_config.json` (if Claude was installed from the Microsoft Store, also the copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`). Keep other servers; add this inside `mcpServers`. Backslashes must be doubled in JSON.
```json
{
  "mcpServers": {
    "pdfcraft": {
      "command": "C:\\Program Files\\PdfCraft\\pdfcraft-cli.exe",
      "args": [
        "mcp",
        "--root",
        "C:\\Users\\YOUR_NAME\\Documents\\PdfCraft"
      ]
    }
  }
}
```
**macOS** - edit `~/Library/Application Support/Claude/claude_desktop_config.json`. Use the absolute CLI path (replace `YOUR_NAME`; JSON does not expand `~`). The CLI path is unverified if your install differs: check with `ls ~/.artcraft/bin /Applications/PdfCraft.app/Contents/MacOS`.
```json
{
  "mcpServers": {
    "pdfcraft": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/pdfcraft-cli",
      "args": [
        "mcp",
        "--root",
        "/Users/YOUR_NAME/Documents/PdfCraft"
      ]
    }
  }
}
```
**Launcher** (starts the app with the control channel; needed before the live server can connect):
- Windows: create a Desktop shortcut named "PdfCraft (Claude)" with target `"C:\Program Files\PdfCraft\pdfcraft.exe" --control "C:\Users\YOUR_NAME\.pdfcraft-control.json"`
- macOS: run `open -n -a "/Applications/PdfCraft.app" --args --control "/Users/YOUR_NAME/.pdfcraft-control.json"`, or save it as an app with Script Editor/`osacompile` (the setup script does this for you).

**Claude Code** (any OS; run once):
Windows (PowerShell):
```powershell
claude mcp add --scope user pdfcraft -- "C:\Program Files\PdfCraft\pdfcraft-cli.exe" mcp --root C:\Users\YOUR_NAME\Documents\PdfCraft
```
macOS (Terminal):
```sh
claude mcp add --scope user pdfcraft -- "$HOME/.artcraft/bin/pdfcraft-cli" mcp --root /Users/YOUR_NAME/Documents/PdfCraft
```
Check with `claude mcp list`.

### 4. Authentication
The app picks a random control port and writes `{port, token}` to a control file (`~/.pdfcraft-control.json`). The MCP server itself runs **headless only** (it does not use that channel). Never print the token in chat.

### 5. Finish and verify
1. **Fully quit and reopen Claude** (Desktop: quit from the tray/menu bar, not just close the window).
2. Open PdfCraft with the **"PdfCraft (Claude)" launcher** before asking Claude to use it.
3. In Claude, confirm the server `pdfcraft` is connected and call its `list_commands` tool (or equivalent). A non-empty command list means it works.

### Live, headless or both
- **Live** (`pdfcraft`): drives the open app; the user sees every change and screenshots/UI tools work. Needs the launcher first.
- **Headless** (`pdfcraft-headless`): in-process engine, no window; always starts; works on files; UI/screenshot tools are not available.
- **Both**: register both and pick per task. Switch with `-Mode` / `--mode`, or by editing the config entries above.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Server shows "disconnected" right after Claude starts | Live server exits when PdfCraft is not running | Open PdfCraft with the "PdfCraft (Claude)" launcher, then re-enable the server or restart Claude; or use headless mode |
| "Port already in use" / app opens but Claude cannot connect | Another app or an old PdfCraft instance holds port (control file) | Quit all PdfCraft windows, check the launcher uses the kit port, make sure no other ArtCraft app uses the same port |
| `command not found` / spawn error | CLI path wrong or the CLI is not installed (Windows: not on `PATH`; macOS: separate download) | Use the full absolute path in the config; macOS: run the setup script (it downloads the CLI to `~/.artcraft/bin/`) |
| macOS: "cannot be opened because the developer cannot be verified" | Gatekeeper/quarantine on the app or CLI | Right-click the app, Open; for the CLI: `xattr -dr com.apple.quarantine ~/.artcraft/bin/pdfcraft-cli` |
| `unauthorized` / token or key errors | Auth file missing or stale (control-file) | Restart the app with the launcher so it writes a fresh file; never paste the secret in chat |
| Config edited but nothing changed (Windows) | Claude installed from the Microsoft Store reads the MSIX copy of the config | Edit/update `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\claude_desktop_config.json` too |
| JSON config error / server not listed | Unescaped backslashes or trailing comma | Double the backslashes on Windows; validate the JSON; restore the `.backup-*` file if needed |
| App crashes on start or window is blank | Graphics backend problem | Windows: set `WGPU_BACKEND=vulkan` (or `dx12`) for one launch; avoid `gl` on some AMD drivers |
| Changes do not appear in the window | You are on the headless server | Use the live server (launcher first) |
| Quotes disappear in PowerShell JSON arguments | Windows PowerShell 5.1 quoting for `.exe` | Use `--script`/files where the CLI supports them, or single-quote the whole JSON |

## Operating knowledge

## Connecting

| Mode | Start | Transport | Notes |
|---|---|---|---|
| **MCP headless** | `pdfcraft-cli mcp [--root DIR] [--compact]` | JSON-RPC 2.0 lines on stdin/stdout | No port open. Stops on EOF on stdin. Protocols 2025-06-18, 2025-03-26, 2024-11-05 |
| **CLI one-shot / script** | `pdfcraft-cli run <tool> k=v ...` or `run --script steps.json` | Single process | Each invocation is a **new session**: `doc` ids do not survive between two `run`s |
| **Classic CLI** | `info`, `render`, `text`, `edit`, `combine`, `extract`, `split`, `check`, `audit`, `batch` | Single process | See CLI section |
| **UI control channel** | `pdfcraft --control FILE` (app) + `pdfcraft-cli ui --control FILE <method>` | TCP loopback only (127.0.0.1), token | Drives the real interface. See its section |

### MCP server flags
- `--root DIR` confines **every** read and write to DIR, including symlinks and `..`. Give it once; it must be a folder path. If a folder name starts with `--`, write it as `./--name`. Relative paths resolve inside the root. A path outside the root always gets the same error, `<path> is outside the allowed directory <root>`. On Windows, `\\host\share` shares and `\\?\`/`\\.\` paths are rejected.
- `--compact` shrinks `tools/list` to a core: `doc_open`, `doc_info`, `doc_save`, `doc_close`, `page_render`, `text_extract`, `text_find`, `doc_combine`, `doc_split`, `edit_undo`, plus the 5 common tools (`command_list`, `command_run`, `command_batch`, `doc_inspect`, `render_preview`) and 2 meta-tools. `tool_search` accepts `query`, `category` (name prefix) or `name` to get the full schema; `tool_call` accepts `name` and `arguments`. The other tools remain callable by name.
- There is no authentication: the server is a child process of the agent and does not listen on the network.
- The kit's setup uses `--root <Documents>/PdfCraft` (a dedicated folder; the setup script creates it). Keep copies of PDFs to process inside it. Add or remove `--compact` in the registered args to taste.

### Choosing the channel
1. **MCP tools `pdfcraft` available** → use them; main channel.
2. **A shell on the user's machine** → `pdfcraft-cli run --script ...` or the classic subcommands.
3. **App open and the user should see/touch it** → control channel (`pdfcraft-cli ui`), only if the app was started with `--control` (the "PdfCraft (Claude)" launcher does that).
4. **A cloud-sandbox shell** is another machine: it cannot run the user's app or reach its ports. Use it only to prepare files (JSON scripts, images, texts). A possible exception: the Linux tarball `pdfcraft-cli-<ver>-linux-x86_64.tar.gz` runs inside a Linux sandbox, but needs downloading and the user's consent.
5. **Fallback:** computer use on the PdfCraft GUI (slow).

Paths: Windows log `%APPDATA%\PdfCraft\data\logs\pdfcraft.log` (or `PdfCraftData\` next to the exe when portable); macOS `~/Library/Application Support/PdfCraft/` (unverified). Version: `pdfcraft-cli --version`; help `--help` (also after a subcommand, e.g. `render --help`).

## Ground rules
- **Pages are 1-based** everywhere in the tools (`page`, `pages`, `at`, `to`). Indices returned by tools (paragraphs, lines, images, comments, links, added content, `object_list`) also start at 1. (In the Rust engine API `ObjectTarget` is 0-based, but that concerns neither MCP nor the CLI.)
- **Coordinates:** PDF points (1/72 in) with the **origin at the top-left of the displayed page**, x right, y down. Point `[x, y]`; rectangle `[x0, y0, x1, y1]`. Rotation, crop box and `/UserUnit` are already accounted for.
  - **Exception:** in `text_edit`, positive `dy` moves the paragraph **up** (positive `dx` right). In `object_move`, `offset [dx, dy]` uses y down.
  - In `content_update`, `rotate` is in degrees **counter-clockwise**. In `page_rotate`, positive `degrees` is **clockwise** (multiples of 90).
- **Documents:** `doc_open` returns an integer id `doc`, to pass to every tool. `doc_list` and `doc_inspect` (without `doc`) list the open ones. `render_preview` may omit `doc` only when exactly one document is open.
- **Object references** always come from a list tool before editing; after every edit, undo, redo or save, **re-list**.
  - Existing text: `text_paragraphs` (→ `paragraph`) and `text_lines` (→ `line`).
  - Images and figures: `page_images` (→ `image`, `kind: "image"|"form"`).
  - Object groups: `object_list` (→ `{kind: added|text|image, index}` + `generation`). Pass `generation` to `object_move` so stale references are rejected.
  - Content added by PdfCraft: `content_list` (→ `index` for `content_update`/`content_delete`).
  - Comments: `comment_list` (→ `id`, or `page` + `index`). Links: `link_list`. Form fields: `form_fields` (by name).
- **Undo:** every change stays in memory and is undoable (`edit_undo`/`edit_redo`), even after saving. Mutating tools return a summary (`doc`, `pages`, `dirty`, `undo`, `redo`, ...).
- **Saving (`doc_save`):** without `path` it saves **in place with an incremental update** (original bytes intact, valid signatures kept); with `path` it does a **full rewrite** to a new file; `full: true/false` forces the mode. Writes are atomic (temp file then replace). `doc_close` refuses to close a document with unsaved changes unless `discard_changes: true`. Some operations force a full rewrite: redaction, PDF/A, password change.
- **Arguments:** unknown keys are **rejected** (MCP: JSON-RPC `-32602` listing accepted keys), so typos show up at once. Exception: `command_run` ignores extra keys in `params` and reports them in `warnings`. Failures come back as `isError: true` and the server stays up.
- **Known traps:**
  - Editing existing text is **fragile** on real files (the docs list it among the main gaps), especially CJK and right-to-left text. Always check with a render.
  - Subset fonts contain no new glyphs. If you need new characters use `text_font_files` → `text_font_import` → `text_fonts`, then `text_edit.font_resource`.
  - `page_add_text` and `doc_create` (`text`) accept only WinAnsi characters (Western European; accented Latin letters are fine).
  - `image_edit` with `replace` and `image_save` work only on raster images, not Form XObjects (`kind: "form"`). Inline images (`BI/ID/EI`) are not editable.
  - `object_move`: max 1000 objects, all on the same page. Vertical text, clipping, group resize and alignment are unsupported. A rejection changes nothing.
  - Signed documents cannot be optimized with `batch`/`doc_batch`, and encrypted PDFs that need a password are unsupported by `batch`.
  - Exports are synchronous: no progress, no cancellation.

## MCP tools

**Common tools:**

| Tool | Arguments | Notes |
|---|---|---|
| `command_list` | `doc?`, `filter?`, `enabled_only?` | `{commands:[...]}` with the schema of the tool tied to each command |
| `command_run` | `id`, `params?` | Runs a registry command via its tool. `params` includes `doc`. A command with no headless tool errors |
| `command_batch` | `steps:[{id, params?}]`, `stop_on_error?` (default true) | Returns `completed`, `failed` and `{ok, result?, error?}` per step. Earlier changes before the error stay |
| `doc_inspect` | `doc?` | Info of one document, or `{documents:[...]}` |
| `render_preview` | `doc?`, `page?` (default 1), `max_side?` (1–4096, default 1024) | PNG |

**Main working tools** (full list: `pdfcraft-cli tools` or `tool_search`; 160+ in the source):

| Tool | Required → optional arguments | Notes |
|---|---|---|
| `doc_open` | `path` → `password` | Returns `doc`, pages, whether editable |
| `doc_info` | `doc` | Metadata, pages, bookmarks, fields, fonts, security... (read-only) |
| `doc_save` | `doc` → `path`, `full`, `flatten_fill_sign` | Destructive if overwriting |
| `doc_close` | `doc` → `discard_changes` | |
| `doc_set_info` | `doc`, `key`, `value` | Title/Author/Subject/Keywords (also updates XMP) |
| `page_render` | `doc`, `page` → `dpi` (default 96, max 600), `hide_text_block` | PNG |
| `text_extract` | `doc` → `pages`, `rect` | Reading order |
| `text_find` | `doc`, `query` → `limit` (500), `start_page`, `skip` | Follow `next` if present; check `warnings` |
| `page_rotate` | `doc`, `degrees` → `pages`, `subset` (all/even/odd), `orientation` | |
| `page_delete` / `page_move` / `page_duplicate` | `doc`, `pages` (+ `to` for move) | |
| `page_insert_blank` / `page_insert_file` / `page_replace` | `doc`, `at` (+`path`) / `doc`,`pages`,`path` | |
| `page_extract` | `doc`, `pages` → `out`, `open`, `separate`, `out_dir`, `delete` | |
| `doc_combine` | `paths` → `pages`, `passwords`, `groups`, `field_conflicts`, `out`, `open` | Also accepts images and .txt |
| `doc_split` | `doc`, `out_dir` → `every`, `before`, `bookmarks`, `max_mb` | Files `<name>-partK.pdf` |
| `doc_create` | `from` (blank/images/text) → `paths`, `text`, `pages`, `width`, `height`, `dpi`... | |
| `text_paragraphs` / `text_lines` | `doc`, `page` | Numbers for `text_edit` |
| `text_edit` | `doc`, `page` + (`paragraph` or `line`) → `text`, `font`, `bold`, `italic`, `size`, `color`, `align`, `underline`, `line_spacing`, `dx`, `dy`, `width`, `char_spacing`, `scale`, `font_resource`, `ranges` | `ranges`: up to 1024 disjoint UTF-8 byte ranges |
| `text_edit_layout` | `doc`, `page`, `paragraph` → `text`, `size`, `width`, `font_resource`, `ranges` | Measures a draft without applying it |
| `text_fonts` / `text_font_files` / `text_font_import` | `doc`,`page` / `directories?` / `doc`,`page`,`path` | Fonts for existing text |
| `page_images` / `image_hit` | `doc`, `page` (+ `point`) | `rect` is the full placement, `visible_rect` the visible part |
| `image_edit` | `doc`, `page`, `image`, `action` (move/rotate/flip_horizontal/flip_vertical/replace/delete) → `rect`, `quarters`, `path` | |
| `image_save` | `doc`, `page`, `image`, `path` | Raster only |
| `object_list` / `object_move` | `doc`, `page` / + `objects`, `offset` → `generation` | Atomic group move |
| `page_add_text` | `doc`, `page`, `text` → `at`, `width`, `rect`, `font`, `size`, `bold`, `italic`, `color`, `align` | Page content, not a comment |
| `page_add_image` | `doc`, `page`, `path` → `rect` | PNG, JPEG, TIFF, GIF, BMP |
| `content_list` / `content_update` / `content_delete` | `doc` / `doc`,`page`,`index` ... | Only elements added by PdfCraft |
| `form_fields` / `form_fill` / `form_reset` | `doc` / `doc`,`values` / `doc`→`fields` | `values`: name → string, boolean or array |
| `form_add_field` / `form_set_props` / `form_delete_field` | see schema | Form preparation |
| `comment_list` / `comment_add` / `comment_edit` / `comment_reply` / `comment_delete` | `doc`, `page`, `type`... | `comment_add` with `find` highlights a phrase by searching for it |
| `redact_mark` → `redact_apply` | `doc` → `find`, `rect`, `pattern`, `pages`... | `apply` is destructive |
| `doc_protect` / `doc_unprotect` | `doc` → `open_password`, `permissions_password`, ... | |
| `doc_export_images` / `doc_export_text` / `doc_export_office` | `doc`, `folder` / `path` | Office by extension: .docx, .pptx, .html, .rtf |
| `doc_reduce` / `doc_optimize` / `doc_batch` | `doc`, `path` / `paths`, `out_dir`, `steps` | |
| `ocr_recognize` / `ocr_status` | `doc` → `pages`, `dpi`, `language`, `skip_text_pages` | Need the OCR models installed |
| `edit_undo` / `edit_redo` | `doc` | |

Resources: `pdfcraft://document` (session info) and `pdfcraft://commands` (command catalog). For each open document: `pdfcraft://doc/{doc}/info`, `.../text`, `.../page/{page}/text`, `.../page/{page}/image{?dpi}` (PNG, 1–600 dpi).

## Recipes (MCP; via CLI use the same objects in a script's steps)
```text
# Open, orient, look
doc_open        {"path":"input.pdf"}                       → {"doc":1,...}
doc_info        {"doc":1}
render_preview  {"doc":1,"page":1,"max_side":1024}

# Registry command
command_list    {"doc":1,"filter":"rotate"}
command_run     {"id":"page.rotate","params":{"doc":1,"degrees":90}}

# Edit existing text (paragraph)
text_paragraphs {"doc":1,"page":2}                         → pick "number"
text_edit       {"doc":1,"page":2,"paragraph":3,"text":"New text"}
text_edit       {"doc":1,"page":2,"paragraph":3,"size":12,"color":"#1a1a1a","dx":0,"dy":-10}

# Replace / move an image
page_images     {"doc":1,"page":1}                         → number, kind, rect
image_edit      {"doc":1,"page":1,"image":1,"action":"replace","path":"new-logo.png"}
image_edit      {"doc":1,"page":1,"image":1,"action":"move","rect":[72,72,272,172]}

# Move several objects together
object_list     {"doc":1,"page":1}                         → generation, items
object_move     {"doc":1,"page":1,"generation":0,"objects":[{"kind":"text","index":1},{"kind":"image","index":1}],"offset":[24,18]}

# Add text and an image
page_add_text   {"doc":1,"page":1,"text":"DRAFT","at":[72,40],"size":18,"bold":true,"color":"red"}
page_add_image  {"doc":1,"page":1,"path":"signature.png","rect":[400,700,540,760]}

# Pages
page_rotate     {"doc":1,"pages":[1,3],"degrees":90}
page_move       {"doc":1,"pages":[5],"to":1}
page_delete     {"doc":1,"pages":[4]}
doc_combine     {"paths":["a.pdf","b.pdf"],"out":"merged.pdf"}
page_extract    {"doc":1,"pages":[1,3,5],"out":"extract.pdf"}
doc_split       {"doc":1,"every":10,"out_dir":"parts"}

# Forms
form_fields     {"doc":1}
form_fill       {"doc":1,"values":{"name":"Jane Doe","privacy":true}}

# Comments and export
comment_add     {"doc":1,"page":1,"type":"highlight","find":"amount due"}
doc_export_images {"doc":1,"folder":"png","format":"png","dpi":150}
doc_export_office {"doc":1,"path":"document.docx"}

# Save to a NEW file (never overwrite the original without consent)
doc_save        {"doc":1,"path":"input-edited.pdf"}
```
Take example values (`generation`, indices, rectangles) from the real responses for the document at hand. Field names (`name`, `privacy`) are hypothetical.

## CLI / batch
On Windows replace `pdfcraft-cli` with `& "C:\Program Files\PdfCraft\pdfcraft-cli.exe"` in PowerShell.
```sh
pdfcraft-cli info   file.pdf [--password PW]                 # structure JSON
pdfcraft-cli render file.pdf --page N [--dpi 96] --out x.png # .png .jpg .tif .pam
pdfcraft-cli text   file.pdf [--page N]                      # text, pages separated by form feed
pdfcraft-cli edit   in.pdf --out out.pdf [--rotate 1,3:90] [--delete 2,4] [--move 5:1] [--insert-blank 1] [--title T] [--author A] [--full]
pdfcraft-cli combine a.pdf b.pdf --out combined.pdf
pdfcraft-cli extract in.pdf --pages 1,3,5 --out out.pdf
pdfcraft-cli split   in.pdf (--every N | --before 3,7) [--out-dir DIR]
pdfcraft-cli check   <file or folders> [--timeout 20] [--dpi 36] [--json out.json]
pdfcraft-cli audit   file.pdf                                # fake redactions (fails if any found)
pdfcraft-cli tools                                           # all tools + JSON Schema
pdfcraft-cli run <tool> [key=value ...] [--root DIR] [--out image.png]   # values parsed as JSON
pdfcraft-cli run --script steps.json [--root DIR]
```
- `edit` applies options **in order**, and page numbers refer to the document after the previous options.
- **Script:** a JSON array `[{"tool":"doc_open","args":{"path":"in.pdf"}}, {"tool":"page_render","args":{"doc":1,"page":1},"out":"p1.png"}, ...]`. A script is **one session**: `doc` ids only hold inside it, so always include `doc_open` in the script itself. With `--root` every step's reads and writes, including the `"out"` PNGs, stay inside the root. The script file itself may live elsewhere.
- In PowerShell, quoting `key=value` with JSON is awkward: prefer a script file.

**Batch** (convert, OCR, compress files and folders; never modifies the sources):
```sh
pdfcraft-cli batch scans  --root . --recursive --steps optimize --out-dir compressed
pdfcraft-cli batch inputs --root . --recursive --steps convert,optimize --out-dir pdfs --ppi 150 --quality 80
pdfcraft-cli batch scans  --root . --recursive --steps ocr,optimize --out-dir searchable
```
- **Steps:** `convert` (if present, first), `ocr` and `optimize`, each at most once.
- **Options:** `--ppi` 9–1600 (threshold is 1.5× the value), `--quality` 1–100, `--dpi` 72–600 (default 300), `--language` (default `en`), `--ocr-all-pages`, `--stop-on-error`.
- **Behavior:** existing files are never replaced (outputs become `book (2).pdf`, ...); the JSON report goes to stdout (`completed`, `failed`, `skipped`, `stopped`, `files`) and the process exits non-zero if a file fails. Limits: 1000 paths, 10,000 entries, 64 levels, 512 MiB per file.
- MCP/script equivalent: `doc_batch {"paths":["inputs"],"steps":["convert","optimize"],"recursive":true,"out_dir":"pdfs","ppi":150,"quality":80,"stop_on_error":false}`. `skip_text_pages: false` equals `--ocr-all-pages`.

## UI control channel (running desktop app)
It **exists**, but stays off until the app is started with `--control FILE`:
- Windows: `& "C:\Program Files\PdfCraft\pdfcraft.exe" --control "$env:USERPROFILE\.pdfcraft-control.json" "C:\path\doc.pdf"`
- macOS: `open -n -a /Applications/PdfCraft.app --args --control "$HOME/.pdfcraft-control.json"` (unverified)

- The app listens **only on 127.0.0.1**, on a **port chosen at startup**, and writes `{port, token}` to the control file. The token is random and new at every start. No port is fixed: read it from the file.
- Keep the file in a folder only you can write (your user profile, not a shared folder). `pdfcraft-cli ui` rejects symlinks. If the app cannot write the file it does not start. Never show the token in chat.
- **Protocol** (from the CLI client): JSON-RPC 2.0, one message per line, over TCP. First `{"method":"auth","params":{"token":...}}`, then the `ui.*` method. Easiest is the client:
```sh
pdfcraft-cli ui --control FILE state
pdfcraft-cli ui --control FILE inspect query=rotate          # widgets with labels and positions
pdfcraft-cli ui --control FILE click label="Organize pages"
pdfcraft-cli ui --control FILE key key=K modifiers='["command"]'
pdfcraft-cli ui --control FILE command id=comment.square     # picks the tool, then:
pdfcraft-cli ui --control FILE drag from='[400,300]' to='[600,420]'
pdfcraft-cli ui --control FILE screenshot --out window.png
pdfcraft-cli ui --control FILE resize width=1200 height=800  # 320–8192 logical points
pdfcraft-cli ui --control FILE focus label="Read"
pdfcraft-cli ui --control FILE inspect focused=true
pdfcraft-cli ui --control FILE set key=language value=en
pdfcraft-cli ui --control FILE quit                          # with unsaved changes it asks instead of discarding
```
- **Methods listed by the client:** `state`, `inspect`, `click`, `drag`, `type`, `key`, `command`, `commands`, `set`, `open`, `screenshot`.
- **Documented in the README:** `resize`, `focus`, `quit`, `ui.dock` (operations `open`/`close`/`activate`/`float`/`move`... on the `tools` and `inspector` panels) and `ui.dock.reset`.
- Parameters of `type` and `open` are not documented: check with `--help` or by trial on a test file.

## Commands by family (registry ids)
Only commands that have a linked tool work with `command_run`. Use `command_list {"filter":"..."}` to see which and with what parameters. All ids are valid with `ui command id=...` on the app.

- `file.*`: open, open_recent, clear_recent, close, close_all, save, save_as, revert, properties, pin_folder
- `page.*`: organize, rotate, rotate_ccw, rotate_dialog, delete, duplicate, extract, insert, insert_blank, replace, combine, split, boxes, crop, number, copy, cut, paste
- `edit.*`: undo, redo, find, advanced_search, column_select, edit_text, text, image, link, links_from_urls, remove_links, snapshot, header_footer(.update/.remove), watermark(.update/.remove), background(.update/.remove), bates
- `comment.*`: note, highlight, underline, strikeout, squiggly, replace, caret, freetext, callout, square, circle, line, arrow, polygon, polyline, cloud, ink, eraser, stamp, attach, list, summarize, export, import, flatten, hide_all
- `form.*`: prepare, fields, detect, clear, flatten, export_data, import_data, merge_data, field.properties, tab_order.row/.column/.structure, add.text/.checkbox/.radio/.combo/.list/.button/.date/.image/.signature
- `sign.*`: digital, certify, certify_invisible, validate, panel, fill.text/.check/.cross/.dot/.line/.date/.signature/.initials (+ .change/.remove)
- `protect.*`: password, properties, remove, unlock, remove_hidden · `redact.*`: mark, search, pages, apply, clear, properties, sanitize
- `create.*`: blank, file, images, multiple, multiple_separate, clipboard · `export.*`: docx, pptx, rtf, html, text, image, all_images
- `ocr.*`: recognize, recognize_batch, enhance_camera · `optimize.*`: reduce, advanced · `standards.pdfa` · `doc.compare`
- `measure.*`: distance, perimeter, area, info, scale, snap, export · `a11y.*`: check, report, tags, alt_text, autotag, reading_options
- `actions.*`: wizard, optimize_scans, distribution · `tools.*`: js_console, document_js · `bookmark.*`: add, from_structure
- `view.*` (GUI only): continuous, layout.single/.two_up/.cover, fit_*, read_mode, full_screen, palette, theme(.light/.dark/.system), next_tab/previous_tab, ... · `app.preferences`, `help.*`, `print.dialog`, `ui.dock.reset`

## Recommended workflow
1. **Orient:** choose the channel. Copy PDFs into the `--root` folder and always work on copies.
2. **Open and inspect:** `doc_open` → `doc_info`/`doc_inspect` → `render_preview` of the relevant page to see it before touching it.
3. **Find the tool:** `command_list {"filter":"..."}` or `tool_search` (compact mode). For text, images and objects list the references first (`text_paragraphs`, `page_images`, `object_list`, ...).
4. **Edit** in small atomic steps and re-list after each change. For long sequences use `command_batch` (MCP) or `run --script` (CLI). If something goes wrong, `edit_undo`.
5. **Verify:** `render_preview`/`page_render` of the touched pages, plus `text_find`/`text_extract` if you changed text. Show the preview to the user when it matters.
6. **Save** only when the user agrees, with `doc_save` to a **new** `path` (e.g. `name-edited.pdf`). Never overwrite the original and never use `discard_changes: true` without explicit consent. Finally `doc_close`.
