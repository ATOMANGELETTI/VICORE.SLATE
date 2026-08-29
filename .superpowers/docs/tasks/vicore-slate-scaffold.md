# VICORE Slate Scaffold — Status

**Current phase:** Phase 1 — Task 1 complete (path redesign); ready for Task 2+
**Doing now:** Nothing in-flight
**Model (doing now):** —
**Next:** Task 2 — `.cursor/rules/001-project-overview.mdc`
**Blocked on:** User go-ahead to continue Phase 1 execution

## Checklist

### Phase 1 — Governance, Root Foundation & Tooling (`2026-08-29-vicore-slate-scaffold.phase.01.md`)

- [x] Task 1: `.superpowers/` docs layout + Superpowers path rule — model: Grok 4.6
- [ ] Task 2: `.cursor/rules/001-project-overview.mdc` — model: —
- [ ] Task 3: `.cursor/rules/002-monorepo-and-naming.mdc` — model: —
- [ ] Task 4: `.cursor/rules/003-rust-tauri-guidelines.mdc` — model: —
- [ ] Task 5: `.cursor/rules/004-typescript-react-guidelines.mdc` — model: —
- [ ] Task 6: `.cursor/rules/005-ui-kit-boundaries.mdc` — model: —
- [ ] Task 7: `.cursor/rules/006-app-shell-window-chrome.mdc` — model: —
- [ ] Task 8: `.cursor/rules/007-nord-theming.mdc` — model: —
- [ ] Task 9: `.cursor/rules/008-testing-strategy.mdc` — model: —
- [ ] Task 10: `.cursor/rules/009-portability-and-deferred-scope.mdc` — model: —
- [ ] Task 11: `.cursor/rules/010-git-commit-workflow.mdc` — model: —
- [ ] Task 12: `.cursor/rules/011-release-packaging-layout.mdc` — model: —
- [ ] Task 12b: `.cursor/rules/012-agent-model-selection.mdc` + AGENTS.md preference — model: —
- [ ] Task 13: `.cursor/commands/{new-app,new-package,new-crate}.md` — model: —
- [ ] Task 14: `AGENTS.md` canonical brief + `.cursorrules` pointer (no `CLAUDE.md`) — model: —
- [ ] Task 15: `scripts/sync-crate-versions.ts` (TDD — first real code in the repo) — model: —
- [ ] Task 16: Root `package.json` + `bunfig.toml` (delete stray `.bunfig.toml`) — model: —
- [ ] Task 17: `.tool-versions` (mise) + `rust-toolchain.toml` — model: —
- [ ] Task 18: Cargo workspace (`Cargo.toml`) — model: —
- [ ] Task 19: moon v2 config (`.moon/workspace.yml`, `.moon/toolchains.yml`, `.moon/tasks/common.yml`; delete stray root `moon.yml`) — model: —
- [ ] Task 20: Biome (`biome.jsonc`) — model: —
- [ ] Task 21: Rust lint/format/deny (`rustfmt.toml`, `clippy.toml`, `deny.toml`) — model: —
- [ ] Task 22: Mutation testing + coverage (`cargo-mutants.toml`, `tarpaulin.toml`) — model: —
- [ ] Task 23: Root TypeScript project references + Vitest workspace — model: —
- [ ] Task 24: Husky + Commitlint — model: —
- [ ] Task 25: Changesets (`.changeset/config.json`, `.changeset/README.md`) — model: —
- [ ] Task 26: CI workflow (`.github/workflows/ci.yml`) — model: —
- [ ] Task 27: Phase 1 verification pass — model: —

### Phase 2 — Shared Packages (`2026-08-29-vicore-slate-scaffold.phase.02.md`)

- [ ] Task 1: `packages/config-typescript` — model: —
- [ ] Task 2: `packages/config-vite` — model: —
- [ ] Task 3: `crates/slate-core` stub — model: —
- [ ] Task 4: `packages/ui-kit` bootstrap — model: —
- [ ] Task 5: Nord token files (`src/tokens/`) — model: —
- [ ] Task 6: `src/styles/main.css` — Tailwind v4 theme + light/dark wiring — model: —
- [ ] Task 7: `src/lib/utils.ts` — `cn()` helper — model: —
- [ ] Task 8: `src/primitives/` — shadcn CLI atoms — model: —
- [ ] Task 9: `src/providers/theme-provider/` — model: —
- [ ] Task 10: `src/composites/titlebar/` + `window-controls` — model: —
- [ ] Task 11: `src/composites/toolbar/` — model: —
- [ ] Task 12: `src/composites/app-shell/` — model: —
- [ ] Task 13: `src/composites/context-menu/` — model: —
- [ ] Task 14: Phase 2 verification pass — model: —

### Phase 3 — Template Apps x9 (`2026-08-29-vicore-slate-scaffold.phase.03.md`)

- [ ] Task 1: `scripts/templates/app/` — the template tree — model: —
- [ ] Task 2: `scripts/new-app.ts` generator (TDD) — model: —
- [ ] Task 3: Generate the 8 non-launcher apps — model: —
- [ ] Task 4: Generate `slate-launcher`, then hand-patch its tray extras (TDD for the Rust tray module) — model: —
- [ ] Task 5: Phase 3 verification pass — model: —

### Phase 4 — Release Packaging, Remaining Scripts & Full Verification (`2026-08-29-vicore-slate-scaffold.phase.04.md`)

- [ ] Task 1: Extract `scripts/lib/generate-from-template.ts` + refactor `new-app.ts` onto it — model: —
- [ ] Task 2: `scripts/templates/package/` + `scripts/new-package.ts` — model: —
- [ ] Task 3: `scripts/templates/crate/` + `scripts/new-crate.ts` — model: —
- [ ] Task 4: `scripts/dev.ts` — local dev orchestration — model: —
- [ ] Task 5: `scripts/templates/appdata/configs/settings.toml` + `scripts/changelog.ts` — model: —
- [ ] Task 6: `scripts/package.ts` — assemble `installDir/` — model: —
- [ ] Task 7: Wire the root `moon run :package` task — model: —
- [ ] Task 8: Full-repository verification pass — model: —
