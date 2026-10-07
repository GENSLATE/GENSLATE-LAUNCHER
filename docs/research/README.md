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
