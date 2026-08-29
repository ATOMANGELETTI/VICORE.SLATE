# VICORE Slate Portable Suite Scaffold — Design Spec

**Feature slug:** `vicore-slate-scaffold`
**Status:** Plan written (phases 01–04), not yet executed.

## Vision

Slate is a fully portable, thumbdrive/folder/drive-installable suite of AI-native desktop apps — a "portable AI operating system." Everything the suite needs (dependencies, configs, data, logs) lives inside its own install directory. No OS user-profile paths, no registry, no global installs.

## Layers

- `apps/` — nine independent Tauri v2 applications: `slate-launcher`, `slate-terminal`, `slate-explorer`, `slate-browser`, `slate-editor`, `slate-gallery`, `slate-jukebox`, `slate-player`, `slate-aistudio`.
- `packages/` — shared TypeScript: `@slate/ui-kit` (design system), `@slate/config-typescript`, `@slate/config-vite` (build presets). Only things 2+ apps need identically.
- `crates/` — shared Rust. `slate-core` is a stub for now; `slate-runtime` (portable path resolution), AI/LLM crates, and a plugin-host are explicitly deferred.

## Confirmed architecture decisions

- Frontend: React 19 + TypeScript + Vite 8 + Vitest + Playwright + shadcn/ui + Radix UI primitives (explicit choice over Base UI).
- All 9 apps get the same basic template treatment (custom titlebar, placeholder content, bottom toolbar) in this pass.
- Portable-path runtime crate (`slate-runtime`): stub only.
- AI-native plumbing: fully deferred — `slate-aistudio` is just another template app for now.
- Modular folder + file naming everywhere: apps nest modules under `src/modules/<name>/` and `src-tauri/src/modules/<name>/`; `packages/ui-kit` keeps a `primitives/` (atoms) + `composites/` (shell-level) split, each with one-folder-per-module inside. Dot-segmented names for multi-axis variant files (`theme.nord.polar-night.css`, `tauri.windows.conf.json`).
- Tauri v2 Isolation Pattern applied to all 9 apps as a security-hardening default (pass-through stub bridge for now).
- Granular per-app `capabilities/`: `default.json` + `desktop.json` + `remote-tags.json` for every app, plus `tray-menu.json` for `slate-launcher` only (the only app with a system tray icon/menu).
- No `CLAUDE.md` — `AGENTS.md` (canonical) + `.cursorrules` (thin pointer) only.
- Changesets wired for internal version/changelog tracking across apps, packages, and Rust crates (never published — private/restricted throughout); Rust crates get a thin tracking `package.json` synced into `Cargo.toml` by a script.
- `.tool-versions` (mise) is the single source of truth for local dev-machine Bun/Rust/Node versions; `rust-toolchain.toml` and `.moon/toolchains.yml` mirror it, checked for drift in CI.
- Nord theme (official four families: polar-night, snow-storm, frost, aurora) as separate dot-segmented CSS token files, aggregated, feeding a Tailwind v4 CSS-first `@theme` block with `data-theme` runtime switching. Modern flat, macOS-inspired UI.
- Release packaging (`installDir/`): `launcher/` (top-level), `programs/slate-*/` (exact dev-time name, no renaming), `appdata/{configs,database,logs,docs,resources}`, empty `storage/` — assembled by `scripts/package.ts`.
- Bun over npm everywhere; all automation scripts are Bun-executed TypeScript (`scripts/*.ts`), never `.ps1`.

## Plan structure

Split into 4 sequential phase-plans under this same folder, each independently verifiable:

1. `phase.01.md` — Governance (`.cursor/rules`, `.superpowers/`) + Root Foundation + Tooling.
2. `phase.02.md` — Shared Packages (`config-typescript`, `config-vite`, `ui-kit`, `slate-core` stub).
3. `phase.03.md` — Template Apps x9 (generator-first: `scripts/new-app.ts` + template, then generate, then patch `slate-launcher`).
4. `phase.04.md` — Release Packaging + remaining `scripts/` automation + full-graph verification.

See `progress.md` in this folder for the live status of execution, and `.superpowers/tasks/vicore-slate-scaffold.md` for the current at-a-glance status.
