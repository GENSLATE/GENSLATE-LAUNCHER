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
