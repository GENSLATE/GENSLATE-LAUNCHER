# GENSLATE Launcher — Architecture & Phase 1 Design

Status: draft for review · Date: 2026-10-06 · Path: architectural (spec → plan)

## 1. Goal and intent

Build a **modern, portable "operating system on a stick" launcher for Windows**, in the spirit of the
PortableApps.com Platform, with GENSLATE's own design and capabilities. A user carries one drive
(USB stick or portable SSD) holding their apps and documents and runs it on any Windows PC without
writing anything to the host.

- **Focus order:** GENSLATE apps first; PortableApps.com and portapps.io apps second.
- **Foundation for more apps:** the monorepo and a complete Nord design system are built so that
  later GENSLATE apps reuse them.
- **UI/UX:** identical in look and feel to the design system and launcher in
  `S:\DEVELOPMENT\PROJECTS\GENSLATE-SLATESUITE` (reference screenshots: Design Kit "Colors" and
  "Button" pages). Official Nord: **Polar Night = dark theme, Snow Storm = light theme.**
- **This repo is a clean rebuild.** SLATESUITE is the *specification* (visual contract, rules,
  layout ideas), not a code source. Each piece is written fresh here and audited as it is built.

Success looks like: `bun run dev` opens the real launcher against the mock `installDir`;
`bun run example` serves the design kit in a browser matching the screenshots; `bun run package`
yields a zip that extracts to a working portable install; `bun run check` and `bun run test` pass.

### Scope split (decided)

| Spec | Contents |
|---|---|
| **This spec** | Monorepo foundation, tokens + design system, example webapp, agent folders, portable layout and `paths`, launcher core + shell + UI (scan/launch/library/settings), packaging, CI. Interface-level seams only for the items below. |
| Later spec #2 | App installer/updater (PortableApps.com and portapps.io catalogs, download, hash verify, extract, update). |
| Later spec #3 | Vault (encrypted `storage/vault/`, unlock, auto-lock). |
| Later spec #4 | Safe eject and host-PC cleanup. |

Each later spec gets its own brainstorm → spec → plan. They are in scope for the product, not for
this plan.

## 2. Decisions

| Topic | Decision |
|---|---|
| Relationship to SLATESUITE | Clean rebuild; SLATESUITE is the spec only. |
| Launcher form | Tray-resident popup (global hotkey) that expands into a larger window for tools/settings. |
| Platforms | Windows x64 first. Code stays portable-clean: platform-specific code sits behind Rust traits. |
| Build orchestration | **Turborepo**; **bun** for everything (never npm/npx/pnpm/yarn/node). |
| Rust in Turborepo | One Cargo workspace (root `Cargo.toml`), exposed to Turborepo as a single `@genslate/rust` package (Approach A). |
| Design system scope (phase 1) | All foundations + every component the launcher needs; the kit grows with the apps. |
| Release artifact | Portable zip of a ready-to-run `installDir` in `release/launcher/` (old builds → `release/.archive/`). |
| Attribution | No agent attribution anywhere (no `Co-Authored-By`, no "Generated with" footers, no `claude/` branches), enforced by a script, hooks and CI. |
| Dependencies | **Latest stable of everything**; an older pin needs a recorded compatibility reason. |
| Configs | Every tool config lives in `.config/` if the tool can read it there. |
| Naming | Modular folders (domain → role) and `<subject>.<kind>.<ext>` file names. |
| Tests | Every app/package/crate has `tests/unit/` and `tests/e2e/`; **no test code inside source folders.** |

## 3. Monorepo

_Built in: M1 (verified first in M0)_

```
programs/
  desktop/launcher/        Tauri app: src/ (React UI), src-tauri/ (shell), tests/{unit,e2e}/,
                           other/launcher/ (shipped template), installDir/ (dev mock tree)
  webapp/example/          Design Kit dev server (Vite + React), tests/{unit,e2e}/
crates/
  paths/  launcher-core/  design-tokens/  testing/        each with tests/{unit,e2e}/
packages/
  tokens/  design-system/  tauri-bridge/  config-typescript/  config-vite/   each with tests/{unit,e2e}/
scripts/commands/<cmd>.ts  scripts/lib/                    bun scripts behind root commands
release/  (release/.archive/)
.config/  .claude/  .agents/  .cursor/  .github/  .vscode/
docs/superpowers/specs/    design specs
```

### Root commands (all `bun run <cmd>`)

`setup`, `dev` (native launcher + HMR), `example` (Design Kit on localhost), `build`, `package`,
`test`, `check`, `format`, `tokens`, `deps`, `agents:sync`, `version`, `clean`, `attribution`.
One file per command in `scripts/commands/`.

### Turborepo

`turbo.json` tasks: `tokens`, `typecheck`, `test`, `build`, `check`, `dev` (persistent) and the
`rust:*` tasks. Edges such as `tokens` before `design-system#build`. Rust task `inputs` are
`**/*.rs`, `Cargo.toml`, `Cargo.lock`. A single `@genslate/rust` package runs cargo tasks so cargo's
build lock is not contended; Tauri apps have thin `package.json` scripts for `dev`/`build`/`package`.

### `.config/` rule

All tool config lives in `.config/` when the tool supports a path flag or discovers it there:
Biome, cspell, knip, commitlint, lefthook, cargo-deny/mutants/tarpaulin, TS bases, dependency
exceptions. These stay at the root or beside code because the tool requires it: `package.json`,
`Cargo.toml`, `Cargo.lock`, `bun.lock`, `bunfig.toml`, `rust-toolchain.toml`, `.gitignore`,
`.gitattributes`, `.editorconfig`, a thin `tsconfig.json`, `turbo.json`, per-app `vite.config.ts` and
`tauri.conf.json`. **To verify during planning:** Turborepo `--root-turbo-json`, rustfmt's
`.config/rustfmt.toml` lookup, and Cargo (reads only `.cargo/config.toml`; may need `--config` or a
small root `.cargo/` stub). A file moves only if the tool really supports it, with a header comment
saying where it is wired in.

### Dependency policy

All versions are resolved live at planning time (npm via bun, crates.io, rust toolchain, bun) —
SLATESUITE's pins are not reused. Pins are exact: root `workspaces.catalog` (packages use
`"catalog:"`) and Cargo `[workspace.dependencies]`. Exceptions are recorded with a reason in
`.config/dependency-exceptions.toml`; `bun run check` fails on a stale pin without one. `bun run deps`
reports outdated packages; Renovate stays configured.

### Skeleton fixes

Existing typos to correct: `premissions/` (→ `permissions/`), `tauri.config.json`
(→ `tauri.conf.json`), `crates/paths/src/enviroment.rs` (→ `environment.rs`), `.cursor/extesions.json`
(→ `extensions.json`), `emit.sharedts` (→ `emit.shared.ts`), `.github/codeql/coeql-config.yml`
(→ `codeql-config.yml`). Deprecated `.cursorrules` is removed.

## 4. Naming and structure rules

_Built in: M1 (check script) and M2 (rule files)_

1. **Folders by domain, then role**, e.g. `features/apps/{components,hooks,state,lib}/`; role
   subfolders from the first file so layouts never need reorganizing.
2. **Files `<subject>.<kind>.<ext>`**, kebab-case: `app-row.component.tsx`, `use-launcher.hook.ts`,
   `button.variants.ts`, `button.types.ts`. Rust uses `snake_case.rs`.
3. **One responsibility per file**; split past ~200 lines. A folder has an `index.ts` barrel with
   named exports only.
4. **Rust modules are folders by capability** (`name.rs` next to `name/`, no `mod.rs`) with small
   files (`model.rs`, `error.rs`, `service.rs`), not large `lib.rs` files.
5. **Docs and agent rules**: one topic per file with topic-named files.
6. A check script (`bun run check`) enforces file-name shape and the no-test-code-in-src rule.

### Code conventions (ported from the SLATESUITE rules, re-verified when written)

TypeScript strictest, named exports only, `import type`, no `any`/`!`/enums; React function
components with `ref` as a prop and React Compiler on; token utilities only for styling
(`bg-surface`, `h-control-md`), no hex and no `dark:`; Rust edition 2024, no `unwrap`/`expect`, `unsafe` only in the isolated `platform/` module (§13.2),
`thiserror` errors, `tracing` logs; Conventional Commits with project-id scopes; generated files
never hand-edited.

## 5. Tests

_Built in: M1 (conventions, fixtures, check script); applied in every later milestone_

- Every app, package and crate owns `tests/unit/` and `tests/e2e/`. **No test code in `src/`:** no
  `#[cfg(test)]` modules and no `*.test.ts(x)` beside source.
- **TypeScript:** bun test + Testing Library + happy-dom in `tests/unit/` (mirroring the `src/` tree);
  Playwright in `tests/e2e/` against the example webapp and the launcher UI over the mock IPC; also the
  visual-QA screenshots (both themes).
- **Rust:** Cargo treats each top-level file in `tests/` as a test crate, so each crate has thin
  entries `tests/unit.rs` and `tests/e2e.rs` that declare the modules under `tests/unit/` and
  `tests/e2e/`. Tests use the public API, so modules are designed with public, testable seams.
  `crates/testing` provides shared fixtures (a builder for a fake `installDir` tree, an injectable
  `Environment`).
- **Real-window behavior** (tray, hotkey, transparent popup, WebView2 cache location) cannot be
  automated reliably: it gets a manual QA checklist in `other/launcher/documents/`.

## 6. Portable runtime layout (`genslate-paths`)

_Built in: M5 (`paths`, `config`); the window cache path in M7_

```
<install>/                      any folder, any drive letter
├─ programs/{genslate/<app>/, portableapps.com/<App>/, portapps.io/<app>/}
├─ other/<app>/{configs,database,cache,logs,documents,licenses,resources}   each app's own folder
└─ storage/{users/shared/{Desktop,Documents,Downloads,Music,Pictures,Videos,Recycle Bin}, vault/}
```

- **Template vs runtime.** `programs/desktop/launcher/other/launcher/` is the shipped template
  (default `settings.toml`, docs, licenses, resources), copied to `<install>/other/launcher/` at
  package time. `cache/`, `database/` and `logs/` are created at run time and git-ignored.
- **Root resolution (precedence):** `GENSLATE_INSTALL_DIR` override → install mode (exe at
  `<root>/programs/genslate/<app>/` and `<root>/other/` exists) → dev mode (debug build in the repo
  uses `programs/desktop/launcher/installDir/`) → **fatal, dedicated error screen**. There is **no
  silent fallback to `%APPDATA%`**: nothing is written to the host.
- **Drive-letter safety:** all stored paths (favorites, recents, overrides) are relative to
  `<install>`. A read-only install starts in read-only mode with a banner, with state kept in memory.
- **Host cleanliness:** the window is created in code so WebView2's profile lives in
  `other/launcher/cache/webview/` rather than `%LOCALAPPDATA%`.
- **App compatibility:** launched apps receive PortableApps.com-style environment variables
  (`PortableApps.comDocuments`, `…Music`, `…Pictures`, `…Videos`, …) pointing into
  `storage/users/shared/`. Exact variable names are verified against the platform docs at planning.

### `settings.toml`

`other/launcher/configs/settings.toml` is the user-editable launcher settings file (theme and so
on). Every key is optional, unknown keys are rejected with a clear message, an invalid file logs a
warning and falls back to defaults. It is hot-reloaded. The UI writes to it through `toml_edit`,
preserving comments and formatting. Updates never overwrite it; the package ships defaults and the
file is created on first run if missing.

## 7. Launcher architecture

_Built in: M5 (core), M6 (IPC contract), M7 (shell), M8 (UI)_

### `crates/launcher-core` (no Tauri dependency)

| Module | Responsibility |
|---|---|
| `config/` | `settings.toml` schema, loader, writer, hot-reload watcher |
| `catalog/` | One scanner per source → common `AppEntry` model. GENSLATE: `programs/genslate/<app>/` + app metadata. PortableApps.com: `<App>/App/AppInfo/appinfo.ini` + icon. portapps.io: `<app>/<app>.exe` + config/data folders. Formats are verified against upstream docs at planning. |
| `launch/` | Direct spawn (no shell), working dir, env injection, running-process tracking (needed later for eject) |
| `library/` | Favorites, recents, per-app overrides in `other/launcher/database/`, as relative paths |
| `icons/` | Icon extraction and cache |
| `platform/` | Windows-specific code behind traits |

Installer, vault and eject get **trait seams and reserved IPC namespaces only** in this phase.

### `programs/desktop/launcher/src-tauri` (thin)

Transparent popup window, expand-to-large-window behavior, tray, global hotkey, autostart,
`launcher-icon://` protocol, and IPC commands delegating to core. IPC errors cross as structured
`{code, message}`.

### React UI

`features/{frame,apps,command-bar,rail,status,tools,settings}/`, each with
`components/ hooks/ state/ lib/`. Talks to Rust only through the typed `tauri-bridge` client; a
browser mock implements the same interface so the UI runs in `programs/webapp` and Playwright without
Tauri.

### Error handling

`thiserror` typed errors; config problems degrade to defaults with a warning; a missing install root
is the only fatal error (error screen); the UI maps IPC errors to toasts or inline states.

## 8. Tokens, design system, example webapp

_Built in: M3 (tokens) and M4 (design system + example webapp)_

**Tokens.** `packages/tokens/src/{tokens,themes}/` is the single source: color, layout, typography,
motion, cursor tokens; themes `nord.polar-night` (dark) and `nord.snow-storm` (light). `bun run tokens`
generates CSS variables, Tailwind `@theme inline`, TS, JSON and Rust (`crates/design-tokens`).
Themes swap via `[data-theme]`. A contrast validator gates the themes. Generated files are never
hand-edited; `bun run check` fails on drift.

**Design system (`packages/design-system`).** Base UI + tailwind-variants + Codicons + Inter /
JetBrains Mono (variable fonts). Component layout:
`components/<category>/<name>/{<name>.component.tsx, <name>.variants.ts, <name>.types.ts, index.ts}`,
categories `window, layout, actions, inputs, navigation, display, feedback, overlays`.
Visual contract (written into `.claude/rules/design-system.md` from the SLATESUITE rule, re-verified):
flat at rest, depth only when floating, one accent, macOS feel with VS Code density (13px base,
radii 6/8/12, row 22/28, control 28, titlebar 38, status bar 24), every state in both themes, WCAG 2.2 AA,
reduced-motion/contrast/forced-colors support, custom traffic-light window chrome on every OS,
Tauri-free.

Phase 1 components: all foundations plus title bar + traffic lights, status bar, app shell, sidebar,
panel, card, scroll area, separator, button, icon button, toggle, segmented control, toolbar, text and
search fields, switch/select/checkbox, menus + context menu, popover, tooltip, dialog/sheet, tabs,
command bar/palette, list/tree rows, badge, progress, toast, icon.

**Example webapp (`programs/webapp/example`).** Vite + React, served by `bun run example`. Reproduces
the reference screenshots: category sidebar with pill selection, Ctrl+K component search, theme toggle,
code snippets, state matrices (rest/hover/pressed/focus/disabled/loading), status bar with page count
and runtime. Pages live at `src/pages/<category>/<name>.page.tsx` and are built alongside each
component. It is also the visual-QA target.

## 9. Agent folders

_Built in: M2 (early, so later work follows the rules)_

Rules are authored once in `.claude/rules/*.md`; `bun run agents:sync` generates `.agents/rules/` and
`.cursor/rules/*.mdc`; `bun run check` fails on divergence. Tool-specific content:

- `.claude/` — rules, agents (code-reviewer, security-reviewer, ui-visual-qa, design-system-engineer,
  tauri-rust-engineer, docs-writer, release-manager), skills, commands, hooks (format-on-edit,
  guard-generated, session-start), memory, settings.
- `.agents/` (Antigravity) — workflows, skills, hooks, persona-light config.
- `.cursor/` — commands, hooks, `mcp.json`, `extensions.json`; `.cursorrules` removed.
- `AGENTS.md` and `CLAUDE.md` at the root are concise indexes pointing at the rules.

The exact Antigravity folder conventions are verified against its docs at planning.

## 10. Quality, release, CI

_Built in: M1 (quality gate, baseline CI) and M9 (package, release, full CI)_

- **`bun run check` (via Turborepo):** Biome, tsc, cspell, knip, naming/structure check, token drift,
  agent-sync drift, dependency-exception check, attribution check, `cargo fmt --check`, clippy with
  warnings denied, cargo-deny. Lefthook runs the fast subset on commit; commitlint enforces Conventional
  Commits.
- **CI** mirrors it on Windows (`.github/workflows/`).
- **`bun run package`** builds the launcher and stages a complete `installDir` (exe, `other/launcher/`
  template, empty `programs/` and `storage/` tree), zips it to `release/launcher/`, and rotates old
  builds into `release/.archive/`. `bun run version` bumps every manifest at once.

## 11. Phase 1 build order (authoritative)

Each section above is tagged with the milestone that builds it. A milestone starts only when its
dependencies are done. Docs are written inside the milestone whose behavior they describe, not at the end.

| # | Milestone | Depends on | Delivers |
|---|---|---|---|
| **M0** | **Research and version resolution** | — | Resolve the latest stable of every tool and dependency (npm, crates.io, bun, Rust, Tauri); verify the §12 items (`.config/` support, PortableApps.com and portapps.io formats, Antigravity conventions, Tauri transparent-popup and WebView2 specifics); record findings and any dependency exceptions. Also decide: IPC type generation (tauri-specta or ts-rs), release-age cooldown support, and the performance targets (§13.3). Output: the pinned catalog and updates to this spec. |
| **M1** | **Monorepo foundation** | M0 | Fix skeleton typos (§3); root `package.json` / `Cargo.toml` / `turbo.json` / `bunfig.toml` / toolchain; move configs into `.config/`; `scripts/` framework and the commands that can already work (`setup`, `check`, `format`, `test`, `clean`, `version`, `deps`, `attribution`); test conventions (§5) with the no-test-code-in-`src` check; `crates/testing` fixtures; naming/structure check; workspace lints and strict compiler options from §13.1; security tooling from §13.2 (cargo-deny, dependency audit, secret scan, CI SHA-pinning); lefthook + commitlint; **baseline CI** running `check` + `test` on Windows. |
| **M2** | **Agent rules and tooling** | M1 | `.claude/rules` (naming, tests, TypeScript, Rust, security, performance, Tauri, attribution, design-system contract — encoding all of §13), `AGENTS.md` / `CLAUDE.md` indexes, hooks (format-on-edit, guard-generated, session-start), agents and skills, `bun run agents:sync` with drift check, `.agents/` and `.cursor/`. Done this early so all later work follows the rules. |
| **M3** | **Tokens and TS/Vite config** | M1 | `packages/tokens` (tokens, Polar Night + Snow Storm themes, contrast validator, generator), generated CSS/Tailwind/TS/JSON, `crates/design-tokens`, drift check; `config-typescript`, `config-vite`. |
| **M4** | **Design system + example webapp** (parallel with M5) | M2, M3 | (a) `programs/webapp/example` shell first: Vite dev server, sidebar, Ctrl+K search, theme toggle, page chrome, `bun run example`. (b) Design system foundations. (c) Components in waves, each with its kit page and unit tests, visual QA in both themes per wave: window/layout/actions → inputs → overlays and menus → navigation/display/feedback. |
| **M5** | **Rust core** (parallel with M4) | M1 | `genslate-paths` (all modes unit-tested via injected environment), then `launcher-core` in dependency order: `config` → `catalog` (with `icons`) → `library` → `launch` → `platform` traits. Reserved trait seams for installer, vault and eject. |
| **M6** | **IPC contract and bridge** | M3, M5 | Typed IPC command/event contract (core ↔ UI), `packages/tauri-bridge` client, and the browser mock that implements the same interface. |
| **M7** | **Tauri shell** | M5, M6 | `src-tauri`: window, tray, hotkey, autostart, `launcher-icon://`, commands delegating to core, WebView2 cache path. After this, `bun run dev` opens the real launcher against the mock `installDir`. |
| **M8** | **Launcher UI** | M4, M6 (and M7 for native QA) | Features in order: `frame` → `apps` (source tabs, grouped/searchable list, context menu) → `command-bar` → `rail` (documents) → `status` → `settings` (writes `settings.toml`) → `tools` (expand view; installer, vault and eject appear as "coming soon" entry points). Developed first against the browser mock, then QA'd natively. |
| **M9** | **Packaging, CI, release hardening** | M7, M8 | `bun run package` (staged `installDir` zip, archive rotation), full CI matrix and release workflow, manual QA checklist, `other/launcher/documents/`, code-signing if a certificate is available, final code and security review, threat-model doc. |

**Every milestone ends with the exit gate in §13.5** (check + test green, security review, performance targets, docs).

**Parallelism:** M4 and M5 are independent and can run side by side. M2 can overlap the end of M1.
**Command availability:** `dev` works from M7, `package` from M9; earlier milestones only claim the commands
they can actually run.

## 12. Verification list (executed in M0)

- Latest stable versions of all tools and dependencies.
- Tool support for `.config/` (Turborepo, rustfmt, Cargo, lefthook).
- PortableApps.com `appinfo.ini` and launcher env-var conventions; portapps.io layout.
- Antigravity workspace folder conventions.
- Tauri 2 specifics for a transparent popup + expand behavior on Windows, and WebView2 data-folder control.

## 13. Engineering standards (apply to every milestone)

_Built in: encoded as rules in M2, enforced by tooling from M1, checked at every milestone exit gate_

Security and performance are **first-class requirements, not polish**. Where a standard can be enforced
by a tool (lint, compiler flag, CI check) it is; the rest is written into `.claude/rules/` and reviewed
by the `code-reviewer` and `security-reviewer` agents at each milestone exit. Specific tools and flags
below are confirmed against current docs in M0.

### 13.1 Modern syntax and methods

- **TypeScript (latest stable):** strictest compiler options (`strict`, `noUncheckedIndexedAccess`,
  `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `noImplicitOverride`), ESM only, `satisfies`,
  `as const`, discriminated unions, `import type`; no `any`, non-null `!`, enums, default exports or
  CommonJS. Modern built-ins preferred (`toSorted`, `structuredClone`, `Object.groupBy`, `using` where supported).
- **React 19:** `ref` as a prop, `use`, Actions/`useTransition`, Suspense and error boundaries, React
  Compiler on; no `forwardRef`, no `useEffect` for derived state or data fetching, no reflexive `useMemo`.
- **CSS:** Tailwind v4 CSS-first config; native CSS (`@layer`, container queries, `color-mix`, `:has`)
  over JS where possible; token utilities only.
- **Rust (latest stable, edition 2024):** let-chains, `let … else`, async fn in traits, `LazyLock` /
  `OnceLock`, `std` over third-party crates where equivalent. Workspace `[workspace.lints]` with clippy
  `pedantic` (selected `nursery`) and `unwrap_used`, `expect_used`, `panic`, `todo`, `dbg_macro` denied.
- **Bun:** `bun run`, `bun test`, `bun x`; Bun APIs for scripts (`Bun.file`, `Bun.$`) instead of Node shims.

### 13.2 Security (priority)

- **Tauri hardening:** per-window least-privilege capabilities with no wildcard permissions; strict CSP
  (no `unsafe-inline`, no `unsafe-eval`); no remote content or navigation; `withGlobalTauri` off; no
  `dangerous*` flags; scoped asset protocol; single-instance plugin; custom protocols
  (`launcher-icon://`) sanitize every path; no shell/opener scopes wider than needed.
- **IPC is the trust boundary.** Every command validates every argument in Rust. The UI sends **app ids,
  never raw paths or commands**; Rust resolves the id to an executable the scanner found under
  `programs/`. Processes are spawned with argument arrays, never through a shell.
- **Path safety:** canonicalize and verify every path stays inside `<install>` (traversal, symlinks and
  junctions, UNC paths, 8.3 names, reserved device names, alternate data streams). Config files have
  size limits and unknown keys are rejected.
- **`unsafe`:** `#![forbid(unsafe_code)]` everywhere except an isolated `platform/` module for Windows
  FFI, which prefers the safe wrappers in the `windows` crate; any `unsafe` carries a `// SAFETY:`
  comment and is reviewed explicitly.
- **Supply chain:** exact pins, committed lockfiles, `bun install --frozen-lockfile` in CI, cargo-deny
  (advisories, licenses, bans, sources), dependency audit in `check`, a release-age cooldown on new
  versions if the tool supports it (balances the latest-versions policy), CI actions pinned by commit SHA,
  least-privilege workflow tokens, secret scanning in hooks and CI.
- **Constraints on later specs:** the installer uses HTTPS only, hash/signature verification and
  extraction path validation (no zip-slip); the vault uses audited crypto crates with zeroized key
  material and no home-grown cryptography; eject/cleanup never deletes outside `<install>` or app-owned
  host traces.
- **Release:** sign the executable if a certificate is available (M9); a written threat model lives in
  `other/launcher/documents/security.md`.

### 13.3 Performance (priority)

- **Targets recorded and tracked** (numbers set in M0, measured from M7 on): hotkey-to-visible, cold start to
  interactive, idle memory, and UI bundle size. Bundle-size budgets are checked in CI.
- **Popup feels instant:** the webview is preloaded and kept warm while hidden; non-critical features
  (settings, tools) are lazy-loaded; long lists are virtualized; animations touch only `transform` and
  `opacity`; self-hosted variable fonts only.
- **Removable-drive reality:** USB sticks have slow I/O and limited write endurance. Scans run off the
  main thread with bounded concurrency, are incremental and cached by mtime fingerprint, and file watching
  is debounced. Writes are **atomic (temp file + rename)**, batched and debounced, so an unplug can't
  corrupt state and flash wear stays low. IPC calls never block; large results stream as events.
- **Build:** Rust release profile tuned (LTO, `codegen-units = 1`, `strip`), Turborepo caching with exact
  `inputs`, minimal dependencies.

### 13.4 Proper practices by area

- **Repo:** Conventional Commits, `.changes` changelog entries, semver, protected `main`, CODEOWNERS and PR
  templates, LF line endings via `.gitattributes`, no generated or secret files committed, every rule
  enforced by the check script.
- **Rust:** `thiserror` in libraries (`anyhow` only in binaries), `tracing` structured logs, no global
  mutable state, dependency injection through traits, small public APIs, deterministic and testable
  logic, `cargo nextest` as the runner, coverage via a tool in `.config/`.
- **Front end:** accessibility (WCAG 2.2 AA) as a gate, error boundaries, minimal state with Rust as the
  source of truth (a small store such as `useSyncExternalStore`, no heavy state library unless M0 shows a
  need), no direct network calls from the UI.
- **Tauri shell:** thin, async commands, shared state behind `State<Arc<…>>`, events for push updates, a
  panic hook that logs, graceful shutdown that terminates tracked processes only when asked.
- **Backend (`launcher-core`):** ports-and-adapters (traits at the Windows and filesystem edges), pure
  logic in the core, structured typed errors, cancellation for long tasks, atomic file writes.
- **IPC types:** generated from Rust so the contract has one source of truth (tauri-specta or ts-rs,
  chosen in M0), plus a drift check in `bun run check`.

### 13.5 Milestone exit gate

A milestone is done only when: `bun run check` and `bun run test` are green; its security checklist
(§13.2) is reviewed by the `security-reviewer`; any performance target it touches is measured and
within budget; docs for its behavior are written; and nothing in the milestone breaks the naming (§4)
or test (§5) rules.
