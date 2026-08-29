# Phase 2: Shared Packages Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Prerequisite:** Phase 1 (`.superpowers/plans/vicore-slate-scaffold/phase.01.md`) must be fully verified (its Task 27) before starting this plan — this plan's tasks append to files Phase 1 created (root `tsconfig.json`, `Cargo.toml`'s glob-matched members).

**Goal:** Build `packages/config-typescript`, `packages/config-vite`, the full `packages/ui-kit` design system (Nord tokens, Tailwind v4 theme, primitives via shadcn/Radix, hand-written composites, ThemeProvider), and the `crates/slate-core` stub — every piece independently unit-tested, so Phase 3's 9 apps have a real, working design system and Rust stub to build on.

**Architecture:** `config-typescript` and `config-vite` are pure build-tooling presets consumed by everything else, built first. `crates/slate-core` is built early too since it's independent of the TS work and gives the Rust workspace its first real member. `ui-kit` is built last since it depends on both TS presets; within `ui-kit`, tokens -> theme -> utils -> primitives -> composites -> barrel, since each layer imports the one before it.

**Tech Stack:** TypeScript 5.9 project references, Vite 8 (library mode), Tailwind CSS v4 (CSS-first `@theme`), shadcn CLI + Radix UI, `class-variance-authority`, `clsx` + `tailwind-merge`, Vitest + `@testing-library/react` + jsdom, Playwright, React 19, Rust 1.90 (`thiserror`, `serde`).

## Global Constraints

- One module = one folder with a local `index.ts` barrel (per `.cursor/rules/002-monorepo-and-naming.mdc`).
- `packages/ui-kit` keeps the `primitives/` (atoms) vs `composites/` (shell-level) split — every new module in this plan goes in exactly one of those two buckets, never loose at `src/` top level.
- No `any`; Biome (`noExplicitAny: error`) enforced, no ESLint/Prettier.
- All tests under `tests/unit/**` (Vitest) or `tests/visual/**` (Playwright) mirroring the `src/` module path — never beside the component file.
- ui-kit only holds what 2+ apps need identically (`.cursor/rules/005-ui-kit-boundaries.mdc`) — every module built in this plan already qualifies (titlebar, toolbar, app-shell, context-menu, and all shadcn primitives are needed by all 9 apps).
- No published/publishable output — `crates/slate-core`'s tracking `package.json` stays `"private": true`, never `bun publish`/`cargo publish`.

---

### Task 1: `packages/config-typescript`

**Files:**
- Create: `packages/config-typescript/package.json`
- Create: `packages/config-typescript/tsconfig.base.json`
- Create: `packages/config-typescript/tsconfig.node.json`
- Create: `packages/config-typescript/tsconfig.web.json`

**Interfaces:**
- Consumes: nothing
- Produces: `tsconfig.web.json` (extended by `packages/ui-kit/tsconfig.json` in Task 4, and by every Phase 3 app) and `tsconfig.node.json` (extended by `packages/config-vite/tsconfig.json` in Task 2, and by root `scripts/*.ts`)

- [ ] **Step 1: Write `packages/config-typescript/package.json`**

```json
{
  "name": "@slate/config-typescript",
  "version": "0.1.0",
  "private": true,
  "files": ["tsconfig.base.json", "tsconfig.node.json", "tsconfig.web.json"]
}
```

- [ ] **Step 2: Write `packages/config-typescript/tsconfig.base.json`**

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2023",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "verbatimModuleSyntax": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "composite": true,
    "declaration": true
  }
}
```

- [ ] **Step 3: Write `packages/config-typescript/tsconfig.node.json`**

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2023"],
    "types": ["bun-types"]
  }
}
```

- [ ] **Step 4: Write `packages/config-typescript/tsconfig.web.json`**

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "types": ["vite/client"]
  }
}
```

- [ ] **Step 5: Verify each config is loadable by tsc in isolation**

```bash
bun install
New-Item -ItemType Directory -Force .tmp-tsconfig-check
Set-Content .tmp-tsconfig-check/tsconfig.json '{"extends": "../packages/config-typescript/tsconfig.web.json", "files": []}'
bunx tsc --showConfig -p .tmp-tsconfig-check/tsconfig.json
Remove-Item -Recurse -Force .tmp-tsconfig-check
```

Expected: `tsc --showConfig` prints the fully-resolved merged config (jsx, lib, strict all present) with exit 0.

- [ ] **Step 6: Commit**

```bash
git add packages/config-typescript/
git commit -m "feat: add shared typescript config presets"
```

---

### Task 2: `packages/config-vite`

**Files:**
- Create: `tests/unit/config-vite/create-slate-vite-config.test.ts`
- Create: `packages/config-vite/src/index.ts`
- Create: `packages/config-vite/package.json`
- Create: `packages/config-vite/tsconfig.json`
- Create: `packages/config-vite/vitest.config.ts`
- Modify: `tsconfig.json` (root — add reference)

**Interfaces:**
- Consumes: `@slate/config-typescript` `tsconfig.node.json` (Task 1)
- Produces: `createSlateViteConfig(options?: SlateViteConfigOptions): UserConfig` — consumed by `packages/ui-kit/vite.config.ts` (Task 4) and every Phase 3 app's `vite.config.ts`

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, expect, it } from "vitest";
import { createSlateViteConfig } from "../../../packages/config-vite/src/index";

describe("createSlateViteConfig", () => {
  it("includes the React plugin by default", () => {
    const config = createSlateViteConfig();
    expect(config.plugins).toBeDefined();
    expect(config.plugins?.length).toBeGreaterThan(0);
  });

  it("targets es2023 with sourcemaps on", () => {
    const config = createSlateViteConfig();
    expect(config.build?.target).toBe("es2023");
    expect(config.build?.sourcemap).toBe(true);
  });

  it("merges a caller-supplied extraConfig on top of the base", () => {
    const config = createSlateViteConfig({ extraConfig: { server: { port: 4321 } } });
    expect(config.server?.port).toBe(4321);
    expect(config.server?.strictPort).toBe(true);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun test tests/unit/config-vite/create-slate-vite-config.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `packages/config-vite/package.json`**

```json
{
  "name": "@slate/config-vite",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "dependencies": {
    "vite": "latest",
    "@vitejs/plugin-react": "latest"
  },
  "devDependencies": {
    "@slate/config-typescript": "workspace:*"
  }
}
```

- [ ] **Step 4: Write `packages/config-vite/tsconfig.json`**

```json
{
  "extends": "@slate/config-typescript/tsconfig.node.json",
  "compilerOptions": { "outDir": "./dist", "rootDir": "./src" },
  "include": ["src"]
}
```

- [ ] **Step 5: Write `packages/config-vite/src/index.ts`**

```typescript
import { defineConfig, mergeConfig, type UserConfig } from "vite";
import react from "@vitejs/plugin-react";

export interface SlateViteConfigOptions {
  extraConfig?: UserConfig;
}

export function createSlateViteConfig(options: SlateViteConfigOptions = {}): UserConfig {
  const base: UserConfig = {
    plugins: [react()],
    clearScreen: false,
    server: { strictPort: true },
    build: { target: "es2023", sourcemap: true },
  };

  return options.extraConfig ? mergeConfig(base, options.extraConfig) : base;
}

export default defineConfig(createSlateViteConfig());
```

- [ ] **Step 6: Write `packages/config-vite/vitest.config.ts`**

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: { include: ["../../tests/unit/config-vite/**/*.test.ts"] },
});
```

- [ ] **Step 7: Run the test to verify it passes**

Run: `bun install && bun test tests/unit/config-vite/create-slate-vite-config.test.ts`
Expected: PASS — all three assertions green.

- [ ] **Step 8: Add the root `tsconfig.json` reference**

```json
{
  "files": [],
  "references": [{ "path": "./packages/config-vite" }]
}
```

- [ ] **Step 9: Commit**

```bash
git add packages/config-vite/ tests/unit/config-vite/ tsconfig.json bun.lockb
git commit -m "feat: add shared vite config factory"
```

---

### Task 3: `crates/slate-core` stub

**Files:**
- Create: `crates/slate-core/Cargo.toml`
- Create: `crates/slate-core/src/lib.rs`
- Create: `crates/slate-core/tests/.gitkeep`
- Create: `crates/slate-core/package.json`

**Interfaces:**
- Consumes: `Cargo.toml` workspace (`[workspace.dependencies]`, `[workspace.package]`), `scripts/sync-crate-versions.ts` (Phase 1 Task 15)
- Produces: `slate_core::VERSION: &str`, and the Rust workspace's first real member (Phase 1's globbed `crates/*` member now resolves to something)

- [ ] **Step 1: Write `crates/slate-core/Cargo.toml`**

```toml
[package]
name = "slate-core"
version = "0.1.0"
edition.workspace = true
rust-version.workspace = true

[dependencies]
serde = { workspace = true }
thiserror = { workspace = true }
```

- [ ] **Step 2: Write `crates/slate-core/src/lib.rs` with its inline test**

```rust
//! Shared core crate for the Slate portable app suite.
//!
//! Intentionally a stub for now. Future responsibilities (not yet
//! implemented — see `.cursor/rules/009-portability-and-deferred-scope.mdc`):
//! portable path resolution, shared error types, typed IPC contracts.

/// The crate version, exposed so apps can display/log it without depending
/// on `env!("CARGO_PKG_VERSION")` directly everywhere.
pub const VERSION: &str = env!("CARGO_PKG_VERSION");

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn version_is_not_empty() {
        assert!(!VERSION.is_empty());
    }

    #[test]
    fn version_matches_cargo_toml() {
        assert_eq!(VERSION, "0.1.0");
    }
}
```

- [ ] **Step 3: Reserve the integration-test directory**

```bash
mkdir -p crates/slate-core/tests
New-Item -ItemType File crates/slate-core/tests/.gitkeep
```

- [ ] **Step 4: Write `crates/slate-core/package.json` (Changesets tracking only — no JS content)**

```json
{
  "name": "slate-core",
  "version": "0.1.0",
  "private": true
}
```

- [ ] **Step 5: Run the crate's tests**

Run: `cargo test -p slate-core`
Expected: PASS — 2 tests (`version_is_not_empty`, `version_matches_cargo_toml`).

- [ ] **Step 6: Verify the workspace now has a real member**

Run: `cargo metadata --no-deps --format-version 1 | Select-String '"name":"slate-core"'`
Expected: matches (contrast with Phase 1 Task 18, where the member list was empty).

Run: `cargo clippy -p slate-core -- -D warnings`
Expected: exit 0, zero warnings.

- [ ] **Step 7: Verify version sync works end-to-end**

Run: `bun scripts/sync-crate-versions.ts`
Expected: exit 0, `crates/slate-core/Cargo.toml`'s `version = "0.1.0"` unchanged (already matches `package.json`).

- [ ] **Step 8: Commit**

```bash
git add crates/slate-core/
git commit -m "feat: add slate-core stub crate"
```

---

### Task 4: `packages/ui-kit` bootstrap

**Files:**
- Create: `packages/ui-kit/package.json`
- Create: `packages/ui-kit/tsconfig.json`
- Create: `packages/ui-kit/vite.config.ts`
- Create: `packages/ui-kit/vitest.config.ts`
- Create: `packages/ui-kit/tests/setup.ts`
- Create: `packages/ui-kit/playwright.config.ts`
- Create: `packages/ui-kit/src/index.ts`
- Modify: `tsconfig.json` (root — add reference)

**Interfaces:**
- Consumes: `@slate/config-typescript` (Task 1), `@slate/config-vite` (Task 2)
- Produces: the empty barrel `src/index.ts` every subsequent task in this plan appends an `export *` line to; the `tests/unit/**` Vitest environment every subsequent task's tests run under

- [ ] **Step 1: Write `packages/ui-kit/package.json`**

```json
{
  "name": "@slate/ui-kit",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "dependencies": {
    "react": "latest",
    "react-dom": "latest",
    "@tauri-apps/api": "latest",
    "class-variance-authority": "latest",
    "clsx": "latest",
    "tailwind-merge": "latest",
    "lucide-react": "latest",
    "@radix-ui/react-dialog": "latest",
    "@radix-ui/react-tooltip": "latest",
    "@radix-ui/react-dropdown-menu": "latest",
    "@radix-ui/react-switch": "latest",
    "@radix-ui/react-scroll-area": "latest",
    "@radix-ui/react-context-menu": "latest"
  },
  "devDependencies": {
    "@slate/config-typescript": "workspace:*",
    "@slate/config-vite": "workspace:*",
    "tailwindcss": "latest",
    "@tailwindcss/vite": "latest",
    "vite-plugin-dts": "latest",
    "@vitejs/plugin-react": "latest",
    "vitest": "latest",
    "@testing-library/react": "latest",
    "@testing-library/jest-dom": "latest",
    "jsdom": "latest",
    "@playwright/test": "latest"
  }
}
```

- [ ] **Step 2: Write `packages/ui-kit/tsconfig.json`**

```json
{
  "extends": "@slate/config-typescript/tsconfig.web.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Write `packages/ui-kit/vite.config.ts`**

```typescript
import { resolve } from "node:path";
import { defineConfig, mergeConfig } from "vite";
import dts from "vite-plugin-dts";
import tailwindcss from "@tailwindcss/vite";
import { createSlateViteConfig } from "@slate/config-vite";

export default defineConfig(
  mergeConfig(createSlateViteConfig(), {
    plugins: [tailwindcss(), dts({ rollupTypes: true })],
    build: {
      lib: {
        entry: resolve(__dirname, "src/index.ts"),
        formats: ["es"],
        fileName: "index",
      },
      rollupOptions: {
        external: ["react", "react-dom", "@tauri-apps/api"],
      },
    },
  }),
);
```

- [ ] **Step 4: Write `packages/ui-kit/vitest.config.ts`**

```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./tests/setup.ts"],
    include: ["tests/unit/**/*.test.{ts,tsx}"],
  },
});
```

- [ ] **Step 5: Write `packages/ui-kit/tests/setup.ts`**

```typescript
import "@testing-library/jest-dom/vitest";
```

- [ ] **Step 6: Write `packages/ui-kit/playwright.config.ts`**

```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/visual",
  snapshotDir: "./tests/visual/__snapshots__",
  use: { viewport: { width: 1280, height: 800 } },
});
```

- [ ] **Step 7: Write the starting empty barrel `packages/ui-kit/src/index.ts`**

```typescript
export {};
```

- [ ] **Step 8: Add the root `tsconfig.json` reference**

```json
{
  "files": [],
  "references": [
    { "path": "./packages/config-vite" },
    { "path": "./packages/ui-kit" }
  ]
}
```

- [ ] **Step 9: Verify the empty package builds and typechecks**

Run: `bun install && bunx tsc -b`
Expected: exit 0.

Run: `bun test packages/ui-kit --run` (no test files matched yet — expected "No test files found", exit 0)

- [ ] **Step 10: Commit**

```bash
git add packages/ui-kit/package.json packages/ui-kit/tsconfig.json packages/ui-kit/vite.config.ts packages/ui-kit/vitest.config.ts packages/ui-kit/tests/setup.ts packages/ui-kit/playwright.config.ts packages/ui-kit/src/index.ts tsconfig.json bun.lockb
git commit -m "feat: bootstrap packages/ui-kit"
```

---

### Task 5: Nord token files (`src/tokens/`)

**Files:**
- Create: `packages/ui-kit/tests/unit/tokens/nord-tokens.test.ts`
- Create: `packages/ui-kit/src/tokens/theme.nord.polar-night.css`
- Create: `packages/ui-kit/src/tokens/theme.nord.snow-storm.css`
- Create: `packages/ui-kit/src/tokens/theme.nord.frost.css`
- Create: `packages/ui-kit/src/tokens/theme.nord.aurora.css`
- Create: `packages/ui-kit/src/tokens/theme.nord.css`

**Interfaces:**
- Consumes: nothing
- Produces: `--nord0` through `--nord15` CSS custom properties, consumed by `src/styles/main.css` (Task 6)

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { resolve } from "node:path";

const tokensDir = resolve(__dirname, "../../../src/tokens");
const read = (file: string) => readFileSync(resolve(tokensDir, file), "utf-8");

describe("nord token files", () => {
  it("polar-night defines nord0 through nord3", () => {
    const css = read("theme.nord.polar-night.css");
    for (const name of ["--nord0", "--nord1", "--nord2", "--nord3"]) {
      expect(css).toContain(`${name}:`);
    }
  });

  it("snow-storm defines nord4 through nord6", () => {
    const css = read("theme.nord.snow-storm.css");
    for (const name of ["--nord4", "--nord5", "--nord6"]) {
      expect(css).toContain(`${name}:`);
    }
  });

  it("frost defines nord7 through nord10", () => {
    const css = read("theme.nord.frost.css");
    for (const name of ["--nord7", "--nord8", "--nord9", "--nord10"]) {
      expect(css).toContain(`${name}:`);
    }
  });

  it("aurora defines nord11 through nord15", () => {
    const css = read("theme.nord.aurora.css");
    for (const name of ["--nord11", "--nord12", "--nord13", "--nord14", "--nord15"]) {
      expect(css).toContain(`${name}:`);
    }
  });

  it("aggregator imports all four families", () => {
    const css = read("theme.nord.css");
    for (const file of [
      "theme.nord.polar-night.css",
      "theme.nord.snow-storm.css",
      "theme.nord.frost.css",
      "theme.nord.aurora.css",
    ]) {
      expect(css).toContain(file);
    }
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/tokens/nord-tokens.test.ts`
Expected: FAIL — files don't exist yet.

- [ ] **Step 3: Write `theme.nord.polar-night.css`**

```css
:root {
  --nord0: #2e3440;
  --nord1: #3b4252;
  --nord2: #434c5e;
  --nord3: #4c566a;
}
```

- [ ] **Step 4: Write `theme.nord.snow-storm.css`**

```css
:root {
  --nord4: #d8dee9;
  --nord5: #e5e9f0;
  --nord6: #eceff4;
}
```

- [ ] **Step 5: Write `theme.nord.frost.css`**

```css
:root {
  --nord7: #8fbcbb;
  --nord8: #88c0d0;
  --nord9: #81a1c1;
  --nord10: #5e81ac;
}
```

- [ ] **Step 6: Write `theme.nord.aurora.css`**

```css
:root {
  --nord11: #bf616a;
  --nord12: #d08770;
  --nord13: #ebcb8b;
  --nord14: #a3be8c;
  --nord15: #b48ead;
}
```

- [ ] **Step 7: Write the aggregator `theme.nord.css`**

```css
@import "./theme.nord.polar-night.css";
@import "./theme.nord.snow-storm.css";
@import "./theme.nord.frost.css";
@import "./theme.nord.aurora.css";
```

- [ ] **Step 8: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/tokens/nord-tokens.test.ts`
Expected: PASS — 5 tests green.

- [ ] **Step 9: Commit**

```bash
git add packages/ui-kit/src/tokens/ packages/ui-kit/tests/unit/tokens/
git commit -m "feat: add nord theme token files"
```

---

### Task 6: `src/styles/main.css` — Tailwind v4 theme + light/dark wiring

**Files:**
- Create: `packages/ui-kit/tests/unit/styles/main-css.test.ts`
- Create: `packages/ui-kit/src/styles/main.css`

**Interfaces:**
- Consumes: `src/tokens/theme.nord.css` (Task 5)
- Produces: `--color-surface`, `--color-border`, `--color-accent`, `--color-danger`, `--color-warning`, `--color-success`, `--radius-lg` CSS variables consumed by every composite/primitive component built in Tasks 9–13

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { resolve } from "node:path";

const css = readFileSync(resolve(__dirname, "../../../src/styles/main.css"), "utf-8");

describe("main.css", () => {
  it("imports tailwindcss and the nord token aggregator", () => {
    expect(css).toContain('@import "tailwindcss"');
    expect(css).toContain("theme.nord.css");
  });

  it("defines a @theme block with the semantic color tokens", () => {
    expect(css).toContain("@theme");
    for (const token of ["--color-surface", "--color-border", "--color-accent", "--color-danger", "--radius-lg"]) {
      expect(css).toContain(token);
    }
  });

  it("wires data-theme dark and light overrides", () => {
    expect(css).toContain('[data-theme="dark"]');
    expect(css).toContain('[data-theme="light"]');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/styles/main-css.test.ts`
Expected: FAIL — file doesn't exist.

- [ ] **Step 3: Write `src/styles/main.css`**

```css
@import "tailwindcss";
@import "../tokens/theme.nord.css";

@theme {
  --color-surface: var(--nord6);
  --color-border: var(--nord4);
  --color-accent: var(--nord10);
  --color-accent-frost: var(--nord8);
  --color-danger: var(--nord11);
  --color-warning: var(--nord13);
  --color-success: var(--nord14);
  --radius-lg: 0.75rem;
  --font-sans: "Inter", system-ui, sans-serif;
}

[data-theme="dark"] {
  --color-surface: var(--nord0);
  --color-border: var(--nord3);
}

[data-theme="light"] {
  --color-surface: var(--nord6);
  --color-border: var(--nord4);
}

body {
  background-color: var(--color-surface);
  color-scheme: light dark;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/styles/main-css.test.ts`
Expected: PASS — 3 tests green.

- [ ] **Step 5: Commit**

```bash
git add packages/ui-kit/src/styles/ packages/ui-kit/tests/unit/styles/
git commit -m "feat: add tailwind v4 theme and dark/light wiring"
```

---

### Task 7: `src/lib/utils.ts` — `cn()` helper

**Files:**
- Create: `packages/ui-kit/tests/unit/lib/utils.test.ts`
- Create: `packages/ui-kit/src/lib/utils.ts`
- Modify: `packages/ui-kit/src/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `cn(...inputs: ClassValue[]): string` — consumed by every primitive (Task 8) and composite (Tasks 9–13)

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, expect, it } from "vitest";
import { cn } from "../../../src/lib/utils";

describe("cn", () => {
  it("joins simple class strings", () => {
    expect(cn("p-2", "text-sm")).toBe("p-2 text-sm");
  });

  it("lets a later conflicting tailwind class win", () => {
    expect(cn("p-2", "p-4")).toBe("p-4");
  });

  it("drops falsy values", () => {
    expect(cn("p-2", false && "hidden", undefined, "text-sm")).toBe("p-2 text-sm");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/lib/utils.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/lib/utils.ts`**

```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/lib/utils.test.ts`
Expected: PASS — 3 tests green.

- [ ] **Step 5: Append to the public barrel**

```typescript
export * from "./lib/utils";
```

- [ ] **Step 6: Commit**

```bash
git add packages/ui-kit/src/lib/ packages/ui-kit/tests/unit/lib/ packages/ui-kit/src/index.ts
git commit -m "feat: add cn() class-merging utility"
```

---

### Task 8: `src/primitives/` — shadcn CLI atoms

**Files:**
- Create: `packages/ui-kit/components.json`
- Create: `packages/ui-kit/src/primitives/button/`, `dialog/`, `tooltip/`, `dropdown-menu/`, `switch/`, `scroll-area/` (each: `<name>.tsx` + `index.ts`)
- Modify: `packages/ui-kit/src/index.ts`

**Interfaces:**
- Consumes: `src/lib/utils.ts` (Task 7, shadcn-generated components import `cn` from here)
- Produces: `Button`, `Dialog`+parts, `Tooltip`+parts, `DropdownMenu`+parts, `Switch`, `ScrollArea` — consumed by Phase 3 apps and by this plan's own composites (Tasks 9–13 don't need these directly, but Phase 3 does)

- [ ] **Step 1: Write `components.json`**

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/styles/main.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "src/primitives",
    "utils": "src/lib/utils"
  }
}
```

- [ ] **Step 2: Run the shadcn CLI to generate all six atoms**

```bash
cd packages/ui-kit
bunx shadcn@latest add button dialog tooltip dropdown-menu switch scroll-area
cd ../..
```

Expected: creates flat files `packages/ui-kit/src/primitives/button.tsx`,
`dialog.tsx`, `tooltip.tsx`, `dropdown-menu.tsx`, `switch.tsx`,
`scroll-area.tsx`, each importing `cn` from `src/lib/utils`.

- [ ] **Step 3: Move each generated file into its own module folder with an `index.ts` barrel**

```powershell
foreach ($name in @("button","dialog","tooltip","dropdown-menu","switch","scroll-area")) {
  New-Item -ItemType Directory -Force "packages/ui-kit/src/primitives/$name" | Out-Null
  Move-Item "packages/ui-kit/src/primitives/$name.tsx" "packages/ui-kit/src/primitives/$name/$name.tsx"
  Set-Content "packages/ui-kit/src/primitives/$name/index.ts" "export * from `"./$name`";"
}
```

- [ ] **Step 4: Append each module to the public barrel**

```typescript
export * from "./primitives/button";
export * from "./primitives/dialog";
export * from "./primitives/tooltip";
export * from "./primitives/dropdown-menu";
export * from "./primitives/switch";
export * from "./primitives/scroll-area";
```

- [ ] **Step 5: Verify everything still typechecks after the move**

Run: `bun install && bunx tsc -b`
Expected: exit 0 — the relative imports inside each moved file (`../../lib/utils` etc.) resolve correctly since shadcn used the `utils` alias, not a hardcoded relative path.

If a moved file's import path breaks (shadcn sometimes emits `@/lib/utils`),
fix it to `../../lib/utils` in that file before re-running `tsc -b`.

- [ ] **Step 6: Commit**

```bash
git add packages/ui-kit/components.json packages/ui-kit/src/primitives/ packages/ui-kit/src/index.ts
git commit -m "feat: add shadcn/radix primitive atoms"
```

---

### Task 9: `src/providers/theme-provider/`

**Files:**
- Create: `packages/ui-kit/tests/unit/providers/theme-provider.test.tsx`
- Create: `packages/ui-kit/src/providers/theme-provider/theme-provider.tsx`
- Create: `packages/ui-kit/src/providers/theme-provider/index.ts`
- Modify: `packages/ui-kit/src/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `ThemeProvider`, `useTheme(): { theme: SlateTheme; setTheme: (t: SlateTheme) => void }`, `SlateTheme = "dark" | "light"` — consumed by every Phase 3 app's `src/modules/app/app.tsx`

- [ ] **Step 1: Write the failing test**

```tsx
import { describe, expect, it } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import { ThemeProvider, useTheme } from "../../../src/providers/theme-provider/theme-provider";

function ThemeConsumer() {
  const { theme, setTheme } = useTheme();
  return (
    <button type="button" onClick={() => setTheme(theme === "dark" ? "light" : "dark")}>
      {theme}
    </button>
  );
}

describe("ThemeProvider", () => {
  it("sets data-theme on the document element to the default theme", () => {
    render(
      <ThemeProvider>
        <ThemeConsumer />
      </ThemeProvider>,
    );
    expect(document.documentElement.getAttribute("data-theme")).toBe("dark");
  });

  it("updates data-theme when setTheme is called", () => {
    render(
      <ThemeProvider>
        <ThemeConsumer />
      </ThemeProvider>,
    );
    fireEvent.click(screen.getByRole("button"));
    expect(document.documentElement.getAttribute("data-theme")).toBe("light");
  });

  it("throws when useTheme is called outside a ThemeProvider", () => {
    function Bare() {
      useTheme();
      return null;
    }
    expect(() => render(<Bare />)).toThrow("useTheme must be used within a ThemeProvider");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/providers/theme-provider.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/providers/theme-provider/theme-provider.tsx`**

```tsx
import { createContext, useContext, useEffect, useMemo, useState, type ReactNode } from "react";

export type SlateTheme = "dark" | "light";

interface ThemeContextValue {
  theme: SlateTheme;
  setTheme: (theme: SlateTheme) => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

export interface ThemeProviderProps {
  children: ReactNode;
  defaultTheme?: SlateTheme;
}

export function ThemeProvider({ children, defaultTheme = "dark" }: ThemeProviderProps) {
  const [theme, setTheme] = useState<SlateTheme>(defaultTheme);

  useEffect(() => {
    document.documentElement.setAttribute("data-theme", theme);
  }, [theme]);

  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export function useTheme(): ThemeContextValue {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error("useTheme must be used within a ThemeProvider");
  }
  return context;
}
```

- [ ] **Step 4: Write `src/providers/theme-provider/index.ts`**

```typescript
export * from "./theme-provider";
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/providers/theme-provider.test.tsx`
Expected: PASS — 3 tests green.

- [ ] **Step 6: Append to the public barrel**

```typescript
export * from "./providers/theme-provider";
```

- [ ] **Step 7: Commit**

```bash
git add packages/ui-kit/src/providers/ packages/ui-kit/tests/unit/providers/ packages/ui-kit/src/index.ts
git commit -m "feat: add ThemeProvider"
```

---

### Task 10: `src/composites/titlebar/` + `window-controls`

**Files:**
- Create: `packages/ui-kit/tests/unit/composites/titlebar.test.tsx`
- Create: `packages/ui-kit/src/composites/titlebar/window-controls.tsx`
- Create: `packages/ui-kit/src/composites/titlebar/titlebar.tsx`
- Create: `packages/ui-kit/src/composites/titlebar/index.ts`
- Modify: `packages/ui-kit/src/index.ts`

**Interfaces:**
- Consumes: `cn` (Task 7)
- Produces: `TitleBar({ title, icon?, className? })`, `WindowControls({ className? })` — consumed by Phase 3's `src/modules/app/app.tsx` in every app

- [ ] **Step 1: Write the failing test (mocking `@tauri-apps/api/window`)**

```tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import { TitleBar } from "../../../src/composites/titlebar/titlebar";

const minimize = vi.fn();
const toggleMaximize = vi.fn();
const close = vi.fn();

vi.mock("@tauri-apps/api/window", () => ({
  getCurrentWindow: () => ({ minimize, toggleMaximize, close }),
}));

describe("TitleBar", () => {
  it("renders the given title", () => {
    render(<TitleBar title="Slate Terminal" />);
    expect(screen.getByText("Slate Terminal")).toBeInTheDocument();
  });

  it("calls window.minimize when the minimize button is clicked", () => {
    render(<TitleBar title="Slate Terminal" />);
    fireEvent.click(screen.getByLabelText("Minimize"));
    expect(minimize).toHaveBeenCalledOnce();
  });

  it("calls window.toggleMaximize when the maximize button is clicked", () => {
    render(<TitleBar title="Slate Terminal" />);
    fireEvent.click(screen.getByLabelText("Maximize"));
    expect(toggleMaximize).toHaveBeenCalledOnce();
  });

  it("calls window.close when the close button is clicked", () => {
    render(<TitleBar title="Slate Terminal" />);
    fireEvent.click(screen.getByLabelText("Close"));
    expect(close).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/titlebar.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/composites/titlebar/window-controls.tsx`**

```tsx
import { getCurrentWindow } from "@tauri-apps/api/window";
import { Minus, Square, X } from "lucide-react";
import { cn } from "../../lib/utils";

export interface WindowControlsProps {
  className?: string;
}

export function WindowControls({ className }: WindowControlsProps) {
  const appWindow = getCurrentWindow();

  return (
    <div className={cn("flex items-center gap-1", className)}>
      <button type="button" aria-label="Minimize" onClick={() => appWindow.minimize()}>
        <Minus size={14} />
      </button>
      <button type="button" aria-label="Maximize" onClick={() => appWindow.toggleMaximize()}>
        <Square size={12} />
      </button>
      <button type="button" aria-label="Close" onClick={() => appWindow.close()}>
        <X size={14} />
      </button>
    </div>
  );
}
```

- [ ] **Step 4: Write `src/composites/titlebar/titlebar.tsx`**

```tsx
import type { ReactNode } from "react";
import { WindowControls } from "./window-controls";
import { cn } from "../../lib/utils";

export interface TitleBarProps {
  title: string;
  icon?: ReactNode;
  className?: string;
}

export function TitleBar({ title, icon, className }: TitleBarProps) {
  return (
    <header
      data-tauri-drag-region
      className={cn(
        "flex h-9 items-center justify-between border-b px-3 backdrop-blur",
        className,
      )}
    >
      <div className="flex items-center gap-2 text-sm font-medium">
        {icon}
        <span>{title}</span>
      </div>
      <WindowControls />
    </header>
  );
}
```

- [ ] **Step 5: Write `src/composites/titlebar/index.ts`**

```typescript
export * from "./titlebar";
export * from "./window-controls";
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/titlebar.test.tsx`
Expected: PASS — 4 tests green.

- [ ] **Step 7: Append to the public barrel**

```typescript
export * from "./composites/titlebar";
```

- [ ] **Step 8: Commit**

```bash
git add packages/ui-kit/src/composites/titlebar/ packages/ui-kit/tests/unit/composites/titlebar.test.tsx packages/ui-kit/src/index.ts
git commit -m "feat: add TitleBar and WindowControls composites"
```

---

### Task 11: `src/composites/toolbar/`

**Files:**
- Create: `packages/ui-kit/tests/unit/composites/toolbar.test.tsx`
- Create: `packages/ui-kit/src/composites/toolbar/toolbar.tsx`
- Create: `packages/ui-kit/src/composites/toolbar/index.ts`
- Modify: `packages/ui-kit/src/index.ts`

**Interfaces:**
- Consumes: `cn` (Task 7)
- Produces: `Toolbar({ children, className? })` — consumed by Phase 3's `src/modules/app/app.tsx` (wraps each app's `src/modules/app-toolbar/`)

- [ ] **Step 1: Write the failing test**

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { Toolbar } from "../../../src/composites/toolbar/toolbar";

describe("Toolbar", () => {
  it("renders its children", () => {
    render(
      <Toolbar>
        <span>Status: Ready</span>
      </Toolbar>,
    );
    expect(screen.getByText("Status: Ready")).toBeInTheDocument();
  });

  it("renders as a footer element", () => {
    render(<Toolbar>content</Toolbar>);
    expect(screen.getByText("content").closest("footer")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/toolbar.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/composites/toolbar/toolbar.tsx`**

```tsx
import type { ReactNode } from "react";
import { cn } from "../../lib/utils";

export interface ToolbarProps {
  children: ReactNode;
  className?: string;
}

export function Toolbar({ children, className }: ToolbarProps) {
  return (
    <footer className={cn("flex h-10 items-center gap-2 border-t px-3", className)}>
      {children}
    </footer>
  );
}
```

- [ ] **Step 4: Write `src/composites/toolbar/index.ts`**

```typescript
export * from "./toolbar";
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/toolbar.test.tsx`
Expected: PASS — 2 tests green.

- [ ] **Step 6: Append to the public barrel**

```typescript
export * from "./composites/toolbar";
```

- [ ] **Step 7: Commit**

```bash
git add packages/ui-kit/src/composites/toolbar/ packages/ui-kit/tests/unit/composites/toolbar.test.tsx packages/ui-kit/src/index.ts
git commit -m "feat: add Toolbar composite"
```

---

### Task 12: `src/composites/app-shell/`

**Files:**
- Create: `packages/ui-kit/tests/unit/composites/app-shell.test.tsx`
- Create: `packages/ui-kit/src/composites/app-shell/app-shell.tsx`
- Create: `packages/ui-kit/src/composites/app-shell/index.ts`
- Modify: `packages/ui-kit/src/index.ts`

**Interfaces:**
- Consumes: `cn` (Task 7)
- Produces: `AppShell({ titleBar, toolbar, children, className? })` — consumed by Phase 3's `src/modules/app/app.tsx` in every app (wraps `TitleBar` + main content + `Toolbar`)

- [ ] **Step 1: Write the failing test**

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { AppShell } from "../../../src/composites/app-shell/app-shell";

describe("AppShell", () => {
  it("renders titleBar, children, and toolbar in DOM order", () => {
    render(
      <AppShell titleBar={<div>TITLE</div>} toolbar={<div>TOOLBAR</div>}>
        <div>CONTENT</div>
      </AppShell>,
    );
    const container = screen.getByText("CONTENT").parentElement?.parentElement;
    const text = container?.textContent ?? "";
    expect(text.indexOf("TITLE")).toBeLessThan(text.indexOf("CONTENT"));
    expect(text.indexOf("CONTENT")).toBeLessThan(text.indexOf("TOOLBAR"));
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/app-shell.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/composites/app-shell/app-shell.tsx`**

```tsx
import type { ReactNode } from "react";
import { cn } from "../../lib/utils";

export interface AppShellProps {
  titleBar: ReactNode;
  toolbar: ReactNode;
  children: ReactNode;
  className?: string;
}

export function AppShell({ titleBar, toolbar, children, className }: AppShellProps) {
  return (
    <div className={cn("flex h-screen flex-col", className)}>
      {titleBar}
      <main className="flex-1 overflow-auto">{children}</main>
      {toolbar}
    </div>
  );
}
```

- [ ] **Step 4: Write `src/composites/app-shell/index.ts`**

```typescript
export * from "./app-shell";
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/app-shell.test.tsx`
Expected: PASS — 1 test green.

- [ ] **Step 6: Append to the public barrel**

```typescript
export * from "./composites/app-shell";
```

- [ ] **Step 7: Commit**

```bash
git add packages/ui-kit/src/composites/app-shell/ packages/ui-kit/tests/unit/composites/app-shell.test.tsx packages/ui-kit/src/index.ts
git commit -m "feat: add AppShell composite"
```

---

### Task 13: `src/composites/context-menu/`

**Files:**
- Create: `packages/ui-kit/tests/unit/composites/context-menu.test.tsx`
- Create: `packages/ui-kit/src/composites/context-menu/context-menu.tsx`
- Create: `packages/ui-kit/src/composites/context-menu/index.ts`
- Modify: `packages/ui-kit/src/index.ts`

**Interfaces:**
- Consumes: `cn` (Task 7), `@radix-ui/react-context-menu` (Task 4 dependency)
- Produces: `ContextMenu`, `ContextMenuTrigger`, `ContextMenuContent`, `ContextMenuItem` — the base module Phase 3 apps compose/extend per `.cursor/rules/005-ui-kit-boundaries.mdc`, never edited directly by an app

- [ ] **Step 1: Write the failing test**

```tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import {
  ContextMenu,
  ContextMenuContent,
  ContextMenuItem,
  ContextMenuTrigger,
} from "../../../src/composites/context-menu/context-menu";

describe("ContextMenu", () => {
  it("shows its items after a right-click on the trigger, and calls onSelect", () => {
    const onSelect = vi.fn();
    render(
      <ContextMenu>
        <ContextMenuTrigger>
          <div>Right-click me</div>
        </ContextMenuTrigger>
        <ContextMenuContent>
          <ContextMenuItem onSelect={onSelect}>Rename</ContextMenuItem>
        </ContextMenuContent>
      </ContextMenu>,
    );

    fireEvent.contextMenu(screen.getByText("Right-click me"));
    const item = screen.getByText("Rename");
    expect(item).toBeInTheDocument();

    fireEvent.click(item);
    expect(onSelect).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/context-menu.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/composites/context-menu/context-menu.tsx`**

```tsx
import * as ContextMenuPrimitive from "@radix-ui/react-context-menu";
import type { ReactNode } from "react";
import { cn } from "../../lib/utils";

export const ContextMenu = ContextMenuPrimitive.Root;
export const ContextMenuTrigger = ContextMenuPrimitive.Trigger;

export interface ContextMenuContentProps {
  children: ReactNode;
  className?: string;
}

export function ContextMenuContent({ children, className }: ContextMenuContentProps) {
  return (
    <ContextMenuPrimitive.Portal>
      <ContextMenuPrimitive.Content
        className={cn(
          "min-w-40 rounded-lg border bg-[var(--color-surface)] p-1 shadow-sm",
          className,
        )}
      >
        {children}
      </ContextMenuPrimitive.Content>
    </ContextMenuPrimitive.Portal>
  );
}

export interface ContextMenuItemProps {
  children: ReactNode;
  onSelect?: () => void;
}

export function ContextMenuItem({ children, onSelect }: ContextMenuItemProps) {
  return (
    <ContextMenuPrimitive.Item
      className="cursor-pointer rounded px-2 py-1.5 text-sm outline-none data-[highlighted]:bg-[var(--color-accent-frost)]"
      onSelect={onSelect}
    >
      {children}
    </ContextMenuPrimitive.Item>
  );
}
```

- [ ] **Step 4: Write `src/composites/context-menu/index.ts`**

```typescript
export * from "./context-menu";
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `bun vitest run packages/ui-kit/tests/unit/composites/context-menu.test.tsx`
Expected: PASS — 1 test green.

- [ ] **Step 6: Append to the public barrel**

```typescript
export * from "./composites/context-menu";
```

- [ ] **Step 7: Commit**

```bash
git add packages/ui-kit/src/composites/context-menu/ packages/ui-kit/tests/unit/composites/context-menu.test.tsx packages/ui-kit/src/index.ts
git commit -m "feat: add base ContextMenu composite"
```

---

### Task 14: Phase 2 verification pass

**Files:** none created — verification only

**Interfaces:**
- Consumes: every file created in Tasks 1–13
- Produces: confirmation Phase 2 is complete and Phase 3 (9 template apps) can build on a real, working `@slate/ui-kit` and `slate-core`

- [ ] **Step 1: Full install**

Run: `bun install`
Expected: exit 0.

- [ ] **Step 2: Lint**

Run: `bunx biome check .`
Expected: exit 0.

- [ ] **Step 3: TypeScript across the whole graph**

Run: `bunx tsc -b`
Expected: exit 0 — builds `config-vite` and `ui-kit` (now non-empty).

- [ ] **Step 4: Full ui-kit + config-vite test suite**

Run: `bunx vitest run`
Expected: exit 0 — every test file from Tasks 2, 5, 6, 7, 9, 10, 11, 12, 13 passes (9 files, 20+ tests total).

- [ ] **Step 5: Rust**

Run: `cargo build --workspace && cargo clippy --workspace -- -D warnings && cargo test --workspace`
Expected: all exit 0 — `slate-core` builds, lints clean, and its 2 tests pass.

- [ ] **Step 6: moon**

Run: `moon run :lint :typecheck :test :build`
Expected: exit 0 across `config-typescript`, `config-vite`, `ui-kit` projects (moon picks these up automatically via `.moon/workspace.yml`'s `packages/*` glob from Phase 1).

- [ ] **Step 7: Record a changeset for this phase's work**

Run: `bunx changeset` — select `@slate/ui-kit`, `@slate/config-vite`, and `slate-core` as `minor` bumps (first real functionality), write "Add shared design system, vite config preset, and slate-core stub" as the summary.

Run: `bunx changeset status`
Expected: reports the changeset just created.

- [ ] **Step 8: Final commit**

```bash
git add .changeset/ -A
git commit -m "chore: phase 2 verification and changeset"
```

Phase 2 is complete. Proceed to the Phase 3 plan (Template Apps x9:
`scripts/new-app.ts` generator + `scripts/templates/app/`, then generate
all 9 apps, then hand-patch `slate-launcher`'s tray/isolation extras) once
this verification pass is fully green.
