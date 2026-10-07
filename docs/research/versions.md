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
