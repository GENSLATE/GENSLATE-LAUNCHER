# Plan 1 — Foundation (M0–M2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the empty GENSLATE-LAUNCHER skeleton into a working Turborepo + bun + Cargo monorepo with a full quality gate, a Rust test-fixture crate, CI, and agent rules synced across Claude, Cursor and Antigravity.

**Architecture:** Root bun workspace (exact pins in `workspaces.catalog`) orchestrated by Turborepo; one Cargo workspace exposed to Turborepo as the `@genslate/rust` package (`crates/package.json`). Every root command is a small Bun script in `scripts/commands/` built from pure, unit-tested helpers in `scripts/lib/`. Agent rules are authored once in `.claude/rules/` and generated into `.cursor/rules/` and `.agents/rules/`; the same guard scripts back each tool's hooks, with lefthook and `bun run check` as the tool-independent safety net.

**Tech Stack:** bun 1.4.2 · Turborepo 2.11.7 · TypeScript 7.0.2 · Biome 2.5.15 · cspell 10.3.6 · knip 6.39.0 · secretlint 13.0.7 · lefthook 2.1.16 · commitlint 21.2.3 · Rust 1.99.0 (edition 2024) · cargo-nextest 0.9.146 · cargo-deny 0.20.2 · tempfile 3.27.0

**Spec:** `docs/superpowers/specs/2026-10-07-genslate-launcher-design.md` (§3, §4, §9, §10, §11, §12 rows M0–M2)

## Global Constraints

- **bun only:** `bun install`, `bun run <cmd>`, `bun x <bin>`, `bun test`. Never npm, npx, pnpm, yarn or node — in code, docs, hooks or CI.
- **Latest stable everything, with a 3-day release-age cooldown:** "latest" means the newest stable `x.y.z` published at least 259 200 s (3 days) ago. `bunfig.toml` enforces it for npm (`minimumReleaseAge = 259200`) and `bun run deps` applies the same rule to crates. Older pins need an entry in `.config/dependency-exceptions.toml`.
- **Exact pins:** npm versions live only in the root `workspaces.catalog` (packages use `"catalog:"`); crates only in root `[workspace.dependencies]` as `"=x.y.z"`.
- **Tool config in `.config/`** wherever the tool can read it there; root-only files: `package.json`, `bun.lock`, `bunfig.toml`, `turbo.json`, `tsconfig.json`, `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml`, `rustfmt.toml`, `.cargo/config.toml`, `.gitignore`, `.gitattributes`, `.editorconfig`.
- **Naming:** files `<subject>.<kind>.<ext>` in lowercase kebab-case; Rust files `snake_case.rs`; TypeScript files under any `src/` or `scripts/lib/` must end in a known kind (Task 9).
- **Tests only in `tests/unit/` or `tests/e2e/`** of their project (`scripts/tests/unit/` for scripts); no `#[cfg(test)]`/`#[test]` in Rust `src/`.
- **TypeScript:** strictest options, ESM, named exports only (config files are the only default exports), `import type`, no `any`, no non-null `!`, no enums.
- **Rust:** edition 2024; clippy `pedantic` + `unwrap_used`, `expect_used`, `panic`, `todo`, `dbg_macro` denied; `unsafe_code` forbidden; tests return `Result` instead of unwrapping.
- **Commits:** Conventional Commits `type(scope): subject`, scopes from `scripts/lib/commits/commit.constants.ts`. **No agent attribution anywhere** — no `Co-Authored-By` trailers, no "Generated with" footers, no session links, no `claude/`-style branch names.
- **Windows-first:** paths may contain spaces and backslashes; spawn processes with argument arrays (`Bun.spawn([...])`), never a shell string.

## Review Focus

1. **Windows paths in hook input** — `S:\DEVELOPMENT\...\bun.lock` with backslashes or a lower-case drive letter must still be recognised as the repo's `bun.lock` and blocked. Pinned by tests in Task 7 (`toRepoRelative`) and Task 19.
2. **CRLF files** — a rule authored in a Windows editor with `\r\n` line endings must parse and render identically to an LF file. Pinned by a test in Task 17.
3. **Offline / registry failure** — `bun run check` on a plane must not fail because npm or crates.io is unreachable; unresolved versions are warnings, not failures. Pinned by a test in Task 13.
4. **Arguments with spaces** — commands receiving paths like `Recycle Bin` or a repo under `My Projects` must pass them through intact. Pinned by a test in Task 7 (`capture` round-trip).
5. **Attribution false positives** — an ordinary commit such as `docs(agents): add Claude rules for the cursor tokens` must pass; only real trailers, footers, links and agent identities fail. Pinned by tests in Task 10.

---

## File map

| Path | Responsibility | Task |
|---|---|---|
| `docs/research/{README,versions,tooling,frontend-stack,tauri-webview2,portable-windows}.md` | M0 findings | 1–4 |
| `.gitignore`, `.gitattributes`, `.editorconfig` | Repo text and ignore policy | 5 |
| `package.json`, `bunfig.toml`, `turbo.json`, `tsconfig.json`, `crates/package.json` | Workspace, pins, task graph | 6 |
| `scripts/lib/repo/{repo,files}.util.ts` | Repo root, path conversion, file listing | 7 |
| `scripts/lib/process/{exec,steps}.util.ts` | Spawning commands, running step lists | 7 |
| `.config/{biome,cspell,knip,secretlint}.json`, `.config/cspell/project-words.txt`, `scripts/commands/format.ts` | Lint/format config | 8 |
| `scripts/lib/structure/structure.rules.ts`, `scripts/commands/structure.ts` | Naming + test-location check | 9 |
| `scripts/lib/attribution/attribution.rules.ts`, `scripts/commands/attribution.ts` | Attribution check | 10 |
| `scripts/lib/commits/{commit.constants,change.render}.ts`, `.config/commitlint.config.ts`, `scripts/commands/change.ts` | Commit types/scopes, change entries | 11 |
| `Cargo.toml`, `rust-toolchain.toml`, `rustfmt.toml`, `.cargo/config.toml`, `.config/cargo/deny.toml`, `crates/testing/**` | Rust workspace + fixtures crate | 12 |
| `scripts/lib/semver/semver.util.ts`, `scripts/lib/deps/{deps.registry,deps.policy}.ts`, `scripts/lib/rust/rust-tools.constants.ts`, `.config/dependency-exceptions.toml`, `scripts/commands/deps.ts` | Dependency policy | 13 |
| `scripts/lib/version/version.util.ts`, `scripts/lib/clean/clean.util.ts`, `scripts/commands/{version,clean}.ts` | Version bump, clean | 14 |
| `scripts/commands/{setup,test,check}.ts`, `.config/lefthook.yml` | Orchestration + git hooks | 15 |
| `.vscode/*`, `.github/**` | Editor + CI + repo community files | 16 |
| `scripts/lib/agents/{rule.model,rule.render}.ts` | Rule parsing and per-tool rendering | 17 |
| `scripts/agents/sync.ts`, `scripts/lib/agents/sync.plan.ts` | `agents:sync` + drift check | 18 |
| `scripts/lib/agents/{hook-input.parse,hook-output.util,generated.rules}.ts` | Hook protocol helpers | 19 |
| `scripts/agents/hooks/*.hook.ts`, `.claude/settings.json`, `.cursor/hooks.json`, `.agents/hooks.json` | Guards wired into each tool | 20 |
| `.claude/rules/*.md`, `AGENTS.md`, `CLAUDE.md` | Shared rules (synced) | 21 |
| `.claude/{agents,skills,commands,memory}/**`, `.mcp.json`, `.claude/README.md` | Claude-only tooling | 22 |
| `.cursor/{commands,mcp.json,README.md}`, `.cursorignore`, `.agents/{workflows,skills,README.md}` | Cursor + Antigravity tooling | 23 |
| `README.md`, `scripts/README.md` | Docs + milestone exit | 24 |

Skeleton placeholders that belong to later milestones (`packages/*`, `crates/{paths,launcher-core,design-tokens}`, `programs/desktop/launcher/**`, `scripts/commands/{build,dev,package}.ts`, `.changes/{config.yaml,templates/*,releases/*}`, `CHANGELOG.md`, `LICENSE`, `.git-blame-ignore-revs`) stay as they are, except the empty JSON files Task 5 makes valid.

---

## M0 — Research and versions

### Task 0: Developer toolchain (one-time, needs the user's approval)

This machine has bun 1.4.2 and WebView2 154 but **no Rust toolchain and no MSVC build tools**. Installing host software is the user's call — ask before running these.

- [ ] **Step 1: Install MSVC Build Tools (C++ workload)**

```powershell
winget install --id Microsoft.VisualStudio.BuildTools --source winget --override "--wait --passive --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

- [ ] **Step 2: Install rustup**

```powershell
winget install --id Rustlang.Rustup --source winget
```

- [ ] **Step 3: Open a new terminal and verify**

Run:
```powershell
rustup --version
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -products * -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property displayName
```
Expected: a `rustup 1.29.x` line, and `Visual Studio Build Tools 2026` (or later). The Rust toolchain itself is installed from `rust-toolchain.toml` in Task 12. Nothing to commit.

### Task 1: Record versions and tooling findings

**Files:**
- Create: `docs/research/README.md`, `docs/research/versions.md`, `docs/research/tooling.md`

**Interfaces:**
- Produces: the pin table every later task copies; the `.config/` support matrix Tasks 8, 12 and 15 rely on.

- [ ] **Step 1: Re-verify the npm pins are still the newest eligible versions**

Run (PowerShell or bash, from the repo root):
```bash
bun -e "for (const n of ['turbo','typescript','@biomejs/biome','cspell','knip','secretlint','@secretlint/secretlint-rule-preset-recommend','lefthook','@commitlint/cli','@commitlint/config-conventional','@commitlint/types','@types/bun']) { const j = await (await fetch('https://registry.npmjs.org/' + n.replace('/', '%2F'))).json(); const ok = Object.entries(j.time).filter(([v, t]) => /^\d+\.\d+\.\d+$/.test(v) && Date.now() - Date.parse(t) >= 259200000).sort((a, b) => Date.parse(b[1]) - Date.parse(a[1])); console.log(n, ok[0][0]); }"
```
Expected: the versions in the table below. If a newer eligible version is printed, use it in this file **and** in Task 6's catalog (Task 13's `deps --strict` would otherwise fail later).

- [ ] **Step 2: Write `docs/research/README.md`**

```markdown
# Research notes

Findings from milestone M0 of the master spec (`docs/superpowers/specs/2026-10-07-genslate-launcher-design.md` §11).
Each file answers one group of verification items, cites its sources and states the decision it leads to.
When a finding contradicts the spec, it is raised with the user before the affected milestone starts.

| File | Spec §11 items | Used by |
|---|---|---|
| `versions.md` | 1 | Plan 1 (pins), `bun run deps` |
| `tooling.md` | 2, 8, 9, 10 | Plan 1 |
| `frontend-stack.md` | 3, 4, 7 | Plan 2, Plan 3 (M8) |
| `tauri-webview2.md` | 5, 6 | Plan 3 |
| `portable-windows.md` | 11, 12 | Plan 3 |
```

- [ ] **Step 3: Write `docs/research/versions.md`**

```markdown
# Versions (resolved 2026-10-07)

"Latest" = newest stable `x.y.z` published at least 3 days ago (bunfig `minimumReleaseAge = 259200`).
`bun run deps` re-checks these with the same rule; `bun run check` fails on a stale pin without an
exception in `.config/dependency-exceptions.toml`.

## Toolchains

| Tool | Version | Pinned in |
|---|---|---|
| bun | 1.4.2 | `package.json` `packageManager` |
| Rust | 1.99.0 (2026-09-28) | `rust-toolchain.toml` |
| rustup | 1.29.1 | developer machine (winget) |
| MSVC Build Tools | 2026 (18.x) | developer machine (winget) |

## npm (root `workspaces.catalog`)

| Package | Version | Note |
|---|---|---|
| turbo | 2.11.7 | |
| typescript | 7.0.2 | native (Go) compiler; no programmatic API until 7.1 |
| @biomejs/biome | 2.5.15 | |
| cspell | 10.3.6 | |
| knip | 6.39.0 | 6.40.0 is inside the cooldown |
| secretlint | 13.0.7 | |
| @secretlint/secretlint-rule-preset-recommend | 13.0.7 | |
| lefthook | 2.1.16 | 2.1.17 is inside the cooldown |
| @commitlint/cli | 21.2.3 | |
| @commitlint/config-conventional | 21.2.3 | |
| @commitlint/types | 21.2.3 | |
| @types/bun | 1.4.2 | |

## Crates (root `[workspace.dependencies]`)

| Crate | Version |
|---|---|
| tempfile | 3.27.0 |

## Cargo tools (`scripts/lib/rust/rust-tools.constants.ts`)

| Tool | Version |
|---|---|
| cargo-nextest | 0.9.146 |
| cargo-deny | 0.20.2 |

## GitHub Actions (pinned by commit SHA)

| Action | Tag | SHA |
|---|---|---|
| actions/checkout | v7.0.1 | 3d3c42e5aac5ba805825da76410c181273ba90b1 |
| oven-sh/setup-bun | v2.2.0 | 0c5077e51419868618aeaa5fe8019c62421857d6 |
| Swatinem/rust-cache | v2.9.2 | 6323deb102c322ba6fcbdcafc7e3dddab59af2b6 |
| taiki-e/install-action | v2.87.26 | f7e5d7c961414b23f5b25b2da9294395d08513ad |
| github/codeql-action | v4.38.2 | 2892aa5e19bbd11bc0cff5427e3b750a04d9e3c2 |

## Recorded for later plans (not pinned yet)

Vite 8.3.3 · React 19.3.0 · @types/react 19.3.0 · @vitejs/plugin-react 6.1.2 · babel-plugin-react-compiler 1.0.0 ·
Tailwind CSS 4.3.3 · @base-ui/react 1.8.0 · tailwind-variants 3.3.1 · @vscode/codicons 0.0.46-24 ·
@playwright/test 1.63.0 · happy-dom 20.14.5 · @testing-library/react 16.3.3 · @testing-library/user-event 14.6.7 ·
@tauri-apps/cli and @tauri-apps/api 2.12.1 · tauri 2.12.1 · tauri-build 2.7.1 · tauri-specta 1.0.2 · specta 1.0.5 ·
serde 1.0.229 · thiserror 2.0.21 · toml_edit 0.25.15 · toml 1.1.6 · tracing 0.1.44 · windows 0.62.2.
Plans 2 and 3 re-resolve these with `bun run deps` before pinning.
```

- [ ] **Step 4: Write `docs/research/tooling.md`**

```markdown
# Tooling findings (spec §11 items 2, 8, 9, 10)

## TypeScript 7 (item 2)

- `typescript@7.0.2` is the Go-native compiler; the CLI is still `tsc` (`bun x tsc`).
- 7.0 has **no programmatic API** (planned for 7.1). Tools that need it can depend on the compatibility
  package `@typescript/typescript6` (6.0.2, exposes `tsc6` and the 6.0 API).
- knip 6.x parses with `oxc-parser` and has no TypeScript dependency, so it works with 7.0 as-is.
  Biome does not use TypeScript. **Decision:** pin `typescript@7.0.2` directly; add
  `@typescript/typescript6` only when a tool proves it needs the API (with a dependency exception).
- TS 6/7 changed defaults: `types` defaults to `[]` (list `"bun"` explicitly), `esModuleInterop` is always on,
  `moduleResolution: node10`/`classic`, `baseUrl` and `outFile` are gone.
- Sources: https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/ ·
  https://www.infoworld.com/article/4196378/go-based-typescript-7-0-arrives.html

## `.config/` support (item 8)

| Tool | Reads `.config/`? | How it is wired |
|---|---|---|
| Biome | via flag | `--config-path=.config/biome.json`; editors: `"biome.configurationPath": ".config/biome.json"`; globs in the config are relative to the repo root when the flag is used |
| cspell | via flag | `--config .config/cspell.json`; `globRoot` set to the cwd so ignore paths are repo-relative |
| knip | via flag | `--config .config/knip.json` |
| secretlint | via flag | `--secretlintrc .config/secretlint.json` |
| commitlint | via flag | `--config .config/commitlint.config.ts` |
| lefthook | **auto-discovered** | `.config/lefthook.yml` is a documented location |
| cargo-deny | via flag | `check --config .config/cargo/deny.toml` |
| Turborepo | no | `turbo.json` stays at the root |
| rustfmt | no (only project/parent dirs, `~`, and the *global* `~/.config/rustfmt/`) | `rustfmt.toml` stays at the root so rust-analyzer finds it |
| Cargo | no (`.cargo/config.toml` only) | `.cargo/config.toml` at the root |

Sources: https://lefthook.dev/configuration/ · https://github.com/rust-lang/rustfmt/blob/master/Configurations.md

## bun supply-chain features (item 9)

- `[install] minimumReleaseAge = <seconds>` in `bunfig.toml` (bun ≥ 1.3) filters out versions newer than the
  threshold during resolution; `minimumReleaseAgeExcludes` lists exempt packages. **Decision:** 259 200 s (3 days), no excludes.
- `bun audit` reports known vulnerabilities and exits non-zero when it finds any at or above `--audit-level`.
- Built-ins replace parser dependencies: `Bun.YAML.parse`, `Bun.TOML.parse`, `Bun.Glob`, `Bun.which`, `Bun.spawn`.
- Source: https://bun.sh/docs/pm

## Agent tool formats (item 10)

**Claude Code** — rules in `.claude/rules/*.md` with optional `paths:` front matter; hooks in
`.claude/settings.json` (`PreToolUse`/`PostToolUse`/`SessionStart`, `matcher`, `type: "command"`);
hook stdin is JSON with `tool_name` and `tool_input` (`file_path`, `command`); exit code 2 blocks and
feeds stderr to the agent; `$CLAUDE_PROJECT_DIR` points at the repo. MCP servers in the root `.mcp.json`.

**Cursor** — rules in `.cursor/rules/*.mdc` with front matter `description`, `globs` (comma-separated,
unquoted) and `alwaysApply`; `.cursorrules` is deprecated. Cursor also reads `AGENTS.md`.
Hooks in `.cursor/hooks.json` (`"version": 1`); project hooks run from the project root.
`preToolUse` stdin `{ tool_name, tool_input, tool_use_id, cwd }` (matchers `Shell`, `Read`, `Write`,
`Grep`, `Delete`, `Task`, `MCP:<name>`); `beforeShellExecution` stdin `{ command, cwd, sandbox }`;
`afterFileEdit` stdin `{ file_path, edits }`. Output `{ "permission": "allow"|"deny"|"ask",
"user_message", "agent_message" }`; exit 2 = deny; other non-zero exit codes fail open.
Commands in `.cursor/commands/*.md`; MCP in `.cursor/mcp.json`.

**Antigravity** — folder `.agents/` (plural is now the default; `.agent/` still works). Rules in
`.agents/rules/*.md` (immediate children only) with **required** front matter `trigger`
(`always_on` | `model_decision` | `glob` | `manual`), `description`, and `globs` as one
comma-separated string; 24 KB per file and a 20 000-token budget across always-on rules. Also reads
`AGENTS.md`. Workflows in `.agents/workflows/*.md` (front matter `description`), invoked as `/<name>`.
Skills in `.agents/skills/<name>/SKILL.md`. Hooks in `.agents/hooks.json`:
`{ "<hook-name>": { "enabled": true, "PreToolUse": [...], "PostToolUse": [...] } }`; stdin includes
`toolCall: { name, args }`; PreToolUse answers on stdout with `{ "decision": "allow"|"deny"|"ask", ... }`.
Hooks run in the Antigravity IDE, CLI and 2.0.

Sources: https://cursor.com/docs/hooks · https://www.antigravity.google/docs/rules/ ·
https://antigravity.google/docs/hooks · https://atamel.dev/posts/2025/11-25_customize_antigravity_rules_workflows/
```

- [ ] **Step 5: Commit**

```bash
git add docs/research/README.md docs/research/versions.md docs/research/tooling.md
git commit -m "docs(repo): record m0 version and tooling research"
```

### Task 2: Research the front-end stack (spec §11 items 3, 4, 7)

**Files:**
- Create: `docs/research/frontend-stack.md`

**Interfaces:**
- Produces: confirmed configuration snippets Plan 2 copies verbatim.

Use Context7 (`resolve-library-id` then `query-docs`) first, the official docs site second. Every answer cites its source URL.

- [ ] **Step 1: Answer each question in the document, using this exact outline**

```markdown
# Front-end stack findings (spec §11 items 3, 4, 7)

## Vite 8 + @vitejs/plugin-react 6 + React Compiler (item 3)
- How is the React Compiler enabled with plugin-react 6 on Vite 8 (exact plugin import and options)?
- Does plugin-react 6 still use Babel, or Rolldown/oxc; which package provides the compiler transform?
- Minimal working `vite.config.ts` (paste it), dev server port option, and `build.target` for WebView2.

## Tailwind CSS 4.3 (item 4)
- How is Tailwind wired into Vite 8 (`@tailwindcss/vite` version and usage)?
- Exact syntax for `@theme inline`, `@custom-variant`, `@utility`, and `@source` in 4.3.
- Any 4.x changes to `@theme` namespaces used for spacing, radius, shadow, z-index, cursor tokens.

## Base UI 1.8 (item 4)
- Package name and import style (`@base-ui/react/<component>`?), the `render` prop signature.
- Part names and data attributes for: Menu, ContextMenu, Popover, Tooltip, Dialog, AlertDialog, Toast,
  Tabs, Select, Checkbox, Switch, Slider, ScrollArea, Toolbar, Toggle/ToggleGroup, RadioGroup, NumberField.
- Which components are missing (e.g. command palette, tree) and must be composed by us.

## CSP without `unsafe-inline` (item 7)
- Do React 19 `style` props, Base UI positioning (inline `style` for popups) and Tailwind v4 output work
  under `style-src 'self'` with no `unsafe-inline` in WebView2? (Style attributes set through the CSSOM
  are allowed; `<style>` elements and `style=""` in HTML are not.)
- Does Vite's dev server inject inline styles/scripts that require a separate `devCsp`?
- Conclusion: the exact production `csp` and `devCsp` strings Plan 3 uses.

## Decisions
- One bullet per question above: what Plan 2/3 will do.
```

- [ ] **Step 2: Self-check**

Every question has an answer and a source link; the "Decisions" list has one bullet per question; nothing says "unknown" without a proposed way to find out.

- [ ] **Step 3: Commit**

```bash
git add docs/research/frontend-stack.md
git commit -m "docs(repo): record front-end stack research"
```

### Task 3: Research Tauri and WebView2 (spec §11 items 5, 6)

**Files:**
- Create: `docs/research/tauri-webview2.md`

- [ ] **Step 1: Answer each question, using this exact outline**

```markdown
# Tauri 2.12 and WebView2 findings (spec §11 items 5, 6)

## tauri-specta 1.0 (item 5)
- Exact crate set and features (`tauri-specta`, `specta`, `specta-typescript`?) and the builder code that
  exports TypeScript bindings for commands and events. Paste a minimal example.
- How to run the export in debug builds only and fail CI on drift (command used by `bun run check`).

## Transparent tray popup (item 5)
- Window options for a frameless, transparent, always-on-top, skip-taskbar popup on Windows
  (`transparent`, `decorations`, `shadow`, `skipTaskbar`, `alwaysOnTop`, `focus`), and known WebView2
  transparency caveats.
- Tray icon API (tauri 2.12 `tray` module): left-click vs right-click events and icon position (for anchoring
  the popup above the tray).
- Plugins and versions: global-shortcut, autostart, single-instance, opener/shell (if needed); their
  capability permission identifiers.
- Registering a custom URI scheme protocol (`launcher-icon://`) and how WebView2 exposes it
  (`http://launcher-icon.localhost`).

## WebView2 runtime selection and data folder (item 6)
- How to detect an installed Evergreen runtime before creating a window (API or registry key) from Rust.
- How to point WebView2 at a fixed-version runtime folder at startup (env var
  `WEBVIEW2_BROWSER_EXECUTABLE_FOLDER` and/or Tauri config/builder), and whether it must be set before the
  first webview is created.
- How to set the user-data folder per window in Tauri 2.12 (`data_directory` on `WebviewWindowBuilder`?).
- Fixed-version runtime download source and size for x64; licence terms for redistribution.

## Decisions
- One bullet per question above: what Plan 3 will do.
```

- [ ] **Step 2: Self-check** — every question answered with a source link; decisions listed.

- [ ] **Step 3: Commit**

```bash
git add docs/research/tauri-webview2.md
git commit -m "docs(repo): record tauri and webview2 research"
```

### Task 4: Research portable formats and Windows APIs (spec §11 items 11, 12)

**Files:**
- Create: `docs/research/portable-windows.md`

- [ ] **Step 1: Answer each question, using this exact outline**

```markdown
# Portable app formats and Windows API findings (spec §11 items 11, 12)

## PortableApps.com Format (item 11)
- Folder layout of an installed app (`<App>/App/AppInfo/appinfo.ini`, `<App>/App/AppInfo/appicon*.png|ico`,
  `<App>/<App>.exe`, `<App>/Data/`). Paste a real `appinfo.ini`.
- Sections and keys the launcher needs: `[Details] Name, AppID, Publisher, Category, Description`,
  `[Version] DisplayVersion`, `[Control] Start, Icons`, extra `[Control] Start1/Name1` entries.
- Official category list.
- Environment variables the PortableApps.com Platform sets for launched apps (`PortableApps.comDocuments`,
  `PortableApps.comMusic`, `PortableApps.comPictures`, `PortableApps.comVideos`, `PortableApps.comRoot`,
  …) — exact names and meanings.

## portapps.io (item 11)
- Folder layout of an installed portapps app (`<app>-portable.exe`, `app/`, `data/`, `<app>-portable.yml`),
  how the launcher should find the executable, name and icon.

## Windows eject and process close (item 12)
- API sequence to eject a removable or USB drive given its letter (`CM_Locate_DevNode` /
  `CM_Get_Parent` / `CM_Request_Device_EjectW` via SetupAPI/CfgMgr32), and how a veto reports the blocking
  process (`PNP_VETO_TYPE`, veto name).
- Which `windows` crate (0.62) features expose those functions.
- Graceful close of a process tree started by the launcher (`WM_CLOSE` to top-level windows, wait, then
  `TerminateProcess`), and using a Job Object to track children.

## Decisions
- One bullet per question above: what Plan 3 will do.
```

- [ ] **Step 2: Self-check** — every question answered with a source link; decisions listed.

- [ ] **Step 3: Commit**

```bash
git add docs/research/portable-windows.md
git commit -m "docs(repo): record portable format and windows api research"
```

---
## M1 — Monorepo foundation

### Task 5: Repo hygiene (skeleton fixes, ignore and text policy)

**Files:**
- Rename, delete and create the files listed in the steps below
- Create/overwrite: `.gitignore`, `.gitattributes`, `.editorconfig`

**Interfaces:**
- Produces: a repo where every tracked JSON file parses, typos are fixed, and generated/build output is ignored.

- [ ] **Step 1: Fix misspelled and misplaced names (`git mv`)**

```bash
git mv programs/desktop/launcher/src-tauri/tauri.config.json programs/desktop/launcher/src-tauri/tauri.conf.json
git mv crates/paths/src/enviroment.rs crates/paths/src/environment.rs
git mv crates/design-tokens/src/generated/theme-colors.rs crates/design-tokens/src/generated/theme_colors.rs
git mv packages/tokens/scripts/emit/emit.sharedts packages/tokens/scripts/emit/emit.shared.ts
git mv .github/codeql/coeql-config.yml .github/codeql/codeql-config.yml
git mv .claude/memory/lession-learned.md .claude/memory/lessons-learned.md
git mv lefthook.yml .config/lefthook.yml
mkdir -p .cargo && git mv .config/cargo/config.toml .cargo/config.toml
rmdir programs/desktop/launcher/src-tauri/premissions/autogenerated programs/desktop/launcher/src-tauri/premissions packages/design-system/src/utlis 2>/dev/null || true
```

(`.cargo/config.toml` must sit at the root — Cargo reads nothing from `.config/`; see `docs/research/tooling.md`. The `premissions/` and `utlis/` folders are empty and untracked; Tauri and Plan 2 create the correctly spelled ones.)

- [ ] **Step 2: Remove empty files that no tool reads, that are generated, or that a later milestone recreates**

```bash
git rm -q bun.lock Cargo.lock target/.gitkeep .cursorrules .claudeignore .agentignore \
  .claude/mcp.json .claude/rules/changes.mdc .claude/memory/agent-memory.md .claude/memory/specifications.md \
  .claude/agents/.gitkeep .claude/hooks/.gitkeep .claude/logs/.gitkeep .claude/personas/.gitkeep .claude/skills/.gitkeep .claude/workflows/.gitkeep \
  .cursor/settings.json .cursor/extesions.json .cursor/rules/changes.md \
  .cursor/agents/.gitkeep .cursor/logs/.gitkeep .cursor/personas/.gitkeep .cursor/skills/.gitkeep .cursor/workflows/.gitkeep \
  .agents/config.json .agents/mcp.json .agents/rules/changes.md .agents/commands/change.md \
  .agents/agents/.gitkeep .agents/hooks/.gitkeep .agents/logs/.gitkeep .agents/memory/.gitkeep .agents/personas/.gitkeep \
  .changes/.schema.json .config/cargo/mutants.toml .config/cargo/tarpaulin.toml \
  programs/desktop/launcher/src-tauri/gen/schemas/capabilities.json \
  .github/dependabot.yml .github/workflows/cd.yml .github/workflows/release.yml .github/workflows/verify-changes.yml \
  .github/ISSUE_TEMPLATE/custom_form.yml .github/ISSUE_TEMPLATE/legacy_bug.yml .github/ISSUE_TEMPLATE/legacy_feature.yml \
  .github/PULL_REQUEST_TEMPLATE/bugfix.md .github/PULL_REQUEST_TEMPLATE/feature.md .github/PULL_REQUEST_TEMPLATE/hotfix.md \
  .github/mergify.yml .github/labeler.yml .github/release-drafter.yml .github/settings.yml .github/stale.yml .github/FUNDING.yml \
  .github/DISCUSSION_TEMPLATE/general.md .github/DISCUSSION_TEMPLATE/ideas.md .github/DISCUSSION_TEMPLATE/q-and-a.yml \
  .github/scripts/.gitkeep
git mv .github/PULL_REQUEST_TEMPLATE/pull_request_template.md .github/pull_request_template.md
```

Why each group goes (all of these files are 0 bytes):

| Removed | Reason |
|---|---|
| `bun.lock`, `Cargo.lock` | Empty lockfiles are invalid; Task 6 and Task 12 regenerate them. |
| `target/.gitkeep`, `src-tauri/gen/**` | Build/generated output; now git-ignored. |
| `.cursorrules` | Deprecated by Cursor (`.cursor/rules/*.mdc` replaces it). |
| `.claudeignore`, `.agentignore` | Not read by Claude Code or Antigravity; the `.claude/settings.json` deny rules (Task 20) do this job. |
| `.claude/mcp.json`, `.agents/mcp.json` | Claude Code reads the root `.mcp.json` (Task 22); Antigravity's MCP config is global. |
| `*/personas`, `*/logs`, `.claude/workflows`, `.cursor/{agents,skills,workflows}`, `.agents/{agents,commands,memory}`, `.agents/config.json`, `.cursor/settings.json`, `.claude/hooks` | Not features of those tools; hook scripts live in `scripts/agents/hooks/` (Task 20). |
| `*/rules/changes.*` | The change-entry rule is folded into `git-and-attribution.md` (Task 21); `.cursor/rules/` and `.agents/rules/` become fully generated. |
| `.claude/memory/{agent-memory,specifications}.md` | Replaced by `decisions.md`, `lessons-learned.md`, `active-context.md` (Task 22). |
| `.cursor/extesions.json` | Recommendations move to `.vscode/extensions.json`, which Cursor reads (Task 16). |
| `.changes/.schema.json` | Change-entry fields are validated in code from `commit.constants.ts` (Task 11). |
| `.config/cargo/{mutants,tarpaulin}.toml` | Not in the spec; tarpaulin does not support Windows. Re-add when coverage/mutation testing is specified. |
| `.github/dependabot.yml` | The spec chose Renovate (Task 16). |
| `.github/workflows/{cd,release,verify-changes}.yml` | Empty workflows are invalid on GitHub. `ci.yml` is written in Task 16; the release workflow arrives in M10. |
| `.github/{mergify,labeler,release-drafter,settings,stale,FUNDING}.yml`, `DISCUSSION_TEMPLATE/*`, extra PR templates, legacy/custom issue forms, `.github/scripts/` | Need GitHub apps or actions the spec does not include; empty forms render as errors. The default PR template moves to `.github/pull_request_template.md` (filled in Task 16). |

- [ ] **Step 3: Make every remaining empty JSON placeholder valid**

```bash
for f in $(git ls-files '*.json'); do [ -s "$f" ] || printf '{}\n' > "$f"; done
git ls-files '*.json' | while read -r f; do [ -s "$f" ] || echo "STILL EMPTY: $f"; done
```
Expected: no `STILL EMPTY` lines. (Later tasks and plans overwrite these `{}` placeholders with real content.)

- [ ] **Step 4: Add the backups folder to the dev mock drive**

```bash
mkdir -p programs/desktop/launcher/installDir/storage/backups
touch programs/desktop/launcher/installDir/storage/backups/.gitkeep
```

- [ ] **Step 5: Write `.gitignore`**

```gitignore
# Dependencies
node_modules/

# Build output and caches
target/
dist/
.turbo/
*.tsbuildinfo

# Test output
coverage/
playwright-report/
test-results/

# Releases: keep the folder skeleton, ignore the archives
release/**
!release/.gitkeep
!release/.archive/
!release/.archive/.gitkeep

# Tauri generated schemas
**/src-tauri/gen/

# Launcher runtime state (template and dev mock drive)
programs/desktop/launcher/other/launcher/cache/**
!programs/desktop/launcher/other/launcher/cache/.gitkeep
programs/desktop/launcher/other/launcher/database/**
!programs/desktop/launcher/other/launcher/database/.gitkeep
programs/desktop/launcher/other/launcher/logs/**
!programs/desktop/launcher/other/launcher/logs/.gitkeep
programs/desktop/launcher/installDir/other/**
!programs/desktop/launcher/installDir/other/.gitkeep
programs/desktop/launcher/installDir/storage/backups/**
!programs/desktop/launcher/installDir/storage/backups/.gitkeep

# Local environment and machine files
.env
.env.*
!.env.example
.claude/settings.local.json
*.log
.DS_Store
Thumbs.db
```

- [ ] **Step 6: Write `.gitattributes`**

```gitattributes
# Normalise every text file to LF in the repository and the working tree.
* text=auto eol=lf

# Binary assets
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.webp binary
*.ico binary
*.icns binary
*.woff binary
*.woff2 binary
*.ttf binary
*.otf binary
*.zip binary
*.exe binary
*.dll binary

# Lockfiles are generated: collapse them in diffs.
bun.lock linguist-generated=true -diff
Cargo.lock linguist-generated=true -diff
```

- [ ] **Step 7: Write `.editorconfig`**

```editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.rs]
indent_size = 4

[*.md]
trim_trailing_whitespace = false
```

- [ ] **Step 8: Renormalise line endings and verify**

```bash
git add --renormalize .
git status --short | head -50
git ls-files --eol | grep -v "i/lf\|i/none\|i/-text" | head
```
Expected: the renames, deletions and new files above; the last command prints nothing (every text file is LF in the index).

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "chore(repo): fix skeleton names, drop dead placeholders and set text policy"
```

### Task 6: Root workspace, pins and task graph

**Files:**
- Create/overwrite: `package.json`, `bunfig.toml`, `turbo.json`, `tsconfig.json`, `crates/package.json`
- Generated: `bun.lock`

**Interfaces:**
- Produces: the `workspaces.catalog` every later task installs from; Turborepo tasks `build`, `typecheck`, `test`, `rust:fmt`, `rust:lint`, `rust:deny`; the `@genslate/rust` package. Root `scripts` entries are added by the task that creates each command, so no command is listed before it works.

- [ ] **Step 1: Write `package.json`**

```json
{
  "name": "genslate",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "description": "GENSLATE: a portable app launcher for Windows and the shared GENSLATE design system.",
  "packageManager": "bun@1.4.2",
  "engines": {
    "bun": ">=1.4.2"
  },
  "workspaces": {
    "packages": ["crates"],
    "catalog": {
      "@biomejs/biome": "2.5.15",
      "@commitlint/cli": "21.2.3",
      "@commitlint/config-conventional": "21.2.3",
      "@commitlint/types": "21.2.3",
      "@secretlint/secretlint-rule-preset-recommend": "13.0.7",
      "@types/bun": "1.4.2",
      "cspell": "10.3.6",
      "knip": "6.39.0",
      "lefthook": "2.1.16",
      "secretlint": "13.0.7",
      "turbo": "2.11.7",
      "typescript": "7.0.2"
    }
  },
  "scripts": {},
  "devDependencies": {
    "@biomejs/biome": "catalog:",
    "@commitlint/cli": "catalog:",
    "@commitlint/config-conventional": "catalog:",
    "@commitlint/types": "catalog:",
    "@secretlint/secretlint-rule-preset-recommend": "catalog:",
    "@types/bun": "catalog:",
    "cspell": "catalog:",
    "knip": "catalog:",
    "lefthook": "catalog:",
    "secretlint": "catalog:",
    "turbo": "catalog:",
    "typescript": "catalog:"
  }
}
```

- [ ] **Step 2: Write `bunfig.toml`**

```toml
# bun configuration — https://bun.sh/docs/runtime/bunfig

[install]
# Every version is pinned exactly; never write ranges.
exact = true
# Supply-chain cooldown: ignore versions published less than 3 days ago.
# Must equal COOLDOWN_SECONDS in scripts/lib/deps/deps.registry.ts (a test enforces it).
minimumReleaseAge = 259200

[test]
# The root only runs the scripts' tests; workspaces run their own tests through Turborepo.
root = "./scripts/tests"
```

- [ ] **Step 3: Write `turbo.json`**

```json
{
  "$schema": "https://turborepo.com/schema.json",
  "ui": "stream",
  "globalDependencies": ["tsconfig.json", "bunfig.toml"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "rust:fmt": {
      "inputs": [
        "$TURBO_ROOT$/crates/**/*.rs",
        "$TURBO_ROOT$/programs/**/*.rs",
        "$TURBO_ROOT$/rustfmt.toml"
      ],
      "outputs": []
    },
    "rust:lint": {
      "inputs": [
        "$TURBO_ROOT$/crates/**/*.rs",
        "$TURBO_ROOT$/crates/**/Cargo.toml",
        "$TURBO_ROOT$/programs/**/*.rs",
        "$TURBO_ROOT$/programs/**/Cargo.toml",
        "$TURBO_ROOT$/Cargo.toml",
        "$TURBO_ROOT$/Cargo.lock",
        "$TURBO_ROOT$/.cargo/config.toml",
        "$TURBO_ROOT$/rust-toolchain.toml"
      ],
      "outputs": []
    },
    "rust:deny": {
      "cache": false,
      "outputs": []
    },
    "@genslate/rust#build": {
      "cache": false,
      "outputs": []
    },
    "@genslate/rust#test": {
      "inputs": [
        "$TURBO_ROOT$/crates/**",
        "$TURBO_ROOT$/programs/**/*.rs",
        "$TURBO_ROOT$/programs/**/Cargo.toml",
        "$TURBO_ROOT$/Cargo.toml",
        "$TURBO_ROOT$/Cargo.lock",
        "$TURBO_ROOT$/.cargo/config.toml",
        "$TURBO_ROOT$/rust-toolchain.toml"
      ],
      "outputs": []
    }
  }
}
```

(`rust:deny` and the Rust build are never cached: the advisory database changes daily, and Cargo keeps its own incremental cache in `target/`.)

- [ ] **Step 4: Write `tsconfig.json` (root scripts and `.config/*.ts`; Plan 2 moves the shared options into `@genslate/config-typescript`)**

```json
{
  "compilerOptions": {
    "target": "esnext",
    "lib": ["esnext"],
    "module": "preserve",
    "moduleResolution": "bundler",
    "moduleDetection": "force",
    "types": ["bun"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["scripts/**/*.ts", ".config/**/*.ts"]
}
```

- [ ] **Step 5: Write `crates/package.json` (the Cargo workspace as one Turborepo package; Cargo finds the root `Cargo.toml` by walking up from `crates/`)**

```json
{
  "name": "@genslate/rust",
  "version": "0.1.0",
  "private": true,
  "description": "The GENSLATE Cargo workspace as one Turborepo package.",
  "scripts": {
    "build": "cargo build --workspace --locked",
    "test": "cargo nextest run --workspace --locked --no-tests=pass",
    "rust:fmt": "cargo fmt --all --check",
    "rust:lint": "cargo clippy --workspace --all-targets --locked -- -D warnings",
    "rust:deny": "cargo deny --manifest-path ../Cargo.toml --workspace check --config ../.config/cargo/deny.toml"
  }
}
```

- [ ] **Step 6: Install and verify the toolchain resolves**

Run:
```bash
bun install
bun x turbo --version
bun x tsc --version
bun x biome --version
```
Expected: `bun install` succeeds and writes `bun.lock`; then `2.11.7`, `Version 7.0.2`, `Version: 2.5.15`. If install reports a version as too recent for `minimumReleaseAge`, re-run Task 1 Step 1 and use the version it prints.

- [ ] **Step 7: Commit**

```bash
git add package.json bunfig.toml turbo.json tsconfig.json crates/package.json bun.lock
git commit -m "build(repo): add bun workspace, exact pins and turborepo task graph"
```

### Task 7: Script library core (repo paths, files, processes, steps)

**Files:**
- Create: `scripts/lib/repo/repo.util.ts`, `scripts/lib/repo/files.util.ts`, `scripts/lib/process/exec.util.ts`, `scripts/lib/process/steps.util.ts`
- Test: `scripts/tests/unit/repo/repo.util.test.ts`, `scripts/tests/unit/repo/files.util.test.ts`, `scripts/tests/unit/process/exec.util.test.ts`, `scripts/tests/unit/process/steps.util.test.ts`

**Interfaces:**
- Produces:
  - `REPO_ROOT: string`; `repoPath(...parts: string[]): string`; `toRepoRelative(path: string, root?: string): string | null`
  - `listRepoFiles(): Promise<string[]>` (POSIX, repo-relative, tracked + untracked-not-ignored, existing files only, sorted)
  - `run(argv: readonly string[], options?: RunOptions): Promise<number>`; `capture(argv: readonly string[], options?: RunOptions): Promise<RunResult>` with `RunOptions { cwd?: string; env?: Record<string, string> }` and `RunResult { exitCode: number; stdout: string; stderr: string }`
  - `Step { name: string; run: () => Promise<boolean> }`; `commandStep(name: string, argv: readonly string[]): Step`; `runSteps(steps: readonly Step[], log?: (line: string) => void): Promise<StepOutcome[]>` with `StepOutcome { name: string; ok: boolean; ms: number }`; `finish(outcomes: readonly StepOutcome[]): never`

- [ ] **Step 1: Write the failing tests**

`scripts/tests/unit/repo/repo.util.test.ts`
```ts
import { describe, expect, test } from 'bun:test';
import { join, resolve } from 'node:path';

import { REPO_ROOT, repoPath, toRepoRelative } from '../../../lib/repo/repo.util';

describe('toRepoRelative', () => {
  const root = resolve('/work/genslate');

  test('turns an absolute path inside the root into a POSIX relative path', () => {
    expect(toRepoRelative(join(root, 'packages', 'tokens', 'a.ts'), root)).toBe('packages/tokens/a.ts');
  });

  test('resolves relative input against the root', () => {
    expect(toRepoRelative('crates/testing/src/lib.rs', root)).toBe('crates/testing/src/lib.rs');
  });

  test('returns null for paths outside the root', () => {
    expect(toRepoRelative(resolve(root, '..', 'elsewhere.txt'), root)).toBeNull();
  });

  test.skipIf(process.platform !== 'win32')(
    'accepts backslashes and a different drive-letter case',
    () => {
      const winRoot = 'S:\\DEVELOPMENT\\PROJECTS\\GENSLATE-LAUNCHER';
      expect(toRepoRelative('s:\\DEVELOPMENT\\PROJECTS\\GENSLATE-LAUNCHER\\bun.lock', winRoot)).toBe(
        'bun.lock',
      );
    },
  );
});

describe('repoPath', () => {
  test('is anchored at the repository root', () => {
    expect(repoPath('package.json')).toBe(join(REPO_ROOT, 'package.json'));
  });

  test('REPO_ROOT holds the root package.json', async () => {
    expect(await Bun.file(join(REPO_ROOT, 'package.json')).exists()).toBe(true);
  });
});
```

`scripts/tests/unit/repo/files.util.test.ts`
```ts
import { expect, test } from 'bun:test';

import { listRepoFiles } from '../../../lib/repo/files.util';

test('lists repo files as POSIX paths and skips ignored folders', async () => {
  const files = await listRepoFiles();
  expect(files).toContain('package.json');
  expect(files).toContain('scripts/lib/repo/files.util.ts');
  expect(files.some((file) => file.includes('node_modules/'))).toBe(false);
  expect(files.some((file) => file.includes('\\'))).toBe(false);
});
```

`scripts/tests/unit/process/exec.util.test.ts`
```ts
import { expect, test } from 'bun:test';

import { capture, run } from '../../../lib/process/exec.util';

test('capture returns stdout, stderr and the exit code', async () => {
  const result = await capture(['bun', '-e', 'console.log("out"); console.error("err"); process.exit(3)']);
  expect(result.stdout.trim()).toBe('out');
  expect(result.stderr.trim()).toBe('err');
  expect(result.exitCode).toBe(3);
});

test('arguments containing spaces arrive intact (no shell)', async () => {
  const result = await capture([
    'bun',
    '-e',
    'console.log(JSON.stringify(process.argv))',
    'Recycle Bin',
    'My Projects\\a b.txt',
  ]);
  const argv = JSON.parse(result.stdout) as string[];
  expect(argv).toContain('Recycle Bin');
  expect(argv).toContain('My Projects\\a b.txt');
});

test('run resolves to the exit code', async () => {
  expect(await run(['bun', '-e', 'process.exit(0)'])).toBe(0);
  expect(await run(['bun', '-e', 'process.exit(5)'])).toBe(5);
});
```

`scripts/tests/unit/process/steps.util.test.ts`
```ts
import { expect, test } from 'bun:test';

import { commandStep, runSteps } from '../../../lib/process/steps.util';

test('runSteps runs every step, even after a failure or a throw', async () => {
  const lines: string[] = [];
  const outcomes = await runSteps(
    [
      { name: 'a', run: () => Promise.resolve(true) },
      { name: 'b', run: () => Promise.resolve(false) },
      { name: 'c', run: () => Promise.reject(new Error('boom')) },
      { name: 'd', run: () => Promise.resolve(true) },
    ],
    (line) => lines.push(line),
  );
  expect(outcomes.map((outcome) => [outcome.name, outcome.ok])).toEqual([
    ['a', true],
    ['b', false],
    ['c', false],
    ['d', true],
  ]);
  expect(lines.some((line) => line.includes('boom'))).toBe(true);
});

test('commandStep succeeds only on exit code 0', async () => {
  expect(await commandStep('ok', ['bun', '-e', 'process.exit(0)']).run()).toBe(true);
  expect(await commandStep('no', ['bun', '-e', 'process.exit(1)']).run()).toBe(false);
});
```

- [ ] **Step 2: Run the tests to see them fail**

Run: `bun test ./scripts/tests/unit/repo ./scripts/tests/unit/process`
Expected: FAIL — `Cannot find module '../../../lib/repo/repo.util'` (and the others).

- [ ] **Step 3: Implement `scripts/lib/repo/repo.util.ts`**

```ts
import { isAbsolute, relative, resolve } from 'node:path';

/** Absolute path of the repository root (this file lives in scripts/lib/repo/). */
export const REPO_ROOT = resolve(import.meta.dir, '..', '..', '..');

/** Joins path parts onto the repository root. */
export function repoPath(...parts: string[]): string {
  return resolve(REPO_ROOT, ...parts);
}

/**
 * Converts an absolute or root-relative path to a repo-relative POSIX path.
 * Returns null when the path is outside the root. Drive-letter case and backslashes are tolerated.
 */
export function toRepoRelative(path: string, root: string = REPO_ROOT): string | null {
  const rel = relative(root, resolve(root, path));
  if (rel.startsWith('..') || isAbsolute(rel)) {
    return null;
  }
  return rel.replaceAll('\\', '/');
}
```

- [ ] **Step 4: Implement `scripts/lib/process/exec.util.ts`**

```ts
import { REPO_ROOT } from '../repo/repo.util';

export interface RunOptions {
  readonly cwd?: string;
  readonly env?: Record<string, string>;
}

export interface RunResult {
  readonly exitCode: number;
  readonly stdout: string;
  readonly stderr: string;
}

function environment(options: RunOptions): Record<string, string | undefined> {
  return { ...process.env, ...options.env };
}

/** Runs a command (argument array, no shell) with inherited stdio; resolves to its exit code. */
export async function run(argv: readonly string[], options: RunOptions = {}): Promise<number> {
  const child = Bun.spawn([...argv], {
    cwd: options.cwd ?? REPO_ROOT,
    env: environment(options),
    stdio: ['inherit', 'inherit', 'inherit'],
  });
  return await child.exited;
}

/** Runs a command (argument array, no shell) and captures its output. */
export async function capture(argv: readonly string[], options: RunOptions = {}): Promise<RunResult> {
  const child = Bun.spawn([...argv], {
    cwd: options.cwd ?? REPO_ROOT,
    env: environment(options),
    stdout: 'pipe',
    stderr: 'pipe',
  });
  const [stdout, stderr, exitCode] = await Promise.all([
    new Response(child.stdout).text(),
    new Response(child.stderr).text(),
    child.exited,
  ]);
  return { exitCode, stdout, stderr };
}
```

- [ ] **Step 5: Implement `scripts/lib/repo/files.util.ts`**

```ts
import { existsSync } from 'node:fs';

import { capture } from '../process/exec.util';
import { repoPath } from './repo.util';

/** Files git tracks or sees as untracked-but-not-ignored, as sorted repo-relative POSIX paths. */
export async function listRepoFiles(): Promise<string[]> {
  const result = await capture(['git', 'ls-files', '--cached', '--others', '--exclude-standard', '-z']);
  if (result.exitCode !== 0) {
    throw new Error(`git ls-files failed: ${result.stderr.trim()}`);
  }
  const unique = new Set(result.stdout.split('\0').filter((path) => path.length > 0));
  return [...unique].filter((path) => existsSync(repoPath(path))).toSorted();
}
```

- [ ] **Step 6: Implement `scripts/lib/process/steps.util.ts`**

```ts
import { run } from './exec.util';

export interface Step {
  readonly name: string;
  readonly run: () => Promise<boolean>;
}

export interface StepOutcome {
  readonly name: string;
  readonly ok: boolean;
  readonly ms: number;
}

/** A step that runs a command and succeeds on exit code 0. */
export function commandStep(name: string, argv: readonly string[]): Step {
  return { name, run: async () => (await run(argv)) === 0 };
}

/** Runs every step in order — a failure never stops later steps — and reports each outcome. */
export async function runSteps(
  steps: readonly Step[],
  log: (line: string) => void = console.log,
): Promise<StepOutcome[]> {
  const outcomes: StepOutcome[] = [];
  for (const step of steps) {
    log(`\n▶ ${step.name}`);
    const started = performance.now();
    let ok = false;
    try {
      ok = await step.run();
    } catch (error) {
      log(`  ${step.name} threw: ${error instanceof Error ? error.message : String(error)}`);
    }
    const ms = Math.round(performance.now() - started);
    log(`${ok ? '✔' : '✖'} ${step.name} (${ms} ms)`);
    outcomes.push({ name: step.name, ok, ms });
  }
  return outcomes;
}

/** Prints a summary and exits: 0 when every step passed, 1 otherwise. */
export function finish(outcomes: readonly StepOutcome[]): never {
  const failed = outcomes.filter((outcome) => !outcome.ok);
  if (failed.length === 0) {
    console.log(`\n✔ all ${outcomes.length} steps passed`);
    process.exit(0);
  }
  const names = failed.map((outcome) => outcome.name).join(', ');
  console.error(`\n✖ ${failed.length} of ${outcomes.length} steps failed: ${names}`);
  process.exit(1);
}
```

- [ ] **Step 7: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/repo ./scripts/tests/unit/process`
Expected: PASS (the Windows-only drive-letter test runs on this machine).

- [ ] **Step 8: Typecheck**

Run: `bun x tsc -p tsconfig.json`
Expected: no output, exit code 0.

- [ ] **Step 9: Commit**

```bash
git add scripts/lib scripts/tests
git commit -m "feat(scripts): add repo, process and step helpers for root commands"
```

### Task 8: Lint, spell-check and format configuration

**Files:**
- Overwrite: `.config/biome.json`, `.config/cspell.json`, `.config/cspell/project-words.txt`, `.config/knip.json`
- Create: `.config/secretlint.json`, `scripts/commands/format.ts`
- Modify: `package.json` (`scripts.format`)

**Interfaces:**
- Consumes: `commandStep`, `runSteps`, `finish` (Task 7).
- Produces: `bun run format`; the exact lint commands Task 15's `check` runs.

- [ ] **Step 1: Write `.config/biome.json`**

```json
{
  "$schema": "../node_modules/@biomejs/biome/configuration_schema.json",
  "root": true,
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true,
    "defaultBranch": "main"
  },
  "files": {
    "ignoreUnknown": true,
    "includes": [
      "**",
      "!!**/node_modules",
      "!!**/target",
      "!!**/dist",
      "!!release",
      "!!**/src-tauri/gen",
      "!!.cursor/hooks/state",
      "!**/generated",
      "!**/*.generated.ts",
      "!.cursor/rules",
      "!.agents/rules"
    ]
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineEnding": "lf",
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "jsxQuoteStyle": "double",
      "quoteProperties": "asNeeded",
      "semicolons": "always",
      "trailingCommas": "all",
      "arrowParentheses": "always",
      "bracketSpacing": true,
      "bracketSameLine": false
    }
  },
  "json": {
    "formatter": { "trailingCommas": "none" }
  },
  "css": {
    "parser": { "tailwindDirectives": true },
    "formatter": { "quoteStyle": "single" }
  },
  "assist": {
    "enabled": true,
    "actions": {
      "source": { "organizeImports": "on" }
    }
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "complexity": {
        "noForEach": "error"
      },
      "correctness": {
        "noUnusedImports": "error",
        "noUnusedVariables": "error"
      },
      "style": {
        "noDefaultExport": "error",
        "noEnum": "error",
        "noNonNullAssertion": "error",
        "noParameterAssign": "error",
        "useConst": "error",
        "useExportType": "error",
        "useImportType": "error",
        "useNodejsImportProtocol": "error",
        "useTemplate": "error"
      },
      "suspicious": {
        "noExplicitAny": "error"
      }
    }
  },
  "overrides": [
    {
      "includes": [".config/**/*.ts", "**/*.config.ts"],
      "linter": {
        "rules": {
          "style": { "noDefaultExport": "off" }
        }
      }
    }
  ]
}
```

If Biome reports an unknown rule name, look the rule up with `bun x biome explain <ruleName>` and use the group it names; do not drop the rule.

- [ ] **Step 2: Write `.config/cspell.json`**

```json
{
  "$schema": "https://raw.githubusercontent.com/streetsidesoftware/cspell/main/cspell.schema.json",
  "version": "0.2",
  "language": "en",
  "globRoot": "${cwd}",
  "useGitignore": true,
  "dictionaryDefinitions": [
    { "name": "project-words", "path": "./cspell/project-words.txt", "addWords": true }
  ],
  "dictionaries": [
    "project-words",
    "typescript",
    "node",
    "rust",
    "softwareTerms",
    "companies",
    "misc",
    "bash",
    "powershell",
    "css",
    "html"
  ],
  "ignorePaths": [
    "**/node_modules/**",
    "**/target/**",
    "**/dist/**",
    "**/bun.lock",
    "**/Cargo.lock",
    "**/generated/**",
    "**/*.generated.ts",
    "**/src-tauri/gen/**",
    ".git/**",
    "release/**",
    ".cursor/hooks/state/**"
  ]
}
```

- [ ] **Step 3: Write `.config/cspell/project-words.txt` (one word per line, sorted)**

```text
Angeletti
appinfo
biomejs
binstall
codicon
codicons
commitlint
crt
cspell
genslate
GENSLATE
gitkeep
knip
lefthook
nextest
nord
portableapps
portapps
renormalise
rustc
rustflags
rustfmt
rustup
secretlint
secretlintrc
SLATESUITE
specta
tauri
tempfile
thiserror
turborepo
vswhere
webview
winget
```

- [ ] **Step 4: Write `.config/knip.json`**

```json
{
  "$schema": "https://unpkg.com/knip@6/schema.json",
  "ignoreExportsUsedInFile": true,
  "workspaces": {
    ".": {
      "entry": [
        "scripts/commands/*.ts",
        "scripts/agents/**/*.ts",
        "scripts/tests/**/*.test.ts",
        ".config/commitlint.config.ts"
      ],
      "project": ["scripts/**/*.ts", ".config/**/*.ts"]
    },
    "crates": {}
  },
  "ignoreDependencies": [
    "@biomejs/biome",
    "@commitlint/cli",
    "@commitlint/config-conventional",
    "@secretlint/secretlint-rule-preset-recommend",
    "cspell",
    "knip",
    "lefthook",
    "secretlint",
    "turbo",
    "typescript"
  ]
}
```

(These CLIs are invoked with `bun x` from `scripts/commands/*.ts`, which knip cannot see.)

- [ ] **Step 5: Write `.config/secretlint.json`**

```json
{
  "rules": [{ "id": "@secretlint/secretlint-rule-preset-recommend" }]
}
```

- [ ] **Step 6: Write `scripts/commands/format.ts`**

```ts
// bun run format — rewrite files with Biome (Task 12 adds rustfmt).
import { commandStep, finish, runSteps } from '../lib/process/steps.util';

const steps = [
  commandStep('biome', ['bun', 'x', 'biome', 'check', '--config-path=.config/biome.json', '--write', '.']),
];

finish(await runSteps(steps));
```

Add to `package.json` `scripts`: `"format": "bun scripts/commands/format.ts"`.

- [ ] **Step 7: Format the repo, then run Biome and cspell in check mode**

Run:
```bash
bun run format
bun x biome check --config-path=.config/biome.json --error-on-warnings .
bun x cspell lint --config .config/cspell.json --no-progress --dot "**"
```
Expected: `format` passes; Biome reports no errors. For each cspell finding: if it is a real typo, fix it; if it is a legitimate project term, add it to `project-words.txt` (sorted, one per line). Re-run cspell until it reports `Issues found: 0`.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "build(repo): configure biome, cspell, knip and secretlint in .config"
```

### Task 9: Structure check (naming and test location)

**Files:**
- Create: `scripts/lib/structure/structure.rules.ts`, `scripts/commands/structure.ts`
- Test: `scripts/tests/unit/structure/structure.rules.test.ts`

**Interfaces:**
- Consumes: `listRepoFiles`, `repoPath`, `toRepoRelative` (Task 7).
- Produces: `checkPath(path: string, content?: string): Violation[]`, `Violation { path: string; rule: StructureRule; message: string }`, `StructureRule = 'name-case' | 'ts-kind' | 'test-location' | 'rust-test-in-src'`, `KINDS: ReadonlySet<string>`; command `bun scripts/commands/structure.ts [files…]` (no arguments = whole repo).

- [ ] **Step 1: Write the failing tests** — `scripts/tests/unit/structure/structure.rules.test.ts`

```ts
import { describe, expect, test } from 'bun:test';

import { checkPath } from '../../../lib/structure/structure.rules';

const rules = (path: string, content = '') => checkPath(path, content).map((v) => v.rule);

describe('checkPath', () => {
  test('accepts well-named files', () => {
    expect(rules('packages/design-system/src/components/actions/button/button.component.tsx')).toEqual([]);
    expect(rules('packages/tokens/src/index.ts')).toEqual([]);
    expect(rules('programs/desktop/launcher/src/main.tsx')).toEqual([]);
    expect(rules('packages/tokens/src/env.d.ts')).toEqual([]);
    expect(rules('scripts/commands/check.ts')).toEqual([]);
    expect(rules('scripts/lib/repo/repo.util.ts')).toEqual([]);
    expect(rules('crates/paths/Cargo.toml')).toEqual([]);
    expect(rules('programs/desktop/launcher/README.md')).toEqual([]);
    expect(rules('crates/paths/src/install_root.rs')).toEqual([]);
    expect(rules('programs/desktop/launcher/src-tauri/build.rs')).toEqual([]);
  });

  test('requires lowercase kebab-case file and folder names', () => {
    expect(rules('packages/tokens/src/Theme.types.ts')).toEqual(['name-case']);
    expect(rules('packages/Design/src/index.ts')).toEqual(['name-case']);
  });

  test('requires snake_case Rust files', () => {
    expect(rules('crates/design-tokens/src/generated/theme-colors.rs')).toEqual(['name-case']);
  });

  test('requires a known kind for TypeScript in src/ and scripts/lib/', () => {
    expect(rules('packages/tokens/src/colors.ts')).toEqual(['ts-kind']);
    expect(rules('packages/tokens/src/colors.banana.ts')).toEqual(['ts-kind']);
    expect(rules('scripts/lib/helpers.ts')).toEqual(['ts-kind']);
  });

  test('keeps test files in tests/unit or tests/e2e', () => {
    expect(rules('packages/tokens/src/colors.test.ts')).toEqual(['test-location']);
    expect(rules('packages/tokens/tests/colors.test.ts')).toEqual(['test-location']);
    expect(rules('packages/tokens/tests/unit/colors.test.ts')).toEqual([]);
    expect(rules('programs/webapp/design-system/tests/e2e/colors.spec.ts')).toEqual([]);
  });

  test('rejects Rust test code inside src/', () => {
    const code = '#[cfg(test)]\nmod tests {\n    #[test]\n    fn works() {}\n}\n';
    expect(rules('crates/paths/src/lib.rs', code)).toEqual(['rust-test-in-src']);
    expect(rules('crates/paths/tests/unit/resolver.rs', code)).toEqual([]);
  });

  test('ignores folders outside the checked roots and excluded trees', () => {
    expect(rules('.github/ISSUE_TEMPLATE/bug_report.yml')).toEqual([]);
    expect(rules('node_modules/some-lib/Bad.ts')).toEqual([]);
    expect(rules('programs/desktop/launcher/installDir/storage/users/shared/Recycle Bin/.gitkeep')).toEqual([]);
    expect(rules('programs/desktop/launcher/other/launcher/licenses/genslate.LICENSE.md')).toEqual([]);
    expect(rules('programs/desktop/launcher/src-tauri/gen/schemas/Desktop-Schema.json')).toEqual([]);
  });
});
```

- [ ] **Step 2: Run to see it fail**

Run: `bun test ./scripts/tests/unit/structure`
Expected: FAIL — cannot find module `structure.rules`.

- [ ] **Step 3: Implement `scripts/lib/structure/structure.rules.ts`**

```ts
export type StructureRule = 'name-case' | 'ts-kind' | 'test-location' | 'rust-test-in-src';

export interface Violation {
  readonly path: string;
  readonly rule: StructureRule;
  readonly message: string;
}

/** Only these trees follow the naming rules (dot-folders and .github follow their tools' conventions). */
const CHECKED_ROOTS = ['crates/', 'packages/', 'programs/', 'scripts/'] as const;

const EXCLUDED: readonly RegExp[] = [
  /(^|\/)node_modules\//,
  /^programs\/[^/]+\/[^/]+\/installDir\//,
  /^programs\/[^/]+\/[^/]+\/other\//,
  /\/src-tauri\/(gen|icons|permissions)\//,
  /(^|\/)\.gitkeep$/,
];

/** The `<kind>` segment allowed in `<subject>.<kind>.ts(x)` under src/ and scripts/lib/. */
export const KINDS: ReadonlySet<string> = new Set([
  'client',
  'command',
  'component',
  'config',
  'constants',
  'context',
  'emitter',
  'env',
  'generated',
  'hook',
  'keys',
  'mock',
  'model',
  'page',
  'parse',
  'plan',
  'plugin',
  'policy',
  'provider',
  'recipe',
  'registry',
  'render',
  'rules',
  'schema',
  'section',
  'service',
  'state',
  'store',
  'theme',
  'tokens',
  'types',
  'util',
  'variants',
]);

const ALLOWED_NAMES: ReadonlySet<string> = new Set([
  'README.md',
  'CHANGELOG.md',
  'LICENSE',
  'Cargo.toml',
  'SKILL.md',
]);
const KIND_EXEMPT: ReadonlySet<string> = new Set(['index.ts', 'main.ts', 'main.tsx']);
const FILE_NAME = /^[a-z0-9]+(?:[-.][a-z0-9]+)*$/;
const RUST_FILE = /^[a-z0-9]+(?:_[a-z0-9]+)*\.rs$/;
const FOLDER_NAME = /^[a-z0-9]+(?:[-_.][a-z0-9]+)*$/;
const TEST_FILE = /\.(test|spec)\.tsx?$/;
const RUST_TEST_ATTRIBUTE = /#\[\s*(?:cfg\s*\(\s*test\s*\)|test)\s*\]/;

function isChecked(path: string): boolean {
  return CHECKED_ROOTS.some((root) => path.startsWith(root)) && !EXCLUDED.some((re) => re.test(path));
}

/** Checks one repo-relative POSIX path (and, for Rust, its content) against the structure rules. */
export function checkPath(path: string, content = ''): Violation[] {
  if (!isChecked(path)) {
    return [];
  }
  const violations: Violation[] = [];
  const add = (rule: StructureRule, message: string) => {
    violations.push({ path, rule, message });
  };
  const segments = path.split('/');
  const file = segments.at(-1) ?? '';
  const folders = segments.slice(0, -1);

  for (const folder of folders) {
    if (!FOLDER_NAME.test(folder)) {
      add('name-case', `folder "${folder}" must be lowercase kebab-case (snake_case in Rust crates)`);
    }
  }

  if (file.endsWith('.rs')) {
    if (!RUST_FILE.test(file)) {
      add('name-case', 'Rust files use snake_case.rs');
    }
  } else if (!ALLOWED_NAMES.has(file) && !FILE_NAME.test(file)) {
    add('name-case', 'file names are lowercase kebab-case: <subject>.<kind>.<ext>');
  }

  const inKindScope = folders.includes('src') || path.startsWith('scripts/lib/');
  const isTypeScript = /\.tsx?$/.test(file) && !file.endsWith('.d.ts');
  if (inKindScope && isTypeScript && !KIND_EXEMPT.has(file) && !TEST_FILE.test(file)) {
    const parts = file.replace(/\.tsx?$/, '').split('.');
    const kind = parts.at(-1) ?? '';
    if (parts.length < 2 || !KINDS.has(kind)) {
      add('ts-kind', `name must end in a known kind, e.g. button.component.tsx (kinds: ${[...KINDS].join(', ')})`);
    }
  }

  if (TEST_FILE.test(file)) {
    const testsAt = folders.indexOf('tests');
    const area = testsAt >= 0 ? folders[testsAt + 1] : undefined;
    if (area !== 'unit' && area !== 'e2e') {
      add('test-location', "test files live in the project's tests/unit/ or tests/e2e/ folder");
    }
  }

  if (file.endsWith('.rs') && folders.includes('src') && RUST_TEST_ATTRIBUTE.test(content)) {
    add('rust-test-in-src', 'Rust tests live in tests/unit/ or tests/e2e/, not in src/');
  }

  return violations;
}
```

- [ ] **Step 4: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/structure`
Expected: PASS.

- [ ] **Step 5: Implement `scripts/commands/structure.ts`**

```ts
// Structure check: file naming and test location. Usage: bun scripts/commands/structure.ts [files…]
import { listRepoFiles } from '../lib/repo/files.util';
import { repoPath, toRepoRelative } from '../lib/repo/repo.util';
import { checkPath, type Violation } from '../lib/structure/structure.rules';

const args = process.argv.slice(2);
const paths =
  args.length > 0
    ? args.map((arg) => toRepoRelative(arg)).filter((path): path is string => path !== null)
    : await listRepoFiles();

const violations: Violation[] = [];
for (const path of paths) {
  const file = Bun.file(repoPath(path));
  const content = path.endsWith('.rs') && (await file.exists()) ? await file.text() : '';
  violations.push(...checkPath(path, content));
}

for (const violation of violations) {
  console.error(`${violation.path}: [${violation.rule}] ${violation.message}`);
}
console.log(`structure: ${paths.length} files checked, ${violations.length} problems`);
process.exit(violations.length === 0 ? 0 : 1);
```

- [ ] **Step 6: Run it on the whole repo**

Run: `bun scripts/commands/structure.ts`
Expected: `structure: N files checked, 0 problems`, exit code 0 (Task 5 fixed the only offenders).

- [ ] **Step 7: Commit**

```bash
git add scripts/lib/structure scripts/commands/structure.ts scripts/tests/unit/structure
git commit -m "feat(scripts): add naming and test-location structure check"
```

### Task 10: Attribution check

**Files:**
- Create: `scripts/lib/attribution/attribution.rules.ts`, `scripts/commands/attribution.ts`
- Test: `scripts/tests/unit/attribution/attribution.rules.test.ts`
- Modify: `package.json` (`scripts.attribution`)

**Interfaces:**
- Consumes: `capture` (Task 7).
- Produces: `findAttribution(text: string): string[]`, `isAgentBranch(name: string): boolean`, `isAgentIdentity(identity: string): boolean`, `attributionProblems(command: string): string[]`, `parseGitLog(output: string): Commit[]` with `Commit { hash: string; author: string; committer: string; message: string }`; command `bun run attribution [--message-file <path>] [--range <rev-range>]`.

- [ ] **Step 1: Write the failing tests** — `scripts/tests/unit/attribution/attribution.rules.test.ts`

```ts
import { describe, expect, test } from 'bun:test';

import {
  attributionProblems,
  findAttribution,
  isAgentBranch,
  isAgentIdentity,
  parseGitLog,
} from '../../../lib/attribution/attribution.rules';

describe('findAttribution', () => {
  test('flags agent co-author trailers and e-mail addresses', () => {
    const found = findAttribution(
      'feat(launcher): add tray\n\nCo-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>',
    );
    expect(found).toContain('agent co-author trailer');
    expect(found).toContain('agent e-mail address');
  });

  test('flags "Generated with" footers and the robot emoji', () => {
    const found = findAttribution('🤖 Generated with [Claude Code](https://claude.com/claude-code)');
    expect(found).toContain('"Generated with" footer');
    expect(found).toContain('robot emoji');
  });

  test('flags agent session links', () => {
    expect(findAttribution('see https://claude.ai/code/session_123')).toEqual(['agent session link']);
  });

  test('allows ordinary messages that mention tools by name', () => {
    expect(findAttribution('docs(agents): add Claude rules for the cursor tokens')).toEqual([]);
    expect(findAttribution('fix: pair work\n\nCo-authored-by: Jane Doe <jane@example.com>')).toEqual([]);
  });
});

describe('branches and identities', () => {
  test('agent branch prefixes are rejected', () => {
    expect(isAgentBranch('claude/feature-x')).toBe(true);
    expect(isAgentBranch('cursor-fix')).toBe(true);
    expect(isAgentBranch('feature/claude-rules')).toBe(false);
    expect(isAgentBranch('main')).toBe(false);
  });

  test('agent identities are rejected', () => {
    expect(isAgentIdentity('Claude <noreply@anthropic.com>')).toBe(true);
    expect(isAgentIdentity('ANGELETTI <336796155+GENSLATE@users.noreply.github.com>')).toBe(false);
  });
});

describe('attributionProblems (shell commands)', () => {
  test('blocks commits carrying attribution', () => {
    expect(
      attributionProblems('git commit -m "feat: x" -m "Co-Authored-By: Claude <noreply@anthropic.com>"'),
    ).not.toEqual([]);
  });

  test('blocks agent branch names', () => {
    expect(attributionProblems('git checkout -b claude/new-thing')).toEqual([
      'agent branch name: claude/new-thing',
    ]);
    expect(attributionProblems('git switch -c cursor-fix')).toEqual(['agent branch name: cursor-fix']);
  });

  test('allows ordinary commands', () => {
    expect(attributionProblems('git commit -m "docs(agents): add Claude rules"')).toEqual([]);
    expect(attributionProblems('git switch -c feature/tray')).toEqual([]);
    expect(attributionProblems('ls -la')).toEqual([]);
  });
});

test('parseGitLog splits records and fields', () => {
  const output = [
    ['a1', 'Ann <ann@x.io>', 'Ann <ann@x.io>', 'feat: one\n'].join('\x1f'),
    ['b2', 'Bob <bob@x.io>', 'Bob <bob@x.io>', 'fix: two\n\nbody\n'].join('\x1f'),
  ].join('\x1e');
  expect(parseGitLog(output)).toEqual([
    { hash: 'a1', author: 'Ann <ann@x.io>', committer: 'Ann <ann@x.io>', message: 'feat: one' },
    { hash: 'b2', author: 'Bob <bob@x.io>', committer: 'Bob <bob@x.io>', message: 'fix: two\n\nbody' },
  ]);
});
```

- [ ] **Step 2: Run to see it fail**

Run: `bun test ./scripts/tests/unit/attribution`
Expected: FAIL — cannot find module `attribution.rules`.

- [ ] **Step 3: Implement `scripts/lib/attribution/attribution.rules.ts`**

```ts
/** Names of AI agents and their vendors that must never be credited in this repo. */
const AGENTS =
  '(?:claude|anthropic|cursor|copilot|gemini|antigravity|openai|chatgpt|codex|devin|aider)';

const PATTERNS: readonly { readonly label: string; readonly pattern: RegExp }[] = [
  { label: 'agent co-author trailer', pattern: new RegExp(`^\\s*co-authored-by:.*${AGENTS}`, 'im') },
  {
    label: '"Generated with" footer',
    pattern: new RegExp(`generated\\s+(?:with|by|using)\\b.*${AGENTS}`, 'i'),
  },
  { label: 'robot emoji', pattern: /\u{1F916}/u },
  { label: 'agent session link', pattern: /https?:\/\/(?:claude\.ai|claude\.com)\/(?:code|chat|share)\b/i },
  { label: 'agent e-mail address', pattern: /noreply@anthropic\.com/i },
];

const AGENT_BRANCH = new RegExp(`^${AGENTS}[/-]`, 'i');
const AGENT_IDENTITY = new RegExp(`\\b${AGENTS}\\b`, 'i');
const COMMIT_COMMAND = /\bgit\b[\s\S]*\bcommit\b/;
const BRANCH_COMMAND =
  /\bgit\s+(?:checkout\s+-[bB]|switch\s+-[cC]|branch(?:\s+-[mM])?)\s+([^\s;&|]+)/g;

export interface Commit {
  readonly hash: string;
  readonly author: string;
  readonly committer: string;
  readonly message: string;
}

/** Labels of every attribution pattern found in `text` (empty when clean). */
export function findAttribution(text: string): string[] {
  return PATTERNS.filter(({ pattern }) => pattern.test(text)).map(({ label }) => label);
}

export function isAgentBranch(name: string): boolean {
  return AGENT_BRANCH.test(name);
}

export function isAgentIdentity(identity: string): boolean {
  return AGENT_IDENTITY.test(identity);
}

/** Problems in a shell command an agent is about to run (used by the guard hooks). */
export function attributionProblems(command: string): string[] {
  const problems: string[] = [];
  if (COMMIT_COMMAND.test(command)) {
    problems.push(...findAttribution(command));
  }
  for (const match of command.matchAll(BRANCH_COMMAND)) {
    const name = match[1];
    if (name !== undefined && isAgentBranch(name)) {
      problems.push(`agent branch name: ${name}`);
    }
  }
  return problems;
}

/** Parses `git log --format=%H%x1f%an <%ae>%x1f%cn <%ce>%x1f%B%x1e`. */
export function parseGitLog(output: string): Commit[] {
  return output
    .split('\x1e')
    .map((record) => record.replace(/^\s+/, ''))
    .filter((record) => record.length > 0)
    .map((record) => {
      const [hash = '', author = '', committer = '', message = ''] = record.split('\x1f');
      return { hash, author, committer, message: message.trim() };
    });
}
```

- [ ] **Step 4: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/attribution`
Expected: PASS.

- [ ] **Step 5: Implement `scripts/commands/attribution.ts`**

```ts
// bun run attribution — reject agent attribution in commit messages, identities and branch names.
// Usage: attribution.ts [--message-file <path>] [--range <rev-range>]
import { parseArgs } from 'node:util';

import {
  findAttribution,
  isAgentBranch,
  isAgentIdentity,
  parseGitLog,
} from '../lib/attribution/attribution.rules';
import { capture } from '../lib/process/exec.util';

const FORMAT = '--format=%H%x1f%an <%ae>%x1f%cn <%ce>%x1f%B%x1e';

const { values } = parseArgs({
  args: process.argv.slice(2),
  options: { 'message-file': { type: 'string' }, range: { type: 'string' } },
});

const problems: string[] = [];
const messageFile = values['message-file'];

if (messageFile !== undefined) {
  const message = await Bun.file(messageFile).text();
  for (const label of findAttribution(message)) {
    problems.push(`commit message: ${label}`);
  }
} else {
  const range = values.range;
  const logArgs =
    range === undefined ? ['git', 'log', '--max-count=100', FORMAT] : ['git', 'log', range, FORMAT];
  const log = await capture(logArgs);
  if (log.exitCode !== 0) {
    console.error(`git log failed: ${log.stderr.trim()}`);
    process.exit(1);
  }
  for (const commit of parseGitLog(log.stdout)) {
    const short = commit.hash.slice(0, 7);
    for (const label of findAttribution(commit.message)) {
      problems.push(`${short}: ${label}`);
    }
    if (isAgentIdentity(commit.author)) {
      problems.push(`${short}: agent author ${commit.author}`);
    }
    if (isAgentIdentity(commit.committer)) {
      problems.push(`${short}: agent committer ${commit.committer}`);
    }
  }
  const branch = (await capture(['git', 'branch', '--show-current'])).stdout.trim();
  if (isAgentBranch(branch)) {
    problems.push(`agent branch name: ${branch}`);
  }
}

for (const problem of problems) {
  console.error(`✖ ${problem}`);
}
console.log(problems.length === 0 ? 'attribution: clean' : `attribution: ${problems.length} problems`);
process.exit(problems.length === 0 ? 0 : 1);
```

Add to `package.json` `scripts`: `"attribution": "bun scripts/commands/attribution.ts"`.

- [ ] **Step 6: Run it against the real history**

Run: `bun run attribution`
Expected: `attribution: clean`, exit 0.

- [ ] **Step 7: Commit**

```bash
git add scripts/lib/attribution scripts/commands/attribution.ts scripts/tests/unit/attribution package.json
git commit -m "feat(scripts): add attribution check for commits, identities and branches"
```

### Task 11: Commit conventions, commitlint and change entries

**Files:**
- Create: `scripts/lib/commits/commit.constants.ts`, `scripts/lib/commits/change.render.ts`, `scripts/commands/change.ts`
- Overwrite: `.config/commitlint.config.ts`
- Test: `scripts/tests/unit/commits/change.render.test.ts`
- Modify: `package.json` (`scripts.change`)

**Interfaces:**
- Produces: `COMMIT_TYPES`, `COMMIT_SCOPES` (readonly tuples), `CommitType`, `CommitScope`, `isCommitType(value: string): value is CommitType`, `isCommitScope(value: string): value is CommitScope`; `ChangeEntry { type: CommitType; scope: CommitScope; summary: string; date: Date }`; `parseChangeArgs(argv: readonly string[], date: Date): ChangeEntry | { error: string }`; `renderChange(entry: ChangeEntry): { fileName: string; content: string }`; command `bun run change <type> <scope> <summary…>` writing `.changes/unreleased/<fileName>`.

- [ ] **Step 1: Write the failing tests** — `scripts/tests/unit/commits/change.render.test.ts`

```ts
import { describe, expect, test } from 'bun:test';

import { parseChangeArgs, renderChange } from '../../../lib/commits/change.render';

const date = new Date(Date.UTC(2026, 9, 7, 13, 5, 9));

describe('parseChangeArgs', () => {
  test('builds an entry from type, scope and summary words', () => {
    expect(parseChangeArgs(['feat', 'launcher', 'Add', 'the', 'tray'], date)).toEqual({
      type: 'feat',
      scope: 'launcher',
      summary: 'Add the tray',
      date,
    });
  });

  test('rejects unknown types and scopes and a missing summary', () => {
    expect(parseChangeArgs(['oops', 'launcher', 'x'], date)).toHaveProperty('error');
    expect(parseChangeArgs(['feat', 'nowhere', 'x'], date)).toHaveProperty('error');
    expect(parseChangeArgs(['feat', 'launcher'], date)).toHaveProperty('error');
  });
});

describe('renderChange', () => {
  test('renders a dated, slugged file with front matter', () => {
    expect(
      renderChange({ type: 'feat', scope: 'launcher', summary: 'Add the tray popup!', date }),
    ).toEqual({
      fileName: '20261007-130509-launcher-add-the-tray-popup.md',
      content: '---\ntype: feat\nscope: launcher\n---\n\nAdd the tray popup!\n',
    });
  });

  test('caps the slug at 48 characters without a trailing dash', () => {
    const { fileName } = renderChange({
      type: 'fix',
      scope: 'scripts',
      summary: 'A very long summary that keeps going well past the limit of the slug',
      date,
    });
    const slug = fileName.replace('20261007-130509-scripts-', '').replace('.md', '');
    expect(slug.length).toBeLessThanOrEqual(48);
    expect(slug.endsWith('-')).toBe(false);
  });
});
```

- [ ] **Step 2: Run to see it fail**

Run: `bun test ./scripts/tests/unit/commits`
Expected: FAIL — cannot find module `change.render`.

- [ ] **Step 3: Implement `scripts/lib/commits/commit.constants.ts`**

```ts
/** Conventional Commit types allowed in this repo (commitlint and change entries share them). */
export const COMMIT_TYPES = [
  'feat',
  'fix',
  'perf',
  'refactor',
  'docs',
  'test',
  'build',
  'ci',
  'chore',
  'style',
  'revert',
] as const;

/** Commit scopes: one per project plus cross-cutting areas. Add a scope when a project is created. */
export const COMMIT_SCOPES = [
  'launcher',
  'start',
  'design-kit',
  'tokens',
  'design-system',
  'tauri-bridge',
  'config-typescript',
  'config-vite',
  'paths',
  'platform',
  'launcher-core',
  'design-tokens',
  'testing',
  'scripts',
  'repo',
  'ci',
  'deps',
  'agents',
  'docs',
  'release',
] as const;

export type CommitType = (typeof COMMIT_TYPES)[number];
export type CommitScope = (typeof COMMIT_SCOPES)[number];

export function isCommitType(value: string): value is CommitType {
  return (COMMIT_TYPES as readonly string[]).includes(value);
}

export function isCommitScope(value: string): value is CommitScope {
  return (COMMIT_SCOPES as readonly string[]).includes(value);
}
```

- [ ] **Step 4: Implement `scripts/lib/commits/change.render.ts`**

```ts
import {
  COMMIT_SCOPES,
  COMMIT_TYPES,
  type CommitScope,
  type CommitType,
  isCommitScope,
  isCommitType,
} from './commit.constants';

export interface ChangeEntry {
  readonly type: CommitType;
  readonly scope: CommitScope;
  readonly summary: string;
  readonly date: Date;
}

const SLUG_MAX = 48;

export function parseChangeArgs(argv: readonly string[], date: Date): ChangeEntry | { error: string } {
  const [type = '', scope = '', ...words] = argv;
  if (!isCommitType(type)) {
    return { error: `unknown type "${type}" (allowed: ${COMMIT_TYPES.join(', ')})` };
  }
  if (!isCommitScope(scope)) {
    return { error: `unknown scope "${scope}" (allowed: ${COMMIT_SCOPES.join(', ')})` };
  }
  const summary = words.join(' ').trim();
  if (summary.length === 0) {
    return { error: 'a summary is required: bun run change <type> <scope> <summary…>' };
  }
  return { type, scope, summary, date };
}

function slug(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+/, '')
    .slice(0, SLUG_MAX)
    .replace(/-+$/, '');
}

function stamp(date: Date): string {
  const iso = date.toISOString(); // e.g. 2026-10-07T13:05:09.000Z
  return `${iso.slice(0, 10).replaceAll('-', '')}-${iso.slice(11, 19).replaceAll(':', '')}`;
}

export function renderChange(entry: ChangeEntry): { fileName: string; content: string } {
  return {
    fileName: `${stamp(entry.date)}-${entry.scope}-${slug(entry.summary)}.md`,
    content: `---\ntype: ${entry.type}\nscope: ${entry.scope}\n---\n\n${entry.summary}\n`,
  };
}
```

- [ ] **Step 5: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/commits`
Expected: PASS.

- [ ] **Step 6: Implement `scripts/commands/change.ts`**

```ts
// bun run change <type> <scope> <summary…> — add a changelog entry to .changes/unreleased/.
import { parseChangeArgs, renderChange } from '../lib/commits/change.render';
import { repoPath } from '../lib/repo/repo.util';

const entry = parseChangeArgs(process.argv.slice(2), new Date());
if ('error' in entry) {
  console.error(`✖ ${entry.error}`);
  process.exit(1);
}
const { fileName, content } = renderChange(entry);
await Bun.write(repoPath('.changes', 'unreleased', fileName), content);
console.log(`✔ wrote .changes/unreleased/${fileName}`);
```

Add to `package.json` `scripts`: `"change": "bun scripts/commands/change.ts"`.

- [ ] **Step 7: Write `.config/commitlint.config.ts`**

```ts
// commitlint configuration — run with `bun x commitlint --config .config/commitlint.config.ts`.
import type { UserConfig } from '@commitlint/types';

import { COMMIT_SCOPES, COMMIT_TYPES } from '../scripts/lib/commits/commit.constants';

const config: UserConfig = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [...COMMIT_TYPES]],
    'scope-enum': [2, 'always', [...COMMIT_SCOPES]],
    'scope-empty': [2, 'never'],
    'header-max-length': [2, 'always', 100],
    'body-max-line-length': [2, 'always', 100],
  },
};

export default config;
```

- [ ] **Step 8: Verify commitlint accepts and rejects the right messages**

Run:
```bash
printf 'feat(launcher): add the tray popup\n' | bun x commitlint --config .config/commitlint.config.ts; echo "exit=$?"
printf 'Add stuff\n' | bun x commitlint --config .config/commitlint.config.ts; echo "exit=$?"
printf 'feat(nowhere): add stuff\n' | bun x commitlint --config .config/commitlint.config.ts; echo "exit=$?"
```
Expected: the first prints only `exit=0`; the second and third print problems (`type may not be empty` / `scope must be one of …`) and `exit=1`.

- [ ] **Step 9: Typecheck and commit**

```bash
bun x tsc -p tsconfig.json
git add scripts/lib/commits scripts/commands/change.ts scripts/tests/unit/commits .config/commitlint.config.ts package.json
git commit -m "feat(scripts): add commit types and scopes, commitlint config and change entries"
```

### Task 12: Rust workspace and the `genslate-testing` fixtures crate

**Files:**
- Overwrite: `Cargo.toml`, `rust-toolchain.toml`, `rustfmt.toml`, `.cargo/config.toml`, `.config/cargo/deny.toml`, `crates/testing/Cargo.toml`, `crates/testing/src/lib.rs`
- Create: `crates/testing/src/install_dir.rs`, `crates/testing/tests/unit.rs`
- Test: `crates/testing/tests/unit/install_dir.rs`
- Modify: `scripts/commands/format.ts` (add rustfmt)
- Generated: `Cargo.lock`

**Interfaces:**
- Produces (Rust, crate `genslate_testing`):
  - `pub const SHARED_FOLDERS: [&str; 7]` — `Desktop, Documents, Downloads, Music, Pictures, Videos, Recycle Bin`
  - `pub struct InstallDir` with `pub fn root(&self) -> &Path` and `pub fn path(&self, relative: &str) -> PathBuf`; the temp tree is deleted on drop
  - `pub struct InstallDirBuilder` with `new()`, `genslate_app(self, id: &str)`, `portableapps_app(self, folder: &str)`, `portapps_app(self, folder: &str)`, `file(self, relative: &str, contents: &str)` (all `-> Self`) and `build(self) -> std::io::Result<InstallDir>`
  - Layout built every time (spec §5.1): `Start.exe`, `programs/{genslate,portableapps.com,portapps.io}/`, `other/launcher/{configs,database,cache,logs}/`, `storage/users/shared/<SHARED_FOLDERS>/`, `storage/backups/`, `storage/vault/`
- Plan 3 (M5) consumes this crate from `genslate-paths` and `genslate-launcher-core` tests.

- [ ] **Step 1: Write the toolchain files**

`rust-toolchain.toml`
```toml
[toolchain]
channel = "1.99.0"
components = ["rustfmt", "clippy"]
targets = ["x86_64-pc-windows-msvc"]
profile = "minimal"
```

`rustfmt.toml`
```toml
# rustfmt stays at the repo root: rustfmt and rust-analyzer only search the project and its parents.
edition = "2024"
style_edition = "2024"
max_width = 100
newline_style = "Unix"
use_field_init_shorthand = true
use_try_shorthand = true
```

`.cargo/config.toml`
```toml
# Cargo reads only .cargo/config.toml, never .config/ (docs/research/tooling.md).

[target.x86_64-pc-windows-msvc]
# Link the C runtime statically so GENSLATE executables run on any Windows PC
# without the Visual C++ Redistributable (portability requirement, spec §1).
rustflags = ["-C", "target-feature=+crt-static"]
```

`Cargo.toml`
```toml
[workspace]
resolver = "3"
# Members are listed explicitly and grow as each crate is implemented (later plans add theirs).
members = ["crates/testing"]

[workspace.package]
version = "0.1.0"
edition = "2024"
rust-version = "1.99"
authors = ["GENSLATE"]
publish = false

[workspace.dependencies]
# Exact pins only ("=x.y.z"); `bun run deps` checks them against crates.io.
genslate-testing = { path = "crates/testing" }
tempfile = "=3.27.0"

[workspace.lints.rust]
unsafe_code = "forbid"

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
dbg_macro = "deny"
expect_used = "deny"
panic = "deny"
todo = "deny"
unimplemented = "deny"
unwrap_used = "deny"

[profile.release]
codegen-units = 1
lto = true
strip = true
```

`.config/cargo/deny.toml`
```toml
# cargo-deny — run through `bun run check` as
# `cargo deny --manifest-path Cargo.toml --workspace check --config .config/cargo/deny.toml`.

[graph]
targets = ["x86_64-pc-windows-msvc"]
all-features = true

[advisories]
yanked = "deny"
ignore = []

[licenses]
confidence-threshold = 0.9
allow = [
  "Apache-2.0",
  "Apache-2.0 WITH LLVM-exception",
  "BSD-2-Clause",
  "BSD-3-Clause",
  "BSL-1.0",
  "CC0-1.0",
  "ISC",
  "MIT",
  "MPL-2.0",
  "Unicode-3.0",
  "Zlib",
]

[licenses.private]
# Our own crates are publish = false.
ignore = true

[bans]
multiple-versions = "warn"
wildcards = "deny"
allow-wildcard-paths = true

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

- [ ] **Step 2: Install the pinned toolchain**

Run: `rustup toolchain install` (reads `rust-toolchain.toml`), then `cargo --version` and `cargo clippy --version`.
Expected: `cargo 1.99.0 …` and `clippy 0.1.99 …`.

- [ ] **Step 3: Create the crate skeleton so the tests can compile against it**

`crates/testing/Cargo.toml`
```toml
[package]
name = "genslate-testing"
description = "Shared test fixtures for GENSLATE crates (fake portable install trees)."
version.workspace = true
edition.workspace = true
rust-version.workspace = true
authors.workspace = true
publish.workspace = true

[dependencies]
tempfile.workspace = true

[lints]
workspace = true
```

`crates/testing/src/lib.rs`
```rust
//! Shared test fixtures for GENSLATE crates.
```

- [ ] **Step 4: Write the failing tests**

`crates/testing/tests/unit.rs`
```rust
//! Unit tests for `genslate-testing` — one module per source file, under `tests/unit/`.

#[path = "unit/install_dir.rs"]
mod install_dir;
```

`crates/testing/tests/unit/install_dir.rs`
```rust
use std::error::Error;
use std::fs;

use genslate_testing::{InstallDirBuilder, SHARED_FOLDERS};

type TestResult = Result<(), Box<dyn Error>>;

#[test]
fn builds_the_portable_layout() -> TestResult {
    let dir = InstallDirBuilder::new().build()?;
    for relative in [
        "programs/genslate",
        "programs/portableapps.com",
        "programs/portapps.io",
        "other/launcher/configs",
        "other/launcher/database",
        "other/launcher/cache",
        "other/launcher/logs",
        "storage/backups",
        "storage/vault",
    ] {
        assert!(dir.path(relative).is_dir(), "missing {relative}");
    }
    for folder in SHARED_FOLDERS {
        assert!(
            dir.path("storage/users/shared").join(folder).is_dir(),
            "missing shared folder {folder}"
        );
    }
    assert!(dir.path("Start.exe").is_file());
    Ok(())
}

#[test]
fn creates_one_folder_per_app_and_source() -> TestResult {
    let dir = InstallDirBuilder::new()
        .genslate_app("launcher")
        .portableapps_app("FirefoxPortable")
        .portapps_app("vscodium-portable")
        .build()?;
    assert!(dir.path("programs/genslate/launcher").is_dir());
    assert!(dir.path("programs/portableapps.com/FirefoxPortable").is_dir());
    assert!(dir.path("programs/portapps.io/vscodium-portable").is_dir());
    Ok(())
}

#[test]
fn writes_files_relative_to_the_root() -> TestResult {
    let dir = InstallDirBuilder::new()
        .file("other/launcher/configs/settings.toml", "[appearance]\ntheme = \"system\"\n")
        .build()?;
    let text = fs::read_to_string(dir.path("other/launcher/configs/settings.toml"))?;
    assert_eq!(text, "[appearance]\ntheme = \"system\"\n");
    Ok(())
}

#[test]
fn rejects_paths_that_leave_the_root() {
    assert!(InstallDirBuilder::new().file("../escape.txt", "x").build().is_err());
    assert!(InstallDirBuilder::new().genslate_app("../escape").build().is_err());
    assert!(InstallDirBuilder::new().file("", "x").build().is_err());
}

#[test]
fn deletes_the_tree_on_drop() -> TestResult {
    let dir = InstallDirBuilder::new().build()?;
    let root = dir.root().to_path_buf();
    drop(dir);
    assert!(!root.exists());
    Ok(())
}
```

- [ ] **Step 5: Run to see it fail**

Run: `cargo test -p genslate-testing`
Expected: FAIL to compile — `unresolved imports genslate_testing::InstallDirBuilder, genslate_testing::SHARED_FOLDERS`.

- [ ] **Step 6: Implement**

`crates/testing/src/lib.rs`
```rust
//! Shared test fixtures for GENSLATE crates.
//!
//! [`InstallDirBuilder`] creates a throw-away portable install tree (spec §5.1) in a temp folder.

mod install_dir;

pub use install_dir::{InstallDir, InstallDirBuilder, SHARED_FOLDERS};
```

`crates/testing/src/install_dir.rs`
```rust
use std::fs;
use std::io;
use std::path::{Component, Path, PathBuf};

use tempfile::TempDir;

/// The user folders inside `storage/users/shared/`.
pub const SHARED_FOLDERS: [&str; 7] = [
    "Desktop",
    "Documents",
    "Downloads",
    "Music",
    "Pictures",
    "Videos",
    "Recycle Bin",
];

const LAYOUT_DIRS: [&str; 9] = [
    "programs/genslate",
    "programs/portableapps.com",
    "programs/portapps.io",
    "other/launcher/configs",
    "other/launcher/database",
    "other/launcher/cache",
    "other/launcher/logs",
    "storage/backups",
    "storage/vault",
];

#[derive(Debug, Clone, Copy)]
enum AppSource {
    Genslate,
    PortableApps,
    Portapps,
}

impl AppSource {
    fn folder(self) -> &'static str {
        match self {
            Self::Genslate => "programs/genslate",
            Self::PortableApps => "programs/portableapps.com",
            Self::Portapps => "programs/portapps.io",
        }
    }
}

/// A portable install tree in a temporary folder, deleted when dropped.
#[derive(Debug)]
pub struct InstallDir {
    root: TempDir,
}

impl InstallDir {
    /// The install root (`<install>`).
    #[must_use]
    pub fn root(&self) -> &Path {
        self.root.path()
    }

    /// `relative` joined onto the install root (forward slashes are fine on Windows).
    #[must_use]
    pub fn path(&self, relative: &str) -> PathBuf {
        self.root.path().join(relative)
    }
}

/// Builds an [`InstallDir`] with the standard layout plus the apps and files you add.
#[derive(Debug, Default)]
pub struct InstallDirBuilder {
    apps: Vec<(AppSource, String)>,
    files: Vec<(String, String)>,
}

impl InstallDirBuilder {
    /// An empty builder: `build` creates only the standard layout.
    #[must_use]
    pub fn new() -> Self {
        Self::default()
    }

    /// Adds `programs/genslate/<id>/`.
    #[must_use]
    pub fn genslate_app(mut self, id: &str) -> Self {
        self.apps.push((AppSource::Genslate, id.to_owned()));
        self
    }

    /// Adds `programs/portableapps.com/<folder>/`.
    #[must_use]
    pub fn portableapps_app(mut self, folder: &str) -> Self {
        self.apps.push((AppSource::PortableApps, folder.to_owned()));
        self
    }

    /// Adds `programs/portapps.io/<folder>/`.
    #[must_use]
    pub fn portapps_app(mut self, folder: &str) -> Self {
        self.apps.push((AppSource::Portapps, folder.to_owned()));
        self
    }

    /// Adds a file at `relative` (parent folders are created).
    #[must_use]
    pub fn file(mut self, relative: &str, contents: &str) -> Self {
        self.files.push((relative.to_owned(), contents.to_owned()));
        self
    }

    /// Creates the tree.
    ///
    /// # Errors
    /// Returns [`io::ErrorKind::InvalidInput`] when an app or file path is empty, absolute or
    /// leaves the root, and any I/O error from creating the tree.
    pub fn build(self) -> io::Result<InstallDir> {
        let root = TempDir::new()?;
        let base = root.path();
        for dir in LAYOUT_DIRS {
            fs::create_dir_all(base.join(dir))?;
        }
        for folder in SHARED_FOLDERS {
            fs::create_dir_all(base.join("storage/users/shared").join(folder))?;
        }
        fs::write(base.join("Start.exe"), b"")?;
        for (source, name) in &self.apps {
            let relative = checked(name)?;
            fs::create_dir_all(base.join(source.folder()).join(relative))?;
        }
        for (relative, contents) in &self.files {
            let path = base.join(checked(relative)?);
            if let Some(parent) = path.parent() {
                fs::create_dir_all(parent)?;
            }
            fs::write(path, contents)?;
        }
        Ok(InstallDir { root })
    }
}

/// Accepts only non-empty relative paths made of normal components.
fn checked(relative: &str) -> io::Result<&Path> {
    let path = Path::new(relative);
    let inside = !relative.is_empty()
        && path
            .components()
            .all(|component| matches!(component, Component::Normal(_) | Component::CurDir));
    if inside {
        Ok(path)
    } else {
        Err(io::Error::new(
            io::ErrorKind::InvalidInput,
            format!("path must stay inside the install root: {relative:?}"),
        ))
    }
}
```

- [ ] **Step 7: Run the tests to see them pass**

Run: `cargo test -p genslate-testing`
Expected: `5 passed`. This also writes `Cargo.lock`.

- [ ] **Step 8: Install the pinned cargo tools and run the Rust tasks through Turborepo**

Run:
```bash
cargo install --locked cargo-nextest@0.9.146 cargo-deny@0.20.2
bun x turbo run test rust:fmt rust:lint rust:deny --filter=@genslate/rust
```
Expected: nextest runs the 5 tests; `cargo fmt --check` and clippy (`-D warnings`, pedantic) report nothing; cargo-deny prints `advisories ok, bans ok, licenses ok, sources ok`. If clippy flags pedantic lints, fix the code (do not add `allow`s).

- [ ] **Step 9: Add rustfmt to `bun run format`** — replace `scripts/commands/format.ts` with:

```ts
// bun run format — rewrite files with Biome and rustfmt.
import { commandStep, finish, runSteps } from '../lib/process/steps.util';

const steps = [
  commandStep('biome', ['bun', 'x', 'biome', 'check', '--config-path=.config/biome.json', '--write', '.']),
  commandStep('rustfmt', ['cargo', 'fmt', '--all']),
];

finish(await runSteps(steps));
```

Run: `bun run format` — Expected: both steps pass.

- [ ] **Step 10: Commit**

```bash
git add Cargo.toml Cargo.lock rust-toolchain.toml rustfmt.toml .cargo/config.toml .config/cargo/deny.toml crates/testing scripts/commands/format.ts
git commit -m "feat(testing): add cargo workspace and portable install-tree fixtures"
```

### Task 13: Dependency policy (`bun run deps`)

**Files:**
- Create: `scripts/lib/semver/semver.util.ts`, `scripts/lib/deps/deps.registry.ts`, `scripts/lib/deps/deps.policy.ts`, `scripts/lib/rust/rust-tools.constants.ts`, `.config/dependency-exceptions.toml`, `scripts/commands/deps.ts`
- Test: `scripts/tests/unit/semver/semver.util.test.ts`, `scripts/tests/unit/deps/deps.registry.test.ts`, `scripts/tests/unit/deps/deps.policy.test.ts`
- Modify: `package.json` (`scripts.deps`)

**Interfaces:**
- Consumes: `repoPath` (Task 7).
- Produces:
  - `parseVersion(text: string): Version | null` (`Version = readonly [number, number, number]`, stable `x.y.z` only); `compareVersions(a: string, b: string): number`
  - `COOLDOWN_SECONDS = 259_200`; `PublishedVersion { version: string; publishedAt: Date }`; `newestEligible(versions, now: Date, cooldownSeconds?): string | null`; `npmVersions(name)`, `crateVersions(name)` → `Promise<PublishedVersion[]>`; `rustStableVersion(): Promise<string>`
  - `Ecosystem = 'npm' | 'crates' | 'tool'`; `Pin { ecosystem; name; spec }`; `DependencyException { ecosystem; name; version; reason }`; `Finding` (kinds `current | outdated | too-new | excepted | not-exact | unknown | stale-exception`); `exactVersion(pin)`, `evaluatePins(pins, latest: ReadonlyMap<string, string>, exceptions)`, `isFailure(finding)`, `pinKey(ecosystem, name)`, `parseExceptions(toml)`, `readCatalogPins(manifest)`, `readCargoPins(manifest)`, `readToolPins(packageManager, toolchain, tools)`
  - `RUST_TOOLS = { 'cargo-nextest': '0.9.146', 'cargo-deny': '0.20.2' }`
  - command `bun run deps [--strict]`

- [ ] **Step 1: Write the failing tests**

`scripts/tests/unit/semver/semver.util.test.ts`
```ts
import { expect, test } from 'bun:test';

import { compareVersions, parseVersion } from '../../../lib/semver/semver.util';

test('parses stable x.y.z versions only', () => {
  expect(parseVersion('2.11.7')).toEqual([2, 11, 7]);
  expect(parseVersion('7.0.0-dev.20260707.2')).toBeNull();
  expect(parseVersion('^1.2.3')).toBeNull();
  expect(parseVersion('=1.2.3')).toBeNull();
});

test('compares numerically, not lexically', () => {
  expect(compareVersions('2.10.0', '2.9.9')).toBeGreaterThan(0);
  expect(compareVersions('1.0.0', '1.0.0')).toBe(0);
  expect(compareVersions('0.9.146', '0.10.0')).toBeLessThan(0);
});

test('throws on non-stable input', () => {
  expect(() => compareVersions('1.0.0-rc.1', '1.0.0')).toThrow();
});
```

`scripts/tests/unit/deps/deps.registry.test.ts`
```ts
import { expect, test } from 'bun:test';

import { COOLDOWN_SECONDS, newestEligible } from '../../../lib/deps/deps.registry';
import { repoPath } from '../../../lib/repo/repo.util';

const now = new Date('2026-10-07T12:00:00Z');
const daysAgo = (days: number) => new Date(now.getTime() - days * 86_400_000);

test('picks the highest stable version older than the cooldown', () => {
  expect(
    newestEligible(
      [
        { version: '6.38.0', publishedAt: daysAgo(20) },
        { version: '6.39.0', publishedAt: daysAgo(7) },
        { version: '6.40.0', publishedAt: daysAgo(1) },
        { version: '7.0.0-beta.1', publishedAt: daysAgo(10) },
      ],
      now,
    ),
  ).toBe('6.39.0');
});

test('orders by version, not by publish date (backports)', () => {
  expect(
    newestEligible(
      [
        { version: '2.0.0', publishedAt: daysAgo(30) },
        { version: '1.9.9', publishedAt: daysAgo(5) },
      ],
      now,
    ),
  ).toBe('2.0.0');
});

test('returns null when nothing is eligible', () => {
  expect(newestEligible([{ version: '1.0.0', publishedAt: daysAgo(1) }], now)).toBeNull();
});

test('COOLDOWN_SECONDS matches bunfig.toml minimumReleaseAge', async () => {
  const bunfig = Bun.TOML.parse(await Bun.file(repoPath('bunfig.toml')).text()) as {
    install: { minimumReleaseAge: number };
  };
  expect(bunfig.install.minimumReleaseAge).toBe(COOLDOWN_SECONDS);
});
```

`scripts/tests/unit/deps/deps.policy.test.ts`
```ts
import { describe, expect, test } from 'bun:test';

import {
  type DependencyException,
  evaluatePins,
  exactVersion,
  isFailure,
  type Pin,
  parseExceptions,
  pinKey,
  readCargoPins,
  readCatalogPins,
  readToolPins,
} from '../../../lib/deps/deps.policy';

const npm = (name: string, spec: string): Pin => ({ ecosystem: 'npm', name, spec });
const crate = (name: string, spec: string): Pin => ({ ecosystem: 'crates', name, spec });
const latest = (entries: [Pin, string][]) =>
  new Map(entries.map(([pin, version]) => [pinKey(pin.ecosystem, pin.name), version]));
const kinds = (findings: ReturnType<typeof evaluatePins>) => findings.map((finding) => finding.kind);

describe('exactVersion', () => {
  test('npm pins are bare x.y.z; crate pins are =x.y.z', () => {
    expect(exactVersion(npm('turbo', '2.11.7'))).toBe('2.11.7');
    expect(exactVersion(npm('turbo', '^2.11.7'))).toBeNull();
    expect(exactVersion(crate('tempfile', '=3.27.0'))).toBe('3.27.0');
    expect(exactVersion(crate('tempfile', '3.27.0'))).toBeNull();
  });
});

describe('evaluatePins', () => {
  const turbo = npm('turbo', '2.11.7');

  test('current, outdated and too-new pins', () => {
    expect(kinds(evaluatePins([turbo], latest([[turbo, '2.11.7']]), []))).toEqual(['current']);
    expect(kinds(evaluatePins([turbo], latest([[turbo, '2.12.0']]), []))).toEqual(['outdated']);
    expect(kinds(evaluatePins([turbo], latest([[turbo, '2.11.0']]), []))).toEqual(['too-new']);
  });

  test('an exception for the exact pinned version allows an older pin', () => {
    const exception: DependencyException = {
      ecosystem: 'npm',
      name: 'turbo',
      version: '2.11.7',
      reason: 'Waiting for upstream fix vercel/turbo#1',
    };
    const findings = evaluatePins([turbo], latest([[turbo, '2.12.0']]), [exception]);
    expect(kinds(findings)).toEqual(['excepted']);
    expect(findings.some(isFailure)).toBe(false);
  });

  test('stale exceptions fail', () => {
    const forLatest: DependencyException = { ecosystem: 'npm', name: 'turbo', version: '2.11.7', reason: 'no longer needed at all' };
    expect(kinds(evaluatePins([turbo], latest([[turbo, '2.11.7']]), [forLatest]))).toEqual([
      'current',
      'stale-exception',
    ]);
    const orphan: DependencyException = { ecosystem: 'npm', name: 'left-pad', version: '1.0.0', reason: 'nothing pins this anymore' };
    expect(kinds(evaluatePins([turbo], latest([[turbo, '2.11.7']]), [orphan]))).toEqual([
      'current',
      'stale-exception',
    ]);
  });

  test('an unreachable registry is reported but is not a failure (offline)', () => {
    const findings = evaluatePins([turbo], new Map(), []);
    expect(kinds(findings)).toEqual(['unknown']);
    expect(findings.some(isFailure)).toBe(false);
  });

  test('ranges are failures', () => {
    const findings = evaluatePins([npm('knip', '^6.39.0')], new Map(), []);
    expect(kinds(findings)).toEqual(['not-exact']);
    expect(findings.every(isFailure)).toBe(true);
  });
});

describe('parseExceptions', () => {
  test('reads [[exception]] tables', () => {
    expect(
      parseExceptions(
        '[[exception]]\necosystem = "crates"\nname = "windows"\nversion = "0.61.3"\nreason = "tauri 2.12 still links windows 0.61"\n',
      ),
    ).toEqual([
      { ecosystem: 'crates', name: 'windows', version: '0.61.3', reason: 'tauri 2.12 still links windows 0.61' },
    ]);
  });

  test('an empty file means no exceptions', () => {
    expect(parseExceptions('# nothing here\n')).toEqual([]);
  });

  test('rejects bad entries', () => {
    expect(() => parseExceptions('[[exception]]\necosystem = "pip"\nname = "x"\nversion = "1.0.0"\nreason = "long enough reason"\n')).toThrow();
    expect(() => parseExceptions('[[exception]]\necosystem = "npm"\nname = "x"\nversion = "^1"\nreason = "long enough reason"\n')).toThrow();
    expect(() => parseExceptions('[[exception]]\necosystem = "npm"\nname = "x"\nversion = "1.0.0"\nreason = "short"\n')).toThrow();
  });
});

describe('manifest readers', () => {
  test('catalog pins come from workspaces.catalog', () => {
    expect(readCatalogPins({ workspaces: { catalog: { turbo: '2.11.7' } } })).toEqual([npm('turbo', '2.11.7')]);
    expect(readCatalogPins({})).toEqual([]);
  });

  test('cargo pins skip path-only workspace crates and honour package renames', () => {
    expect(
      readCargoPins({
        workspace: {
          dependencies: {
            'genslate-testing': { path: 'crates/testing' },
            tempfile: '=3.27.0',
            serde: { version: '=1.0.229', features: ['derive'] },
            win: { package: 'windows', version: '=0.62.2' },
          },
        },
      }),
    ).toEqual([crate('tempfile', '=3.27.0'), crate('serde', '=1.0.229'), crate('windows', '=0.62.2')]);
  });

  test('tool pins come from packageManager, rust-toolchain.toml and the cargo tool list', () => {
    expect(
      readToolPins('bun@1.4.2', { toolchain: { channel: '1.99.0' } }, { 'cargo-deny': '0.20.2' }),
    ).toEqual([
      { ecosystem: 'tool', name: 'bun', spec: '1.4.2' },
      { ecosystem: 'tool', name: 'rust', spec: '1.99.0' },
      { ecosystem: 'tool', name: 'cargo-deny', spec: '0.20.2' },
    ]);
  });
});
```

- [ ] **Step 2: Run to see them fail**

Run: `bun test ./scripts/tests/unit/semver ./scripts/tests/unit/deps`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement `scripts/lib/semver/semver.util.ts`**

```ts
export type Version = readonly [number, number, number];

const STABLE = /^(\d+)\.(\d+)\.(\d+)$/;

/** Parses a stable `x.y.z` version; anything else (ranges, prereleases, `=` prefixes) is null. */
export function parseVersion(text: string): Version | null {
  const match = STABLE.exec(text);
  if (match === null) {
    return null;
  }
  return [Number(match[1]), Number(match[2]), Number(match[3])];
}

/** Negative when a < b, 0 when equal, positive when a > b. Throws on non-stable input. */
export function compareVersions(a: string, b: string): number {
  const left = parseVersion(a);
  const right = parseVersion(b);
  if (left === null || right === null) {
    throw new Error(`not a stable x.y.z version: ${left === null ? a : b}`);
  }
  return left[0] - right[0] || left[1] - right[1] || left[2] - right[2];
}
```

- [ ] **Step 4: Implement `scripts/lib/deps/deps.registry.ts`**

```ts
import { compareVersions, parseVersion } from '../semver/semver.util';

/** Supply-chain cooldown in seconds; must equal bunfig.toml [install] minimumReleaseAge. */
export const COOLDOWN_SECONDS = 259_200;

const TIMEOUT_MS = 20_000;
const USER_AGENT = 'genslate-deps (https://github.com/GENSLATE)';

export interface PublishedVersion {
  readonly version: string;
  readonly publishedAt: Date;
}

/** The highest stable version published at least `cooldownSeconds` before `now`. */
export function newestEligible(
  versions: readonly PublishedVersion[],
  now: Date,
  cooldownSeconds: number = COOLDOWN_SECONDS,
): string | null {
  const cutoff = now.getTime() - cooldownSeconds * 1000;
  const eligible = versions.filter(
    (entry) => parseVersion(entry.version) !== null && entry.publishedAt.getTime() <= cutoff,
  );
  const [newest] = eligible.toSorted((a, b) => compareVersions(b.version, a.version));
  return newest?.version ?? null;
}

async function getJson(url: string, headers: Record<string, string> = {}): Promise<unknown> {
  const response = await fetch(url, { headers, signal: AbortSignal.timeout(TIMEOUT_MS) });
  if (!response.ok) {
    throw new Error(`${response.status} ${response.statusText} for ${url}`);
  }
  return await response.json();
}

export async function npmVersions(name: string): Promise<PublishedVersion[]> {
  const body = (await getJson(`https://registry.npmjs.org/${name.replace('/', '%2F')}`)) as {
    time?: Record<string, string>;
  };
  return Object.entries(body.time ?? {})
    .filter(([version]) => version !== 'created' && version !== 'modified')
    .map(([version, published]) => ({ version, publishedAt: new Date(published) }));
}

export async function crateVersions(name: string): Promise<PublishedVersion[]> {
  const body = (await getJson(`https://crates.io/api/v1/crates/${name}/versions`, {
    'User-Agent': USER_AGENT,
  })) as { versions?: { num: string; created_at: string; yanked: boolean }[] };
  return (body.versions ?? [])
    .filter((entry) => !entry.yanked)
    .map((entry) => ({ version: entry.num, publishedAt: new Date(entry.created_at) }));
}

export async function rustStableVersion(): Promise<string> {
  const response = await fetch('https://static.rust-lang.org/dist/channel-rust-stable.toml', {
    signal: AbortSignal.timeout(TIMEOUT_MS),
  });
  const text = await response.text();
  const match = /\[pkg\.rust\]\s*\r?\nversion = "(\d+\.\d+\.\d+)/.exec(text);
  const version = match?.[1];
  if (version === undefined) {
    throw new Error('could not read the Rust stable version');
  }
  return version;
}
```

- [ ] **Step 5: Implement `scripts/lib/rust/rust-tools.constants.ts`**

```ts
/**
 * Cargo subcommands the repo needs, pinned exactly. `bun run setup` installs these versions;
 * `.github/actions/setup-env/action.yml` installs the same ones (a test keeps them in sync).
 */
export const RUST_TOOLS = {
  'cargo-nextest': '0.9.146',
  'cargo-deny': '0.20.2',
} as const satisfies Record<string, string>;
```

- [ ] **Step 6: Implement `scripts/lib/deps/deps.policy.ts`**

```ts
import { compareVersions, parseVersion } from '../semver/semver.util';

export type Ecosystem = 'npm' | 'crates' | 'tool';

export interface Pin {
  readonly ecosystem: Ecosystem;
  readonly name: string;
  readonly spec: string;
}

export interface DependencyException {
  readonly ecosystem: Ecosystem;
  readonly name: string;
  readonly version: string;
  readonly reason: string;
}

export type Finding =
  | { readonly kind: 'current'; readonly pin: Pin }
  | { readonly kind: 'outdated'; readonly pin: Pin; readonly latest: string }
  | { readonly kind: 'too-new'; readonly pin: Pin; readonly latest: string }
  | { readonly kind: 'excepted'; readonly pin: Pin; readonly latest: string; readonly reason: string }
  | { readonly kind: 'not-exact'; readonly pin: Pin }
  | { readonly kind: 'unknown'; readonly pin: Pin }
  | { readonly kind: 'stale-exception'; readonly exception: DependencyException; readonly why: string };

const FAILING: ReadonlySet<Finding['kind']> = new Set([
  'outdated',
  'too-new',
  'not-exact',
  'stale-exception',
]);

export function pinKey(ecosystem: Ecosystem, name: string): string {
  return `${ecosystem}:${name}`;
}

export function isFailure(finding: Finding): boolean {
  return FAILING.has(finding.kind);
}

/** The pinned version, or null when the spec is not an exact pin (`x.y.z`; crates `=x.y.z`). */
export function exactVersion(pin: Pin): string | null {
  if (pin.ecosystem === 'crates') {
    const version = pin.spec.startsWith('=') ? pin.spec.slice(1) : null;
    return version !== null && parseVersion(version) !== null ? version : null;
  }
  return parseVersion(pin.spec) === null ? null : pin.spec;
}

export function evaluatePins(
  pins: readonly Pin[],
  latest: ReadonlyMap<string, string>,
  exceptions: readonly DependencyException[],
): Finding[] {
  const findings: Finding[] = [];
  const findException = (pin: Pin) =>
    exceptions.find((entry) => entry.ecosystem === pin.ecosystem && entry.name === pin.name);

  for (const pin of pins) {
    const version = exactVersion(pin);
    if (version === null) {
      findings.push({ kind: 'not-exact', pin });
      continue;
    }
    const newest = latest.get(pinKey(pin.ecosystem, pin.name));
    if (newest === undefined) {
      findings.push({ kind: 'unknown', pin });
      continue;
    }
    const exception = findException(pin);
    const order = compareVersions(version, newest);
    if (order === 0) {
      findings.push({ kind: 'current', pin });
      if (exception !== undefined) {
        findings.push({ kind: 'stale-exception', exception, why: `${pin.name} is already on the latest version` });
      }
    } else if (order > 0) {
      findings.push({ kind: 'too-new', pin, latest: newest });
    } else if (exception !== undefined && exception.version === version) {
      findings.push({ kind: 'excepted', pin, latest: newest, reason: exception.reason });
    } else {
      if (exception !== undefined) {
        findings.push({
          kind: 'stale-exception',
          exception,
          why: `exception is for ${exception.version} but ${pin.name} is pinned at ${version}`,
        });
      }
      findings.push({ kind: 'outdated', pin, latest: newest });
    }
  }

  for (const exception of exceptions) {
    const pinned = pins.some((pin) => pin.ecosystem === exception.ecosystem && pin.name === exception.name);
    if (!pinned) {
      findings.push({ kind: 'stale-exception', exception, why: 'no such dependency is pinned' });
    }
  }
  return findings;
}

/** Parses `.config/dependency-exceptions.toml` (`[[exception]]` tables). */
export function parseExceptions(source: string): DependencyException[] {
  const data = Bun.TOML.parse(source) as { exception?: unknown };
  const entries = data.exception ?? [];
  if (!Array.isArray(entries)) {
    throw new Error('dependency exceptions must be [[exception]] tables');
  }
  return entries.map((entry: unknown, index) => {
    const label = `exception ${index + 1}`;
    if (typeof entry !== 'object' || entry === null) {
      throw new Error(`${label}: not a table`);
    }
    const { ecosystem, name, version, reason } = entry as Record<string, unknown>;
    if (ecosystem !== 'npm' && ecosystem !== 'crates' && ecosystem !== 'tool') {
      throw new Error(`${label}: ecosystem must be npm, crates or tool`);
    }
    if (typeof name !== 'string' || name.length === 0) {
      throw new Error(`${label}: name is required`);
    }
    if (typeof version !== 'string' || parseVersion(version) === null) {
      throw new Error(`${label} (${name}): version must be an exact x.y.z`);
    }
    if (typeof reason !== 'string' || reason.trim().length < 10) {
      throw new Error(`${label} (${name}): give a real reason (at least 10 characters)`);
    }
    return { ecosystem, name, version, reason: reason.trim() };
  });
}

export function readCatalogPins(manifest: unknown): Pin[] {
  const catalog =
    (manifest as { workspaces?: { catalog?: Record<string, string> } }).workspaces?.catalog ?? {};
  return Object.entries(catalog).map(([name, spec]) => ({ ecosystem: 'npm', name, spec }));
}

export function readCargoPins(manifest: unknown): Pin[] {
  const dependencies =
    (manifest as { workspace?: { dependencies?: Record<string, unknown> } }).workspace?.dependencies ?? {};
  const pins: Pin[] = [];
  for (const [key, value] of Object.entries(dependencies)) {
    if (typeof value === 'string') {
      pins.push({ ecosystem: 'crates', name: key, spec: value });
      continue;
    }
    if (typeof value !== 'object' || value === null) {
      continue;
    }
    const table = value as { version?: unknown; package?: unknown };
    if (typeof table.version !== 'string') {
      continue; // path-only workspace crate: not a registry pin
    }
    const name = typeof table.package === 'string' ? table.package : key;
    pins.push({ ecosystem: 'crates', name, spec: table.version });
  }
  return pins;
}

export function readToolPins(
  packageManager: string | undefined,
  toolchain: unknown,
  tools: Readonly<Record<string, string>>,
): Pin[] {
  const pins: Pin[] = [];
  const bun = packageManager?.startsWith('bun@') ? packageManager.slice('bun@'.length) : undefined;
  if (bun !== undefined) {
    pins.push({ ecosystem: 'tool', name: 'bun', spec: bun });
  }
  const channel = (toolchain as { toolchain?: { channel?: unknown } }).toolchain?.channel;
  if (typeof channel === 'string') {
    pins.push({ ecosystem: 'tool', name: 'rust', spec: channel });
  }
  for (const [name, spec] of Object.entries(tools)) {
    pins.push({ ecosystem: 'tool', name, spec });
  }
  return pins;
}
```

- [ ] **Step 7: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/semver ./scripts/tests/unit/deps`
Expected: PASS.

- [ ] **Step 8: Write `.config/dependency-exceptions.toml`**

```toml
# Dependency exceptions — an older pin is allowed only with a reason (spec §10.1).
# `bun run deps --strict` (part of `bun run check`) fails on an outdated pin without an entry
# here, and on an entry that no longer matches a pin.
#
# [[exception]]
# ecosystem = "npm"   # npm | crates | tool
# name = "some-package"
# version = "1.2.3"   # the exact pinned version this exception allows
# reason = "Why the latest version cannot be used yet, with a link to the upstream issue."
```

- [ ] **Step 9: Implement `scripts/commands/deps.ts`**

```ts
// bun run deps [--strict] — compare every pin with the newest eligible release.
// "Eligible" = stable and older than the bunfig cooldown. Unreachable registries are warnings.
import { parseArgs } from 'node:util';

import {
  crateVersions,
  newestEligible,
  npmVersions,
  rustStableVersion,
} from '../lib/deps/deps.registry';
import {
  evaluatePins,
  type Finding,
  isFailure,
  type Pin,
  parseExceptions,
  pinKey,
  readCargoPins,
  readCatalogPins,
  readToolPins,
} from '../lib/deps/deps.policy';
import { repoPath } from '../lib/repo/repo.util';
import { RUST_TOOLS } from '../lib/rust/rust-tools.constants';

const { values } = parseArgs({ args: process.argv.slice(2), options: { strict: { type: 'boolean' } } });
const now = new Date();

const readText = (path: string) => Bun.file(repoPath(path)).text();
const manifest = JSON.parse(await readText('package.json')) as { packageManager?: string };
const pins: Pin[] = [
  ...readCatalogPins(manifest),
  ...readCargoPins(Bun.TOML.parse(await readText('Cargo.toml'))),
  ...readToolPins(manifest.packageManager, Bun.TOML.parse(await readText('rust-toolchain.toml')), RUST_TOOLS),
];
const exceptions = parseExceptions(await readText('.config/dependency-exceptions.toml'));

async function latestFor(pin: Pin): Promise<string | null> {
  if (pin.ecosystem === 'npm') return newestEligible(await npmVersions(pin.name), now);
  if (pin.ecosystem === 'crates') return newestEligible(await crateVersions(pin.name), now);
  if (pin.name === 'bun') return newestEligible(await npmVersions('bun'), now);
  if (pin.name === 'rust') return await rustStableVersion();
  return newestEligible(await crateVersions(pin.name), now);
}

const latest = new Map<string, string>();
await Promise.all(
  pins.map(async (pin) => {
    try {
      const version = await latestFor(pin);
      if (version !== null) latest.set(pinKey(pin.ecosystem, pin.name), version);
    } catch (error) {
      console.warn(`⚠ ${pin.ecosystem} ${pin.name}: ${error instanceof Error ? error.message : String(error)}`);
    }
  }),
);

function describe(finding: Finding): string {
  switch (finding.kind) {
    case 'current':
      return `✔ ${finding.pin.ecosystem} ${finding.pin.name} ${finding.pin.spec}`;
    case 'outdated':
      return `✖ ${finding.pin.ecosystem} ${finding.pin.name} ${finding.pin.spec} → ${finding.latest} (outdated)`;
    case 'too-new':
      return `✖ ${finding.pin.ecosystem} ${finding.pin.name} ${finding.pin.spec} is newer than the cooldown allows (latest eligible ${finding.latest})`;
    case 'excepted':
      return `• ${finding.pin.ecosystem} ${finding.pin.name} ${finding.pin.spec} (latest ${finding.latest}; exception: ${finding.reason})`;
    case 'not-exact':
      return `✖ ${finding.pin.ecosystem} ${finding.pin.name} "${finding.pin.spec}" is not an exact pin`;
    case 'unknown':
      return `⚠ ${finding.pin.ecosystem} ${finding.pin.name} ${finding.pin.spec} (latest unknown — offline?)`;
    case 'stale-exception':
      return `✖ exception ${finding.exception.ecosystem} ${finding.exception.name}: ${finding.why}`;
  }
}

const findings = evaluatePins(pins, latest, exceptions);
for (const finding of findings) {
  console.log(describe(finding));
}
const failures = findings.filter(isFailure).length;
console.log(`deps: ${pins.length} pins, ${failures} problems`);
process.exit(values.strict === true && failures > 0 ? 1 : 0);
```

Add to `package.json` `scripts`: `"deps": "bun scripts/commands/deps.ts"`.

- [ ] **Step 10: Run it**

Run: `bun run deps --strict`
Expected: every pin `✔` (or `⚠` for a registry that did not answer) and `deps: N pins, 0 problems`. An `✖ … outdated` line means a newer eligible release appeared since Task 1: update the pin (catalog, `Cargo.toml`, `rust-toolchain.toml`, `packageManager` or `RUST_TOOLS`), run `bun install` / `cargo update`, and re-run.

- [ ] **Step 11: Typecheck and commit**

```bash
bun x tsc -p tsconfig.json
git add scripts/lib/semver scripts/lib/deps scripts/lib/rust scripts/commands/deps.ts scripts/tests/unit/semver scripts/tests/unit/deps .config/dependency-exceptions.toml package.json
git commit -m "feat(scripts): add dependency freshness policy with cooldown and exceptions"
```

### Task 14: `bun run version` and `bun run clean`

**Files:**
- Create: `scripts/lib/version/version.util.ts`, `scripts/lib/clean/clean.util.ts`, `scripts/commands/clean.ts`
- Overwrite: `scripts/commands/version.ts` (the skeleton's empty file)
- Test: `scripts/tests/unit/version/version.util.test.ts`, `scripts/tests/unit/clean/clean.util.test.ts`
- Modify: `package.json` (`scripts.version`, `scripts.clean`)

**Interfaces:**
- Consumes: `parseVersion`, `compareVersions` (Task 13); `REPO_ROOT`, `repoPath` (Task 7); `run` (Task 7).
- Produces: `nextVersion(current: string, request: string): string`; `setJsonVersion(text: string, version: string): string`; `setCargoWorkspaceVersion(text: string, version: string): string`; `cleanTargets(options: { all: boolean }): string[]`; `nestedCleanGlobs(options: { all: boolean }): string[]`; commands `bun run version <major|minor|patch|x.y.z>` and `bun run clean [--all]`.

- [ ] **Step 1: Write the failing tests**

`scripts/tests/unit/version/version.util.test.ts`
```ts
import { describe, expect, test } from 'bun:test';

import { nextVersion, setCargoWorkspaceVersion, setJsonVersion } from '../../../lib/version/version.util';

describe('nextVersion', () => {
  test('bumps by kind', () => {
    expect(nextVersion('0.1.0', 'patch')).toBe('0.1.1');
    expect(nextVersion('0.1.9', 'minor')).toBe('0.2.0');
    expect(nextVersion('0.9.3', 'major')).toBe('1.0.0');
  });

  test('accepts an explicit higher version only', () => {
    expect(nextVersion('0.1.0', '0.3.0')).toBe('0.3.0');
    expect(() => nextVersion('0.3.0', '0.2.0')).toThrow();
    expect(() => nextVersion('0.3.0', 'banana')).toThrow();
  });
});

describe('setJsonVersion', () => {
  test('replaces the top-level version and keeps formatting', () => {
    const text = '{\n  "name": "x",\n  "version": "0.1.0",\n  "dependencies": {}\n}\n';
    expect(setJsonVersion(text, '0.2.0')).toBe('{\n  "name": "x",\n  "version": "0.2.0",\n  "dependencies": {}\n}\n');
  });

  test('leaves non-semver versions (tauri.conf.json "../package.json") and missing versions alone', () => {
    const tauri = '{\n  "version": "../package.json"\n}\n';
    expect(setJsonVersion(tauri, '0.2.0')).toBe(tauri);
    expect(setJsonVersion('{}\n', '0.2.0')).toBe('{}\n');
  });
});

describe('setCargoWorkspaceVersion', () => {
  test('changes only [workspace.package] version', () => {
    const text = '[workspace]\nmembers = []\n\n[workspace.package]\nversion = "0.1.0"\nedition = "2024"\n\n[workspace.dependencies]\nx = { version = "=1.0.0" }\n';
    expect(setCargoWorkspaceVersion(text, '0.2.0')).toBe(text.replace('version = "0.1.0"', 'version = "0.2.0"'));
  });

  test('keeps CRLF line endings', () => {
    const text = '[workspace.package]\r\nversion = "0.1.0"\r\n';
    expect(setCargoWorkspaceVersion(text, '1.0.0')).toBe('[workspace.package]\r\nversion = "1.0.0"\r\n');
  });

  test('throws when there is no [workspace.package] version', () => {
    expect(() => setCargoWorkspaceVersion('[workspace]\n', '1.0.0')).toThrow();
  });
});
```

`scripts/tests/unit/clean/clean.util.test.ts`
```ts
import { expect, test } from 'bun:test';
import { isAbsolute } from 'node:path';

import { cleanTargets, nestedCleanGlobs } from '../../../lib/clean/clean.util';

test('default clean keeps node_modules; --all removes it', () => {
  expect(cleanTargets({ all: false })).not.toContain('node_modules');
  expect(cleanTargets({ all: true })).toContain('node_modules');
  expect(nestedCleanGlobs({ all: false }).some((glob) => glob.endsWith('node_modules'))).toBe(false);
  expect(nestedCleanGlobs({ all: true }).some((glob) => glob.endsWith('node_modules'))).toBe(true);
});

test('every target stays inside the repo', () => {
  for (const path of [...cleanTargets({ all: true }), ...nestedCleanGlobs({ all: true })]) {
    expect(isAbsolute(path)).toBe(false);
    expect(path.split('/')).not.toContain('..');
  }
});
```

- [ ] **Step 2: Run to see them fail**

Run: `bun test ./scripts/tests/unit/version ./scripts/tests/unit/clean`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement `scripts/lib/version/version.util.ts`**

```ts
import { compareVersions, parseVersion } from '../semver/semver.util';

const KINDS = ['major', 'minor', 'patch'] as const;
type BumpKind = (typeof KINDS)[number];

const isBumpKind = (value: string): value is BumpKind => (KINDS as readonly string[]).includes(value);

/** The next version: a bump kind (`major`, `minor`, `patch`) or an explicit higher `x.y.z`. */
export function nextVersion(current: string, request: string): string {
  const parsed = parseVersion(current);
  if (parsed === null) {
    throw new Error(`current version is not x.y.z: ${current}`);
  }
  const [major, minor, patch] = parsed;
  if (isBumpKind(request)) {
    if (request === 'major') return `${major + 1}.0.0`;
    if (request === 'minor') return `${major}.${minor + 1}.0`;
    return `${major}.${minor}.${patch + 1}`;
  }
  if (parseVersion(request) === null) {
    throw new Error(`expected major, minor, patch or x.y.z, got "${request}"`);
  }
  if (compareVersions(request, current) <= 0) {
    throw new Error(`${request} is not higher than ${current}`);
  }
  return request;
}

/** Replaces the top-level `"version"` when it is a semver string; otherwise returns `text` as is. */
export function setJsonVersion(text: string, version: string): string {
  const data = JSON.parse(text) as { version?: unknown };
  if (typeof data.version !== 'string' || parseVersion(data.version) === null) {
    return text;
  }
  return text.replace(/("version"\s*:\s*")[^"]*(")/, `$1${version}$2`);
}

/** Replaces `version` inside `[workspace.package]`, keeping every other line byte-for-byte. */
export function setCargoWorkspaceVersion(text: string, version: string): string {
  const lines = text.split('\n');
  const start = lines.findIndex((line) => line.trim() === '[workspace.package]');
  if (start < 0) {
    throw new Error('Cargo.toml has no [workspace.package] table');
  }
  for (let index = start + 1; index < lines.length; index += 1) {
    const line = lines[index] ?? '';
    if (line.trim().startsWith('[')) {
      break;
    }
    if (/^\s*version\s*=/.test(line)) {
      lines[index] = line.replace(/"[^"]*"/, `"${version}"`);
      return lines.join('\n');
    }
  }
  throw new Error('[workspace.package] has no version key');
}
```

- [ ] **Step 4: Implement `scripts/lib/clean/clean.util.ts`**

```ts
interface CleanOptions {
  readonly all: boolean;
}

/** Top-level paths removed by `bun run clean` (repo-relative). */
export function cleanTargets(options: CleanOptions): string[] {
  const targets = ['target', '.turbo', 'coverage', 'playwright-report', 'test-results', 'node_modules/.cache'];
  return options.all ? [...targets, 'node_modules'] : targets;
}

/** Nested build output removed by `bun run clean` (repo-relative globs). */
export function nestedCleanGlobs(options: CleanOptions): string[] {
  const globs = ['packages/*/dist', 'programs/*/*/dist', 'packages/*/.turbo', 'programs/*/*/.turbo', 'crates/.turbo'];
  return options.all
    ? [...globs, 'packages/*/node_modules', 'programs/*/*/node_modules', 'crates/node_modules']
    : globs;
}
```

- [ ] **Step 5: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/version ./scripts/tests/unit/clean`
Expected: PASS.

- [ ] **Step 6: Implement `scripts/commands/version.ts`**

```ts
// bun run version <major|minor|patch|x.y.z> — bump every manifest at once, then refresh lockfiles.
import { run } from '../lib/process/exec.util';
import { REPO_ROOT, repoPath } from '../lib/repo/repo.util';
import { nextVersion, setCargoWorkspaceVersion, setJsonVersion } from '../lib/version/version.util';

const request = process.argv[2];
if (request === undefined) {
  console.error('usage: bun run version <major|minor|patch|x.y.z>');
  process.exit(1);
}

const root = JSON.parse(await Bun.file(repoPath('package.json')).text()) as {
  version: string;
  workspaces?: { packages?: string[] };
};
const next = nextVersion(root.version, request);

const jsonFiles = ['package.json'];
for (const pattern of root.workspaces?.packages ?? []) {
  for await (const file of new Bun.Glob(`${pattern}/package.json`).scan({ cwd: REPO_ROOT })) {
    jsonFiles.push(file);
  }
}
for await (const file of new Bun.Glob('programs/**/src-tauri/tauri.conf.json').scan({ cwd: REPO_ROOT })) {
  jsonFiles.push(file);
}

for (const file of jsonFiles) {
  const text = await Bun.file(repoPath(file)).text();
  const updated = setJsonVersion(text, next);
  if (updated !== text) {
    await Bun.write(repoPath(file), updated);
    console.log(`✔ ${file}`);
  }
}

const cargo = await Bun.file(repoPath('Cargo.toml')).text();
await Bun.write(repoPath('Cargo.toml'), setCargoWorkspaceVersion(cargo, next));
console.log('✔ Cargo.toml [workspace.package]');

const refreshed = (await run(['bun', 'install'])) === 0 && (await run(['cargo', 'update', '--workspace'])) === 0;
console.log(`version ${root.version} → ${next}${refreshed ? '' : ' (lockfile refresh failed — run bun install and cargo update --workspace)'}`);
process.exit(refreshed ? 0 : 1);
```

- [ ] **Step 7: Implement `scripts/commands/clean.ts`**

```ts
// bun run clean [--all] — remove build output and caches (--all also removes node_modules).
import { rm } from 'node:fs/promises';
import { parseArgs } from 'node:util';

import { cleanTargets, nestedCleanGlobs } from '../lib/clean/clean.util';
import { REPO_ROOT, repoPath } from '../lib/repo/repo.util';

const { values } = parseArgs({ args: process.argv.slice(2), options: { all: { type: 'boolean' } } });
const options = { all: values.all === true };

const paths = [...cleanTargets(options)];
for (const glob of nestedCleanGlobs(options)) {
  for await (const match of new Bun.Glob(glob).scan({ cwd: REPO_ROOT, onlyFiles: false })) {
    paths.push(match);
  }
}

for (const path of paths) {
  await rm(repoPath(path), { recursive: true, force: true });
  console.log(`✔ removed ${path}`);
}
```

Add to `package.json` `scripts`: `"version": "bun scripts/commands/version.ts"`, `"clean": "bun scripts/commands/clean.ts"`.

- [ ] **Step 8: Try them**

Run:
```bash
bun run version patch && git diff --stat && git checkout -- package.json crates/package.json Cargo.toml Cargo.lock bun.lock
bun run clean
```
Expected: `version 0.1.0 → 0.1.1` with `package.json`, `crates/package.json` and `Cargo.toml` changed (then reverted by the checkout); `clean` removes `target` and the caches. Run `cargo build --workspace` afterwards to restore `target/` before the next task.

- [ ] **Step 9: Typecheck and commit**

```bash
bun x tsc -p tsconfig.json
git add scripts/lib/version scripts/lib/clean scripts/commands/version.ts scripts/commands/clean.ts scripts/tests/unit/version scripts/tests/unit/clean package.json
git commit -m "feat(scripts): add version bump across manifests and clean command"
```

### Task 15: `setup`, `test` and `check` commands, and git hooks

**Files:**
- Create: `scripts/commands/setup.ts`, `scripts/commands/test.ts`, `scripts/commands/check.ts`
- Overwrite: `.config/lefthook.yml`
- Modify: `package.json` (`scripts.setup`, `scripts.test`, `scripts.check`)

**Interfaces:**
- Consumes: `commandStep`, `runSteps`, `finish`, `run`, `capture` (Task 7); `RUST_TOOLS` (Task 13); `repoPath` (Task 7).
- Produces: `bun run setup`, `bun run test`, `bun run check`; the `CHECK_STEPS` list Task 18 extends with the agent-rules drift check.

- [ ] **Step 1: Implement `scripts/commands/setup.ts`**

```ts
// bun run setup — prepare a fresh clone: Rust toolchain, dependencies, cargo tools, git hooks.
import { capture, run } from '../lib/process/exec.util';
import { commandStep, finish, runSteps, type Step } from '../lib/process/steps.util';
import { repoPath } from '../lib/repo/repo.util';
import { RUST_TOOLS } from '../lib/rust/rust-tools.constants';

if (Bun.which('rustup') === null) {
  console.error(
    '✖ rustup is not installed. Install it and the MSVC Build Tools first:\n' +
      '  winget install --id Rustlang.Rustup --source winget\n' +
      '  winget install --id Microsoft.VisualStudio.BuildTools --source winget --override "--wait --passive --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"',
  );
  process.exit(1);
}

async function installRustTools(): Promise<boolean> {
  for (const [tool, version] of Object.entries(RUST_TOOLS)) {
    const installed = await capture(['cargo', tool.replace(/^cargo-/, ''), '--version']);
    if (installed.exitCode === 0 && installed.stdout.includes(version)) {
      console.log(`  ${tool} ${version} already installed`);
      continue;
    }
    if ((await run(['cargo', 'install', '--locked', `${tool}@${version}`])) !== 0) {
      return false;
    }
  }
  return true;
}

const lockfile = await Bun.file(repoPath('bun.lock')).exists();
const steps: Step[] = [
  commandStep('rust toolchain (rust-toolchain.toml)', ['rustup', 'toolchain', 'install']),
  commandStep('dependencies', lockfile ? ['bun', 'install', '--frozen-lockfile'] : ['bun', 'install']),
  { name: 'cargo tools', run: installRustTools },
  commandStep('git hooks (lefthook)', ['bun', 'x', 'lefthook', 'install']),
];

finish(await runSteps(steps));
```

- [ ] **Step 2: Implement `scripts/commands/test.ts`**

```ts
// bun run test — the scripts' tests, then every workspace's tests through Turborepo.
import { commandStep, finish, runSteps } from '../lib/process/steps.util';

finish(
  await runSteps([
    commandStep('scripts (bun test)', ['bun', 'test']),
    commandStep('workspaces (turbo test)', ['bun', 'x', 'turbo', 'run', 'test']),
  ]),
);
```

- [ ] **Step 3: Implement `scripts/commands/check.ts`**

```ts
// bun run check — the full quality gate (spec §10.4). Every step runs; the summary lists failures.
import { commandStep, finish, runSteps, type Step } from '../lib/process/steps.util';

export const CHECK_STEPS: readonly Step[] = [
  commandStep('biome', ['bun', 'x', 'biome', 'check', '--config-path=.config/biome.json', '--error-on-warnings', '.']),
  commandStep('typescript (scripts)', ['bun', 'x', 'tsc', '-p', 'tsconfig.json']),
  commandStep('cspell', ['bun', 'x', 'cspell', 'lint', '--config', '.config/cspell.json', '--no-progress', '--dot', '**']),
  commandStep('knip', ['bun', 'x', 'knip', '--config', '.config/knip.json']),
  commandStep('structure', ['bun', 'scripts/commands/structure.ts']),
  commandStep('attribution', ['bun', 'scripts/commands/attribution.ts']),
  commandStep('secretlint', [
    'bun', 'x', 'secretlint', '--secretlintrc', '.config/secretlint.json', '--secretlintignore', '.gitignore', '**/*',
  ]),
  commandStep('bun audit', ['bun', 'audit', '--audit-level=moderate']),
  commandStep('dependencies', ['bun', 'scripts/commands/deps.ts', '--strict']),
  commandStep('turbo (typecheck, rustfmt, clippy, cargo-deny)', [
    'bun', 'x', 'turbo', 'run', 'typecheck', 'rust:fmt', 'rust:lint', 'rust:deny',
  ]),
];

if (import.meta.main) {
  finish(await runSteps(CHECK_STEPS));
}
```

Add to `package.json` `scripts`: `"setup": "bun scripts/commands/setup.ts"`, `"test": "bun scripts/commands/test.ts"`, `"check": "bun scripts/commands/check.ts"`.

- [ ] **Step 4: Write `.config/lefthook.yml`**

```yaml
# Git hooks — lefthook reads .config/lefthook.yml (docs/research/tooling.md).
# A fast subset of `bun run check`; CI runs the full gate.

pre-commit:
  parallel: true
  jobs:
    - name: biome
      glob: "*.{ts,tsx,js,mjs,cjs,json,jsonc,css}"
      run: bun x biome check --config-path=.config/biome.json --write --no-errors-on-unmatched {staged_files}
      stage_fixed: true
    - name: rustfmt
      glob: "*.rs"
      run: rustfmt --edition 2024 {staged_files}
      stage_fixed: true
    - name: cspell
      run: bun x cspell lint --config .config/cspell.json --no-progress --no-must-find-files --dot {staged_files}
    - name: structure
      run: bun scripts/commands/structure.ts {staged_files}
    - name: secretlint
      run: bun x secretlint --secretlintrc .config/secretlint.json {staged_files}

commit-msg:
  jobs:
    - name: commitlint
      run: bun x commitlint --config .config/commitlint.config.ts --edit {1}
    - name: attribution
      run: bun scripts/commands/attribution.ts --message-file {1}

pre-push:
  jobs:
    - name: attribution
      run: bun scripts/commands/attribution.ts
```

- [ ] **Step 5: Run setup, test and check**

Run:
```bash
bun run setup
bun run test
bun run check
```
Expected: each ends with `✔ all N steps passed`. Fix what fails; typical first-run findings are cspell words (add to `project-words.txt`) and knip unused exports (remove the export or use it). Do not weaken a rule to pass.

- [ ] **Step 6: Commit (this commit runs the new hooks)**

```bash
git add scripts/commands/setup.ts scripts/commands/test.ts scripts/commands/check.ts .config/lefthook.yml package.json .config/cspell/project-words.txt
git commit -m "feat(scripts): add setup, test and check commands with lefthook git hooks"
```
Expected: lefthook prints the pre-commit and commit-msg jobs as passed.

### Task 16: Editor settings, CI and repository files

**Files:**
- Overwrite: `.vscode/settings.json`, `.vscode/extensions.json`, `.vscode/tasks.json`, `.vscode/launch.json`, `.github/actions/setup-env/action.yml`, `.github/workflows/ci.yml`, `.github/codeql/codeql-config.yml`, `.github/renovate.json`, `.github/CODEOWNERS`, `.github/SECURITY.md`, `.github/CONTRIBUTING.md`, `.github/CODE_OF_CONDUCT.md`, `.github/SUPPORT.md`, `.github/pull_request_template.md`, `.github/ISSUE_TEMPLATE/bug_report.yml`, `.github/ISSUE_TEMPLATE/feature_request.yml`, `.github/ISSUE_TEMPLATE/config.yml`
- Create: `.github/workflows/codeql.yml`
- Test: `scripts/tests/unit/rust/rust-tools.constants.test.ts`

**Interfaces:**
- Consumes: `RUST_TOOLS` (Task 13); `repoPath` (Task 7).
- Produces: CI that runs `bun run check` and `bun run test` on Windows for every push and PR.

- [ ] **Step 1: Write the failing consistency test** — `scripts/tests/unit/rust/rust-tools.constants.test.ts`

```ts
import { expect, test } from 'bun:test';

import { repoPath } from '../../../lib/repo/repo.util';
import { RUST_TOOLS } from '../../../lib/rust/rust-tools.constants';

test('CI installs the same cargo tool versions as bun run setup', async () => {
  const action = Bun.YAML.parse(
    await Bun.file(repoPath('.github/actions/setup-env/action.yml')).text(),
  ) as { runs: { steps: { uses?: string; with?: { tool?: string } }[] } };
  const install = action.runs.steps.find((step) => step.uses?.startsWith('taiki-e/install-action@'));
  const tools = (install?.with?.tool ?? '').split(',').map((entry) => entry.trim());
  expect(tools.toSorted()).toEqual(
    Object.entries(RUST_TOOLS)
      .map(([name, version]) => `${name}@${version}`)
      .toSorted(),
  );
});
```

Run: `bun test ./scripts/tests/unit/rust` — Expected: FAIL (the action file is still `{}`/empty YAML).

- [ ] **Step 2: Write `.github/actions/setup-env/action.yml`**

```yaml
name: Set up GENSLATE environment
description: Install bun (from packageManager), the pinned Rust toolchain, cargo tools and dependencies.
runs:
  using: composite
  steps:
    - uses: oven-sh/setup-bun@0c5077e51419868618aeaa5fe8019c62421857d6 # v2.2.0
      with:
        bun-version-file: package.json
    - name: Install the Rust toolchain from rust-toolchain.toml
      shell: pwsh
      run: rustup toolchain install
    - uses: Swatinem/rust-cache@6323deb102c322ba6fcbdcafc7e3dddab59af2b6 # v2.9.2
    - uses: taiki-e/install-action@f7e5d7c961414b23f5b25b2da9294395d08513ad # v2.87.26
      with:
        tool: cargo-nextest@0.9.146,cargo-deny@0.20.2
    - name: Install dependencies
      shell: pwsh
      run: bun install --frozen-lockfile
```

Run: `bun test ./scripts/tests/unit/rust` — Expected: PASS.

- [ ] **Step 3: Write `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  check:
    name: Check and test (Windows)
    runs-on: windows-latest
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: ./.github/actions/setup-env
      - name: Attribution (commits in this pull request)
        if: github.event_name == 'pull_request'
        shell: pwsh
        env:
          BASE_REF: ${{ github.base_ref }}
        run: bun run attribution --range "origin/$env:BASE_REF..HEAD"
      - name: Check
        shell: pwsh
        run: bun run check
      - name: Test
        shell: pwsh
        run: bun run test
```

- [ ] **Step 4: Write `.github/workflows/codeql.yml` and `.github/codeql/codeql-config.yml`**

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "23 4 * * 1"

permissions:
  contents: read

jobs:
  analyze:
    name: Analyze (${{ matrix.language }})
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      security-events: write
    strategy:
      fail-fast: false
      matrix:
        language: [actions, javascript-typescript, rust]
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: github/codeql-action/init@2892aa5e19bbd11bc0cff5427e3b750a04d9e3c2 # v4.38.2
        with:
          languages: ${{ matrix.language }}
          build-mode: none
          config-file: ./.github/codeql/codeql-config.yml
      - uses: github/codeql-action/analyze@2892aa5e19bbd11bc0cff5427e3b750a04d9e3c2 # v4.38.2
        with:
          category: "/language:${{ matrix.language }}"
```

`.github/codeql/codeql-config.yml`
```yaml
name: GENSLATE CodeQL configuration
paths-ignore:
  - "**/node_modules"
  - "**/target"
  - "**/generated"
  - "**/*.generated.ts"
  - release
```

- [ ] **Step 5: Write `.github/renovate.json`**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", "helpers:pinGitHubActionDigests"],
  "rangeStrategy": "pin",
  "minimumReleaseAge": "3 days",
  "semanticCommits": "enabled",
  "semanticCommitType": "build",
  "semanticCommitScope": "deps",
  "labels": ["deps"],
  "lockFileMaintenance": { "enabled": true, "schedule": ["before 6am on monday"] },
  "packageRules": [{ "matchManagers": ["github-actions"], "semanticCommitScope": "ci" }]
}
```

- [ ] **Step 6: Write the repository community files**

`.github/CODEOWNERS`
```text
# Default owner for everything in this repository.
* @GENSLATE
```

`.github/SECURITY.md`
```markdown
# Security policy

## Supported versions

Only the latest release of the GENSLATE launcher receives security fixes.

## Reporting a vulnerability

Please report vulnerabilities privately through GitHub: open the repository's **Security** tab and choose
**Report a vulnerability**. Do not open a public issue. You will get an acknowledgement within 7 days and a
fix or mitigation plan within 30 days for confirmed issues.

The launcher's threat model lives in `programs/desktop/launcher/other/launcher/documents/security.md`.
```

`.github/CONTRIBUTING.md`
```markdown
# Contributing to GENSLATE

## Setup

1. Windows 10/11 x64 with Git, bun 1.4.2, rustup and the MSVC Build Tools (see `docs/research/versions.md`).
2. `bun run setup` — installs the Rust toolchain, dependencies, cargo tools and git hooks.

Use **bun only** (`bun install`, `bun run <cmd>`, `bun x <bin>`, `bun test`) — never npm, npx, pnpm or yarn.

## Everyday commands

| Command | What it does |
|---|---|
| `bun run check` | Full quality gate (lint, types, spelling, structure, attribution, secrets, audit, dependency freshness, rustfmt, clippy, cargo-deny) |
| `bun run test` | All tests (bun test + cargo nextest) |
| `bun run format` | Biome + rustfmt |
| `bun run change <type> <scope> <summary>` | Add a changelog entry to `.changes/unreleased/` |
| `bun run deps` | Show outdated dependencies |

## Commits and pull requests

- Conventional Commits: `type(scope): subject`. Types and scopes are listed in
  `scripts/lib/commits/commit.constants.ts`; commitlint enforces them.
- Add a change entry (`bun run change …`) for every user-visible change.
- **No AI-agent attribution** of any kind: no `Co-Authored-By` agent trailers, no "Generated with" footers,
  no agent session links, no agent-named branches. Hooks and CI reject them.
- A pull request is ready when `bun run check` and `bun run test` pass.
```

`.github/CODE_OF_CONDUCT.md`
```markdown
# Code of conduct

This project follows the [Contributor Covenant, version 2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

Report unacceptable behaviour privately to the maintainer, [@GENSLATE](https://github.com/GENSLATE), through
GitHub. Reports are handled confidentially.
```

`.github/SUPPORT.md`
```markdown
# Getting help

- **Bugs:** open an issue with the *Bug report* form.
- **Ideas:** open an issue with the *Feature request* form.
- **Security problems:** see [SECURITY.md](SECURITY.md) — never report them in public issues.
```

`.github/pull_request_template.md`
```markdown
## What and why

<!-- One or two sentences: what changes and why. Link the issue if there is one. -->

## How it was tested

<!-- Commands run, screenshots for UI changes (both Polar Night and Snow Storm). -->

## Checklist

- [ ] `bun run check` passes
- [ ] `bun run test` passes
- [ ] Change entry added (`bun run change <type> <scope> <summary>`) for user-visible changes
- [ ] Docs updated when behaviour changed
- [ ] No agent attribution in commits or this description
```

`.github/ISSUE_TEMPLATE/bug_report.yml`
```yaml
name: Bug report
description: Something in GENSLATE does not work as expected.
labels: [bug]
body:
  - type: textarea
    id: what-happened
    attributes:
      label: What happened?
      description: Include what you expected to happen instead.
    validations:
      required: true
  - type: textarea
    id: steps
    attributes:
      label: Steps to reproduce
      placeholder: "1. Plug in the drive\n2. Run Start.exe\n3. …"
    validations:
      required: true
  - type: input
    id: version
    attributes:
      label: GENSLATE version
      placeholder: "0.1.0"
    validations:
      required: true
  - type: input
    id: windows
    attributes:
      label: Windows version
      placeholder: "Windows 11 24H2"
  - type: textarea
    id: logs
    attributes:
      label: Logs
      description: Attach files from `other/launcher/logs/` on the drive if relevant.
      render: text
```

`.github/ISSUE_TEMPLATE/feature_request.yml`
```yaml
name: Feature request
description: Suggest an idea for GENSLATE.
labels: [enhancement]
body:
  - type: textarea
    id: problem
    attributes:
      label: What problem would this solve?
    validations:
      required: true
  - type: textarea
    id: idea
    attributes:
      label: What would you like to happen?
    validations:
      required: true
  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives you considered
```

`.github/ISSUE_TEMPLATE/config.yml`
```yaml
blank_issues_enabled: false
contact_links:
  - name: Report a security vulnerability
    url: https://github.com/GENSLATE/GENSLATE-LAUNCHER/security/advisories/new
    about: Please report security problems privately.
```

- [ ] **Step 7: Write the editor settings (Cursor reads `.vscode/`)**

`.vscode/settings.json`
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "biomejs.biome",
  "editor.codeActionsOnSave": {
    "source.fixAll.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "biome.enabled": true,
  "biome.requireConfiguration": true,
  "biome.configurationPath": ".config/biome.json",
  "[rust]": { "editor.defaultFormatter": "rust-lang.rust-analyzer" },
  "[toml]": { "editor.defaultFormatter": "tamasfe.even-better-toml" },
  "[markdown]": { "editor.defaultFormatter": null },
  "rust-analyzer.check.command": "clippy",
  "rust-analyzer.cargo.targetDir": true,
  "cSpell.import": ["${workspaceFolder}/.config/cspell.json"],
  "files.eol": "\n",
  "files.insertFinalNewline": true,
  "files.trimTrailingWhitespace": true,
  "search.exclude": {
    "**/node_modules": true,
    "**/target": true,
    "bun.lock": true,
    "Cargo.lock": true
  }
}
```

`.vscode/extensions.json`
```json
{
  "recommendations": [
    "biomejs.biome",
    "rust-lang.rust-analyzer",
    "tauri-apps.tauri-vscode",
    "tamasfe.even-better-toml",
    "streetsidesoftware.code-spell-checker",
    "bradlc.vscode-tailwindcss",
    "oven.bun-vscode",
    "editorconfig.editorconfig",
    "vadimcn.vscode-lldb"
  ],
  "unwantedRecommendations": ["esbenp.prettier-vscode", "dbaeumer.vscode-eslint"]
}
```

`.vscode/tasks.json`
```json
{
  "version": "2.0.0",
  "tasks": [
    { "label": "check", "type": "shell", "command": "bun run check", "problemMatcher": [] },
    { "label": "test", "type": "shell", "command": "bun run test", "problemMatcher": [], "group": "test" },
    { "label": "format", "type": "shell", "command": "bun run format", "problemMatcher": [] }
  ]
}
```

`.vscode/launch.json` (Plan 3 adds the Tauri debug configurations)
```json
{
  "version": "0.2.0",
  "configurations": []
}
```

- [ ] **Step 8: Validate and commit**

Run:
```bash
bun test ./scripts/tests/unit/rust
bun run check
```
Expected: PASS and `✔ all N steps passed`.

```bash
git add .vscode .github scripts/tests/unit/rust
git commit -m "ci(repo): add windows ci, codeql, renovate, editor settings and community files"
```

---

## M2 — Agent rules and tooling

### Task 17: Rule model and per-tool renderers

**Files:**
- Create: `scripts/lib/agents/rule.model.ts`, `scripts/lib/agents/rule.render.ts`
- Test: `scripts/tests/unit/agents/rule.model.test.ts`, `scripts/tests/unit/agents/rule.render.test.ts`

**Interfaces:**
- Produces:
  - `Rule { name: string; description: string; paths: readonly string[]; body: string }`
  - `parseRule(fileName: string, source: string): Rule` — front matter keys: `description` (required), `paths` (optional non-empty list of globs); anything else throws; CRLF and LF parse identically
  - `budgetProblems(rules: readonly Rule[]): string[]` — Antigravity limits: one rule ≤ 24 000 bytes; all always-on rules together ≤ 60 000 bytes
  - `renderCursorRule(rule: Rule): string`, `renderAntigravityRule(rule: Rule): string`

Source format (`.claude/rules/<name>.md`):
```markdown
---
description: One line that tells Cursor and Antigravity when the rule matters.
paths:
  - "**/*.rs"
---

# Title

Body…
```
Claude Code reads `paths` natively. A rule without `paths` is always on in every tool.

- [ ] **Step 1: Write the failing tests**

`scripts/tests/unit/agents/rule.model.test.ts`
```ts
import { describe, expect, test } from 'bun:test';

import { budgetProblems, parseRule, type Rule } from '../../../lib/agents/rule.model';

const scoped = '---\ndescription: Rust conventions\npaths:\n  - "**/*.rs"\n  - "**/Cargo.toml"\n---\n\n# Rust\n\nUse edition 2024.\n';

describe('parseRule', () => {
  test('reads description, paths and body', () => {
    expect(parseRule('rust.md', scoped)).toEqual({
      name: 'rust',
      description: 'Rust conventions',
      paths: ['**/*.rs', '**/Cargo.toml'],
      body: '# Rust\n\nUse edition 2024.',
    });
  });

  test('a rule without paths is always on', () => {
    expect(parseRule('project.md', '---\ndescription: Project basics\n---\n# Project\n').paths).toEqual([]);
  });

  test('CRLF files parse exactly like LF files', () => {
    expect(parseRule('rust.md', scoped.replaceAll('\n', '\r\n'))).toEqual(parseRule('rust.md', scoped));
  });

  test('rejects missing front matter, missing description, unknown keys and bad paths', () => {
    expect(() => parseRule('a.md', '# No front matter\n')).toThrow('front matter');
    expect(() => parseRule('a.md', '---\npaths:\n  - "x"\n---\nbody\n')).toThrow('description');
    expect(() => parseRule('a.md', '---\ndescription: x\nglobs: "*.ts"\n---\nbody\n')).toThrow('unknown');
    expect(() => parseRule('a.md', '---\ndescription: x\npaths: []\n---\nbody\n')).toThrow('paths');
  });
});

describe('budgetProblems', () => {
  const rule = (name: string, size: number, paths: string[] = []): Rule => ({
    name,
    description: name,
    paths,
    body: 'x'.repeat(size),
  });

  test('accepts rules within the Antigravity limits', () => {
    expect(budgetProblems([rule('a', 1000), rule('b', 2000, ['**/*.rs'])])).toEqual([]);
  });

  test('flags a single oversized rule and an oversized always-on total', () => {
    expect(budgetProblems([rule('huge', 25_000, ['**/*.rs'])])).toHaveLength(1);
    expect(budgetProblems([rule('a', 20_000), rule('b', 20_000), rule('c', 21_000)])).toHaveLength(1);
  });
});
```

`scripts/tests/unit/agents/rule.render.test.ts`
```ts
import { describe, expect, test } from 'bun:test';

import type { Rule } from '../../../lib/agents/rule.model';
import { renderAntigravityRule, renderCursorRule } from '../../../lib/agents/rule.render';

const notice = (name: string) =>
  `<!-- Generated by \`bun run agents:sync\` from .claude/rules/${name}.md. Do not edit. -->`;
const scoped: Rule = {
  name: 'rust',
  description: 'Rust conventions',
  paths: ['**/*.rs', '**/Cargo.toml'],
  body: '# Rust\n\nUse edition 2024.',
};
const always: Rule = { name: 'project', description: 'Project basics', paths: [], body: '# Project' };

describe('renderCursorRule', () => {
  test('scoped rules become glob-attached .mdc rules', () => {
    expect(renderCursorRule(scoped)).toBe(
      `---\ndescription: "Rust conventions"\nglobs: **/*.rs,**/Cargo.toml\nalwaysApply: false\n---\n\n${notice('rust')}\n\n# Rust\n\nUse edition 2024.\n`,
    );
  });

  test('unscoped rules always apply', () => {
    expect(renderCursorRule(always)).toBe(
      `---\ndescription: "Project basics"\nalwaysApply: true\n---\n\n${notice('project')}\n\n# Project\n`,
    );
  });
});

describe('renderAntigravityRule', () => {
  test('scoped rules use the glob trigger with one comma-separated string', () => {
    expect(renderAntigravityRule(scoped)).toBe(
      `---\ntrigger: glob\ndescription: "Rust conventions"\nglobs: "**/*.rs, **/Cargo.toml"\n---\n\n${notice('rust')}\n\n# Rust\n\nUse edition 2024.\n`,
    );
  });

  test('unscoped rules are always on', () => {
    expect(renderAntigravityRule(always)).toBe(
      `---\ntrigger: always_on\ndescription: "Project basics"\n---\n\n${notice('project')}\n\n# Project\n`,
    );
  });
});
```

- [ ] **Step 2: Run to see them fail**

Run: `bun test ./scripts/tests/unit/agents`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement `scripts/lib/agents/rule.model.ts`**

```ts
export interface Rule {
  readonly name: string;
  readonly description: string;
  readonly paths: readonly string[];
  readonly body: string;
}

/** Antigravity limits (docs/research/tooling.md): 24 KB per rule; ~20k tokens for always-on rules. */
const RULE_MAX_BYTES = 24_000;
const ALWAYS_ON_MAX_BYTES = 60_000;

const FRONT_MATTER = /^---\n([\s\S]*?)\n---\n?([\s\S]*)$/;

const isGlobList = (value: unknown): value is string[] =>
  Array.isArray(value) && value.length > 0 && value.every((item) => typeof item === 'string' && item.length > 0);

/** Parses `.claude/rules/<fileName>` (front matter: `description`, optional `paths`). */
export function parseRule(fileName: string, source: string): Rule {
  const match = FRONT_MATTER.exec(source.replaceAll('\r\n', '\n'));
  if (match === null) {
    throw new Error(`${fileName}: missing front matter (--- description / paths ---)`);
  }
  const data: unknown = Bun.YAML.parse(match[1] ?? '');
  if (typeof data !== 'object' || data === null || Array.isArray(data)) {
    throw new Error(`${fileName}: front matter must be a mapping`);
  }
  const { description, paths, ...rest } = data as Record<string, unknown>;
  const unknownKeys = Object.keys(rest);
  if (unknownKeys.length > 0) {
    throw new Error(`${fileName}: unknown front-matter keys: ${unknownKeys.join(', ')}`);
  }
  if (typeof description !== 'string' || description.trim().length === 0) {
    throw new Error(`${fileName}: "description" is required`);
  }
  let globs: readonly string[] = [];
  if (paths !== undefined) {
    if (!isGlobList(paths)) {
      throw new Error(`${fileName}: "paths" must be a non-empty list of globs`);
    }
    globs = paths;
  }
  return {
    name: fileName.replace(/\.md$/, ''),
    description: description.trim(),
    paths: globs,
    body: (match[2] ?? '').trim(),
  };
}

const bytes = (text: string) => new TextEncoder().encode(text).length;

/** Problems with the Antigravity rule budgets (empty when within limits). */
export function budgetProblems(rules: readonly Rule[]): string[] {
  const problems = rules
    .filter((rule) => bytes(rule.body) > RULE_MAX_BYTES)
    .map((rule) => `${rule.name}.md is over ${RULE_MAX_BYTES} bytes; split it`);
  const alwaysOn = rules.filter((rule) => rule.paths.length === 0).reduce((sum, rule) => sum + bytes(rule.body), 0);
  if (alwaysOn > ALWAYS_ON_MAX_BYTES) {
    problems.push(`always-on rules total ${alwaysOn} bytes (limit ${ALWAYS_ON_MAX_BYTES}); scope some with paths`);
  }
  return problems;
}
```

- [ ] **Step 4: Implement `scripts/lib/agents/rule.render.ts`**

```ts
import type { Rule } from './rule.model';

const notice = (rule: Rule) =>
  `<!-- Generated by \`bun run agents:sync\` from .claude/rules/${rule.name}.md. Do not edit. -->`;

/** JSON strings are valid YAML double-quoted scalars. */
const quoted = (text: string) => JSON.stringify(text);

function document(frontMatter: readonly string[], rule: Rule): string {
  return ['---', ...frontMatter, '---', '', notice(rule), '', rule.body, ''].join('\n');
}

/** `.cursor/rules/<name>.mdc` — globs are comma-separated and unquoted (Cursor's format). */
export function renderCursorRule(rule: Rule): string {
  const frontMatter =
    rule.paths.length > 0
      ? [`description: ${quoted(rule.description)}`, `globs: ${rule.paths.join(',')}`, 'alwaysApply: false']
      : [`description: ${quoted(rule.description)}`, 'alwaysApply: true'];
  return document(frontMatter, rule);
}

/** `.agents/rules/<name>.md` — Antigravity requires `trigger`; globs are one comma-separated string. */
export function renderAntigravityRule(rule: Rule): string {
  const frontMatter =
    rule.paths.length > 0
      ? ['trigger: glob', `description: ${quoted(rule.description)}`, `globs: ${quoted(rule.paths.join(', '))}`]
      : ['trigger: always_on', `description: ${quoted(rule.description)}`];
  return document(frontMatter, rule);
}
```

- [ ] **Step 5: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/agents`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add scripts/lib/agents scripts/tests/unit/agents
git commit -m "feat(agents): add rule model with cursor and antigravity renderers"
```

### Task 18: `bun run agents:sync` with drift check

**Files:**
- Create: `scripts/lib/agents/sync.plan.ts`, `scripts/agents/sync.ts`
- Test: `scripts/tests/unit/agents/sync.plan.test.ts`
- Modify: `scripts/commands/check.ts` (add the drift step), `.config/lefthook.yml` (pre-commit job), `package.json` (`scripts["agents:sync"]`)

**Interfaces:**
- Consumes: `parseRule`, `budgetProblems`, `renderCursorRule`, `renderAntigravityRule` (Task 17); `repoPath` (Task 7).
- Produces: `SyncTarget { dir: string; extension: string; render: (rule: Rule) => string }`, `SYNC_TARGETS`, `planSync(rules, targets?): Map<string, string>` (repo-relative path → content), `diffSync(planned, existing): { write: string[]; remove: string[] }`; command `bun run agents:sync [--check]`.

- [ ] **Step 1: Write the failing tests** — `scripts/tests/unit/agents/sync.plan.test.ts`

```ts
import { describe, expect, test } from 'bun:test';

import type { Rule } from '../../../lib/agents/rule.model';
import { diffSync, planSync } from '../../../lib/agents/sync.plan';

const rules: Rule[] = [
  { name: 'project', description: 'Project basics', paths: [], body: '# Project' },
  { name: 'rust', description: 'Rust', paths: ['**/*.rs'], body: '# Rust' },
];

describe('planSync', () => {
  test('plans one Cursor and one Antigravity file per rule', () => {
    expect([...planSync(rules).keys()].toSorted()).toEqual([
      '.agents/rules/project.md',
      '.agents/rules/rust.md',
      '.cursor/rules/project.mdc',
      '.cursor/rules/rust.mdc',
    ]);
  });
});

describe('diffSync', () => {
  const planned = planSync(rules);

  test('nothing to do when everything matches', () => {
    expect(diffSync(planned, new Map(planned))).toEqual({ write: [], remove: [] });
  });

  test('writes missing and changed files, removes files with no source rule', () => {
    const existing = new Map(planned);
    existing.delete('.cursor/rules/rust.mdc');
    existing.set('.agents/rules/project.md', 'hand-edited');
    existing.set('.cursor/rules/old.mdc', 'stale');
    expect(diffSync(planned, existing)).toEqual({
      write: ['.agents/rules/project.md', '.cursor/rules/rust.mdc'],
      remove: ['.cursor/rules/old.mdc'],
    });
  });
});
```

- [ ] **Step 2: Run to see it fail**

Run: `bun test ./scripts/tests/unit/agents/sync.plan.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `scripts/lib/agents/sync.plan.ts`**

```ts
import type { Rule } from './rule.model';
import { renderAntigravityRule, renderCursorRule } from './rule.render';

export interface SyncTarget {
  readonly dir: string;
  readonly extension: string;
  readonly render: (rule: Rule) => string;
}

/** Folders fully owned by `agents:sync` (only files with the target extension are managed). */
export const SYNC_TARGETS: readonly SyncTarget[] = [
  { dir: '.cursor/rules', extension: '.mdc', render: renderCursorRule },
  { dir: '.agents/rules', extension: '.md', render: renderAntigravityRule },
];

export function planSync(rules: readonly Rule[], targets: readonly SyncTarget[] = SYNC_TARGETS): Map<string, string> {
  const planned = new Map<string, string>();
  for (const target of targets) {
    for (const rule of rules) {
      planned.set(`${target.dir}/${rule.name}${target.extension}`, target.render(rule));
    }
  }
  return planned;
}

export function diffSync(
  planned: ReadonlyMap<string, string>,
  existing: ReadonlyMap<string, string>,
): { write: string[]; remove: string[] } {
  const write = [...planned].filter(([path, content]) => existing.get(path) !== content).map(([path]) => path);
  const remove = [...existing.keys()].filter((path) => !planned.has(path));
  return { write: write.toSorted(), remove: remove.toSorted() };
}
```

- [ ] **Step 4: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/agents`
Expected: PASS.

- [ ] **Step 5: Implement `scripts/agents/sync.ts`**

```ts
// bun run agents:sync [--check] — generate Cursor and Antigravity rules from .claude/rules/*.md.
// --check: fail (exit 1) when the generated files differ from what the sources produce.
import { existsSync } from 'node:fs';
import { mkdir, rm } from 'node:fs/promises';

import { budgetProblems, parseRule, type Rule } from '../lib/agents/rule.model';
import { diffSync, planSync, SYNC_TARGETS } from '../lib/agents/sync.plan';
import { repoPath } from '../lib/repo/repo.util';

const check = process.argv.includes('--check');
const sourceDir = repoPath('.claude', 'rules');

const rules: Rule[] = [];
for await (const file of new Bun.Glob('*.md').scan({ cwd: sourceDir })) {
  rules.push(parseRule(file, await Bun.file(repoPath('.claude', 'rules', file)).text()));
}
rules.sort((a, b) => a.name.localeCompare(b.name));

const budget = budgetProblems(rules);
if (budget.length > 0) {
  for (const problem of budget) console.error(`✖ ${problem}`);
  process.exit(1);
}

const existing = new Map<string, string>();
for (const target of SYNC_TARGETS) {
  const dir = repoPath(target.dir);
  if (!existsSync(dir)) continue;
  for await (const file of new Bun.Glob(`*${target.extension}`).scan({ cwd: dir })) {
    const text = await Bun.file(repoPath(target.dir, file)).text();
    existing.set(`${target.dir}/${file}`, text.replaceAll('\r\n', '\n'));
  }
}

const planned = planSync(rules);
const diff = diffSync(planned, existing);

if (check) {
  if (diff.write.length + diff.remove.length > 0) {
    for (const path of diff.write) console.error(`✖ out of date: ${path}`);
    for (const path of diff.remove) console.error(`✖ no source rule: ${path}`);
    console.error('agent rules are out of sync — run: bun run agents:sync');
    process.exit(1);
  }
  console.log(`agents:sync: ${rules.length} rules in sync`);
  process.exit(0);
}

for (const target of SYNC_TARGETS) {
  await mkdir(repoPath(target.dir), { recursive: true });
}
for (const path of diff.write) {
  await Bun.write(repoPath(path), planned.get(path) ?? '');
}
for (const path of diff.remove) {
  await rm(repoPath(path));
}
console.log(`agents:sync: ${rules.length} rules → ${diff.write.length} written, ${diff.remove.length} removed`);
```

Add to `package.json` `scripts`: `"agents:sync": "bun scripts/agents/sync.ts"`.

- [ ] **Step 6: Wire the drift check into `check` and the pre-commit hook**

In `scripts/commands/check.ts`, add this entry to `CHECK_STEPS` directly after the `attribution` step:
```ts
  commandStep('agent rules in sync', ['bun', 'scripts/agents/sync.ts', '--check']),
```

In `.config/lefthook.yml`, add this job to `pre-commit.jobs`:
```yaml
    - name: agents-sync
      glob: ".claude/rules/*.md"
      run: bun scripts/agents/sync.ts --check
```

- [ ] **Step 7: Run it**

Run: `bun run agents:sync && bun scripts/agents/sync.ts --check`
Expected: `agents:sync: 0 rules → 0 written, 0 removed` then `agents:sync: 0 rules in sync` (Task 21 adds the rules).

- [ ] **Step 8: Commit**

```bash
git add scripts/lib/agents/sync.plan.ts scripts/agents/sync.ts scripts/tests/unit/agents/sync.plan.test.ts scripts/commands/check.ts .config/lefthook.yml package.json
git commit -m "feat(agents): add agents:sync with drift check in check and pre-commit"
```

### Task 19: Hook protocol helpers (input, output, generated paths)

**Files:**
- Create: `scripts/lib/agents/hook-input.parse.ts`, `scripts/lib/agents/hook-output.util.ts`, `scripts/lib/agents/generated.rules.ts`
- Test: `scripts/tests/unit/agents/hook-input.parse.test.ts`, `scripts/tests/unit/agents/hook-output.util.test.ts`, `scripts/tests/unit/agents/generated.rules.test.ts`

**Interfaces:**
- Consumes: `toRepoRelative` (Task 7).
- Produces:
  - `AgentTool = 'claude' | 'cursor' | 'antigravity'`; `parseAgentTool(argv: readonly string[]): AgentTool` (`--tool <name>`, default `claude`)
  - `HookInput { toolName: string | null; filePaths: readonly string[]; command: string | null }`; `parseHookInput(raw: string): HookInput` — understands Claude (`tool_name`, `tool_input.file_path`, `tool_input.command`), Cursor (`tool_name`, `tool_input.*`, root `file_path`, root `command`) and Antigravity (`toolCall.name`, `toolCall.args.*`) payloads
  - `isReadOnlyTool(name: string | null): boolean` — true for read/view/list/search tools, which the generated-file guard always allows (Antigravity's matcher is `*`)
  - `HookDecision { stdout: string; stderr: string; exitCode: number }`; `denyDecision(tool, message)`, `allowDecision(tool)`, `passDecision(tool)`; `emit(decision): Promise<never>`
  - `generatedFix(repoRelativePath: string): string | null`; `generatedProblems(filePaths: readonly string[], root: string): string[]`

Per-tool protocol (from `docs/research/tooling.md`):

| Tool | Deny | Allow | Post-edit "pass" |
|---|---|---|---|
| claude | message on stderr, exit 2 | nothing, exit 0 | nothing, exit 0 |
| cursor | stdout `{"permission":"deny","user_message":m,"agent_message":m}`, stderr m, exit 2 | stdout `{"permission":"allow"}`, exit 0 | nothing, exit 0 |
| antigravity | stdout `{"decision":"deny","reason":m}`, exit 0 | stdout `{"decision":"allow"}`, exit 0 | stdout `{}`, exit 0 |

- [ ] **Step 1: Write the failing tests**

`scripts/tests/unit/agents/hook-input.parse.test.ts`
```ts
import { describe, expect, test } from 'bun:test';

import { isReadOnlyTool, parseAgentTool, parseHookInput } from '../../../lib/agents/hook-input.parse';

describe('parseHookInput', () => {
  test('Claude Edit and Bash payloads', () => {
    expect(
      parseHookInput(JSON.stringify({ tool_name: 'Edit', tool_input: { file_path: 'C:\\r\\bun.lock', old_string: 'a' }, transcript_path: 'C:\\t.jsonl' })),
    ).toEqual({ toolName: 'Edit', filePaths: ['C:\\r\\bun.lock'], command: null });
    expect(parseHookInput(JSON.stringify({ tool_name: 'Bash', tool_input: { command: 'git status' } }))).toEqual({
      toolName: 'Bash',
      filePaths: [],
      command: 'git status',
    });
  });

  test('Cursor preToolUse, beforeShellExecution and afterFileEdit payloads', () => {
    expect(parseHookInput(JSON.stringify({ tool_name: 'Write', tool_input: { path: 'Cargo.lock' }, cwd: '/r' }))).toEqual({
      toolName: 'Write',
      filePaths: ['Cargo.lock'],
      command: null,
    });
    expect(parseHookInput(JSON.stringify({ command: 'git checkout -b x', cwd: '/r', sandbox: false }))).toEqual({
      toolName: null,
      filePaths: [],
      command: 'git checkout -b x',
    });
    expect(
      parseHookInput(JSON.stringify({ file_path: '/r/a.ts', edits: [{ old_string: 'a', new_string: 'b' }] })),
    ).toEqual({ toolName: null, filePaths: ['/r/a.ts'], command: null });
  });

  test('Antigravity toolCall payloads', () => {
    expect(
      parseHookInput(JSON.stringify({ conversationId: 'c', toolCall: { name: 'write_to_file', args: { TargetFile: 'S:\\r\\bun.lock' } } })),
    ).toEqual({ toolName: 'write_to_file', filePaths: ['S:\\r\\bun.lock'], command: null });
    expect(
      parseHookInput(JSON.stringify({ toolCall: { name: 'run_command', args: { CommandLine: 'git commit -m x' } } })),
    ).toEqual({ toolName: 'run_command', filePaths: [], command: 'git commit -m x' });
  });

  test('invalid JSON yields an empty input', () => {
    expect(parseHookInput('not json')).toEqual({ toolName: null, filePaths: [], command: null });
  });
});

test('read-only tools are recognised so the generated-file guard lets them read', () => {
  expect(isReadOnlyTool('view_file')).toBe(true);
  expect(isReadOnlyTool('Read')).toBe(true);
  expect(isReadOnlyTool('list_dir')).toBe(true);
  expect(isReadOnlyTool('grep_search')).toBe(true);
  expect(isReadOnlyTool('write_to_file')).toBe(false);
  expect(isReadOnlyTool('Edit')).toBe(false);
  expect(isReadOnlyTool(null)).toBe(false);
});

test('parseAgentTool reads --tool and defaults to claude', () => {
  expect(parseAgentTool(['--tool', 'cursor'])).toBe('cursor');
  expect(parseAgentTool(['--tool', 'antigravity'])).toBe('antigravity');
  expect(parseAgentTool([])).toBe('claude');
  expect(() => parseAgentTool(['--tool', 'vim'])).toThrow();
});
```

`scripts/tests/unit/agents/hook-output.util.test.ts`
```ts
import { expect, test } from 'bun:test';

import { allowDecision, denyDecision, passDecision } from '../../../lib/agents/hook-output.util';

test('deny speaks each tool protocol', () => {
  expect(denyDecision('claude', 'no')).toEqual({ stdout: '', stderr: 'no', exitCode: 2 });
  expect(denyDecision('cursor', 'no')).toEqual({
    stdout: '{"permission":"deny","user_message":"no","agent_message":"no"}',
    stderr: 'no',
    exitCode: 2,
  });
  expect(denyDecision('antigravity', 'no')).toEqual({
    stdout: '{"decision":"deny","reason":"no"}',
    stderr: '',
    exitCode: 0,
  });
});

test('allow and pass speak each tool protocol', () => {
  expect(allowDecision('claude')).toEqual({ stdout: '', stderr: '', exitCode: 0 });
  expect(allowDecision('cursor')).toEqual({ stdout: '{"permission":"allow"}', stderr: '', exitCode: 0 });
  expect(allowDecision('antigravity')).toEqual({ stdout: '{"decision":"allow"}', stderr: '', exitCode: 0 });
  expect(passDecision('cursor')).toEqual({ stdout: '', stderr: '', exitCode: 0 });
  expect(passDecision('antigravity')).toEqual({ stdout: '{}', stderr: '', exitCode: 0 });
});
```

`scripts/tests/unit/agents/generated.rules.test.ts`
```ts
import { describe, expect, test } from 'bun:test';
import { join, resolve } from 'node:path';

import { generatedFix, generatedProblems } from '../../../lib/agents/generated.rules';

describe('generatedFix', () => {
  test('names the source to edit for each generated area', () => {
    expect(generatedFix('packages/tokens/src/generated/css/tokens.css')).toContain('bun run tokens');
    expect(generatedFix('crates/design-tokens/src/generated/tokens.rs')).toContain('bun run tokens');
    expect(generatedFix('packages/tauri-bridge/src/ipc/bindings.generated.ts')).not.toBeNull();
    expect(generatedFix('programs/desktop/launcher/src-tauri/gen/schemas/x.json')).not.toBeNull();
    expect(generatedFix('bun.lock')).toContain('bun install');
    expect(generatedFix('Cargo.lock')).not.toBeNull();
    expect(generatedFix('.cursor/rules/rust.mdc')).toContain('agents:sync');
    expect(generatedFix('.agents/rules/rust.md')).toContain('agents:sync');
  });

  test('ordinary files are editable', () => {
    expect(generatedFix('scripts/lib/repo/repo.util.ts')).toBeNull();
    expect(generatedFix('.claude/rules/rust.md')).toBeNull();
    expect(generatedFix('packages/tokens/src/tokens/color.tokens.ts')).toBeNull();
  });
});

describe('generatedProblems', () => {
  const root = resolve('/work/genslate');

  test('maps absolute paths into the repo and ignores outside paths', () => {
    expect(generatedProblems([join(root, 'bun.lock'), resolve(root, '..', 'bun.lock'), join(root, 'README.md')], root)).toHaveLength(1);
  });

  test.skipIf(process.platform !== 'win32')('handles backslashes and drive-letter case', () => {
    const winRoot = 'S:\\DEVELOPMENT\\PROJECTS\\GENSLATE-LAUNCHER';
    expect(generatedProblems(['s:\\DEVELOPMENT\\PROJECTS\\GENSLATE-LAUNCHER\\Cargo.lock'], winRoot)).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run to see them fail**

Run: `bun test ./scripts/tests/unit/agents`
Expected: FAIL — the three new modules are not found.

- [ ] **Step 3: Implement `scripts/lib/agents/hook-input.parse.ts`**

```ts
export type AgentTool = 'claude' | 'cursor' | 'antigravity';

export interface HookInput {
  readonly toolName: string | null;
  readonly filePaths: readonly string[];
  readonly command: string | null;
}

const TOOLS: readonly AgentTool[] = ['claude', 'cursor', 'antigravity'];
const PATH_KEY = /^(?:file_?path|path|target_?file|absolute_?path)$/i;
const COMMAND_KEY = /^(?:command|command_?line|cmd)$/i;
const READ_ONLY_TOOL = /read|view|list|search|grep|glob|find|fetch/i;
const MAX_DEPTH = 4;

/** Read/view/list/search tools never change files, so file guards let them through. */
export function isReadOnlyTool(name: string | null): boolean {
  return name !== null && READ_ONLY_TOOL.test(name);
}

/** `--tool <claude|cursor|antigravity>`; defaults to claude. */
export function parseAgentTool(argv: readonly string[]): AgentTool {
  const index = argv.indexOf('--tool');
  if (index < 0) {
    return 'claude';
  }
  const value = argv[index + 1];
  const tool = TOOLS.find((name) => name === value);
  if (tool === undefined) {
    throw new Error(`unknown --tool "${value ?? ''}" (expected ${TOOLS.join(', ')})`);
  }
  return tool;
}

function collect(value: unknown, depth: number, paths: string[], commands: string[]): void {
  if (depth > MAX_DEPTH || typeof value !== 'object' || value === null) {
    return;
  }
  for (const [key, child] of Object.entries(value)) {
    if (typeof child === 'string') {
      if (PATH_KEY.test(key)) paths.push(child);
      else if (COMMAND_KEY.test(key)) commands.push(child);
    } else {
      collect(child, depth + 1, paths, commands);
    }
  }
}

/** Extracts file paths and the shell command from a Claude, Cursor or Antigravity hook payload. */
export function parseHookInput(raw: string): HookInput {
  let data: unknown;
  try {
    data = JSON.parse(raw);
  } catch {
    return { toolName: null, filePaths: [], command: null };
  }
  const record =
    typeof data === 'object' && data !== null
      ? (data as { tool_name?: unknown; toolCall?: { name?: unknown } })
      : {};
  const claudeOrCursor = typeof record.tool_name === 'string' ? record.tool_name : null;
  const antigravity = typeof record.toolCall?.name === 'string' ? record.toolCall.name : null;
  const paths: string[] = [];
  const commands: string[] = [];
  collect(data, 0, paths, commands);
  return {
    toolName: claudeOrCursor ?? antigravity,
    filePaths: [...new Set(paths)],
    command: commands[0] ?? null,
  };
}
```

- [ ] **Step 4: Implement `scripts/lib/agents/hook-output.util.ts`**

```ts
import type { AgentTool } from './hook-input.parse';

export interface HookDecision {
  readonly stdout: string;
  readonly stderr: string;
  readonly exitCode: number;
}

export function denyDecision(tool: AgentTool, message: string): HookDecision {
  switch (tool) {
    case 'claude':
      return { stdout: '', stderr: message, exitCode: 2 };
    case 'cursor':
      return {
        stdout: JSON.stringify({ permission: 'deny', user_message: message, agent_message: message }),
        stderr: message,
        exitCode: 2,
      };
    case 'antigravity':
      return { stdout: JSON.stringify({ decision: 'deny', reason: message }), stderr: '', exitCode: 0 };
  }
}

export function allowDecision(tool: AgentTool): HookDecision {
  switch (tool) {
    case 'claude':
      return { stdout: '', stderr: '', exitCode: 0 };
    case 'cursor':
      return { stdout: JSON.stringify({ permission: 'allow' }), stderr: '', exitCode: 0 };
    case 'antigravity':
      return { stdout: JSON.stringify({ decision: 'allow' }), stderr: '', exitCode: 0 };
  }
}

/** For post-edit hooks that never block. */
export function passDecision(tool: AgentTool): HookDecision {
  return { stdout: tool === 'antigravity' ? '{}' : '', stderr: '', exitCode: 0 };
}

/** Writes the decision and exits with its code. */
export async function emit(decision: HookDecision): Promise<never> {
  if (decision.stdout.length > 0) await Bun.write(Bun.stdout, `${decision.stdout}\n`);
  if (decision.stderr.length > 0) await Bun.write(Bun.stderr, `${decision.stderr}\n`);
  process.exit(decision.exitCode);
}
```

- [ ] **Step 5: Implement `scripts/lib/agents/generated.rules.ts`**

```ts
import { toRepoRelative } from '../repo/repo.util';

const TOKENS = 'edit packages/tokens/src and run `bun run tokens`';
const RULES = 'edit .claude/rules/ and run `bun run agents:sync`';

const GENERATED: readonly { readonly pattern: RegExp; readonly fix: string }[] = [
  { pattern: /^packages\/tokens\/src\/generated\//, fix: TOKENS },
  { pattern: /^crates\/design-tokens\/src\/generated\//, fix: TOKENS },
  { pattern: /\.generated\.ts$/, fix: 'edit the generator input and re-run its generator' },
  { pattern: /(^|\/)src-tauri\/gen\//, fix: 'Tauri writes this folder; change the source config instead' },
  { pattern: /^bun\.lock$/, fix: 'change package.json and run `bun install`' },
  { pattern: /^Cargo\.lock$/, fix: 'change Cargo.toml and let cargo update the lockfile' },
  { pattern: /^\.cursor\/rules\//, fix: RULES },
  { pattern: /^\.agents\/rules\//, fix: RULES },
];

/** How to change a generated file properly, or null when the path is an ordinary source file. */
export function generatedFix(repoRelativePath: string): string | null {
  return GENERATED.find(({ pattern }) => pattern.test(repoRelativePath))?.fix ?? null;
}

/** One message per generated file among `filePaths` (absolute or root-relative; outside paths ignored). */
export function generatedProblems(filePaths: readonly string[], root: string): string[] {
  const problems: string[] = [];
  for (const path of filePaths) {
    const relative = toRepoRelative(path, root);
    const fix = relative === null ? null : generatedFix(relative);
    if (relative !== null && fix !== null) {
      problems.push(`${relative} is generated — ${fix}.`);
    }
  }
  return problems;
}
```

- [ ] **Step 6: Run the tests to see them pass**

Run: `bun test ./scripts/tests/unit/agents`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add scripts/lib/agents scripts/tests/unit/agents
git commit -m "feat(agents): add hook input, output and generated-path helpers for all agent tools"
```

### Task 20: Guard hooks wired into Claude, Cursor and Antigravity

**Files:**
- Create: `scripts/agents/hooks/guard-generated.hook.ts`, `scripts/agents/hooks/guard-attribution.hook.ts`, `scripts/agents/hooks/format-on-edit.hook.ts`, `scripts/agents/hooks/session-start.hook.ts`, `.cursor/hooks.json`
- Overwrite: `.claude/settings.json`, `.agents/hooks.json`

**Interfaces:**
- Consumes: Task 19 helpers; `attributionProblems` (Task 10); `REPO_ROOT`, `repoPath`, `toRepoRelative` (Task 7); `capture` (Task 7).
- Produces: one script per guard, each taking `--tool <claude|cursor|antigravity>`.

- [ ] **Step 1: Write the hook scripts**

`scripts/agents/hooks/guard-generated.hook.ts`
```ts
// Blocks agent edits to generated files. Wired into Claude (PreToolUse), Cursor (preToolUse) and
// Antigravity (PreToolUse). Usage: bun scripts/agents/hooks/guard-generated.hook.ts --tool <tool>
import { generatedProblems } from '../../lib/agents/generated.rules';
import { isReadOnlyTool, parseAgentTool, parseHookInput } from '../../lib/agents/hook-input.parse';
import { allowDecision, denyDecision, emit } from '../../lib/agents/hook-output.util';
import { REPO_ROOT } from '../../lib/repo/repo.util';

const tool = parseAgentTool(process.argv.slice(2));
const input = parseHookInput(await Bun.stdin.text());
const problems = isReadOnlyTool(input.toolName) ? [] : generatedProblems(input.filePaths, REPO_ROOT);

await emit(
  problems.length > 0
    ? denyDecision(tool, `Blocked: generated file.\n${problems.join('\n')}`)
    : allowDecision(tool),
);
```

`scripts/agents/hooks/guard-attribution.hook.ts`
```ts
// Blocks shell commands that would add agent attribution (commit trailers/footers, agent branch names).
// Wired into Claude (PreToolUse: Bash), Cursor (beforeShellExecution) and Antigravity (PreToolUse).
import { attributionProblems } from '../../lib/attribution/attribution.rules';
import { parseAgentTool, parseHookInput } from '../../lib/agents/hook-input.parse';
import { allowDecision, denyDecision, emit } from '../../lib/agents/hook-output.util';

const tool = parseAgentTool(process.argv.slice(2));
const { command } = parseHookInput(await Bun.stdin.text());
const problems = command === null ? [] : attributionProblems(command);

await emit(
  problems.length > 0
    ? denyDecision(
        tool,
        `Blocked: agent attribution is not allowed in this repo (${problems.join('; ')}). ` +
          'Write a plain Conventional Commit message and a non-agent branch name.',
      )
    : allowDecision(tool),
);
```

`scripts/agents/hooks/format-on-edit.hook.ts`
```ts
// Formats files an agent just edited (Biome for web files, rustfmt for Rust). Never blocks.
// Wired into Claude (PostToolUse), Cursor (afterFileEdit) and Antigravity (PostToolUse).
import { existsSync } from 'node:fs';

import { generatedFix } from '../../lib/agents/generated.rules';
import { parseAgentTool, parseHookInput } from '../../lib/agents/hook-input.parse';
import { emit, passDecision } from '../../lib/agents/hook-output.util';
import { capture } from '../../lib/process/exec.util';
import { repoPath, toRepoRelative } from '../../lib/repo/repo.util';

const BIOME_FILES = /\.(?:ts|tsx|js|mjs|cjs|json|jsonc|css)$/;

const tool = parseAgentTool(process.argv.slice(2));
const { filePaths } = parseHookInput(await Bun.stdin.text());

for (const path of filePaths) {
  const relative = toRepoRelative(path);
  if (relative === null || generatedFix(relative) !== null || !existsSync(repoPath(relative))) {
    continue;
  }
  if (BIOME_FILES.test(relative)) {
    await capture([
      'bun', 'x', 'biome', 'format', '--config-path=.config/biome.json', '--write', '--no-errors-on-unmatched', relative,
    ]);
  } else if (relative.endsWith('.rs') && Bun.which('rustfmt') !== null) {
    await capture(['rustfmt', '--edition', '2024', relative]);
  }
}

await emit(passDecision(tool));
```

`scripts/agents/hooks/session-start.hook.ts`
```ts
// Claude Code SessionStart hook: prints short repo context (stdout is added to the session).
import { capture } from '../../lib/process/exec.util';
import { repoPath } from '../../lib/repo/repo.util';

const branch = (await capture(['git', 'branch', '--show-current'])).stdout.trim() || '(detached)';
const changes = (await capture(['git', 'status', '--porcelain'])).stdout.split('\n').filter(Boolean).length;
const activeContext = Bun.file(repoPath('.claude', 'memory', 'active-context.md'));
const context = (await activeContext.exists()) ? (await activeContext.text()).trim() : '';

console.log(
  [
    `GENSLATE session — branch ${branch}, ${changes} uncommitted change(s).`,
    'Rules: AGENTS.md and .claude/rules/. Spec and plans: docs/superpowers/. Use bun only; no agent attribution.',
    context,
  ]
    .filter((line) => line.length > 0)
    .join('\n'),
);
```

- [ ] **Step 2: Try each guard with sample payloads for every tool**

Run (bash, from the repo root):
```bash
printf '{"tool_name":"Edit","tool_input":{"file_path":"%s/bun.lock"}}' "$PWD" | bun scripts/agents/hooks/guard-generated.hook.ts --tool claude; echo "exit=$?"
printf '{"tool_name":"Edit","tool_input":{"file_path":"%s/README.md"}}' "$PWD" | bun scripts/agents/hooks/guard-generated.hook.ts --tool claude; echo "exit=$?"
printf '{"command":"git checkout -b claude/x","cwd":"."}' | bun scripts/agents/hooks/guard-attribution.hook.ts --tool cursor; echo "exit=$?"
printf '{"toolCall":{"name":"write_to_file","args":{"TargetFile":"Cargo.lock"}}}' | bun scripts/agents/hooks/guard-generated.hook.ts --tool antigravity; echo "exit=$?"
printf '{"toolCall":{"name":"run_command","args":{"CommandLine":"git status"}}}' | bun scripts/agents/hooks/guard-attribution.hook.ts --tool antigravity; echo "exit=$?"
```
Expected, in order: a `Blocked: generated file … bun.lock … bun install` message and `exit=2`; `exit=0`; `{"permission":"deny",…}` and `exit=2`; `{"decision":"deny",…}` and `exit=0`; `{"decision":"allow"}` and `exit=0`.

- [ ] **Step 3: Write `.claude/settings.json`**

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true
  },
  "attribution": {
    "commit": "",
    "pr": ""
  },
  "permissions": {
    "allow": [
      "Bash(bun run:*)",
      "Bash(bun test:*)",
      "Bash(bun x biome:*)",
      "Bash(bun x tsc:*)",
      "Bash(bun x turbo:*)",
      "Bash(bun scripts/:*)",
      "Bash(cargo build:*)",
      "Bash(cargo check:*)",
      "Bash(cargo clippy:*)",
      "Bash(cargo fmt:*)",
      "Bash(cargo nextest:*)",
      "Bash(cargo test:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git show:*)",
      "Bash(git branch:*)"
    ],
    "deny": [
      "Bash(npm:*)",
      "Bash(npx:*)",
      "Bash(pnpm:*)",
      "Bash(yarn:*)",
      "Edit(bun.lock)",
      "Edit(Cargo.lock)",
      "Edit(**/generated/**)",
      "Edit(**/*.generated.ts)",
      "Edit(**/src-tauri/gen/**)",
      "Edit(.cursor/rules/**)",
      "Edit(.agents/rules/**)"
    ]
  },
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          { "type": "command", "command": "bun \"$CLAUDE_PROJECT_DIR/scripts/agents/hooks/session-start.hook.ts\"" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write|NotebookEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bun \"$CLAUDE_PROJECT_DIR/scripts/agents/hooks/guard-generated.hook.ts\" --tool claude"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bun \"$CLAUDE_PROJECT_DIR/scripts/agents/hooks/guard-attribution.hook.ts\" --tool claude"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bun \"$CLAUDE_PROJECT_DIR/scripts/agents/hooks/format-on-edit.hook.ts\" --tool claude"
          }
        ]
      }
    ]
  }
}
```

(`attribution` with empty strings turns off Claude Code's own co-author trailer and PR footer; the guard is the second line of defence.)

- [ ] **Step 4: Write `.cursor/hooks.json`** (project hooks run from the repo root)

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "matcher": "Write|Delete",
        "command": "bun scripts/agents/hooks/guard-generated.hook.ts --tool cursor"
      }
    ],
    "beforeShellExecution": [
      { "command": "bun scripts/agents/hooks/guard-attribution.hook.ts --tool cursor" }
    ],
    "afterFileEdit": [
      { "command": "bun scripts/agents/hooks/format-on-edit.hook.ts --tool cursor" }
    ]
  }
}
```

- [ ] **Step 5: Write `.agents/hooks.json`**

```json
{
  "genslate-guards": {
    "enabled": true,
    "PreToolUse": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "bun scripts/agents/hooks/guard-generated.hook.ts --tool antigravity" },
          { "type": "command", "command": "bun scripts/agents/hooks/guard-attribution.hook.ts --tool antigravity" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "bun scripts/agents/hooks/format-on-edit.hook.ts --tool antigravity" }
        ]
      }
    ]
  }
}
```

- [ ] **Step 6: Verify in each tool (manual, once)**

1. **Claude Code:** restart the session in this repo; ask it to "add a blank line to bun.lock". Expected: the edit is blocked with the guard message.
2. **Cursor:** open the repo; in Agent mode ask the same. Expected: blocked. Open *Settings → Hooks* to confirm the three hooks are listed.
3. **Antigravity:** open the repo; ask the agent the same. Expected: blocked. If Antigravity reports a schema error for `.agents/hooks.json`, open its hooks docs (`https://antigravity.google/docs/hooks`), adjust the file to the documented shape, re-test, and note the difference in `docs/research/tooling.md`.

- [ ] **Step 7: Run check and commit**

```bash
bun run check
git add scripts/agents/hooks .claude/settings.json .cursor/hooks.json .agents/hooks.json
git commit -m "feat(agents): wire generated-file, attribution and format guards into claude, cursor and antigravity"
```

### Task 21: Shared rules, `AGENTS.md` and `CLAUDE.md`

**Files:**
- Create: `.claude/rules/{project,monorepo-structure,naming,commands,dependencies,typescript,react,rust,tauri-ipc,security,performance,design-system,testing,git-and-attribution,docs,verify-latest}.md`
- Overwrite: `AGENTS.md`, `CLAUDE.md`
- Generated by the sync: `.cursor/rules/*.mdc`, `.agents/rules/*.md`

**Interfaces:**
- Consumes: `bun run agents:sync` (Task 18).
- Produces: the single rule source all three tools follow. Seven rules are always on (`project`, `monorepo-structure`, `naming`, `commands`, `security`, `git-and-attribution`, `verify-latest`); the rest are path-scoped.

- [ ] **Step 1: Write the always-on rules**

`.claude/rules/project.md`
```markdown
---
description: What GENSLATE is, where the spec and plans live, the SLATESUITE reference and the definition of done.
---

# Project

- **Product:** a portable "operating system on a stick" for Windows 10/11 x64. A tray launcher runs
  GENSLATE apps first and PortableApps.com and portapps.io apps second, plus the user's documents — all inside
  one install folder on a USB stick or SSD. Nothing is written to the host except what spec §5.3 allows.
- **Source of truth:** `docs/superpowers/specs/2026-10-07-genslate-launcher-design.md`. Each phase has a
  plan in `docs/superpowers/plans/`; follow it task by task. If reality contradicts the spec or plan, stop and
  ask the user instead of improvising.
- **Reference, not source:** `S:\DEVELOPMENT\PROJECTS\GENSLATE-SLATESUITE` defines how the design system and
  launcher look and behave. Read it to match them exactly; never copy files from it.
- **Themes:** official Nord — Polar Night is the dark theme, Snow Storm the light theme.
- **Definition of done:** `bun run check` and `bun run test` pass; UI changes are checked in both themes;
  docs are updated when behaviour changes; user-visible changes have a change entry (`bun run change`).
```

`.claude/rules/monorepo-structure.md`
```markdown
---
description: Where code lives in the monorepo and where new files go.
---

# Monorepo structure

| Path | Holds |
|---|---|
| `programs/desktop/<app>/` | Tauri apps: `src/` (React UI), `src-tauri/` (thin Rust shell), `tests/{unit,e2e}/`, `other/<app>/` (shipped template) |
| `programs/desktop/launcher/installDir/` | Mock portable drive used by dev builds (spec §5.1) |
| `programs/desktop/start/` | `Start.exe` (start, eject and restore modes) |
| `programs/webapp/<name>/` | Web apps; `design-system/` is the Design Kit |
| `crates/<name>/` | Shared Rust crates (logic lives here, never in `src-tauri`) |
| `packages/<name>/` | Shared TypeScript packages (`tokens`, `design-system`, `tauri-bridge`, configs) |
| `scripts/commands/<cmd>.ts` | One file per root command; helpers in `scripts/lib/`, tests in `scripts/tests/unit/` |
| `scripts/agents/` | `agents:sync` and the hook scripts shared by every agent tool |
| `docs/superpowers/{specs,plans}/`, `docs/research/` | Design, plans and verified research |
| `.config/` | Tool configuration (when the tool can read it there) |
| `release/` | Packaged builds (`release/.archive/` for older ones) |

- Workspaces are listed explicitly (root `package.json` `workspaces.packages`, root `Cargo.toml` `members`);
  add a project there when you create it, and add its commit scope to `scripts/lib/commits/commit.constants.ts`.
- Put tool configuration in `.config/` and pass its path; only files a tool insists on stay at the root.
- Do not add top-level folders that the spec does not describe.
```

`.claude/rules/naming.md`
```markdown
---
description: File and folder naming, file size and module layout (enforced by the structure check).
---

# Naming and file layout

- Files: lowercase kebab-case `<subject>.<kind>.<ext>` — `app-row.component.tsx`, `use-theme.hook.ts`,
  `button.variants.ts`, `button.types.ts`, `repo.util.ts`, `nord.polar-night.theme.ts`.
- TypeScript under any `src/` or `scripts/lib/` must end in a known kind: client, command, component, config,
  constants, context, emitter, env, generated, hook, keys, mock, model, page, parse, plan, plugin, policy,
  provider, recipe, registry, render, rules, schema, section, service, state, store, theme, tokens, types,
  util, variants. Exempt: `index.ts`, `main.ts(x)`, `*.d.ts`.
- Rust: `snake_case.rs`. Modules are folders by capability with `name.rs` next to `name/` — never `mod.rs`.
- Folders: by domain, then role — `features/apps/{components,hooks,state,lib}/`.
- One responsibility per file; split a file that passes ~200 lines. `index.ts` barrels use named exports only.
- Tests live only in `tests/unit/` or `tests/e2e/` of their project (`scripts/tests/unit/` for scripts).
- `bun scripts/commands/structure.ts` checks all of this and runs in `bun run check` and the pre-commit hook.
```

`.claude/rules/commands.md`
```markdown
---
description: Use bun for everything and the root commands for every routine task.
---

# Commands

**bun only:** `bun install`, `bun run <cmd>`, `bun x <bin>`, `bun test`. Never npm, npx, pnpm, yarn or node.

| Command | Use it to |
|---|---|
| `bun run setup` | Prepare a fresh clone (Rust toolchain, dependencies, cargo tools, git hooks) |
| `bun run check` | Run the full quality gate before saying work is done |
| `bun run test` | Run every test (bun test + cargo nextest) |
| `bun run format` | Format with Biome and rustfmt |
| `bun run deps` | See outdated dependencies (`--strict` fails on them) |
| `bun run change <type> <scope> <summary>` | Add a changelog entry |
| `bun run agents:sync` | Regenerate Cursor and Antigravity rules after editing `.claude/rules/` |
| `bun run version <major\|minor\|patch\|x.y.z>` | Bump every manifest at once |
| `bun run clean [--all]` | Remove build output (and `node_modules` with `--all`) |
| `bun run attribution` | Check commits and branch names for agent attribution |

Targeted runs: `bun x turbo run <task> --filter=<package>`; `cargo` subcommands may be run directly.
Later milestones add `tokens`, `design-system`, `build`, `dev` and `package` — use them only once they exist.
```

`.claude/rules/security.md`
```markdown
---
description: Security rules that apply to every change (trust boundaries, paths, processes, secrets, supply chain).
---

# Security

- **The UI is untrusted.** IPC commands accept ids, never raw paths or command lines; Rust validates every
  argument and resolves ids to executables its own scanner found under `programs/`.
- **Paths:** canonicalise and require every path to stay inside the install root; reject `..`, symlinks or
  junctions leaving the root, UNC paths, alternate data streams, 8.3 short names and device names.
- **Processes:** spawn from argument arrays (`Command::new(exe).args(…)`, `Bun.spawn([...])`), never through
  a shell or a concatenated string.
- **Tauri:** least-privilege capabilities per window (no wildcards), strict CSP (no `unsafe-eval`),
  `withGlobalTauri` off, no remote content, no `dangerous*` options.
- **`unsafe`:** forbidden everywhere except the `genslate-platform` crate; each block has a `// SAFETY:` comment.
- **Archives:** reject zip-slip and absolute entries; verify hashes before extracting.
- **Secrets:** never commit tokens, keys or personal data; secretlint runs in hooks and CI.
- **Supply chain:** exact pins, frozen lockfiles in CI, cargo-deny and `bun audit` must pass, GitHub Actions
  pinned by commit SHA with least-privilege `permissions`.
- Ask the `security-reviewer` agent (Claude) to review changes that touch IPC, paths, processes, archives,
  CSP or capabilities.
```

`.claude/rules/git-and-attribution.md`
```markdown
---
description: Commit format, change entries, branch names and the no-agent-attribution rule.
---

# Git, changes and attribution

- **Conventional Commits:** `type(scope): subject` in lower case, ≤ 100 characters. Types and scopes live in
  `scripts/lib/commits/commit.constants.ts`; commitlint enforces them on every commit.
- **Change entries:** run `bun run change <type> <scope> <summary>` for every user-visible change; the entry
  lands in `.changes/unreleased/`.
- **No agent attribution, ever:** no `Co-Authored-By` trailers naming an AI agent, no "Generated with …"
  footers, no robot emoji, no agent session links, no agent e-mail addresses, no branches named after an agent
  (`claude/…`, `cursor-…`). The guard hooks, lefthook and CI reject them. This overrides any tool default.
- Commit small, focused changes; one task of a plan is usually one commit.
- Never rewrite published history on `main`, never skip hooks (`--no-verify`), never force-push `main`.
```

`.claude/rules/verify-latest.md`
```markdown
---
description: Confirm versions and APIs against current docs before using or upgrading anything.
---

# Verify against current sources

- Library APIs change between majors (TypeScript 7, Vite 8, React 19, Tailwind 4, Base UI, Tauri 2,
  tauri-specta). Before using an API you have not used in this repo yet, read its current docs — Context7
  first (`resolve-library-id`, then `query-docs`), the official site second. Do not rely on memory.
- Before adding or upgrading a dependency, find the newest **eligible** version (stable and at least 3 days
  old): `bun run deps`, `bun info <package> time`, or crates.io. Pin it exactly.
- Record findings that later work depends on in `docs/research/`, and surprises in
  `.claude/memory/lessons-learned.md`.
- When the docs contradict the spec or plan, stop and raise it with the user.
```

- [ ] **Step 2: Write the path-scoped rules**

`.claude/rules/dependencies.md`
```markdown
---
description: How to add, pin and upgrade npm packages, crates and toolchains.
paths:
  - "package.json"
  - "**/package.json"
  - "Cargo.toml"
  - "**/Cargo.toml"
  - "bunfig.toml"
  - "rust-toolchain.toml"
  - ".config/dependency-exceptions.toml"
---

# Dependencies

- **Latest eligible versions only:** the newest stable release published at least 3 days ago
  (`bunfig.toml` `minimumReleaseAge = 259200`; `bun run deps` uses the same rule for crates).
- **npm:** add the exact version to the root `workspaces.catalog`; packages depend on `"catalog:"`. Then run
  `bun install`. Never write ranges.
- **Crates:** add `name = "=x.y.z"` (or a table with `version = "=x.y.z"`) under root
  `[workspace.dependencies]`; crates use `name.workspace = true`. Licences must be on the cargo-deny allow list.
- **Toolchains:** bun in `package.json` `packageManager`; Rust in `rust-toolchain.toml`; cargo tools in
  `scripts/lib/rust/rust-tools.constants.ts` and `.github/actions/setup-env/action.yml`.
- **Older pins** need an `[[exception]]` in `.config/dependency-exceptions.toml` with a real reason;
  `bun run check` fails otherwise, and fails on exceptions that no longer apply.
- Prefer the standard library and Bun built-ins (`Bun.YAML`, `Bun.TOML`, `Bun.Glob`, `Bun.spawn`) over new
  dependencies; justify every new dependency in the commit body.
```

`.claude/rules/typescript.md`
```markdown
---
description: TypeScript 7 conventions for every .ts and .tsx file.
paths:
  - "**/*.ts"
  - "**/*.tsx"
---

# TypeScript

- Strictest compiler options (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `noPropertyAccessFromIndexSignature`, `verbatimModuleSyntax`, `noImplicitOverride`); never loosen them.
- ESM only. Named exports only — config files (`*.config.ts`, `.config/*.ts`) are the single exception.
- `import type` for types; no `any`, no non-null `!`, no `enum`, no `namespace`, no CommonJS.
- Model data with `readonly` properties, discriminated unions, `as const` and `satisfies`.
- Use modern built-ins: `toSorted`, `toReversed`, `Object.groupBy`, `structuredClone`, `Array.prototype.at`,
  `replaceAll`, `AbortSignal.timeout`.
- Read environment variables with bracket access: `process.env['NAME']`.
- Throw `Error` objects with messages that say what to do; never throw strings.
- Scripts use Bun APIs (`Bun.file`, `Bun.write`, `Bun.spawn` with argument arrays, `Bun.Glob`), not Node shims.
- Biome formats and lints (`bun run format`); `bun x tsc -p <tsconfig>` must report nothing.
```

`.claude/rules/react.md`
```markdown
---
description: React 19 conventions for UI code.
paths:
  - "**/*.tsx"
  - "programs/**/src/**"
  - "packages/design-system/**"
---

# React

- Function components only; `ref` is a normal prop (no `forwardRef`). The React Compiler is on: follow the
  Rules of React and do not add `useMemo`/`useCallback` by reflex.
- No `useEffect` for derived state or data fetching: derive during render, use `use`, Suspense and Actions
  (`useTransition`, `useActionState`). Effects are for synchronising with outside systems only.
- Every app root has an error boundary; async UI has Suspense fallbacks.
- State: Rust is the source of truth; mirror it with a small `useSyncExternalStore` store. No heavy state
  libraries without a recorded reason.
- The UI never calls the network directly and never imports `@tauri-apps/*` outside `packages/tauri-bridge`.
- Accessibility is a release gate: correct roles and names, full keyboard support, visible focus, WCAG 2.2 AA.
```

`.claude/rules/rust.md`
```markdown
---
description: Rust conventions for every crate and Tauri shell.
paths:
  - "**/*.rs"
  - "**/Cargo.toml"
---

# Rust

- Edition 2024, toolchain from `rust-toolchain.toml`. Every crate sets `[lints] workspace = true`
  (clippy pedantic; `unwrap_used`, `expect_used`, `panic`, `todo`, `unimplemented`, `dbg_macro` denied;
  `unsafe_code` forbidden — only `genslate-platform` may opt out, with `// SAFETY:` comments).
- Errors: `thiserror` enums in libraries, `anyhow` only in binaries; return `Result`, never panic.
- Logging: `tracing` with structured fields; no `println!` in libraries.
- Logic lives in `crates/`; `src-tauri` stays thin (commands delegate to crates).
- Modules are folders by capability (`catalog.rs` + `catalog/`), never `mod.rs`; keep files small.
- Inject dependencies through traits (filesystem, platform); no global mutable state.
- File writes are atomic (temp file + rename) and debounced — the drive may be unplugged at any time.
- Public items have doc comments; fallible public functions have an `# Errors` section; builders use `#[must_use]`.
- Prefer `std` (`LazyLock`, `OnceLock`, let-chains, `let … else`) over extra crates.
- Tests live in `tests/unit/` and `tests/e2e/` behind `tests/unit.rs`/`tests/e2e.rs` entry files with
  `#[path = "unit/<module>.rs"] mod <module>;`; they return `Result` instead of unwrapping and use the
  `genslate-testing` fixtures.
```

`.claude/rules/tauri-ipc.md`
```markdown
---
description: Tauri shell and IPC contract rules.
paths:
  - "programs/desktop/**/src-tauri/**"
  - "packages/tauri-bridge/**"
  - "programs/desktop/**/src/ipc/**"
---

# Tauri and IPC

- `src-tauri` is a thin shell: window, tray, hotkey, protocol and commands that delegate to `crates/`.
- Commands are async, take ids and small validated values (never paths or command lines from the UI), and
  return typed results; errors cross as `{ code, message }`.
- TypeScript bindings are generated from Rust with tauri-specta; never hand-edit them. `bun run check` fails
  on binding drift.
- The UI talks to Rust only through `packages/tauri-bridge`; its browser mock implements the same interface and
  must stay in step with every new command and event.
- Capabilities: one file per window, least privilege, no wildcard permissions. CSP stays strict.
- Push updates (catalog changes, process state, progress) as typed events; never block a command on long work.
- The custom `launcher-icon://` protocol sanitises every path it receives.
```

`.claude/rules/performance.md`
```markdown
---
description: Performance budgets and practices for apps, packages and crates.
paths:
  - "programs/**"
  - "packages/**"
  - "crates/**"
---

# Performance

| Budget | Target |
|---|---|
| Hotkey → launcher visible | < 100 ms (webview kept warm while hidden) |
| Cold start → interactive (USB 3) | < 1.5 s |
| Idle memory (launcher + WebView2) | < 150 MB |
| Initial UI JavaScript | < 250 KB gzipped |
| Rescan of 200 apps (warm) | < 300 ms |

- Lazy-load views that are not visible at start; virtualise long lists.
- Animate only `transform` and `opacity`; no layout thrash.
- USB drives are slow and wear out: scan incrementally (mtime fingerprints), bound concurrency, debounce
  watchers and writes, write atomically.
- Never block the UI thread or an IPC command on I/O; stream large results as events.
- Measure before and after any optimisation; record the numbers in the commit or PR.
```

`.claude/rules/design-system.md`
```markdown
---
description: The GENSLATE visual contract and component rules (Nord, macOS feel, VS Code density).
paths:
  - "packages/design-system/**"
  - "packages/tokens/**"
  - "programs/**/src/**"
  - "programs/webapp/**"
---

# Design system contract

The UI must match the SLATESUITE design system and launcher exactly: a modern flat UI with VS Code density,
refined like macOS, themed with official Nord (Polar Night dark, Snow Storm light).

## Principles
1. Chrome recedes, content leads: quiet titlebar, sidebar and status bar; hierarchy from type and spacing.
2. Flat at rest, depth only when floating: hairline borders at rest; popovers, menus, dialogs and toasts get a
   0.5px ring, soft layered shadow and (dark) a top inner highlight.
3. One accent, used sparingly (focus, selection, primary action, one status-bar item). Aurora colours mean status only.
4. macOS feel: 13px Inter base (JetBrains Mono for code), tabular numbers in data, radii 6/8/12, spring easing,
   themed cursors, no text selection on chrome, dimmed chrome when the window is inactive.
5. VS Code density: tree rows 22, sidebar rows 28, menu items 24, controls 28, tabs 36, status bar 24,
   titlebar 38; Codicons at 16px (14px dense).
6. Every state in both themes: rest, hover, pressed, focus-visible, selected, disabled, invalid, loading,
   inactive window — at 100/125/150% scaling, under reduced motion, `prefers-contrast: more` and forced colours.

## Rules
- Style only with token utilities (`bg-surface`, `text-fg-muted`, `h-control-md`); no hex, no Tailwind default
  palette, no `dark:` — themes switch through `html[data-theme]`. Plan 2 adds the full utility table here.
- Components live in `packages/design-system/src/components/<category>/<name>/` as
  `<name>.component.tsx`, `<name>.variants.ts` (`tv()` only), `<name>.types.ts`, `index.ts`.
- Base UI primitives with the `render` prop (never `asChild`); `data-slot` on every part; `ref` as a prop.
- The design system never imports Tauri; window chrome takes callbacks. Traffic lights are the only window
  controls on every OS; every window uses `WindowContextMenu`.
- Every component ships with its Design Kit page (`programs/webapp/design-system`) and unit tests, and is
  reviewed in both themes before its wave is done.
- Generated token files are never edited by hand: edit `packages/tokens/src` and run `bun run tokens`.
```

`.claude/rules/testing.md`
```markdown
---
description: Test-driven workflow, test locations and tools.
paths:
  - "**/tests/**"
  - "scripts/tests/**"
  - "**/*.test.ts"
  - "**/*.test.tsx"
---

# Testing

- Test first: write the failing test, watch it fail for the right reason, write the minimum code, watch it pass.
- Tests live in `tests/unit/` (mirroring `src/`) and `tests/e2e/`; never beside source.
- TypeScript: `bun test` with Testing Library and happy-dom for components (`userEvent.setup()`); Playwright
  for end-to-end and screenshot QA in both themes.
- Rust: `cargo nextest`; one entry file per suite (`tests/unit.rs`) with `#[path]` modules; tests return
  `Result`; build fake install trees with `genslate-testing`.
- Test behaviour through public APIs: roles, names, keyboard, states, callbacks, return values and errors —
  not implementation details or pixels in unit tests.
- Cover the awkward inputs: Windows paths (backslashes, drive-letter case, spaces), CRLF text, offline
  networks, read-only drives, files changing while scanning.
- Things that cannot be automated (tray, hotkey, real eject, WebView2 fallback) go on the manual QA checklist
  in `programs/desktop/launcher/other/launcher/documents/testing.md`.
```

`.claude/rules/docs.md`
```markdown
---
description: Where documentation lives and how to write it.
paths:
  - "**/*.md"
  - "docs/**"
---

# Documentation

- Design decisions: `docs/superpowers/specs/`; implementation plans: `docs/superpowers/plans/`; verified
  research: `docs/research/`. Launcher user and developer docs ship in
  `programs/desktop/launcher/other/launcher/documents/`.
- Update docs in the same change as the behaviour they describe.
- One topic per file with a topic name. Lead with what the reader needs; prefer short sentences and tables.
- Commands in docs always use bun (`bun run …`), never npm.
- Mark generated documents as generated and point to their source.
```

- [ ] **Step 3: Write `AGENTS.md`** (read by Cursor and Antigravity; imported by `CLAUDE.md`)

```markdown
# GENSLATE — agent guide

GENSLATE is a portable "operating system on a stick" for Windows: a Tauri tray launcher for GENSLATE apps
(first) and PortableApps.com and portapps.io apps (second), plus the shared Nord design system every GENSLATE
app uses. Everything runs from one install folder on a USB stick or SSD.

- **Spec:** `docs/superpowers/specs/2026-10-07-genslate-launcher-design.md`
- **Plans:** `docs/superpowers/plans/` · **Research:** `docs/research/`
- **Look-and-behaviour reference:** `S:\DEVELOPMENT\PROJECTS\GENSLATE-SLATESUITE` (never copy files from it)

## Stack

Turborepo + bun workspaces · Rust (one Cargo workspace, edition 2024) · Tauri 2 · Vite · React 19 ·
TypeScript 7 · Tailwind CSS 4 · Base UI · Nord themes (Polar Night dark, Snow Storm light).

## Commands (bun only — never npm, npx, pnpm, yarn or node)

`bun run setup` · `bun run check` · `bun run test` · `bun run format` · `bun run deps` ·
`bun run change <type> <scope> <summary>` · `bun run agents:sync` · `bun run version <kind>` · `bun run clean`

## Golden rules

1. bun for everything.
2. Latest eligible versions (stable, ≥ 3 days old), pinned exactly; older pins need a recorded exception.
3. Never edit generated files: `bun.lock`, `Cargo.lock`, `**/generated/**`, `*.generated.ts`,
   `src-tauri/gen/**`, `.cursor/rules/**`, `.agents/rules/**`.
4. Names `<subject>.<kind>.<ext>` in kebab-case; Rust `snake_case.rs`; tests only in `tests/unit` or `tests/e2e`.
5. Check library APIs against current docs before using them.
6. Conventional Commits with project scopes. **No AI-agent attribution, ever.**
7. Done means `bun run check` and `bun run test` pass and docs are updated.

## Rules

The detailed rules are written once in `.claude/rules/` and generated into `.cursor/rules/` and
`.agents/rules/` by `bun run agents:sync`: project, monorepo-structure, naming, commands, dependencies,
typescript, react, rust, tauri-ipc, security, performance, design-system, testing, git-and-attribution, docs,
verify-latest.
```

- [ ] **Step 4: Write `CLAUDE.md`**

```markdown
@AGENTS.md

# Claude Code

Claude is the main development agent for this repo (Cursor handles editing and small tasks; Antigravity
handles light tasks).

- **Workflow:** brainstorming → spec → writing-plans → subagent-driven development, test first. Follow the
  current plan in `docs/superpowers/plans/` task by task.
- **Rules:** `.claude/rules/` load automatically (path-scoped ones when you touch matching files). After
  editing a rule, run `bun run agents:sync`.
- **Agents:** `code-reviewer`, `security-reviewer`, `ui-visual-qa`, `design-system-engineer`,
  `tauri-rust-engineer`, `docs-writer`, `release-manager`.
- **Skills:** `design-system-component`, `design-tokens`, `tauri-ipc-command`, `rust-crate-module`, `verify-latest`.
- **Commands:** `/check`, `/new-component`, `/new-ipc-command`, `/tokens`, `/deps`, `/audit`, `/release`, `/change`.
- **Hooks** (`scripts/agents/hooks/`): generated-file guard, attribution guard, format on edit, session context.
- **Memory:** record non-obvious decisions in `.claude/memory/decisions.md`, gotchas in
  `.claude/memory/lessons-learned.md`, and keep `.claude/memory/active-context.md` current — the session-start
  hook shows it.
```

- [ ] **Step 5: Generate, check and spell-check**

Run:
```bash
bun run agents:sync
bun scripts/agents/sync.ts --check
bun x cspell lint --config .config/cspell.json --no-progress --dot ".claude/**" "AGENTS.md" "CLAUDE.md"
```
Expected: `16 rules → 32 written, 0 removed`; `16 rules in sync`; cspell clean after adding any legitimate terms to `project-words.txt`. Open one generated file of each kind (`.cursor/rules/rust.mdc`, `.agents/rules/rust.md`) and confirm the front matter matches Task 17.

- [ ] **Step 6: Commit**

```bash
git add .claude/rules AGENTS.md CLAUDE.md .cursor/rules .agents/rules .config/cspell/project-words.txt
git commit -m "docs(agents): add shared rules synced to cursor and antigravity"
```

### Task 22: Claude-only tooling (agents, skills, commands, memory, MCP)

**Files:**
- Create: `.claude/agents/{code-reviewer,security-reviewer,ui-visual-qa,design-system-engineer,tauri-rust-engineer,docs-writer,release-manager}.md`
- Create: `.claude/skills/{design-system-component,design-tokens,tauri-ipc-command,rust-crate-module,verify-latest}/SKILL.md`
- Create: `.claude/commands/{check,new-component,new-ipc-command,tokens,deps,audit,release}.md`; overwrite `.claude/commands/change.md`
- Create: `.claude/memory/decisions.md`, `.claude/memory/active-context.md`; overwrite `.claude/memory/lessons-learned.md`, `.claude/README.md`, `.mcp.json`

**Interfaces:**
- Consumes: the rules (Task 21) and root commands (Tasks 8–18).
- Produces: Claude's per-tool layer (not synced to other tools).

- [ ] **Step 1: Write the agents**

`.claude/agents/code-reviewer.md`
```markdown
---
name: code-reviewer
description: Reviews a diff or branch for correctness, rule compliance and maintainability before a task or milestone is marked done.
tools: Read, Grep, Glob, Bash
---

You review changes in the GENSLATE monorepo. Read `AGENTS.md` and the rules in `.claude/rules/` that match the
changed files, and the plan task being reviewed.

Check, in order:
1. Correctness — does the code do what the task and spec say? Look for unhandled cases (Windows paths, CRLF,
   empty input, offline, read-only drive).
2. Tests — written first, testing behaviour through public APIs, located in `tests/unit|e2e`.
3. Rules — naming, strict TypeScript, Rust lints, no generated-file edits, bun only, no agent attribution.
4. Simplicity — no dead code, no speculative abstractions, files under ~200 lines.

Run `bun run check` and `bun run test` yourself. Report findings ranked by severity, each with the file and
line, what is wrong and a concrete fix. Say clearly when there is nothing to fix.
```

`.claude/agents/security-reviewer.md`
```markdown
---
name: security-reviewer
description: Reviews changes touching IPC, paths, processes, archives, CSP, capabilities, dependencies or CI for security problems.
tools: Read, Grep, Glob, Bash
---

You are the security reviewer for GENSLATE, a portable launcher that runs executables from removable drives.
Read `.claude/rules/security.md` and spec §10.6 first.

Check every change for:
- IPC trust boundary: ids only from the UI; every argument validated in Rust.
- Path confinement to the install root (traversal, symlinks/junctions, UNC, ADS, 8.3 names, device names).
- Process spawning without shells; argument arrays only.
- Archive extraction (zip-slip, absolute entries) and hash verification.
- Tauri capabilities (least privilege, no wildcards), CSP, protocol handlers.
- `unsafe` outside `genslate-platform`, or without `// SAFETY:` comments.
- Supply chain: new dependencies, pins, licences, GitHub Actions pinned by SHA, workflow permissions.
- Secrets or personal data in code, logs or fixtures.

Report each finding with severity (critical/high/medium/low), location, exploit scenario and fix. If nothing
is found, say which areas you checked.
```

`.claude/agents/ui-visual-qa.md`
```markdown
---
name: ui-visual-qa
description: Compares GENSLATE UI screenshots in both Nord themes against the SLATESUITE reference and the design-system contract.
tools: Read, Grep, Glob, Bash
---

You check that GENSLATE UI matches the SLATESUITE design system and launcher exactly. Read
`.claude/rules/design-system.md` and the reference screenshots in
`S:\DEVELOPMENT\PROJECTS\GENSLATE-SLATESUITE\other\resources\screenshots\`.

For each screen or component under review:
1. Take or open screenshots in Polar Night and Snow Storm (Playwright screenshots from the Design Kit or the
   launcher's browser mock).
2. Compare spacing, sizes (rows, controls, titlebar, status bar), radii, colours, typography, icon sizes,
   focus rings and every state in the state matrix.
3. Check reduced motion, `prefers-contrast: more` and 125/150% scaling where relevant.

Report differences as a list: what differs, where, the expected value from the reference or contract, and
the likely fix (token or variant to change).
```

`.claude/agents/design-system-engineer.md`
```markdown
---
name: design-system-engineer
description: Builds tokens and design-system components with their Design Kit pages and tests, matching SLATESUITE exactly.
tools: Read, Grep, Glob, Bash, Edit, Write
---

You build `packages/tokens`, `packages/design-system` and `programs/webapp/design-system`. Follow
`.claude/rules/design-system.md`, `react.md`, `typescript.md` and `testing.md`, and the
`design-system-component` and `design-tokens` skills.

Use SLATESUITE as the visual reference only — read its components to match look and behaviour, then write
fresh code here. Verify Base UI and Tailwind APIs against current docs. Every component gets its variants,
types, index, unit tests and a Design Kit page with a state matrix, and is checked in both themes.
```

`.claude/agents/tauri-rust-engineer.md`
```markdown
---
name: tauri-rust-engineer
description: Builds Rust crates, the Tauri shell, Start.exe and the IPC contract.
tools: Read, Grep, Glob, Bash, Edit, Write
---

You build `crates/*`, `programs/desktop/*/src-tauri` and `programs/desktop/start`. Follow
`.claude/rules/rust.md`, `tauri-ipc.md`, `security.md` and `performance.md`, and the `rust-crate-module`
and `tauri-ipc-command` skills.

Logic goes in crates behind traits; the shell stays thin. Test first with `genslate-testing` fixtures. Verify
Tauri 2 and tauri-specta APIs against current docs before using them. Run `bun run check` and `bun run test`
before reporting done.
```

`.claude/agents/docs-writer.md`
```markdown
---
name: docs-writer
description: Writes and updates GENSLATE documentation so it matches current behaviour.
tools: Read, Grep, Glob, Edit, Write
---

You keep GENSLATE documentation accurate. Follow `.claude/rules/docs.md`. Read the code and the spec section
for the behaviour you document; never describe features that do not exist yet. Write for the named reader
(user, contributor or agent), lead with what they need, use short sentences and tables, and use bun in every
command.
```

`.claude/agents/release-manager.md`
```markdown
---
name: release-manager
description: Prepares a release — version bump, changelog from .changes, packaging and release checks.
tools: Read, Grep, Glob, Bash, Edit, Write
---

You prepare GENSLATE releases. Before anything else, run `bun run check`, `bun run test` and
`bun run deps --strict`; stop and report if any fail. Then bump versions with `bun run version <kind>`,
collect `.changes/unreleased/` entries into the changelog, and (from milestone M10) run `bun run package`.
Never add agent attribution to commits, tags or release notes.
```

- [ ] **Step 2: Write the skills**

`.claude/skills/design-system-component/SKILL.md`
```markdown
---
name: design-system-component
description: Use when adding or changing a component in packages/design-system — creates the component, variants, types, barrel, tests and its Design Kit page.
---

# Design-system component

1. Read `.claude/rules/design-system.md` and the matching SLATESUITE component
   (`S:\DEVELOPMENT\PROJECTS\GENSLATE-SLATESUITE\packages\design-system\src\components\<category>\<name>\`)
   for look and behaviour. Do not copy it.
2. Check the Base UI part names and data attributes for the primitive in the current Base UI docs (Context7).
3. Write the failing test in `packages/design-system/tests/unit/<category>/<name>.test.tsx`: role and name,
   keyboard interaction, states (`data-*`, `aria-*`) and callbacks.
4. Create `src/components/<category>/<name>/`:
   - `<name>.types.ts` — public props (`readonly`, documented)
   - `<name>.variants.ts` — `tv()` recipes using token utilities only
   - `<name>.component.tsx` — Base UI `render` prop, `data-slot` on every part, `ref` as a prop
   - `index.ts` — named exports; add `export * from './<name>';` to the category barrel
5. Make the test pass; run `bun run check`.
6. Add the Design Kit page `programs/webapp/design-system/src/pages/<category>/<name>.page.tsx` with a live
   preview, TSX snippet and state matrix (rest, hover, pressed, focus, disabled, loading as relevant).
7. Ask `ui-visual-qa` to compare both themes with SLATESUITE before marking the component done.
```

`.claude/skills/design-tokens/SKILL.md`
```markdown
---
name: design-tokens
description: Use when adding or changing design tokens or the Nord themes in packages/tokens.
---

# Design tokens

1. Edit only the sources in `packages/tokens/src/` (`tokens/*.tokens.ts`, `themes/nord.*.theme.ts`).
   Never edit `src/generated/**` or `crates/design-tokens/src/generated/**`.
2. Keep Nord official: Polar Night nord0–3 for dark surfaces, Snow Storm nord4–6 for light surfaces and text,
   Frost nord7–10 for accents, Aurora nord11–15 for status only.
3. Run `bun run tokens`, then the tokens tests. The contrast validator must pass (WCAG 2.2 AA) in both themes.
4. If you added a utility, document it in the utility table of `.claude/rules/design-system.md` and run
   `bun run agents:sync`.
5. Check affected components in both themes in the Design Kit.
```

`.claude/skills/tauri-ipc-command/SKILL.md`
```markdown
---
name: tauri-ipc-command
description: Use when adding or changing a Tauri command or event — Rust core, shell command, generated bindings, bridge client, browser mock and tests.
---

# Tauri IPC command

1. Read `.claude/rules/tauri-ipc.md` and `security.md`. Decide the arguments: ids and small validated values
   only — never paths or command lines from the UI.
2. Test first in the core crate (`crates/launcher-core/tests/unit/…`) using `genslate-testing` fixtures;
   implement the behaviour in the crate.
3. Add the async command in `src-tauri` that validates arguments and delegates to the crate; errors map to
   `{ code, message }`.
4. Register it with tauri-specta and regenerate the TypeScript bindings (never edit them by hand).
5. Add the method to the `packages/tauri-bridge` client and the same behaviour to its browser mock; test both.
6. Add the permission to the right capability file (least privilege).
7. Run `bun run check` (binding drift, clippy, types) and `bun run test`.
```

`.claude/skills/rust-crate-module/SKILL.md`
```markdown
---
name: rust-crate-module
description: Use when creating a Rust crate or adding a module to one.
---

# Rust crate or module

**New crate**
1. `crates/<name>/Cargo.toml` with `name = "genslate-<name>"`, workspace `version/edition/rust-version/authors/publish`,
   dependencies as `x.workspace = true`, and `[lints] workspace = true`.
2. Add it to root `Cargo.toml` `members` (and `[workspace.dependencies]` if other crates use it) and its scope
   to `scripts/lib/commits/commit.constants.ts`.
3. `src/lib.rs` with a `//!` crate doc and `pub use` of the public API only.

**New module**
1. `src/<module>.rs` (and `src/<module>/` for sub-modules) — never `mod.rs`.
2. Test first: `tests/unit/<module>.rs`, declared in `tests/unit.rs` with
   `#[path = "unit/<module>.rs"] mod <module>;`. Tests return `Result` and use `genslate-testing`.
3. Errors with `thiserror`; `tracing` for logs; `# Errors` docs on fallible public functions.
4. Run `cargo nextest run -p genslate-<name>`, then `bun run check`.
```

`.claude/skills/verify-latest/SKILL.md`
```markdown
---
name: verify-latest
description: Use before adding or upgrading a dependency or using a library API for the first time in this repo.
---

# Verify latest versions and APIs

1. **Version:** run `bun run deps`, or for a new package `bun info <name> time` / crates.io, and take the newest
   stable version published at least 3 days ago.
2. **Docs:** query Context7 (`resolve-library-id`, then `query-docs` with the exact API) or read the official
   docs for that version. Note breaking changes since your training data.
3. **Pin:** add the exact version (catalog or `[workspace.dependencies]`), run `bun install` or cargo.
4. **Record:** add findings later work depends on to `docs/research/`, and surprises to
   `.claude/memory/lessons-learned.md`.
5. If the current docs contradict the spec or plan, stop and ask the user.
```

- [ ] **Step 3: Write the commands**

`.claude/commands/check.md`
```markdown
---
description: Run the full quality gate and fix what fails.
---

Run `bun run check`. For each failing step, read the output, fix the cause (never weaken a rule or add a
suppression to pass), and re-run until every step passes. Then run `bun run test`. Report what you fixed.
```

`.claude/commands/new-component.md`
```markdown
---
description: Create a design-system component with its tests and Design Kit page.
argument-hint: <category>/<name> — e.g. actions/button
---

Create the design-system component `$ARGUMENTS` using the `design-system-component` skill. Match the
SLATESUITE component's look and behaviour without copying it, test first, add its Design Kit page, and finish
with `bun run check` and `bun run test`.
```

`.claude/commands/new-ipc-command.md`
```markdown
---
description: Add a Tauri IPC command end to end (core, shell, bindings, bridge, mock, tests).
argument-hint: <command_name> — what it does
---

Add the IPC command described here: $ARGUMENTS. Follow the `tauri-ipc-command` skill step by step, test first,
and ask the `security-reviewer` agent to review the change before reporting done.
```

`.claude/commands/tokens.md`
```markdown
---
description: Change design tokens and regenerate every output.
argument-hint: what to change
---

Make this token change: $ARGUMENTS. Follow the `design-tokens` skill: edit only `packages/tokens/src`, run
`bun run tokens`, keep the contrast validator green, and check affected components in both themes.
```

`.claude/commands/deps.md`
```markdown
---
description: Report outdated dependencies and propose upgrades.
---

Run `bun run deps`. For each outdated pin, use the `verify-latest` skill to read the release notes and
breaking changes, then propose the upgrade (or an exception with a reason) as a list. Do not change pins
until the user agrees.
```

`.claude/commands/audit.md`
```markdown
---
description: Security and quality audit of the current branch.
---

Run `bun run check` and `bun run test`, then ask the `security-reviewer` agent to review the changes on this
branch against `main`, and the `code-reviewer` agent to review correctness and rule compliance. Summarise the
findings ranked by severity.
```

`.claude/commands/release.md`
```markdown
---
description: Prepare a release with the release-manager agent.
argument-hint: <major|minor|patch|x.y.z>
---

Ask the `release-manager` agent to prepare release `$ARGUMENTS`: green gates first, then version bump,
changelog from `.changes/unreleased/`, and packaging (from milestone M10). No agent attribution anywhere.
```

`.claude/commands/change.md`
```markdown
---
description: Add a changelog entry for the current work.
argument-hint: <type> <scope> <summary>
---

Run `bun run change $ARGUMENTS`. If no arguments were given, look at the staged changes, pick the type and
scope from `scripts/lib/commits/commit.constants.ts`, write a one-line user-facing summary, and run the command.
```

- [ ] **Step 4: Write memory, README and MCP**

`.claude/memory/decisions.md`
```markdown
# Decisions

Non-obvious choices and why. Newest first. The master spec's decision table (§2) is the baseline.

- 2026-10-07 — "Latest" means newest stable release at least 3 days old (bunfig `minimumReleaseAge`), so
  knip is pinned at 6.39.0 and lefthook at 2.1.16 instead of the newest builds.
- 2026-10-07 — TypeScript 7.0 pinned directly; it has no programmatic API, but no current tool needs it
  (knip uses oxc). Add `@typescript/typescript6` only when a tool proves it needs the API.
- 2026-10-07 — Static CRT linking (`+crt-static`) so executables run on PCs without the VC++ redistributable.
```

`.claude/memory/lessons-learned.md`
```markdown
# Lessons learned

Gotchas worth remembering. Newest first.

- Biome does not auto-discover `.config/biome.json`; always pass `--config-path=.config/biome.json`
  (editors: `biome.configurationPath`).
- Cargo reads only `.cargo/config.toml`, and rustfmt only project/parent folders — those configs stay at the root.
- On Windows, always spawn with argument arrays; paths with spaces (`Recycle Bin`) break string commands.
- Integration-test crate roots resolve `mod x;` next to the root file, so use `#[path = "unit/x.rs"]`.
```

`.claude/memory/active-context.md`
```markdown
# Active context

- **Phase:** Plan 1 — Foundation (M0–M2), `docs/superpowers/plans/2026-10-07-plan-1-foundation.md`.
- **Next:** Plan 2 — tokens, design system and Design Kit (M3–M4), written after Plan 1 completes.
```

`.claude/README.md`
```markdown
# .claude

Claude Code configuration for GENSLATE. Claude is the main development agent.

| Path | Contents |
|---|---|
| `rules/` | **Shared rules — the single source.** Synced to `.cursor/rules/` and `.agents/rules/` by `bun run agents:sync`. |
| `agents/` | Sub-agents: code-reviewer, security-reviewer, ui-visual-qa, design-system-engineer, tauri-rust-engineer, docs-writer, release-manager |
| `skills/` | Step-by-step skills for recurring work |
| `commands/` | Slash commands |
| `memory/` | Decisions, lessons learned and the active context |
| `settings.json` | Permissions, hooks (scripts in `scripts/agents/hooks/`) and attribution settings |

Only `rules/` is shared with the other tools; everything else here is Claude-only.
```

`.mcp.json`
```json
{
  "mcpServers": {
    "context7": {
      "type": "http",
      "url": "https://mcp.context7.com/mcp"
    }
  }
}
```

- [ ] **Step 5: Check and commit**

Run: `bun run check` — Expected: all steps pass (add any legitimate new words to `project-words.txt`).

```bash
git add .claude .mcp.json .config/cspell/project-words.txt
git commit -m "docs(agents): add claude agents, skills, commands and memory"
```

### Task 23: Cursor and Antigravity tooling

**Files:**
- Create: `.cursor/commands/{check,new-component,tokens}.md`; overwrite `.cursor/commands/change.md`, `.cursor/mcp.json`, `.cursor/README.md`, `.cursorignore`
- Create: `.agents/workflows/{check,run-tests,fix-ui,format,tokens,new-change}.md`, `.agents/skills/quality-gate/SKILL.md`, `.agents/skills/ui-fixes/SKILL.md`; overwrite `.agents/README.md`
- Delete: `.agents/skills/.gitkeep`, `.agents/workflows/.gitkeep`

**Interfaces:**
- Consumes: the generated rules (Task 21), hooks (Task 20), root commands.
- Produces: each tool's own layer, sized to its role (Cursor: editing and basic tasks; Antigravity: light tasks).

- [ ] **Step 1: Cursor files**

`.cursor/commands/check.md`
```markdown
Run `bun run check`. Fix each failing step at its cause (never weaken a rule), re-run until it passes, then run `bun run test`.
```

`.cursor/commands/new-component.md`
```markdown
Create the design-system component named in the request. Follow `.cursor/rules/design-system.mdc`: files
`<name>.component.tsx`, `<name>.variants.ts`, `<name>.types.ts`, `index.ts` in
`packages/design-system/src/components/<category>/<name>/`, a unit test in
`packages/design-system/tests/unit/<category>/<name>.test.tsx`, and a Design Kit page. Finish with `bun run check`.
```

`.cursor/commands/tokens.md`
```markdown
Edit design tokens only in `packages/tokens/src/`, then run `bun run tokens` and `bun run test`. Never edit generated files.
```

`.cursor/commands/change.md`
```markdown
Add a changelog entry: `bun run change <type> <scope> <summary>` (types and scopes: `scripts/lib/commits/commit.constants.ts`).
```

`.cursor/mcp.json`
```json
{
  "mcpServers": {
    "context7": {
      "url": "https://mcp.context7.com/mcp"
    }
  }
}
```

`.cursor/README.md`
```markdown
# .cursor

Cursor configuration for GENSLATE. Cursor is used for editing and basic tasks.

| Path | Contents |
|---|---|
| `rules/*.mdc` | **Generated** from `.claude/rules/` by `bun run agents:sync` — do not edit here |
| `commands/` | Cursor commands: check, new-component, tokens, change |
| `hooks.json` | Generated-file guard, attribution guard and format-on-edit (scripts in `scripts/agents/hooks/`) |
| `mcp.json` | Context7 docs server |

Cursor also reads `AGENTS.md` at the repo root and the editor settings in `.vscode/`.
```

`.cursorignore`
```gitignore
node_modules/
target/
dist/
.turbo/
release/
bun.lock
Cargo.lock
**/src-tauri/gen/
```

- [ ] **Step 2: Antigravity files**

`.agents/workflows/check.md`
```markdown
---
description: Run the full quality gate and fix what fails.
---

1. Run `bun run check`.
2. For each failing step, fix the cause (never weaken a rule) and re-run.
3. Run `bun run test` and report the result.
```

`.agents/workflows/run-tests.md`
```markdown
---
description: Run all tests and summarise failures.
---

1. Run `bun run test`.
2. For each failure, show the test name, the assertion and the most likely cause. Do not change code unless asked.
```

`.agents/workflows/fix-ui.md`
```markdown
---
description: Fix a small UI issue in a GENSLATE screen or component.
---

1. Read `.agents/rules/design-system.md`.
2. Find the component or screen and the token utilities it uses; fix the issue with token utilities only
   (no hex, no `dark:`).
3. Check the result in both themes (Polar Night and Snow Storm).
4. Run `bun run check`.
```

`.agents/workflows/format.md`
```markdown
---
description: Format the repository.
---

1. Run `bun run format`.
2. Report which files changed.
```

`.agents/workflows/tokens.md`
```markdown
---
description: Regenerate design tokens after editing packages/tokens/src.
---

1. Confirm only files under `packages/tokens/src/` were edited (never `generated/`).
2. Run `bun run tokens`, then `bun run test`.
```

`.agents/workflows/new-change.md`
```markdown
---
description: Add a changelog entry for the current change.
---

1. Pick the type and scope from `scripts/lib/commits/commit.constants.ts`.
2. Run `bun run change <type> <scope> <one-line summary>`.
```

`.agents/skills/quality-gate/SKILL.md`
```markdown
---
name: quality-gate
description: Use before saying any task is done — runs the GENSLATE quality gate and tests.
---

# Quality gate

1. `bun run check` — every step must pass.
2. `bun run test` — every test must pass.
3. If anything fails, fix the cause; never disable a rule, skip a hook or edit generated files.
4. Report the commands you ran and their results.
```

`.agents/skills/ui-fixes/SKILL.md`
```markdown
---
name: ui-fixes
description: Use for small visual fixes — spacing, colour, alignment or state styling in GENSLATE UI.
---

# UI fixes

1. Read `.agents/rules/design-system.md`; the target look is the SLATESUITE design system.
2. Change variants (`*.variants.ts`) or token utilities, never raw colours or `dark:` classes.
3. Keep every state working: hover, pressed, focus-visible, disabled, selected.
4. Check both themes, then run `bun run check`.
```

`.agents/README.md`
```markdown
# .agents

Google Antigravity configuration for GENSLATE. Antigravity is used for light tasks.

| Path | Contents |
|---|---|
| `rules/*.md` | **Generated** from `.claude/rules/` by `bun run agents:sync` — do not edit here |
| `workflows/` | Slash workflows: /check, /run-tests, /fix-ui, /format, /tokens, /new-change |
| `skills/` | Light skills: quality-gate, ui-fixes |
| `hooks.json` | Generated-file guard, attribution guard and format-on-edit (scripts in `scripts/agents/hooks/`) |

Antigravity also reads `AGENTS.md` at the repo root.
```

Then: `git rm -q --ignore-unmatch .agents/skills/.gitkeep .agents/workflows/.gitkeep`

- [ ] **Step 3: Verify in the tools (manual, once)**

1. Cursor: *Settings → Rules* lists the 16 generated rules; typing `/` in chat lists the four commands.
2. Antigravity: the *Customizations* panel lists the rules and the six workflows; `/check` runs the workflow.

- [ ] **Step 4: Check and commit**

```bash
bun run check
git add .cursor .cursorignore .agents .config/cspell/project-words.txt
git commit -m "docs(agents): add cursor commands and antigravity workflows and skills"
```

### Task 24: Docs and milestone exit

**Files:**
- Overwrite: `README.md`, `scripts/README.md`
- Modify: `.claude/memory/active-context.md`

- [ ] **Step 1: Write `README.md`**

````markdown
# GENSLATE

A portable "operating system on a stick" for Windows: carry your apps and documents on a USB stick or SSD
and use them on any Windows 10/11 PC. The GENSLATE launcher puts GENSLATE apps first and runs
PortableApps.com and portapps.io apps too. Every GENSLATE app shares one Nord design system.

> Status: foundation in progress. See `docs/superpowers/specs/2026-10-07-genslate-launcher-design.md`.

## Develop

Requirements: Windows 10/11 x64, Git, bun 1.4.2, rustup and the MSVC Build Tools.

```bash
bun run setup   # Rust toolchain, dependencies, cargo tools, git hooks
bun run check   # full quality gate
bun run test    # all tests
```

Use bun for everything — never npm, npx, pnpm or yarn. See `.github/CONTRIBUTING.md` for the workflow and
`AGENTS.md` for the rules every contributor and AI agent follows.

## Repository

| Path | Contents |
|---|---|
| `programs/` | Apps: the Tauri launcher, `Start.exe`, and the Design Kit web app |
| `crates/` | Shared Rust crates |
| `packages/` | Shared TypeScript packages (tokens, design system, Tauri bridge, configs) |
| `scripts/` | The bun scripts behind every `bun run` command |
| `docs/` | Spec, plans and research |
| `release/` | Packaged builds |
````

- [ ] **Step 2: Write `scripts/README.md`**

```markdown
# scripts

The bun scripts behind the root `bun run` commands.

| Path | Contents |
|---|---|
| `commands/<cmd>.ts` | One file per root command (`check`, `test`, `format`, `setup`, `deps`, `version`, `clean`, `attribution`, `change`) plus `structure.ts` |
| `agents/sync.ts` | `bun run agents:sync` — generates Cursor and Antigravity rules from `.claude/rules/` |
| `agents/hooks/*.hook.ts` | Guard and format hooks shared by Claude Code, Cursor and Antigravity (`--tool <name>`) |
| `lib/<area>/` | Pure helpers the commands are built from (`<subject>.<kind>.ts`) |
| `tests/unit/<area>/` | Tests for `lib/` (`bun test` from the repo root) |

Rules: Bun APIs only, processes spawned from argument arrays (never a shell), logic in `lib/` with tests,
commands kept thin. `build.ts`, `dev.ts` and `package.ts` are filled in by later milestones.
```

- [ ] **Step 3: Run the exit gate**

Run:
```bash
bun run check
bun run test
bun run deps --strict
bun run attribution
```
Expected: all pass.

- [ ] **Step 4: Security and code review**

In Claude Code, run `/audit` (it asks `security-reviewer` and `code-reviewer` to review the branch against
the commit before Task 5). Fix every critical/high finding, re-run Step 3, and list medium/low findings for
the user.

- [ ] **Step 5: Update the active context and commit**

Replace `.claude/memory/active-context.md` with:
```markdown
# Active context

- **Done:** Plan 1 — Foundation (M0–M2). Monorepo, quality gate, CI, Rust fixtures crate and agent tooling
  are in place.
- **Next:** write Plan 2 — tokens, design system and Design Kit (M3–M4) — from the master spec and
  `docs/research/frontend-stack.md`.
```

```bash
git add README.md scripts/README.md .claude/memory/active-context.md
git commit -m "docs(repo): document the foundation and close milestone m2"
```

---

## Self-review notes (for the executor)

- Every task leaves `bun run check` passing from Task 15 onward; if an earlier task's command is not wired
  yet, run its own verification steps instead.
- Plan 2 (M3–M4) and Plan 3 (M5–M10) are written after this plan completes, using `docs/research/` from M0.
