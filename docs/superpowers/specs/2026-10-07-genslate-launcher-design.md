# GENSLATE Launcher — Master Design Spec

Status: approved in brainstorming, awaiting written-spec review · Date: 2026-10-07 · Path: architectural

This is the master spec for the GENSLATE-LAUNCHER monorepo: its foundation, the shared Nord design system
with its Design Kit web app, and the portable launcher. It is implemented through **three plans**
(Foundation → Design system → Launcher), each written just before it runs (§12).

## 1. Intent

Build a **modern, portable "operating system on a stick" for Windows**, in the spirit of the
PortableApps.com Platform, with GENSLATE's own design and capabilities. A user carries one drive (USB
stick or portable SSD) with their apps and documents and uses it on any Windows 10/11 PC.

- **Focus order:** GENSLATE apps first; PortableApps.com and portapps.io apps second.
- **Foundation for more apps:** the monorepo and the design system are built so later GENSLATE apps reuse
  them unchanged. Only the launcher is built now.
- **Look and feel:** identical to the design system and launcher in `S:\DEVELOPMENT\PROJECTS\GENSLATE-SLATESUITE`
  (reference screenshots: Design Kit "Colors" and "Button" pages, and SLATESUITE's
  `other/resources/screenshots/`). Theme: **official Nord — Polar Night = dark, Snow Storm = light.**
- **Clean rebuild.** SLATESUITE is the visual and behavioral **reference only**; no files are copied. Every
  piece is written fresh here, at the latest versions, and audited as it is built.
- **Latest everything, modern practice.** Every dependency and tool is the latest stable release, and every
  API pattern is confirmed against current official docs before use (§10).

### Success criteria

- `bun run design-system` serves the Design Kit on localhost and matches the reference screenshots in both themes.
- `bun run dev` opens the real launcher (native window + HMR) against the mock `installDir`.
- `bun run package` produces a zip that, extracted anywhere, runs via `Start.exe` with nothing installed on the host.
- The launcher scans, groups, searches and launches GENSLATE, PortableApps.com and portapps.io apps; edits
  settings; backs up and restores the drive; and safely ejects it.
- `bun run check` and `bun run test` are green; performance targets (§10.3) are met.

## 2. Decisions

| Topic | Decision |
|---|---|
| Relationship to SLATESUITE | Clean rebuild; SLATESUITE is reference only. |
| Prior spec (git `3be621f`) | Discarded; this spec starts fresh. |
| Spec / plan structure | One master spec; one implementation plan per phase (Foundation, Design system, Launcher). |
| Architecture | **Approach A:** logic in shared crates, thin binaries (Tauri launcher, `Start.exe`). |
| Platform | Windows 10/11 x64. Windows-specific code sits behind traits in `genslate-platform`. |
| Orchestration / package manager | Turborepo; **bun for everything** (never npm, npx, pnpm, yarn or node). |
| Rust in Turborepo | One Cargo workspace exposed as a single `@genslate/rust` package. |
| Launcher form | Tray-resident popup (global hotkey, tray click) that expands into a tools view. |
| v1 features | Core launcher + **Settings UI**, **Backup & restore**, **Safe eject + cleanup**. |
| Deferred (own specs later) | App manager (catalog install/update), vault, multiple named profiles, AI tools, diagnostics. |
| Profiles | Single `shared` profile; code addresses folders by profile id so named profiles can come later. |
| Backup contents | Everything in the install except `storage/backups/` and `other/*/cache/`. |
| WebView2 | System runtime first; bundled fixed-version runtime as fallback. |
| Entry point | Root `Start.exe` stub (also the eject and restore helper). |
| Eject finish | `Start.exe` copied to `%TEMP%` performs the eject, then deletes itself. |
| Release | Portable `.zip` of `installDir` (plus a `-full` variant with the WebView2 fallback). |
| Design system scope | Full parity with SLATESUITE (all foundations + 44 components), built launcher-first. |
| Design Kit | `programs/webapp/design-system/`, static build for sharing; no launcher preview. |
| Agent rules | Authored once, synced to Claude, Cursor and Antigravity; tool-specific extras not synced. |
| Attribution | No agent attribution anywhere (no `Co-Authored-By`, no "Generated with", no `claude/` branches). |

## 3. Monorepo

_Built in: Plan 1 (M1)_

```
programs/
  desktop/launcher/          Tauri app: src/ (React UI), src-tauri/ (thin shell), tests/{unit,e2e}/,
                             other/launcher/ (shipped template), installDir/ (dev mock drive)
  desktop/start/             Rust bin crate → Start.exe (modes: start | eject | restore), tests/{unit,e2e}/
  webapp/design-system/      Design Kit (Vite + React), tests/{unit,e2e}/
crates/
  paths/  platform/  launcher-core/  design-tokens/  testing/      each with tests/{unit,e2e}/
packages/
  tokens/  design-system/  tauri-bridge/  config-typescript/  config-vite/   each with tests/{unit,e2e}/
scripts/
  commands/<cmd>.ts          one file per root command (Bun APIs: Bun.$, Bun.file)
  lib/                       shared script helpers
  agents/                    agents:sync and the shared guard scripts (§9)
release/launcher/  release/.archive/
docs/superpowers/{specs,plans}/   docs/research/
.config/  .claude/  .agents/  .cursor/  .github/  .vscode/  .changes/  .genslate/
```

### Names

| Unit | Package / crate name | Output |
|---|---|---|
| `programs/desktop/launcher` | `@genslate/launcher`; crate `genslate-launcher` | `genslate-launcher.exe` |
| `programs/desktop/start` | crate `genslate-start` | `Start.exe` |
| `programs/webapp/design-system` | `@genslate/webapp-design-system` | static site |
| `packages/*` | `@genslate/tokens`, `@genslate/design-system`, `@genslate/tauri-bridge`, `@genslate/config-typescript`, `@genslate/config-vite` | — |
| `crates/*` | `genslate-paths`, `genslate-platform`, `genslate-launcher-core`, `genslate-design-tokens`, `genslate-testing` | — |
| Cargo workspace (Turborepo view) | `@genslate/rust` | — |

### Toolchain files

- Root `package.json`: `workspaces` = `packages/*`, `programs/*/*`; every npm dependency pinned exactly in
  `workspaces.catalog`, packages reference `"catalog:"`.
- Root `Cargo.toml`: one workspace (`crates/*`, `programs/desktop/*/src-tauri`, `programs/desktop/start`),
  `[workspace.dependencies]` exact pins, `[workspace.lints]`, tuned release profile.
- `turbo.json`, `bunfig.toml`, `rust-toolchain.toml`, `.editorconfig`, `.gitattributes` (LF), `.gitignore`,
  a thin root `tsconfig.json`.
- **`.config/` rule:** every tool config lives in `.config/` when the tool can read it there (Biome, cspell,
  knip, commitlint, lefthook, cargo-deny / mutants / tarpaulin, rustfmt, dependency exceptions). Files stay at
  the root only when the tool requires it; M0 confirms each tool (§11).

### Root commands (all `bun run <cmd>`)

| Command | Purpose | Works from |
|---|---|---|
| `setup` | Install deps (frozen), git hooks, Rust toolchain via rustup | M1 |
| `check` | Full quality gate (§10.4) | M1 |
| `test` | bun tests + cargo nextest + Playwright | M1 |
| `format` | Biome write + cargo fmt | M1 |
| `clean` | Remove build output and caches | M1 |
| `version` | Bump every manifest (package.json, Cargo, tauri.conf) at once | M1 |
| `deps` | Report outdated npm packages, crates, bun and Rust | M1 |
| `attribution` | Reject agent authors, trailers, footers and branch names | M1 |
| `agents:sync` | Generate Cursor and Antigravity rules from `.claude/rules/` | M2 |
| `tokens` | Regenerate design tokens | M3 |
| `design-system` / `design-system:build` | Design Kit dev server / static build | M4 |
| `build` | Build every project | M4 (grows per milestone) |
| `dev` | Native launcher with HMR | M7 |
| `package` | Portable release zips | M10 |

### Skeleton fixes

Rename `premissions/` → `permissions/`, `tauri.config.json` → `tauri.conf.json`, `enviroment.rs` →
`environment.rs`, `emit.sharedts` → `emit.shared.ts`, `coeql-config.yml` → `codeql-config.yml`,
`lession-learned.md` → `lessons-learned.md`. Move `.cursor/extesions.json` content into
`.vscode/extensions.json`. Remove deprecated `.cursorrules`, `.claude/mcp.json` (Claude Code reads root
`.mcp.json`) and folders no tool reads (`personas/`, `logs/`, unused `workflows/` under `.claude/` and `.cursor/`).

## 4. Conventions

- **Files:** `<subject>.<kind>.<ext>`, kebab-case: `app-row.component.tsx`, `use-theme.hook.ts`,
  `button.variants.ts`, `button.types.ts`, `cn.util.ts`, `nord.polar-night.theme.ts`. Rust: `snake_case.rs`.
- **Folders:** by domain, then role (`features/apps/{components,hooks,state,lib}/`).
- **One responsibility per file**; split past ~200 lines. Barrels (`index.ts`) use named exports only.
- **Rust modules:** folders by capability (`name.rs` beside `name/`, no `mod.rs`), small files.
- **Tests** live only in `tests/unit/` and `tests/e2e/` of each project; no test code in `src/`.
- **Commits:** Conventional Commits; scopes = project ids plus `repo, ci, deps, agents, docs, release`;
  changelog entries via `.changes/`. No agent attribution.
- A structure check in `bun run check` enforces file-name shape and test location.

## 5. Portable runtime

_Built in: Plan 3 (M5, M7)_

### 5.1 Drive layout

```
<install>/                                   any folder, any drive letter
├─ Start.exe
├─ programs/
│  ├─ genslate/<app>/                        GENSLATE apps; launcher = genslate-launcher.exe + app.manifest.toml
│  ├─ genslate/.runtime/webview2/            optional bundled WebView2 fixed runtime (fallback)
│  ├─ portableapps.com/<App>/                PortableApps.com Format apps
│  └─ portapps.io/<app>/                     portapps.io apps
├─ other/<app>/{configs,database,cache,logs,documents,licenses,resources}
└─ storage/
   ├─ users/shared/{Desktop,Documents,Downloads,Music,Pictures,Videos,Recycle Bin}
   ├─ backups/
   └─ vault/                                 reserved for the later vault spec
```

- **Template vs runtime:** `programs/desktop/launcher/other/launcher/` is the shipped template (default
  `settings.toml`, documents, licenses, resources), copied to `<install>/other/launcher/` at package time.
  `cache/`, `database/` and `logs/` are created at run time and git-ignored.
- **Dev mock:** `programs/desktop/launcher/installDir/` mirrors the layout above and is what debug builds use.
- **GENSLATE app manifest:** `programs/genslate/<app>/app.manifest.toml` with `id`, `name`, `version`,
  `executable`, `description`, `category`, `icon` (all paths relative to the app folder).

### 5.2 Install-root resolution (in order)

1. `GENSLATE_INSTALL_DIR` environment override.
2. **Install mode:** the executable sits at `<root>/programs/genslate/<app>/` and `<root>/other/` exists.
3. **Dev mode:** a debug build inside the repo uses `programs/desktop/launcher/installDir/`.
4. Otherwise: **fatal error screen.** There is never a fallback to `%APPDATA%` or any host folder.

### 5.3 Portability rules

- All stored paths (favorites, recents, overrides) are relative to `<install>`; drive-letter changes are harmless.
- A read-only install runs in read-only mode with a banner; state is kept in memory.
- The WebView2 user-data folder is `other/launcher/cache/webview/` (window created in code to set it).
- **WebView2 selection:** before any window exists, Rust checks for an installed system runtime; if none is
  available it points WebView2 at `programs/genslate/.runtime/webview2/`; if that is missing too, a native
  error dialog explains what to do. Mechanism confirmed in M0.
- Launched PortableApps.com apps receive the platform's environment variables (documents, music, pictures,
  videos, …) pointing into `storage/users/shared/`; exact names confirmed in M0.
- Host writes are limited to: the optional autostart `Run` entry (off by default) and the self-deleting
  `%TEMP%` helper during eject/restore. Cleanup removes both.

## 6. Rust architecture

_Built in: Plan 3 (M5–M7, M9)_

### 6.1 Crates (no Tauri dependency)

| Crate | Responsibility |
|---|---|
| `genslate-paths` | Install-root resolution (§5.2) with an injectable `Environment`; per-app dirs; **path safety**: canonicalize and require every path to stay inside the root, rejecting `..`, symlinks/junctions leaving the root, UNC paths, alternate data streams, 8.3 short-name tricks and reserved device names. |
| `genslate-platform` | Traits: `ProcessSpawner` (spawn + track + graceful close), `DriveInfo`, `Ejector`, `IconExtractor`, `SingleInstance`, `ShellOpen`, `WebViewRuntime`. Windows implementations via the `windows` crate are the **only** `unsafe` code in the repo. Test doubles live in `genslate-testing`. |
| `genslate-launcher-core` | Modules: `config` (settings schema, loader, comment-preserving `toml_edit` writer, debounced hot reload), `catalog` (one scanner per source → common `AppEntry`; incremental, mtime-fingerprinted), `icons` (extract + cache), `library` (favorites, recents, overrides), `launch` (spawn with env injection, running-process tracking), `backup` (create, verify, retention, restore plan), `eject` (orchestration up to hand-off). Ports-and-adapters: depends on `genslate-platform` traits, never on Windows APIs directly. |
| `genslate-design-tokens` | Generated token constants (window sizes, colors for native surfaces). |
| `genslate-testing` | Fake `installDir` builder, fake `Environment`, platform test doubles, fixtures. |

### 6.2 Binaries

- **`genslate-launcher` (`src-tauri`)** — thin: window, tray, tray-menu window, global hotkey, autostart,
  `launcher-icon://` protocol, WebView2 selection, typed IPC commands that delegate to `launcher-core`,
  events for push updates. Shared state behind `State<Arc<…>>`; a panic hook that logs; graceful shutdown.
- **`genslate-start` (`Start.exe`)** — GUI subsystem (no console), tiny:
  - `start` (default): locate and start `programs/genslate/launcher/genslate-launcher.exe`, or focus the running instance.
  - `--eject`: when run from the drive, copy itself to `%TEMP%` and re-run from there; wait for the launcher to
    exit; eject the drive (`CM_Request_Device_Eject` family, confirmed in M0); report the vetoing process on
    failure; show a "safe to remove" notification; delete itself.
  - `--restore <backup>`: same temp hand-off; verify the backup; extract to a staging folder on the same
    drive; swap in by renames; relaunch the launcher. A failure part-way leaves the original install untouched.

### 6.3 State on disk

- `other/launcher/configs/settings.toml` — user-editable settings (§7.3).
- `other/launcher/database/library.json` — favorites, recents, per-app overrides (relative paths).
- `other/launcher/database/catalog-cache.json` — scan fingerprints for incremental rescans.
- `other/launcher/logs/` — rotating `tracing` logs.
- Every write is **atomic** (temp file + rename) and debounced, protecting against unplug mid-write and limiting flash wear.

### 6.4 Errors

`thiserror` enums per crate (`anyhow` only in binaries). Config problems degrade to defaults with a
warning. A missing install root is the only fatal error. IPC errors cross as `{ code, message }`.

## 7. Launcher

_Built in: Plan 3 (M6–M9)_

### 7.1 Form (matches the SLATESUITE launcher)

- Transparent, frameless tray popup anchored above the taskbar tray; **460 px** wide, expanding left to
  **920 px** for the tools view via the clip-path spring reveal. Opens on the global hotkey
  (default `Ctrl+Alt+Space`) or a tray click; the tray right-click menu is a small design-system window.
- **Frame ("bezel"):** one continuous flat surface around a recessed **well**:
  - **Titlebar:** traffic lights (close and minimise hide to the tray, zoom disabled), GENSLATE mark +
    wordmark + "Launcher", pin-on-top button. The whole bar drags.
  - **Well:** source tabs **GENSLATE · PortableApps.com · portapps.io** with counts; grouped list
    (Favorites, Recent, then categories; collapsible group headers with counts); **44 px two-line rows**
    (icon, name, description) with a running-app dot; keyboard navigation via `aria-activedescendant`.
  - **Documents rail** (right): "Shared" profile avatar, drive label, and the six user folders.
  - **Command bar** (under the well): search + slash commands (`/settings`, `/backup`, `/restore`, `/eject`,
    `/open <folder>`, `/help`).
  - **Tools button** (under the rail) and **status bar** (drive name, free space, app count; mode selectable).
- **App context menu:** Launch, Run with arguments, Favorite, Open folder, Hide, Properties.
- **Subviews in the well:** app details/properties, run-with-arguments, help (shortcut list).

### 7.2 Tools view (expanded)

Real in v1: **Settings**, **Backup & restore**, **Safe eject**. Shown as "coming soon" tiles: App manager,
Diagnostics, AI tools. Tools views are lazy-loaded.

### 7.3 Settings

- `other/launcher/configs/settings.toml` is the **single source of truth**; the Settings UI reads and writes it.
- Hand edits hot-reload. UI writes go through `toml_edit`, preserving the user's comments and formatting.
- Every key is optional; unknown or invalid keys log a warning, show a non-blocking notice, and fall back to defaults.
- The package ships a fully commented default file; first run creates it if missing; updates never overwrite it.
- Initial schema:

| Table | Keys (default) |
|---|---|
| `[appearance]` | `theme` (default `"system"`, which follows the Windows app mode: dark → Polar Night, light → Snow Storm; or force `"polar-night"` / `"snow-storm"`) |
| `[behavior]` | `autostart` (`false`), `hide_on_blur` (`true`), `start_hidden` (`false`) |
| `[keybindings]` | `toggle` (`"Ctrl+Alt+Space"`), `focus_search` (`"mod+k"`), `toggle_tools` (`"mod+t"`), `toggle_pin` (`"mod+p"`), `toggle_favorite` (`"mod+d"`) |
| `[apps]` | `hidden` (`[]`), `categories` (per-app category overrides, `{}`) |
| `[status_bar]` | `mode` (default `"drive"`; also `"apps"` · `"clock"`) |
| `[backup]` | `retention` (`5`) |

### 7.4 Backup & restore

- **Backup:** shows estimated size and checks free space first; writes
  `storage/backups/genslate-backup-<YYYYMMDD-HHMMSS>.zip` containing everything in the install except
  `storage/backups/` and `other/*/cache/`, plus a `manifest.json` (version, date, file list with SHA-256).
  Progress streams to the UI; cancellable (partial file removed). Retention keeps the newest `retention` backups.
  Archives are written as zip64 (installs can exceed 4 GB) and streamed, never held in memory; files open
  for writing by running apps are reported in the result rather than silently skipped.
- **Restore:** pick a backup → verify manifest hashes → confirm (alert dialog naming what will be replaced) →
  launcher closes apps it launched, then hands off to `Start.exe --restore` (§6.2) and exits.
- Extraction rejects zip-slip, absolute paths and entries outside the install root.
- Note: same-drive backups protect against mistakes and corruption, not drive loss; copying to another
  location is a later feature.

### 7.5 Safe eject + cleanup

1. Triggered from the tools view, the tray menu or `/eject`.
2. List running apps the launcher started; ask to close them (graceful close; optional force after a timeout).
3. Flush state; remove the launcher's own host traces (autostart entry, its temp files).
4. Hand off to `Start.exe --eject` (§6.2) and exit.
5. The helper ejects, reports a veto with the blocking process name, notifies "safe to remove", deletes itself.

Cleanup never touches other software's host traces and never deletes outside `<install>` except its own temp helper.

### 7.6 Data flow

```
React feature ──► @genslate/tauri-bridge (generated types) ──► Tauri command ──► launcher-core ──► genslate-platform
      ▲                                                                                    │
      └──────────────── Tauri events (catalog changed, process state, progress) ◄──────────┘
```

- **IPC types are generated from Rust with tauri-specta**; `check` fails on binding drift.
- The bridge has a **browser mock** implementing the same interface (fake apps, drive info), used by
  `vite dev` in a browser, unit tests and Playwright.
- **Trust boundary:** the UI sends **ids, never paths or command lines**. Rust validates every argument and
  resolves ids to executables its own scanner found under `programs/`.
- Front-end state: Rust is the source of truth; a small `useSyncExternalStore` store mirrors it. No direct
  network calls from the UI.

## 8. Tokens, design system and Design Kit

_Built in: Plan 2 (M3, M4)_

### 8.1 Tokens (`packages/tokens`)

- Sources in `src/tokens/` and `src/themes/`: Nord primitives (nord0–nord15); semantic roles (surfaces,
  foreground, borders, accent, focus, selection, fills, controls, `danger`/`warning`/`success`/`info`);
  chrome roles (titlebar, command center, status bar, tabs, tooltip, traffic lights, scrollbar); scales
  (type, radius, shadow, motion, z-index, layout sizes, cursors).
- Themes: `nord.polar-night` (dark), `nord.snow-storm` (light), switched by `html[data-theme]`; transitions
  suppressed while `data-theme-switching` is set. No `dark:` variants anywhere.
- `bun run tokens` generates CSS variables, a Tailwind v4 `@theme inline` block, TypeScript, JSON and Rust.
- **Contrast validator** fails the build on any WCAG 2.2 AA failure in either theme. `check` fails on drift.

### 8.2 Visual contract (written into `.claude/rules/design-system.md`)

- Chrome recedes, content leads. **Flat at rest; depth only when floating** (0.5 px ring, soft layered
  shadow, top inner highlight in dark). **One accent**, used sparingly; Aurora colors only mean status.
- **macOS feel, VS Code density:** 13 px Inter base (JetBrains Mono for code), tabular numbers in data;
  radii 6 / 8 / 12; tree rows 22, sidebar rows 28, menu items 24, controls 28, tabs 36, status bar 24,
  titlebar 38; Codicons 16 px (14 dense); themed cursors; spring easing; animate transform and opacity only.
- **Every state in both themes:** rest, hover, pressed, focus-visible, selected, disabled, invalid, loading,
  inactive window; at 100 / 125 / 150 % scaling; under reduced motion, `prefers-contrast: more` and forced colors.
- Custom traffic-light window chrome on every window; no native caption buttons.

### 8.3 Design system (`packages/design-system`)

- React 19 + Base UI + tailwind-variants + Tailwind v4; self-hosted variable fonts; Codicons (Lucide only
  where no codicon fits).
- Layout: `src/components/<category>/<name>/{<name>.component.tsx, <name>.variants.ts, <name>.types.ts, index.ts}`;
  shared `recipes/` (`popupSurface`, `field`, `listItem`), `providers/` (theme, platform, window state,
  cursor), `hooks/`, `utils/` (`cn`, shortcuts), `styles/` (base, utilities, variants, animations, fonts).
- Rules: `ref` as a prop; Base UI `render` prop for polymorphism; `data-slot` on every part; token utilities
  only; named exports only; **never imports Tauri** (window chrome takes callbacks); accessibility strings via
  a `labels` prop; WAI-ARIA keyboard patterns.
- **Build waves (launcher-first):**

| Wave | Components |
|---|---|
| 1 | Foundations (colors, typography, spacing & elevation, motion, icons, cursors) · window: title bar, traffic lights, status bar, app shell, window context menu · layout: sidebar, panel, card, scroll area, separator |
| 2 | Actions: button, icon button, toggle button, segmented control, toolbar · display: icon, kbd, avatar, color swatch, code block |
| 3 | Overlays: menu, context menu, popover, tooltip, dialog, alert dialog, toast, command palette · feedback: badge, banner, progress bar, spinner, skeleton, empty state, feature teaser |
| 4 | Inputs: text field, search field, number field, textarea, checkbox, checkbox group, radio group, switch, select, slider, field · navigation: tabs, tree, breadcrumbs |

Launcher UI work (M8) may start once waves 1–3 pass their exit gate.

### 8.4 Design Kit (`programs/webapp/design-system`)

- Plain Vite + React web app. `bun run design-system` serves it on localhost; `bun run design-system:build`
  emits a static site.
- Reproduces the reference screenshots: traffic-light titlebar with sidebar toggle and "**GENSLATE** Design
  Kit"; centered **Ctrl+K** component search; theme toggle and settings on the right; category sidebar
  (Foundations, Window, Layout, Actions, Inputs, …) with pill selection; pages with breadcrumb, title, lede,
  sections containing a live preview, a copyable TSX snippet and a **state matrix** (rest, hover, pressed,
  focus, disabled, loading); status bar with the GENSLATE accent item, theme, runtime, page count and version.
- Pages live at `src/pages/<category>/<name>.page.tsx`; every component lands with its page and unit tests.
- Each wave passes Playwright screenshot QA in both themes, reviewed by `ui-visual-qa` against SLATESUITE.

## 9. Agent folders

_Built in: Plan 1 (M2)_

### 9.1 Synced vs per-tool

| Synced to all three tools (one source) | Per tool, never synced |
|---|---|
| **All rules:** authored in `.claude/rules/*.md` (frontmatter `paths:` for scoping); `bun run agents:sync` generates `.cursor/rules/*.mdc` (`paths` → `globs`, unscoped → `alwaysApply: true`, plus `description`) and `.agents/rules/*.md`. Generated files carry a "generated — do not edit" header; `check` fails on drift. | **Claude:** agents, skills, commands, hooks, memory, settings |
| **`AGENTS.md`:** the shared index (stack, layout, commands, golden rules) read by Cursor and Antigravity; `CLAUDE.md` imports it with `@AGENTS.md` and adds only Claude-specific notes. | **Cursor:** commands, `hooks.json`, `mcp.json` |
| **Guard scripts** in `scripts/agents/` (generated-file guard, attribution guard, format-on-edit); each tool's own hook config calls the same script. | **Antigravity:** workflows and light skills |

**Final safety net, tool-independent:** lefthook on commit and `bun run check` in CI enforce the same rules,
so nothing depends on one tool's hooks having run.

### 9.2 Rules (one topic per file)

`project`, `monorepo-structure`, `naming`, `commands`, `dependencies`, `typescript`, `react`, `rust`,
`tauri-ipc`, `security`, `performance`, `design-system`, `testing`, `git-and-attribution`, `docs`,
`verify-latest` (confirm versions and APIs against current docs before use).

### 9.3 `.claude/` (main development agent)

| Area | Contents |
|---|---|
| `settings.json` | Permission allowlist (bun, cargo, read-only git), deny list (lockfiles, generated files), hook wiring. |
| `agents/` | `code-reviewer`, `security-reviewer`, `ui-visual-qa`, `design-system-engineer`, `tauri-rust-engineer`, `docs-writer`, `release-manager`. |
| `skills/` | `design-system-component`, `design-tokens`, `tauri-ipc-command`, `rust-crate-module`, `verify-latest`. |
| `commands/` | `/check`, `/new-component`, `/new-ipc-command`, `/tokens`, `/deps`, `/audit`, `/release`, `/change`. |
| `hooks/` | Wiring for format-on-edit, generated-file guard, attribution guard, session-start context. |
| `memory/` | `decisions.md`, `lessons-learned.md`, `active-context.md`. |
| MCP | Context7 in root `.mcp.json`. |

### 9.4 `.cursor/` (editing and basic tasks)

Generated rules; `hooks.json` (format-on-edit, generated-file guard); `commands/` (`check`,
`new-component`, `tokens`, `change`); `mcp.json` (Context7). Extension recommendations in `.vscode/extensions.json`.

### 9.5 `.agents/` (Antigravity, light tasks)

Generated rules; `workflows/` (`check`, `run-tests`, `fix-ui`, `format`, `tokens`, `new-change`); light
`skills/` (`quality-gate`, `ui-fixes`).

## 10. Engineering standards (every milestone)

### 10.1 Latest versions

- Latest stable of everything — npm packages, crates, bun, Rust, Tauri, WebView2 runtime — resolved live in
  M0 and pinned exactly. Snapshot on 2026-10-07 (re-resolved in M0): bun 1.4.2, Rust 1.99.0, TypeScript 7.0,
  Vite 8.3, React 19.3, Tauri 2.12, Tailwind CSS 4.3, Turborepo 2.11, Base UI 1.8, tailwind-variants 3.3,
  Biome 2.5, Playwright 1.63, tauri-specta 1.0, thiserror 2.0, toml_edit 0.25, windows 0.62.
- An older pin is allowed only with a recorded reason in `.config/dependency-exceptions.toml`; `check` fails
  on a stale pin without one. `bun run deps` reports outdated items; Renovate keeps them current.

### 10.2 Modern syntax and practice (confirmed against official docs in M0, then written into rules)

- **TypeScript 7:** strictest options (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `verbatimModuleSyntax`, `noImplicitOverride`), ESM only, `satisfies`, `as const`, discriminated unions,
  `import type`; no `any`, non-null `!`, enums, default exports or CommonJS.
- **React 19:** `ref` as a prop, `use`, Actions / `useTransition`, Suspense and error boundaries, React
  Compiler on; no `forwardRef`, no `useEffect` for derived state or data fetching, no reflexive `useMemo`.
- **Vite 8 / Tailwind 4:** CSS-first Tailwind config; native CSS (`@layer`, container queries, `color-mix`,
  `:has`) over JavaScript where possible.
- **Rust (edition 2024):** let-chains, `let … else`, async fn in traits, `LazyLock`/`OnceLock`, std over
  third-party where equivalent; `[workspace.lints]` with clippy `pedantic` (plus selected `nursery`) and
  `unwrap_used`, `expect_used`, `panic`, `todo`, `dbg_macro` denied; `#![forbid(unsafe_code)]` in every
  crate except `genslate-platform`; `tracing` for logs.
- **Tauri 2:** async commands, typed events, per-window capabilities, plugins only where needed.
- **Bun:** `bun run`, `bun test`, `bun x`; Bun APIs in scripts.

### 10.3 Performance targets

| Target | Budget |
|---|---|
| Hotkey → popup visible | < 100 ms (webview kept warm while hidden) |
| Cold start → interactive (USB 3) | < 1.5 s |
| Idle memory (launcher + WebView2) | < 150 MB |
| Initial UI JavaScript | < 250 KB gzipped (tools views lazy-loaded) |
| Rescan of 200 apps (warm) | < 300 ms (incremental, mtime-keyed, bounded concurrency) |

Long lists virtualized; animations on transform/opacity only; atomic debounced writes; Rust release
profile with LTO, `codegen-units = 1`, `strip`; Turborepo caching with exact `inputs`. Budgets are measured
from M7 and bundle size is checked in CI.

### 10.4 Quality gate (`bun run check`)

Biome · TypeScript 7 typecheck · cspell · knip · naming/structure + test-location check · token drift ·
agent-rules drift · IPC-binding drift · dependency-exception check · attribution check · `cargo fmt --check`
· clippy (warnings denied) · cargo-deny · `bun audit` · secret scan · bundle-size budgets.
Lefthook runs the fast subset on commit; commitlint enforces Conventional Commits.

### 10.5 Testing

- **TypeScript:** `bun test` + Testing Library + happy-dom in `tests/unit/` (mirroring `src/`); Playwright in
  `tests/e2e/` against the Design Kit and the launcher UI over the browser mock, including screenshot
  comparisons in both themes.
- **Rust:** `cargo nextest`; each crate has `tests/unit.rs` and `tests/e2e.rs` entry files declaring modules in
  `tests/unit/` and `tests/e2e/`; tests use public APIs and `genslate-testing` fixtures.
- **Manual QA checklist** (in `other/launcher/documents/`): tray, hotkey, real eject, WebView2 fallback,
  running from a USB stick, drive-letter change, read-only drive.

### 10.6 Security

- **Tauri:** per-window least-privilege capabilities, no wildcards; strict CSP with no `unsafe-eval` and, if
  M0 confirms the UI stack allows it, no `unsafe-inline`; `withGlobalTauri` off; `freezePrototype` on; no
  remote content or navigation; no `dangerous*` options; single-instance; `launcher-icon://` sanitizes every path.
- **IPC:** ids only; every argument validated in Rust; processes spawned from argument arrays, never a shell;
  all paths confined to the install root (§6.1).
- **Data:** size limits on config and database files; restore guards against zip-slip and verifies hashes first.
- **`unsafe`:** only in `genslate-platform`, each block with a `// SAFETY:` comment, reviewed explicitly.
- **Supply chain:** exact pins, committed lockfiles, `bun install --frozen-lockfile` in CI, cargo-deny
  (advisories, licenses, bans, sources), `bun audit`, a release-age cooldown if bun supports it (confirmed in
  M0), GitHub Actions pinned by commit SHA, least-privilege workflow tokens, secret scanning.
- **Threat model:** `other/launcher/documents/security.md`.

### 10.7 Release and CI

- `bun run package` stages a complete `installDir` (`Start.exe`, launcher, `other/launcher/` template, empty
  `programs/` and `storage/` trees) and writes `release/launcher/genslate-launcher-<version>-win-x64.zip`,
  `…-win-x64-full.zip` (adds the WebView2 fallback runtime) and `SHA256SUMS`; older builds rotate to
  `release/.archive/`.
- `bun run version` bumps every manifest; `.changes/` produces the changelog.
- CI on a Windows runner: frozen install → check → test → build → Playwright. A version tag drafts a GitHub
  release with both zips and checksums.

### 10.8 Milestone exit gate

A milestone is done only when `bun run check` and `bun run test` are green; the `security-reviewer` has
reviewed its changes; any performance target it touches is measured and within budget; docs for its
behavior are written; and nothing breaks §4.

## 11. M0 verification list

Each item is confirmed against current official docs (Context7 / web) and recorded in `docs/research/`:

1. Latest stable versions of every dependency and tool (§10.1).
2. TypeScript 7 compiler options and tooling integration (Biome, knip, Vite).
3. Vite 8 + `@vitejs/plugin-react` 6 + React Compiler wiring.
4. Tailwind CSS 4.3 and Base UI 1.8 APIs used by the design system.
5. tauri-specta 1.0 command/event generation; Tauri 2.12 transparent popup, tray, global shortcut,
   autostart, single-instance and custom protocol APIs.
6. WebView2: runtime detection and pointing at a fixed-version runtime folder at startup; user-data folder.
7. CSP without `unsafe-inline` for the chosen UI stack.
8. Which tools read config from `.config/` (Turborepo, rustfmt, Cargo, lefthook, Biome, cspell, knip, commitlint).
9. Bun release-age cooldown support and `bun audit`.
10. Antigravity `.agents/` rules/workflows/skills format; Cursor `.mdc` rules, `hooks.json` and commands format.
11. PortableApps.com Format (`App/AppInfo/appinfo.ini`, icons, launcher environment variables); portapps.io layout.
12. Windows drive eject API and veto reporting; graceful process close.

Findings that contradict this spec are brought back for a decision before the affected milestone starts.

## 12. Phasing

| Plan | # | Milestone | Depends on | Delivers |
|---|---|---|---|---|
| 1 Foundation | M0 | Research and versions | — | §11 findings, pinned catalog and workspace dependencies |
| | M1 | Monorepo foundation | M0 | Skeleton fixes, root configs, `turbo.json`, scripts framework, check/test/format/clean/version/deps/attribution, structure checks, `genslate-testing`, lefthook + commitlint, baseline Windows CI |
| | M2 | Agent rules and tooling | M1 | All rules, `agents:sync` + drift check, `AGENTS.md` / `CLAUDE.md`, guard scripts, per-tool extras (§9) |
| 2 Design system | M3 | Tokens and configs | M1 | `packages/tokens`, `genslate-design-tokens`, `config-typescript`, `config-vite` |
| | M4 | Design system + Design Kit | M2, M3 | Design Kit shell, then waves 1–4 each with pages, tests and visual QA |
| 3 Launcher | M5 | Rust core | M1 (parallel with M4) | `genslate-paths` → `genslate-platform` → `launcher-core` (config, catalog + icons, library, launch) |
| | M6 | IPC contract | M3, M5 | tauri-specta bindings, `packages/tauri-bridge`, browser mock, binding drift check |
| | M7 | Tauri shell + `Start.exe` | M5, M6 | Window, tray, tray menu, hotkey, autostart, icon protocol, WebView2 selection, `Start.exe` start mode; `bun run dev` works |
| | M8 | Launcher UI | M4 waves 1–3, M6, M7 | frame → apps → command bar → documents rail → status bar → tools view → Settings |
| | M9 | Backup, restore, eject | M7, M8 | Core logic, `Start.exe --restore` / `--eject`, UI with progress and confirmations |
| | M10 | Release hardening | M9 | `bun run package` (both zips), full CI + release workflow, manual QA checklist, documents, final code/security/performance review |

Every milestone ends with the exit gate (§10.8).
