# Phase 3: Template Apps x9 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Prerequisite:** Phase 2 (`.superpowers/plans/vicore-slate-scaffold/phase.02.md`) must be fully verified before starting this plan — every app built here depends on `@slate/ui-kit`, `@slate/config-typescript`, `@slate/config-vite`, and `slate-core`.

**Goal:** Build a single `scripts/new-app.ts` generator + `scripts/templates/app/` template (with its own tests), use it to scaffold all 9 apps identically, then hand-patch `slate-launcher`'s tray/isolation-consuming extras — ending with all 9 apps buildable, typecheck-clean, and passing their own unit + e2e smoke tests.

**Architecture:** Generator-first, not copy-paste: `scripts/templates/app/` holds the literal file tree (with `{{name}}`/`{{title}}` placeholders) every app is generated from; `scripts/new-app.ts` reads that tree and does placeholder substitution + directory copy. This keeps the 9 apps identical by construction — if the template changes later, regenerating is safer than hand-editing 9 near-duplicate trees. `slate-launcher` is generated like every other app, then gets 3 additional hand-written pieces (`tray-menu.json`, `src/modules/tray-menu/`, `src-tauri/src/modules/tray/`) that no other app has.

**Tech Stack:** Tauri v2.11.x, React 19, Vite 8 (isolation build + main build), Vitest, Playwright, `@tauri-apps/api`.

## Global Constraints

- Every app: `src/modules/<name>/` (TS) and `src-tauri/src/modules/<name>/` (Rust) nesting — never flat `src/<name>/`.
- Every app: Tauri v2 Isolation Pattern wired (`isolation/`, `vite.isolation.config.ts`, `tauri.conf.json` security.pattern) — pass-through stub logic only.
- Every app: `capabilities/{default,desktop,remote-tags}.json`; `slate-launcher` additionally gets `tray-menu.json`.
- Every app: `decorations: false`, custom `TitleBar`+`WindowControls`+`Toolbar`+`AppShell` from `@slate/ui-kit`, version fetched live via `@tauri-apps/api/app` `getVersion()`, never hardcoded.
- All tests under `tests/unit/modules/<name>/<name>.test.tsx` (Vitest) and `tests/e2e/<name>.spec.ts` (Playwright) — never beside the component.
- No app gets a system tray, custom Rust modules beyond `tray/`, or extra capabilities unless it's `slate-launcher` — the other 8 stay byte-for-byte structurally identical (only name/title differ).

---

### Task 1: `scripts/templates/app/` — the template tree

**Files:**
- Create: `scripts/templates/app/isolation/index.html`
- Create: `scripts/templates/app/isolation/index.ts`
- Create: `scripts/templates/app/src/main.tsx`
- Create: `scripts/templates/app/src/styles/main.css`
- Create: `scripts/templates/app/src/modules/app/app.tsx`
- Create: `scripts/templates/app/src/modules/app/providers.tsx`
- Create: `scripts/templates/app/src/modules/app/index.ts`
- Create: `scripts/templates/app/src/modules/template-view/template-view.tsx`
- Create: `scripts/templates/app/src/modules/template-view/index.ts`
- Create: `scripts/templates/app/src/modules/app-toolbar/app-toolbar.tsx`
- Create: `scripts/templates/app/src/modules/app-toolbar/index.ts`
- Create: `scripts/templates/app/src-tauri/src/main.rs`
- Create: `scripts/templates/app/src-tauri/src/lib.rs`
- Create: `scripts/templates/app/src-tauri/capabilities/default.json`
- Create: `scripts/templates/app/src-tauri/capabilities/desktop.json`
- Create: `scripts/templates/app/src-tauri/capabilities/remote-tags.json`
- Create: `scripts/templates/app/src-tauri/Cargo.toml`
- Create: `scripts/templates/app/src-tauri/tauri.conf.json`
- Create: `scripts/templates/app/src-tauri/tauri.windows.conf.json`
- Create: `scripts/templates/app/src-tauri/tauri.linux.conf.json`
- Create: `scripts/templates/app/src-tauri/build.rs`
- Create: `scripts/templates/app/index.html`
- Create: `scripts/templates/app/package.json`
- Create: `scripts/templates/app/tsconfig.json`
- Create: `scripts/templates/app/vite.config.ts`
- Create: `scripts/templates/app/vite.isolation.config.ts`
- Create: `scripts/templates/app/vitest.config.ts`
- Create: `scripts/templates/app/playwright.config.ts`
- Create: `scripts/templates/app/tests/unit/modules/template-view/template-view.test.tsx`
- Create: `scripts/templates/app/tests/e2e/app.spec.ts`

**Interfaces:**
- Consumes: `@slate/ui-kit` (Phase 2), `@slate/config-typescript`, `@slate/config-vite` (Phase 2)
- Produces: the literal file tree Task 2's `scripts/new-app.ts` copies + substitutes `{{name}}` (kebab-case, e.g. `terminal`) and `{{title}}` (Title Case, e.g. `Terminal`) into

- [ ] **Step 1: Write `scripts/templates/app/isolation/index.html`**

```html
<!doctype html>
<html>
  <head><meta charset="utf-8" /></head>
  <body><script type="module" src="./index.ts"></script></body>
</html>
```

- [ ] **Step 2: Write `scripts/templates/app/isolation/index.ts` (pass-through Isolation Pattern bridge)**

```typescript
// Tauri v2 Isolation Pattern secure bridge. Pass-through stub for now — see
// .cursor/rules/009-portability-and-deferred-scope.mdc. Real validation
// rules get added here once 2+ apps need identical logic (at which point
// this extracts into a shared packages/isolation-bridge).
window.__TAURI_ISOLATION_HOOK__ = (payload: unknown) => payload;
```

- [ ] **Step 3: Write `scripts/templates/app/src/main.tsx`**

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { App } from "./modules/app";
import "./styles/main.css";

const rootElement = document.getElementById("root");
if (!rootElement) throw new Error("Root element not found");

createRoot(rootElement).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

- [ ] **Step 4: Write `scripts/templates/app/src/styles/main.css`**

```css
@import "@slate/ui-kit/src/styles/main.css";

html, body, #root {
  height: 100%;
  margin: 0;
}
```

- [ ] **Step 5: Write `scripts/templates/app/src/modules/app/providers.tsx`**

```tsx
import type { ReactNode } from "react";
import { ThemeProvider } from "@slate/ui-kit";

export function AppProviders({ children }: { children: ReactNode }) {
  return <ThemeProvider>{children}</ThemeProvider>;
}
```

- [ ] **Step 6: Write `scripts/templates/app/src/modules/app/app.tsx`**

```tsx
import { AppShell, TitleBar, Toolbar } from "@slate/ui-kit";
import { AppProviders } from "./providers";
import { TemplateView } from "../template-view";
import { AppToolbar } from "../app-toolbar";

export function App() {
  return (
    <AppProviders>
      <AppShell
        titleBar={<TitleBar title="{{title}}" />}
        toolbar={
          <Toolbar>
            <AppToolbar />
          </Toolbar>
        }
      >
        <TemplateView />
      </AppShell>
    </AppProviders>
  );
}
```

- [ ] **Step 7: Write `scripts/templates/app/src/modules/app/index.ts`**

```typescript
export * from "./app";
```

- [ ] **Step 8: Write `scripts/templates/app/src/modules/template-view/template-view.tsx`**

```tsx
import { useEffect, useState } from "react";
import { getVersion } from "@tauri-apps/api/app";

export function TemplateView() {
  const [version, setVersion] = useState<string>("");

  useEffect(() => {
    getVersion().then(setVersion);
  }, []);

  return (
    <div className="flex h-full flex-col items-center justify-center gap-2">
      <h1 className="text-xl font-semibold">{{title}} Template Application</h1>
      <p className="text-sm text-[var(--nord3)]">App Version {version}</p>
    </div>
  );
}
```

- [ ] **Step 9: Write `scripts/templates/app/src/modules/template-view/index.ts`**

```typescript
export * from "./template-view";
```

- [ ] **Step 10: Write `scripts/templates/app/src/modules/app-toolbar/app-toolbar.tsx`**

```tsx
export function AppToolbar() {
  return <span className="text-xs text-[var(--nord3)]">{{title}} — Ready</span>;
}
```

- [ ] **Step 11: Write `scripts/templates/app/src/modules/app-toolbar/index.ts`**

```typescript
export * from "./app-toolbar";
```

- [ ] **Step 12: Write `scripts/templates/app/src-tauri/src/main.rs`**

```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    slate_{{name}}_lib::run();
}
```

- [ ] **Step 13: Write `scripts/templates/app/src-tauri/src/lib.rs`**

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .run(tauri::generate_context!())
        .expect("error while running slate-{{name}}");
}
```

- [ ] **Step 14: Write `scripts/templates/app/src-tauri/capabilities/default.json`**

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Core + window capabilities — least privilege baseline",
  "windows": ["main"],
  "permissions": ["core:default", "core:window:default"]
}
```

- [ ] **Step 15: Write `scripts/templates/app/src-tauri/capabilities/desktop.json`**

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "desktop",
  "description": "Desktop OS integration",
  "windows": ["main"],
  "permissions": ["core:app:default", "core:path:default"]
}
```

- [ ] **Step 16: Write `scripts/templates/app/src-tauri/capabilities/remote-tags.json`**

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "remote-tags",
  "description": "Remote-domain webview allowlist — empty by default",
  "windows": ["main"],
  "permissions": [],
  "remote": { "urls": [] }
}
```

- [ ] **Step 17: Write `scripts/templates/app/src-tauri/Cargo.toml`**

```toml
[package]
name = "slate-{{name}}"
version = "0.1.0"
edition.workspace = true
rust-version.workspace = true

[lib]
name = "slate_{{name}}_lib"
crate-type = ["staticlib", "cdylib", "rlib"]

[build-dependencies]
tauri-build = "2"

[dependencies]
tauri = { workspace = true }
serde = { workspace = true }
thiserror = { workspace = true }
slate-core = { path = "../../../crates/slate-core" }
```

- [ ] **Step 18: Write `scripts/templates/app/src-tauri/build.rs`**

```rust
fn main() {
    tauri_build::build();
}
```

- [ ] **Step 19: Write `scripts/templates/app/src-tauri/tauri.conf.json`**

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "slate-{{name}}",
  "version": "0.1.0",
  "identifier": "com.vicore.slate.{{name}}",
  "app": {
    "windows": [{ "title": "{{title}}", "width": 960, "height": 640, "decorations": false }],
    "security": {
      "pattern": { "use": "isolation", "options": { "dir": "../dist-isolation" } }
    }
  },
  "build": { "beforeBuildCommand": "bun run build", "frontendDist": "../dist" },
  "bundle": { "active": true, "targets": [] }
}
```

`"targets": []` produces a raw portable output (no MSI/DMG installer) — see
`.cursor/rules/006-app-shell-window-chrome.mdc`.

- [ ] **Step 20: Write `scripts/templates/app/src-tauri/tauri.windows.conf.json`**

```json
{
  "bundle": { "windows": { "webviewInstallMode": { "type": "skip" } } }
}
```

- [ ] **Step 21: Write `scripts/templates/app/src-tauri/tauri.linux.conf.json`**

```json
{
  "bundle": { "linux": { "appimage": { "bundleMediaFramework": false } } }
}
```

- [ ] **Step 22: Write `scripts/templates/app/index.html`**

```html
<!doctype html>
<html>
  <head><meta charset="utf-8" /><title>{{title}}</title></head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 23: Write `scripts/templates/app/package.json`**

```json
{
  "name": "slate-{{name}}",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build && vite build --config vite.isolation.config.ts",
    "test": "vitest run",
    "test:e2e": "playwright test",
    "tauri": "tauri"
  },
  "dependencies": {
    "react": "latest",
    "react-dom": "latest",
    "@tauri-apps/api": "latest",
    "@slate/ui-kit": "workspace:*"
  },
  "devDependencies": {
    "@slate/config-typescript": "workspace:*",
    "@slate/config-vite": "workspace:*",
    "@tauri-apps/cli": "latest",
    "vitest": "latest",
    "@testing-library/react": "latest",
    "@testing-library/jest-dom": "latest",
    "jsdom": "latest",
    "@playwright/test": "latest"
  }
}
```

- [ ] **Step 24: Write `scripts/templates/app/tsconfig.json`**

```json
{
  "extends": "@slate/config-typescript/tsconfig.web.json",
  "compilerOptions": { "outDir": "./dist", "rootDir": "./src" },
  "include": ["src"]
}
```

- [ ] **Step 25: Write `scripts/templates/app/vite.config.ts`**

```typescript
import { createSlateViteConfig } from "@slate/config-vite";
import { defineConfig } from "vite";

export default defineConfig(
  createSlateViteConfig({ extraConfig: { build: { outDir: "dist" } } }),
);
```

- [ ] **Step 26: Write `scripts/templates/app/vite.isolation.config.ts`**

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  root: "isolation",
  build: { outDir: "../dist-isolation", emptyOutDir: true },
});
```

- [ ] **Step 27: Write `scripts/templates/app/vitest.config.ts`**

```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: { environment: "jsdom", include: ["tests/unit/**/*.test.{ts,tsx}"] },
});
```

- [ ] **Step 28: Write `scripts/templates/app/playwright.config.ts`**

```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  webServer: { command: "bun run dev", port: 1420, reuseExistingServer: true },
});
```

- [ ] **Step 29: Write `scripts/templates/app/tests/unit/modules/template-view/template-view.test.tsx`**

```tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import { TemplateView } from "../../../../src/modules/template-view/template-view";

vi.mock("@tauri-apps/api/app", () => ({ getVersion: () => Promise.resolve("0.1.0") }));

describe("TemplateView", () => {
  it("shows the template heading and live version", async () => {
    render(<TemplateView />);
    expect(screen.getByText(/Template Application/)).toBeInTheDocument();
    await waitFor(() => expect(screen.getByText(/App Version 0.1.0/)).toBeInTheDocument());
  });
});
```

- [ ] **Step 30: Write `scripts/templates/app/tests/e2e/app.spec.ts`**

```typescript
import { expect, test } from "@playwright/test";

test("app window loads and shows the template title", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByText(/Template Application/)).toBeVisible();
});
```

- [ ] **Step 31: Commit the template**

```bash
git add scripts/templates/app/
git commit -m "feat: add the shared app template tree"
```

---

### Task 2: `scripts/new-app.ts` generator (TDD)

**Files:**
- Create: `tests/unit/new-app.test.ts`
- Create: `scripts/new-app.ts`

**Interfaces:**
- Consumes: `scripts/templates/app/` (Task 1)
- Produces: `generateApp(name: string, targetDir: string): void` — Task 3 runs this 9 times; `.cursor/commands/new-app.md` (Phase 1 Task 13) documents it as the only way to create an app

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, expect, it, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync, readFileSync, existsSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { generateApp } from "../../scripts/new-app";

let targetRoot: string;

beforeEach(() => {
  targetRoot = mkdtempSync(join(tmpdir(), "slate-app-"));
});

afterEach(() => {
  rmSync(targetRoot, { recursive: true, force: true });
});

describe("generateApp", () => {
  it("creates the app directory with the expected files", () => {
    const appDir = join(targetRoot, "slate-terminal");
    generateApp("terminal", appDir);

    expect(existsSync(join(appDir, "src", "modules", "app", "app.tsx"))).toBe(true);
    expect(existsSync(join(appDir, "src-tauri", "capabilities", "default.json"))).toBe(true);
    expect(existsSync(join(appDir, "isolation", "index.ts"))).toBe(true);
  });

  it("substitutes {{name}} and {{title}} in file contents", () => {
    const appDir = join(targetRoot, "slate-terminal");
    generateApp("terminal", appDir);

    const cargoToml = readFileSync(join(appDir, "src-tauri", "Cargo.toml"), "utf-8");
    expect(cargoToml).toContain('name = "slate-terminal"');

    const appTsx = readFileSync(join(appDir, "src", "modules", "app", "app.tsx"), "utf-8");
    expect(appTsx).toContain('title="Terminal"');
  });

  it("title-cases hyphenated names correctly", () => {
    const appDir = join(targetRoot, "slate-aistudio");
    generateApp("aistudio", appDir);

    const appTsx = readFileSync(join(appDir, "src", "modules", "app", "app.tsx"), "utf-8");
    expect(appTsx).toContain('title="Aistudio"');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun test tests/unit/new-app.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write the minimal implementation**

```typescript
import { cpSync, readdirSync, readFileSync, statSync, writeFileSync } from "node:fs";
import { join } from "node:path";

const TEMPLATE_DIR = join(import.meta.dir, "templates", "app");

function toTitleCase(name: string): string {
  return name.charAt(0).toUpperCase() + name.slice(1);
}

function substitute(content: string, name: string, title: string): string {
  return content.replaceAll("{{name}}", name).replaceAll("{{title}}", title);
}

function walkAndSubstitute(dir: string, name: string, title: string): void {
  for (const entry of readdirSync(dir)) {
    const fullPath = join(dir, entry);
    if (statSync(fullPath).isDirectory()) {
      walkAndSubstitute(fullPath, name, title);
    } else {
      const content = readFileSync(fullPath, "utf-8");
      writeFileSync(fullPath, substitute(content, name, title));
    }
  }
}

export function generateApp(name: string, targetDir: string): void {
  const title = toTitleCase(name);
  cpSync(TEMPLATE_DIR, targetDir, { recursive: true });
  walkAndSubstitute(targetDir, name, title);
}

if (import.meta.main) {
  const nameArgIndex = process.argv.indexOf("--name");
  const name = nameArgIndex !== -1 ? process.argv[nameArgIndex + 1] : undefined;
  if (!name) {
    console.error("Usage: bun scripts/new-app.ts --name <name>");
    process.exit(1);
  }
  generateApp(name, join(import.meta.dir, "..", "apps", `slate-${name}`));
  console.log(`Generated apps/slate-${name}`);
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `bun test tests/unit/new-app.test.ts`
Expected: PASS — all three tests green.

- [ ] **Step 5: Commit**

```bash
git add scripts/new-app.ts tests/unit/new-app.test.ts
git commit -m "feat: add new-app generator script"
```

---

### Task 3: Generate the 8 non-launcher apps

**Files:**
- Create: `apps/slate-terminal/**`, `apps/slate-explorer/**`, `apps/slate-browser/**`, `apps/slate-editor/**`, `apps/slate-gallery/**`, `apps/slate-jukebox/**`, `apps/slate-player/**`, `apps/slate-aistudio/**` (all generated, none hand-written)
- Modify: `tsconfig.json` (root — add 8 references)
- Modify: `Cargo.toml` (no change needed — `apps/*/src-tauri` glob already matches)

**Interfaces:**
- Consumes: `scripts/new-app.ts` (Task 2)
- Produces: 8 fully scaffolded apps, each independently buildable/testable

- [ ] **Step 1: Run the generator for each app**

```bash
for name in terminal explorer browser editor gallery jukebox player aistudio; do
  bun scripts/new-app.ts --name $name
done
```

- [ ] **Step 2: Add each new app to root `tsconfig.json`'s references**

```json
{
  "files": [],
  "references": [
    { "path": "./packages/config-vite" },
    { "path": "./packages/ui-kit" },
    { "path": "./apps/slate-terminal" },
    { "path": "./apps/slate-explorer" },
    { "path": "./apps/slate-browser" },
    { "path": "./apps/slate-editor" },
    { "path": "./apps/slate-gallery" },
    { "path": "./apps/slate-jukebox" },
    { "path": "./apps/slate-player" },
    { "path": "./apps/slate-aistudio" }
  ]
}
```

- [ ] **Step 3: Install and typecheck**

Run: `bun install && bunx tsc -b`
Expected: exit 0 — all 8 apps compile against `@slate/ui-kit`.

- [ ] **Step 4: Run each app's unit tests**

Run: `bunx vitest run`
Expected: exit 0 — `TemplateView` test passes identically in all 8 apps (48+ new assertions across 8 files).

- [ ] **Step 5: Build one app's Rust side as a smoke check**

Run: `cargo build -p slate-terminal`
Expected: exit 0.

Run: `cargo clippy -p slate-terminal -- -D warnings`
Expected: exit 0.

- [ ] **Step 6: Commit**

```bash
git add apps/slate-terminal apps/slate-explorer apps/slate-browser apps/slate-editor apps/slate-gallery apps/slate-jukebox apps/slate-player apps/slate-aistudio tsconfig.json bun.lockb Cargo.lock
git commit -m "feat: generate 8 template apps from the shared generator"
```

---

### Task 4: Generate `slate-launcher`, then hand-patch its tray extras (TDD for the Rust tray module)

**Files:**
- Create: `apps/slate-launcher/**` (generated base, same as Task 3)
- Create: `apps/slate-launcher/src-tauri/capabilities/tray-menu.json`
- Create: `apps/slate-launcher/src-tauri/src/modules/tray/mod.rs`
- Create: `apps/slate-launcher/src-tauri/tests/tray.rs`
- Create: `apps/slate-launcher/src/modules/tray-menu/tray-menu.ts`
- Create: `apps/slate-launcher/src/modules/tray-menu/index.ts`
- Modify: `apps/slate-launcher/src-tauri/src/lib.rs`
- Modify: `tsconfig.json` (root — add reference)

**Interfaces:**
- Consumes: `scripts/new-app.ts` (Task 2)
- Produces: `slate_launcher_lib::modules::tray::build_tray(app: &tauri::App) -> tauri::Result<()>` (Rust), `listenForTrayMenuActions(): void` (TS) — nothing outside `slate-launcher` depends on these; they are the one intentional structural difference from the other 8 apps

- [ ] **Step 1: Generate the base app identically to the others**

```bash
bun scripts/new-app.ts --name launcher
```

- [ ] **Step 2: Write `apps/slate-launcher/src-tauri/capabilities/tray-menu.json`**

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "tray-menu",
  "description": "System tray icon and menu — slate-launcher only",
  "windows": ["main"],
  "permissions": ["core:tray:default", "core:menu:default"]
}
```

- [ ] **Step 3: Write the failing Rust integration test for the tray module**

```rust
// apps/slate-launcher/src-tauri/tests/tray.rs
use slate_launcher_lib::modules::tray::tray_menu_ids;

#[test]
fn tray_menu_has_open_and_quit_items() {
    let ids = tray_menu_ids();
    assert!(ids.contains(&"open"));
    assert!(ids.contains(&"quit"));
}
```

- [ ] **Step 4: Run the test to verify it fails**

Run: `cargo test -p slate-launcher --test tray`
Expected: FAIL — `slate_launcher_lib::modules` doesn't exist yet (the generated `lib.rs` has no `modules` declaration).

- [ ] **Step 5: Write `apps/slate-launcher/src-tauri/src/modules/tray/mod.rs`**

```rust
use tauri::menu::{Menu, MenuItem};
use tauri::tray::TrayIconBuilder;
use tauri::{AppHandle, Manager, Runtime};

/// The stable set of menu item ids the tray exposes — kept as a plain
/// function (not a struct) so both the builder below and the test above
/// have one source of truth for the id strings.
pub fn tray_menu_ids() -> [&'static str; 2] {
    ["open", "quit"]
}

pub fn build_tray<R: Runtime>(app: &AppHandle<R>) -> tauri::Result<()> {
    let open_item = MenuItem::with_id(app, "open", "Open Launcher", true, None::<&str>)?;
    let quit_item = MenuItem::with_id(app, "quit", "Quit", true, None::<&str>)?;
    let menu = Menu::with_items(app, &[&open_item, &quit_item])?;

    TrayIconBuilder::new()
        .menu(&menu)
        .on_menu_event(move |app, event| {
            if event.id() == "open" {
                if let Some(window) = app.get_webview_window("main") {
                    let _ = window.show();
                    let _ = window.set_focus();
                }
            } else if event.id() == "quit" {
                app.exit(0);
            }
        })
        .build(app)?;

    Ok(())
}
```

- [ ] **Step 6: Wire the module into `lib.rs`**

```rust
pub mod modules;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .setup(|app| {
            modules::tray::build_tray(app.handle())?;
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running slate-launcher");
}
```

And create `apps/slate-launcher/src-tauri/src/modules/mod.rs`:

```rust
pub mod tray;
```

- [ ] **Step 7: Run the test to verify it passes**

Run: `cargo test -p slate-launcher --test tray`
Expected: PASS.

Run: `cargo clippy -p slate-launcher -- -D warnings`
Expected: exit 0.

- [ ] **Step 8: Write the failing TS test for the tray-menu IPC listener**

```tsx
// apps/slate-launcher/tests/unit/modules/tray-menu/tray-menu.test.ts
import { describe, expect, it, vi } from "vitest";
import { listenForTrayMenuActions } from "../../../../src/modules/tray-menu/tray-menu";

const listen = vi.fn().mockResolvedValue(() => {});

vi.mock("@tauri-apps/api/event", () => ({ listen: (...args: unknown[]) => listen(...args) }));

describe("listenForTrayMenuActions", () => {
  it("subscribes to the tray-menu-action event", async () => {
    await listenForTrayMenuActions();
    expect(listen).toHaveBeenCalledWith("tray-menu-action", expect.any(Function));
  });
});
```

- [ ] **Step 9: Run the test to verify it fails**

Run: `bunx vitest run apps/slate-launcher/tests/unit/modules/tray-menu/tray-menu.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 10: Write `apps/slate-launcher/src/modules/tray-menu/tray-menu.ts`**

```typescript
import { listen } from "@tauri-apps/api/event";

export async function listenForTrayMenuActions(): Promise<void> {
  await listen<string>("tray-menu-action", (event) => {
    console.log("tray menu action:", event.payload);
  });
}
```

- [ ] **Step 11: Write `apps/slate-launcher/src/modules/tray-menu/index.ts`**

```typescript
export * from "./tray-menu";
```

- [ ] **Step 12: Run the test to verify it passes**

Run: `bunx vitest run apps/slate-launcher/tests/unit/modules/tray-menu/tray-menu.test.ts`
Expected: PASS.

- [ ] **Step 13: Add `slate-launcher` to root `tsconfig.json` references**

```json
{ "path": "./apps/slate-launcher" }
```

- [ ] **Step 14: Full typecheck + install**

Run: `bun install && bunx tsc -b`
Expected: exit 0.

- [ ] **Step 15: Commit**

```bash
git add apps/slate-launcher tsconfig.json bun.lockb Cargo.lock
git commit -m "feat: generate slate-launcher and add its tray-specific modules"
```

---

### Task 5: Phase 3 verification pass

**Files:** none created — verification only

**Interfaces:**
- Consumes: every file created in Tasks 1–4
- Produces: confirmation all 9 apps are structurally sound before Phase 4 packages them

- [ ] **Step 1: Full install**

Run: `bun install`
Expected: exit 0.

- [ ] **Step 2: Lint**

Run: `bunx biome check .`
Expected: exit 0.

- [ ] **Step 3: TypeScript across the whole graph (all 9 apps + 3 packages)**

Run: `bunx tsc -b`
Expected: exit 0.

- [ ] **Step 4: Full test suite**

Run: `bunx vitest run`
Expected: exit 0 — 9 `TemplateView`/`App` unit test files plus `slate-launcher`'s `tray-menu` test, all green.

- [ ] **Step 5: Rust across the whole workspace**

Run: `cargo build --workspace && cargo clippy --workspace -- -D warnings && cargo test --workspace`
Expected: all exit 0 — includes `slate-core`, all 9 `slate-*` app crates, and `slate-launcher`'s tray integration test.

- [ ] **Step 6: moon across the whole graph**

Run: `moon run :lint :typecheck :test :build`
Expected: exit 0 across all 9 apps + 3 packages.

- [ ] **Step 7: One real `tauri dev` smoke check**

Run: `cd apps/slate-terminal && bun run tauri dev` (start, confirm the window opens showing "Terminal Template Application" and a version number, then stop with Ctrl+C)
Expected: window renders correctly; `cd ../..` back to repo root afterward.

- [ ] **Step 8: One Playwright e2e run**

Run: `cd apps/slate-terminal && bunx playwright install --with-deps chromium && bunx playwright test && cd ../..`
Expected: `app.spec.ts` passes.

- [ ] **Step 9: Record a changeset**

Run: `bunx changeset` — select all 9 apps as `minor` bumps, summary "Add template shape to all 9 apps".

- [ ] **Step 10: Final commit**

```bash
git add .changeset/ -A
git commit -m "chore: phase 3 verification and changeset"
```

Phase 3 is complete. Proceed to the Phase 4 plan (Release Packaging + remaining
`scripts/` automation + full-graph verification) once this verification pass
is fully green.
