# Phase 4: Release Packaging, Remaining Scripts & Full Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Prerequisite:** Phase 3 (`.superpowers/docs/plans/2026-08-29-vicore-slate-scaffold.phase.03.md`) must be fully verified before starting this plan — `scripts/package.ts` packages the 9 apps Phase 3 built, and `scripts/new-package.ts`/`new-crate.ts` reuse the template-generation helper Phase 3's `scripts/new-app.ts` introduced.

**Goal:** Finish the `scripts/` automation surface (`new-package.ts`, `new-crate.ts`, `dev.ts`, `changelog.ts`, `package.ts`), wire a root `moon run :package` task, and run one final full-graph verification pass across the entire repo — ending with a real `installDir/` produced from at least one built app, proving the whole scaffold from `.cursor/rules/000` through the packaged output works end to end.

**Architecture:** First extract a shared `scripts/lib/generate-from-template.ts` helper and refactor Phase 3's `new-app.ts` onto it (DRY — three generators shouldn't each hand-roll directory-walk-and-substitute). Then build `new-package.ts`/`new-crate.ts` as thin wrappers over that helper, `dev.ts` as a local orchestration script, `changelog.ts` as a small pure aggregator, and `package.ts` as the final assembly script whose core logic (`assembleInstallDir`) is fully unit-testable against a fake monorepo tree in a temp directory — no real Tauri build is required to test its logic, only to smoke-test it for real in the last task.

**Tech Stack:** Bun-executed TypeScript throughout, moon v2 root-project task, `bun:test`.

## Global Constraints

- Every script under `scripts/` is Bun-executed TypeScript — no `.ps1`.
- `scripts/new-package.ts` and `scripts/new-crate.ts` are for **future** packages/crates — `packages/config-typescript`, `config-vite`, `ui-kit`, and `crates/slate-core` were already hand-authored in Phase 2 because each is foundational/unique, not a generic instance of the template.
- No published output anywhere — `scripts/package.ts` only ever writes to a local `installDir/` (gitignored — add it to `.gitignore` in Task 6 if not already present), never publishes.
- All new logic gets a test under `tests/unit/**` before implementation (TDD), same as every prior phase.
- This is the last phase — its final task is a full-repository verification pass, not just this phase's own files.

---

### Task 1: Extract `scripts/lib/generate-from-template.ts` + refactor `new-app.ts` onto it

**Files:**

- Create: `tests/unit/lib/generate-from-template.test.ts`
- Create: `scripts/lib/generate-from-template.ts`
- Modify: `scripts/new-app.ts`

**Interfaces:**

- Consumes: nothing
- Produces: `generateFromTemplate(templateDir: string, targetDir: string, replacements: Record<string, string>): void` — consumed by `new-app.ts` (refactored here), `new-package.ts` (Task 2), `new-crate.ts` (Task 3)

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, expect, it, beforeEach, afterEach } from "bun:test";
import {
  mkdtempSync,
  mkdirSync,
  rmSync,
  writeFileSync,
  readFileSync,
  existsSync,
} from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { generateFromTemplate } from "../../../scripts/lib/generate-from-template";

let root: string;
let templateDir: string;
let targetDir: string;

beforeEach(() => {
  root = mkdtempSync(join(tmpdir(), "slate-template-"));
  templateDir = join(root, "template");
  targetDir = join(root, "target");
  mkdirSync(join(templateDir, "nested"), { recursive: true });
  writeFileSync(join(templateDir, "package.json"), '{"name": "{{name}}"}');
  writeFileSync(
    join(templateDir, "nested", "file.ts"),
    'export const title = "{{title}}";',
  );
});

afterEach(() => {
  rmSync(root, { recursive: true, force: true });
});

describe("generateFromTemplate", () => {
  it("copies the full directory tree to the target", () => {
    generateFromTemplate(templateDir, targetDir, {
      name: "terminal",
      title: "Terminal",
    });
    expect(existsSync(join(targetDir, "package.json"))).toBe(true);
    expect(existsSync(join(targetDir, "nested", "file.ts"))).toBe(true);
  });

  it("substitutes every {{key}} placeholder in every file", () => {
    generateFromTemplate(templateDir, targetDir, {
      name: "terminal",
      title: "Terminal",
    });
    expect(readFileSync(join(targetDir, "package.json"), "utf-8")).toBe(
      '{"name": "terminal"}',
    );
    expect(readFileSync(join(targetDir, "nested", "file.ts"), "utf-8")).toBe(
      'export const title = "Terminal";',
    );
  });

  it("leaves an unrecognized placeholder untouched rather than throwing", () => {
    writeFileSync(join(templateDir, "extra.txt"), "{{unknown}}");
    generateFromTemplate(templateDir, targetDir, {
      name: "terminal",
      title: "Terminal",
    });
    expect(readFileSync(join(targetDir, "extra.txt"), "utf-8")).toBe(
      "{{unknown}}",
    );
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun test tests/unit/lib/generate-from-template.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `scripts/lib/generate-from-template.ts`**

```typescript
import {
  cpSync,
  readdirSync,
  readFileSync,
  statSync,
  writeFileSync,
} from "node:fs";
import { join } from "node:path";

function substitute(
  content: string,
  replacements: Record<string, string>,
): string {
  let result = content;
  for (const [key, value] of Object.entries(replacements)) {
    result = result.replaceAll(`{{${key}}}`, value);
  }
  return result;
}

function walkAndSubstitute(
  dir: string,
  replacements: Record<string, string>,
): void {
  for (const entry of readdirSync(dir)) {
    const fullPath = join(dir, entry);
    if (statSync(fullPath).isDirectory()) {
      walkAndSubstitute(fullPath, replacements);
    } else {
      const content = readFileSync(fullPath, "utf-8");
      writeFileSync(fullPath, substitute(content, replacements));
    }
  }
}

export function generateFromTemplate(
  templateDir: string,
  targetDir: string,
  replacements: Record<string, string>,
): void {
  cpSync(templateDir, targetDir, { recursive: true });
  walkAndSubstitute(targetDir, replacements);
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `bun test tests/unit/lib/generate-from-template.test.ts`
Expected: PASS — all three tests green.

- [ ] **Step 5: Refactor `scripts/new-app.ts` to delegate to the shared helper**

```typescript
import { join } from "node:path";
import { generateFromTemplate } from "./lib/generate-from-template";

const TEMPLATE_DIR = join(import.meta.dir, "templates", "app");

function toTitleCase(name: string): string {
  return name.charAt(0).toUpperCase() + name.slice(1);
}

export function generateApp(name: string, targetDir: string): void {
  generateFromTemplate(TEMPLATE_DIR, targetDir, {
    name,
    title: toTitleCase(name),
  });
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

- [ ] **Step 6: Regression-check the existing `new-app` tests still pass after the refactor**

Run: `bun test tests/unit/new-app.test.ts`
Expected: PASS — same 3 tests from Phase 3 Task 2, now exercising the shared helper underneath.

- [ ] **Step 7: Commit**

```bash
git add scripts/lib/generate-from-template.ts scripts/new-app.ts tests/unit/lib/generate-from-template.test.ts
git commit -m "refactor: extract shared template-generation helper"
```

---

### Task 2: `scripts/templates/package/` + `scripts/new-package.ts`

**Files:**

- Create: `scripts/templates/package/package.json`
- Create: `scripts/templates/package/tsconfig.json`
- Create: `scripts/templates/package/vitest.config.ts`
- Create: `scripts/templates/package/src/index.ts`
- Create: `scripts/templates/package/tests/unit/.gitkeep`
- Create: `tests/unit/new-package.test.ts`
- Create: `scripts/new-package.ts`

**Interfaces:**

- Consumes: `generateFromTemplate` (Task 1)
- Produces: `generatePackage(name: string, targetDir: string): void` — documented as the only way to create a new `packages/<name>` per `.cursor/commands/new-package.md` (Phase 1, Task 13)

- [ ] **Step 1: Write the template files**

`scripts/templates/package/package.json`:

```json
{
  "name": "@slate/{{name}}",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "devDependencies": {
    "@slate/config-typescript": "workspace:*"
  }
}
```

`scripts/templates/package/tsconfig.json`:

```json
{
  "extends": "@slate/config-typescript/tsconfig.node.json",
  "compilerOptions": { "outDir": "./dist", "rootDir": "./src" },
  "include": ["src"]
}
```

`scripts/templates/package/vitest.config.ts`:

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: { include: ["tests/unit/**/*.test.ts"] },
});
```

`scripts/templates/package/src/index.ts`:

```typescript
export {};
```

```bash
mkdir -p scripts/templates/package/tests/unit
New-Item -ItemType File scripts/templates/package/tests/unit/.gitkeep
```

- [ ] **Step 2: Write the failing test**

```typescript
import { describe, expect, it, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync, readFileSync, existsSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { generatePackage } from "../../scripts/new-package";

let targetRoot: string;

beforeEach(() => {
  targetRoot = mkdtempSync(join(tmpdir(), "slate-package-"));
});

afterEach(() => {
  rmSync(targetRoot, { recursive: true, force: true });
});

describe("generatePackage", () => {
  it("creates the package with the expected files and name substitution", () => {
    const dir = join(targetRoot, "logging");
    generatePackage("logging", dir);

    expect(existsSync(join(dir, "src", "index.ts"))).toBe(true);
    const pkg = readFileSync(join(dir, "package.json"), "utf-8");
    expect(pkg).toContain('"@slate/logging"');
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `bun test tests/unit/new-package.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 4: Write `scripts/new-package.ts`**

```typescript
import { join } from "node:path";
import { generateFromTemplate } from "./lib/generate-from-template";

const TEMPLATE_DIR = join(import.meta.dir, "templates", "package");

export function generatePackage(name: string, targetDir: string): void {
  generateFromTemplate(TEMPLATE_DIR, targetDir, { name });
}

if (import.meta.main) {
  const nameArgIndex = process.argv.indexOf("--name");
  const name = nameArgIndex !== -1 ? process.argv[nameArgIndex + 1] : undefined;
  if (!name) {
    console.error("Usage: bun scripts/new-package.ts --name <name>");
    process.exit(1);
  }
  generatePackage(name, join(import.meta.dir, "..", "packages", name));
  console.log(`Generated packages/${name}`);
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `bun test tests/unit/new-package.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add scripts/templates/package/ scripts/new-package.ts tests/unit/new-package.test.ts
git commit -m "feat: add new-package generator script"
```

---

### Task 3: `scripts/templates/crate/` + `scripts/new-crate.ts`

**Files:**

- Create: `scripts/templates/crate/Cargo.toml`
- Create: `scripts/templates/crate/src/lib.rs`
- Create: `scripts/templates/crate/tests/.gitkeep`
- Create: `scripts/templates/crate/package.json`
- Create: `tests/unit/new-crate.test.ts`
- Create: `scripts/new-crate.ts`

**Interfaces:**

- Consumes: `generateFromTemplate` (Task 1)
- Produces: `generateCrate(name: string, targetDir: string): void` — documented as the only way to create a new `crates/<name>` per `.cursor/commands/new-crate.md` (Phase 1, Task 13); every generated crate already has its Changesets-tracking `package.json` from day one (per `.cursor/rules/010-git-commit-workflow.mdc`)

- [ ] **Step 1: Write the template files**

`scripts/templates/crate/Cargo.toml`:

```toml
[package]
name = "{{name}}"
version = "0.1.0"
edition.workspace = true
rust-version.workspace = true

[dependencies]
serde = { workspace = true }
thiserror = { workspace = true }
```

`scripts/templates/crate/src/lib.rs`:

```rust
//! {{name}} — generated crate stub. Replace this doc comment and add real
//! functionality; keep integration tests under `tests/`.

pub const VERSION: &str = env!("CARGO_PKG_VERSION");

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn version_is_not_empty() {
        assert!(!VERSION.is_empty());
    }
}
```

`scripts/templates/crate/package.json`:

```json
{
  "name": "{{name}}",
  "version": "0.1.0",
  "private": true
}
```

```bash
mkdir -p scripts/templates/crate/tests
New-Item -ItemType File scripts/templates/crate/tests/.gitkeep
```

- [ ] **Step 2: Write the failing test**

```typescript
import { describe, expect, it, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync, readFileSync, existsSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { generateCrate } from "../../scripts/new-crate";

let targetRoot: string;

beforeEach(() => {
  targetRoot = mkdtempSync(join(tmpdir(), "slate-crate-"));
});

afterEach(() => {
  rmSync(targetRoot, { recursive: true, force: true });
});

describe("generateCrate", () => {
  it("creates the crate with Cargo.toml, src/lib.rs, and a tracking package.json", () => {
    const dir = join(targetRoot, "slate-plugin-host");
    generateCrate("slate-plugin-host", dir);

    expect(existsSync(join(dir, "src", "lib.rs"))).toBe(true);
    expect(readFileSync(join(dir, "Cargo.toml"), "utf-8")).toContain(
      'name = "slate-plugin-host"',
    );
    expect(readFileSync(join(dir, "package.json"), "utf-8")).toContain(
      '"slate-plugin-host"',
    );
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `bun test tests/unit/new-crate.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 4: Write `scripts/new-crate.ts`**

```typescript
import { join } from "node:path";
import { generateFromTemplate } from "./lib/generate-from-template";

const TEMPLATE_DIR = join(import.meta.dir, "templates", "crate");

export function generateCrate(name: string, targetDir: string): void {
  generateFromTemplate(TEMPLATE_DIR, targetDir, { name });
}

if (import.meta.main) {
  const nameArgIndex = process.argv.indexOf("--name");
  const name = nameArgIndex !== -1 ? process.argv[nameArgIndex + 1] : undefined;
  if (!name) {
    console.error("Usage: bun scripts/new-crate.ts --name <name>");
    process.exit(1);
  }
  generateCrate(name, join(import.meta.dir, "..", "crates", name));
  console.log(`Generated crates/${name}`);
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `bun test tests/unit/new-crate.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add scripts/templates/crate/ scripts/new-crate.ts tests/unit/new-crate.test.ts
git commit -m "feat: add new-crate generator script"
```

---

### Task 4: `scripts/dev.ts` — local dev orchestration

**Files:**

- Create: `tests/unit/dev.test.ts`
- Create: `scripts/dev.ts`

**Interfaces:**

- Consumes: nothing
- Produces: `resolveDevTarget(args: string[], availableApps: string[]): string` (pure, tested) + a thin `main()` that spawns `bun run tauri dev` in the resolved app's directory (not unit-tested — verified manually in Task 8)

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, expect, it } from "bun:test";
import { resolveDevTarget } from "../../scripts/dev";

const AVAILABLE = [
  "slate-launcher",
  "slate-terminal",
  "slate-explorer",
  "slate-browser",
  "slate-editor",
  "slate-gallery",
  "slate-jukebox",
  "slate-player",
  "slate-aistudio",
];

describe("resolveDevTarget", () => {
  it("defaults to slate-launcher when no --app flag is given", () => {
    expect(resolveDevTarget([], AVAILABLE)).toBe("slate-launcher");
  });

  it("resolves --app <name> to the matching slate-<name> app", () => {
    expect(resolveDevTarget(["--app", "terminal"], AVAILABLE)).toBe(
      "slate-terminal",
    );
  });

  it("throws a clear error for an unknown app name", () => {
    expect(() => resolveDevTarget(["--app", "nope"], AVAILABLE)).toThrow(
      "Unknown app: slate-nope",
    );
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun test tests/unit/dev.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `scripts/dev.ts`**

```typescript
import { readdirSync } from "node:fs";
import { join } from "node:path";

export function resolveDevTarget(
  args: string[],
  availableApps: string[],
): string {
  const appArgIndex = args.indexOf("--app");
  if (appArgIndex === -1) return "slate-launcher";

  const requested = `slate-${args[appArgIndex + 1]}`;
  if (!availableApps.includes(requested)) {
    throw new Error(`Unknown app: ${requested}`);
  }
  return requested;
}

if (import.meta.main) {
  const appsDir = join(import.meta.dir, "..", "apps");
  const availableApps = readdirSync(appsDir, { withFileTypes: true })
    .filter((entry) => entry.isDirectory())
    .map((entry) => entry.name);

  const target = resolveDevTarget(process.argv.slice(2), availableApps);
  console.log(`Starting dev server for ${target}...`);

  const proc = Bun.spawn(["bun", "run", "tauri", "dev"], {
    cwd: join(appsDir, target),
    stdio: ["inherit", "inherit", "inherit"],
  });
  process.exit(await proc.exited);
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `bun test tests/unit/dev.test.ts`
Expected: PASS — all three tests green.

- [ ] **Step 5: Commit**

```bash
git add scripts/dev.ts tests/unit/dev.test.ts
git commit -m "feat: add local dev orchestration script"
```

---

### Task 5: `scripts/templates/appdata/configs/settings.toml` + `scripts/changelog.ts`

**Files:**

- Create: `scripts/templates/appdata/configs/settings.toml`
- Create: `tests/unit/changelog.test.ts`
- Create: `scripts/changelog.ts`

**Interfaces:**

- Consumes: nothing
- Produces: `aggregateChangelogs(sources: { name: string; path: string }[], outputPath: string): void` — consumed by `scripts/package.ts` (Task 6) via the file it writes to `.changeset/CHANGES/CHANGELOG.md`

- [ ] **Step 1: Write `scripts/templates/appdata/configs/settings.toml`**

```toml
[general]
theme = "dark"

[window]
# Persisted window state is deferred until crates/slate-runtime exists —
# see .cursor/rules/009-portability-and-deferred-scope.mdc.
remember_position = false
remember_size = false
```

- [ ] **Step 2: Write the failing test**

```typescript
import { describe, expect, it, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync, writeFileSync, readFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { aggregateChangelogs } from "../../scripts/changelog";

let dir: string;

beforeEach(() => {
  dir = mkdtempSync(join(tmpdir(), "slate-changelog-"));
});

afterEach(() => {
  rmSync(dir, { recursive: true, force: true });
});

describe("aggregateChangelogs", () => {
  it("concatenates each source under its own heading, in the given order", () => {
    const uiKitLog = join(dir, "ui-kit.md");
    const terminalLog = join(dir, "terminal.md");
    writeFileSync(uiKitLog, "# @slate/ui-kit\n\n## 0.1.0\n- Initial release\n");
    writeFileSync(
      terminalLog,
      "# slate-terminal\n\n## 0.1.0\n- Initial release\n",
    );

    const outputPath = join(dir, "CHANGELOG.md");
    aggregateChangelogs(
      [
        { name: "@slate/ui-kit", path: uiKitLog },
        { name: "slate-terminal", path: terminalLog },
      ],
      outputPath,
    );

    const result = readFileSync(outputPath, "utf-8");
    const uiKitIndex = result.indexOf("@slate/ui-kit");
    const terminalIndex = result.indexOf("slate-terminal");
    expect(uiKitIndex).toBeGreaterThanOrEqual(0);
    expect(terminalIndex).toBeGreaterThan(uiKitIndex);
  });

  it("skips a source path that doesn't exist yet, without throwing", () => {
    const outputPath = join(dir, "CHANGELOG.md");
    expect(() =>
      aggregateChangelogs(
        [{ name: "missing", path: join(dir, "missing.md") }],
        outputPath,
      ),
    ).not.toThrow();
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `bun test tests/unit/changelog.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 4: Write `scripts/changelog.ts`**

```typescript
import { existsSync, mkdirSync, readFileSync, writeFileSync } from "node:fs";
import { dirname } from "node:path";

export interface ChangelogSource {
  name: string;
  path: string;
}

export function aggregateChangelogs(
  sources: ChangelogSource[],
  outputPath: string,
): void {
  const sections: string[] = [];

  for (const source of sources) {
    if (!existsSync(source.path)) continue;
    sections.push(readFileSync(source.path, "utf-8").trim());
  }

  mkdirSync(dirname(outputPath), { recursive: true });
  writeFileSync(outputPath, `${sections.join("\n\n---\n\n")}\n`);
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `bun test tests/unit/changelog.test.ts`
Expected: PASS — both tests green.

- [ ] **Step 6: Commit**

```bash
git add scripts/templates/appdata/ scripts/changelog.ts tests/unit/changelog.test.ts
git commit -m "feat: add changelog template and aggregator script"
```

---

### Task 6: `scripts/package.ts` — assemble `installDir/`

**Files:**

- Create: `tests/unit/package.test.ts`
- Create: `scripts/package.ts`
- Modify: `.gitignore` (add `installDir/`)

**Interfaces:**

- Consumes: `scripts/templates/appdata/configs/settings.toml` (Task 5), `.changeset/CHANGES/CHANGELOG.md` (produced by `scripts/changelog.ts`, Task 5, run beforehand), the 9 apps' Rust release binaries (Phase 3) and `packages/ui-kit/src/assets/{fonts,icons}/` (Phase 2)
- Produces: `assembleInstallDir(monorepoRoot: string, installDir: string): void` — the exact `installDir/` tree documented in `.cursor/rules/011-release-packaging-layout.mdc`

- [ ] **Step 1: Write the failing test against a fake monorepo tree**

```typescript
import { describe, expect, it, beforeEach, afterEach } from "bun:test";
import {
  mkdtempSync,
  mkdirSync,
  rmSync,
  writeFileSync,
  existsSync,
  readFileSync,
} from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { platform } from "node:os";
import { assembleInstallDir, APPS } from "../../scripts/package";

let root: string;
let installDir: string;

function exeName(name: string): string {
  return platform() === "win32" ? `${name}.exe` : name;
}

beforeEach(() => {
  root = mkdtempSync(join(tmpdir(), "slate-package-root-"));
  installDir = join(root, "installDir");

  for (const app of APPS) {
    const binDir = join(
      root,
      "apps",
      app.name,
      "src-tauri",
      "target",
      "release",
    );
    mkdirSync(binDir, { recursive: true });
    writeFileSync(
      join(binDir, exeName(app.name)),
      `fake binary for ${app.name}`,
    );
  }

  mkdirSync(join(root, "scripts", "templates", "appdata", "configs"), {
    recursive: true,
  });
  writeFileSync(
    join(root, "scripts", "templates", "appdata", "configs", "settings.toml"),
    '[general]\ntheme = "dark"\n',
  );

  writeFileSync(join(root, "README.md"), "# Slate");
  writeFileSync(join(root, "LICENSE"), "MIT");

  mkdirSync(join(root, "packages", "ui-kit", "src", "assets", "fonts"), {
    recursive: true,
  });
  mkdirSync(join(root, "packages", "ui-kit", "src", "assets", "icons"), {
    recursive: true,
  });
  writeFileSync(
    join(root, "packages", "ui-kit", "src", "assets", "fonts", "inter.woff2"),
    "",
  );
  writeFileSync(
    join(root, "packages", "ui-kit", "src", "assets", "icons", "logo.svg"),
    "<svg/>",
  );
});

afterEach(() => {
  rmSync(root, { recursive: true, force: true });
});

describe("assembleInstallDir", () => {
  it("packages slate-launcher at the top level, not under programs/", () => {
    assembleInstallDir(root, installDir);
    expect(
      existsSync(join(installDir, "launcher", exeName("slate-launcher"))),
    ).toBe(true);
    expect(existsSync(join(installDir, "programs", "slate-launcher"))).toBe(
      false,
    );
  });

  it("packages every other app under programs/ with its exact slate-* name", () => {
    assembleInstallDir(root, installDir);
    for (const name of [
      "slate-terminal",
      "slate-explorer",
      "slate-browser",
      "slate-editor",
      "slate-gallery",
      "slate-jukebox",
      "slate-player",
      "slate-aistudio",
    ]) {
      expect(
        existsSync(join(installDir, "programs", name, exeName(name))),
      ).toBe(true);
    }
  });

  it("creates the full appdata/ tree and copies docs + resources", () => {
    assembleInstallDir(root, installDir);
    expect(
      existsSync(join(installDir, "appdata", "configs", "settings.toml")),
    ).toBe(true);
    expect(existsSync(join(installDir, "appdata", "database"))).toBe(true);
    expect(existsSync(join(installDir, "appdata", "logs"))).toBe(true);
    expect(
      readFileSync(join(installDir, "appdata", "docs", "README.md"), "utf-8"),
    ).toBe("# Slate");
    expect(
      readFileSync(join(installDir, "appdata", "docs", "LICENSE.md"), "utf-8"),
    ).toBe("MIT");
    expect(
      existsSync(
        join(installDir, "appdata", "resources", "fonts", "inter.woff2"),
      ),
    ).toBe(true);
    expect(
      existsSync(join(installDir, "appdata", "resources", "icons", "logo.svg")),
    ).toBe(true);
  });

  it("creates an empty storage/ directory", () => {
    assembleInstallDir(root, installDir);
    expect(existsSync(join(installDir, "storage"))).toBe(true);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bun test tests/unit/package.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `scripts/package.ts`**

```typescript
import { copyFileSync, cpSync, existsSync, mkdirSync, rmSync } from "node:fs";
import { join } from "node:path";
import { platform } from "node:os";

export interface AppManifestEntry {
  name: string;
  kind: "launcher" | "program";
}

export const APPS: AppManifestEntry[] = [
  { name: "slate-launcher", kind: "launcher" },
  { name: "slate-terminal", kind: "program" },
  { name: "slate-explorer", kind: "program" },
  { name: "slate-browser", kind: "program" },
  { name: "slate-editor", kind: "program" },
  { name: "slate-gallery", kind: "program" },
  { name: "slate-jukebox", kind: "program" },
  { name: "slate-player", kind: "program" },
  { name: "slate-aistudio", kind: "program" },
];

function exeName(appName: string): string {
  return platform() === "win32" ? `${appName}.exe` : appName;
}

function packageApp(
  monorepoRoot: string,
  installDir: string,
  app: AppManifestEntry,
): void {
  const binaryPath = join(
    monorepoRoot,
    "apps",
    app.name,
    "src-tauri",
    "target",
    "release",
    exeName(app.name),
  );
  const destDir =
    app.kind === "launcher"
      ? join(installDir, "launcher")
      : join(installDir, "programs", app.name);

  mkdirSync(destDir, { recursive: true });
  copyFileSync(binaryPath, join(destDir, exeName(app.name)));
}

function assembleAppdata(monorepoRoot: string, installDir: string): void {
  const appdata = join(installDir, "appdata");
  mkdirSync(join(appdata, "configs"), { recursive: true });
  mkdirSync(join(appdata, "database"), { recursive: true });
  mkdirSync(join(appdata, "logs"), { recursive: true });
  mkdirSync(join(appdata, "docs"), { recursive: true });
  mkdirSync(join(appdata, "resources", "fonts"), { recursive: true });
  mkdirSync(join(appdata, "resources", "icons"), { recursive: true });

  copyFileSync(
    join(
      monorepoRoot,
      "scripts",
      "templates",
      "appdata",
      "configs",
      "settings.toml",
    ),
    join(appdata, "configs", "settings.toml"),
  );
  copyFileSync(
    join(monorepoRoot, "README.md"),
    join(appdata, "docs", "README.md"),
  );
  copyFileSync(
    join(monorepoRoot, "LICENSE"),
    join(appdata, "docs", "LICENSE.md"),
  );

  const changelogPath = join(
    monorepoRoot,
    ".changeset",
    "CHANGES",
    "CHANGELOG.md",
  );
  if (existsSync(changelogPath)) {
    copyFileSync(changelogPath, join(appdata, "docs", "CHANGELOG.md"));
  }

  cpSync(
    join(monorepoRoot, "packages", "ui-kit", "src", "assets", "fonts"),
    join(appdata, "resources", "fonts"),
    { recursive: true },
  );
  cpSync(
    join(monorepoRoot, "packages", "ui-kit", "src", "assets", "icons"),
    join(appdata, "resources", "icons"),
    { recursive: true },
  );
}

export function assembleInstallDir(
  monorepoRoot: string,
  installDir: string,
): void {
  rmSync(installDir, { recursive: true, force: true });
  mkdirSync(installDir, { recursive: true });

  for (const app of APPS) {
    packageApp(monorepoRoot, installDir, app);
  }

  assembleAppdata(monorepoRoot, installDir);
  mkdirSync(join(installDir, "storage"), { recursive: true });
}

if (import.meta.main) {
  const monorepoRoot = join(import.meta.dir, "..");
  assembleInstallDir(monorepoRoot, join(monorepoRoot, "installDir"));
  console.log("Packaged to installDir/");
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `bun test tests/unit/package.test.ts`
Expected: PASS — all four tests green.

- [ ] **Step 5: Add `installDir/` to `.gitignore`**

```
installDir/
```

- [ ] **Step 6: Commit**

```bash
git add scripts/package.ts tests/unit/package.test.ts .gitignore
git commit -m "feat: add installDir/ packaging script"
```

---

### Task 7: Wire the root `moon run :package` task

**Files:**

- Modify: `.moon/workspace.yml` (register `.` as a moon project)
- Create: `moon.yml` (root — this time a real, valid per-project moon v2 file, unlike the stray one Phase 1 deleted)

**Interfaces:**

- Consumes: `scripts/package.ts` (Task 6)
- Produces: the `moon run :package` task `.cursor/rules/011-release-packaging-layout.mdc` and the original architecture plan both call for

- [ ] **Step 1: Register the repo root as a moon project**

```yaml
$schema: "https://moonrepo.dev/schemas/workspace.json"
projects:
  - "apps/*"
  - "packages/*"
  - "."
vcs:
  manager: "git"
  defaultBranch: "main"
```

- [ ] **Step 2: Write the root `moon.yml`**

```yaml
$schema: "https://moonrepo.dev/schemas/project.json"
type: "tool"
tasks:
  package:
    command: "bun scripts/package.ts"
    deps: ["changelog"]
  changelog:
    command: "bun scripts/changelog.ts"
```

- [ ] **Step 3: Verify moon picks up the new root project and task**

Run: `moon project-graph`
Expected: shows a project for `.` (root) alongside every app/package.

Run: `moon run :package`
Expected: exit 0 — runs `changelog` then `package`, producing `installDir/` at the repo root (using whatever release binaries currently exist — see Task 8 for the first real end-to-end run).

- [ ] **Step 4: Commit**

```bash
git add .moon/workspace.yml moon.yml
git commit -m "feat: wire moon run :package task"
```

---

### Task 8: Full-repository verification pass

**Files:** none created — verification only

**Interfaces:**

- Consumes: every file created across all 4 phases
- Produces: proof the entire scaffold — governance, tooling, shared packages, all 9 apps, and release packaging — works end to end

- [ ] **Step 1: Full install**

Run: `bun install`
Expected: exit 0.

- [ ] **Step 2: Lint**

Run: `bunx biome check .`
Expected: exit 0.

- [ ] **Step 3: TypeScript across the entire graph**

Run: `bunx tsc -b`
Expected: exit 0 — all 3 packages + 9 apps.

- [ ] **Step 4: Full test suite**

Run: `bunx vitest run`
Expected: exit 0 — every test file from all 4 phases passes.

- [ ] **Step 5: Rust across the entire workspace**

Run: `cargo build --workspace && cargo clippy --workspace -- -D warnings && cargo test --workspace`
Expected: all exit 0.

- [ ] **Step 6: Toolchain drift check**

Run: `bun scripts/check-toolchain-versions.ts`
Expected: "Toolchain versions in sync."

- [ ] **Step 7: moon across the entire graph**

Run: `moon run :lint :typecheck :test :build`
Expected: exit 0 across every project, including the root `.` project from Task 7.

- [ ] **Step 8: One real Tauri build, to produce a real binary for packaging**

Run: `cd apps/slate-terminal && bun run tauri build && cd ../..`
Expected: exit 0, produces `apps/slate-terminal/src-tauri/target/release/slate-terminal(.exe)`.

- [ ] **Step 9: Run the real packaging script against that one real build**

Run: `bun scripts/changelog.ts && bun scripts/package.ts`
Expected: exit 0. `installDir/programs/slate-terminal/` contains the real binary; the other 7 program dirs and `installDir/launcher/` are created but empty (their binaries weren't built in this pass — acceptable for this verification, since Step 8 only built one app on purpose to keep the check fast). `installDir/appdata/` and `installDir/storage/` match `.cursor/rules/011-release-packaging-layout.mdc` exactly.

- [ ] **Step 10: Record a final changeset and dry-run version**

Run: `bunx changeset` — summarize the whole scaffold as `minor` bumps across every workspace.
Run: `bunx changeset version --snapshot dry-run` (or inspect `bunx changeset status` output) to confirm the tooling resolves versions across apps, packages, and crates without error.

- [ ] **Step 11: Update this feature's status files**

Check off completed tasks in `.superpowers/docs/tasks/vicore-slate-scaffold.md` (all 54 boxes `[x]` once verification passes) and refresh the full status header (Current phase / Doing now / Model (doing now) / Next / Blocked on) to reflect all 4 phases verified. Do not flip step checkboxes inside phase files. Append a final entry to `.superpowers/docs/progress/vicore-slate-scaffold.md` summarizing that the full scaffold is built and verified end to end.

- [ ] **Step 12: Final commit**

```bash
git add .changeset/ -A
git commit -m "chore: phase 4 verification — full scaffold built and verified end to end"
```

All 4 phases are complete. The repository now has a working `.cursor/` governance layer, root tooling, `@slate/ui-kit` design system, `slate-core` stub, all 9 template apps (with the Isolation Pattern, granular capabilities, and `slate-launcher`'s tray extras), and a release-packaging pipeline that produces a real `installDir/` — ready for real feature work to begin on top of this foundation.
