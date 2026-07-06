# DX Baseline

Developer experience and code quality tooling every project must include. Keeps the codebase clean and consistent automatically — no manual enforcement needed. Referenced by `/build-spec` when `dx` is in the active baselines list.

**When this applies:**
- First phase of a project → SCAFFOLD all items
- Subsequent phases → INHERIT; no re-scaffold
- Applies to all projects with a codebase (excludes pure static HTML with no build step)

**Stack-agnostic rule:** Tooling choices below are defaults for JS/TS projects. For other stacks, Claude resolves the equivalent: Python → Ruff (lint+format), Black; Go → gofmt, golangci-lint; Rust → clippy, rustfmt. The requirements (lint, format, pre-commit, type safety) are universal; the tools are stack-specific.

---

## Tier 1 — Auto-Scaffold

### Linting and formatting (JS/TS projects)

**Biome 1.9.x** (replaces ESLint + Prettier in a single tool, 35x faster):

Install with pinned version: `npm install --save-dev --save-exact @biomejs/biome@1.9.4`

The config below targets the 1.9 API. If the project's Biome version differs, verify the schema URL and config shape match the installed version before using.

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "a11y": { "recommended": true },
      "security": { "recommended": true }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "trailingCommas": "es5"
    }
  }
}
```

Fallback: if project already has an established ESLint config that must be preserved, use ESLint + Prettier instead. Never both Biome and ESLint simultaneously.

### TypeScript configuration (TypeScript projects only)

Strict mode is the only acceptable default. Permissive TypeScript is worse than no TypeScript — it gives a false sense of type safety.

```json
// tsconfig.json (key settings — fill remaining from framework template)
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": false,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

Note: `exactOptionalPropertyTypes: true` is valid but breaks many libraries — leave off by default and enable deliberately.

### Package scripts

Every project must have these scripts in `package.json` (or stack equivalent). Names are the convention — the commands behind them vary by stack:

```json
{
  "scripts": {
    "dev": "[start dev server]",
    "build": "[production build]",
    "start": "[start production server]",
    "lint": "biome lint .",
    "lint:fix": "biome lint --fix .",
    "format": "biome format --write .",
    "format:check": "biome format .",
    "typecheck": "tsc --noEmit",
    "test": "[test runner]",
    "test:watch": "[test runner --watch]"
  }
}
```

### Node version pinning (JS/TS projects)

`.nvmrc` in project root pinning the Node LTS version used. Also add `engines` field to `package.json`:

```json
{
  "engines": {
    "node": ">=20.0.0"
  }
}
```

Prevents "works on my machine" failures when the AI agent or a collaborator uses a different Node version.

### Pre-commit hook (Husky + lint-staged)

Runs lint + format on staged files only before every commit. Fast (only touches changed files). Blocks commit on lint errors — if it doesn't block, it doesn't protect.

```json
// package.json additions
{
  "lint-staged": {
    "*.{js,ts,jsx,tsx,json}": ["biome lint --no-errors-on-unmatched", "biome format --write"],
    "*.{css,md}": ["biome format --write"]
  }
}
```

```sh
# .husky/pre-commit
npx lint-staged
```

Install: `npx husky init && npx husky add .husky/pre-commit "npx lint-staged"`

**Important:** pre-commit hooks are bypassed by `--no-verify`. The CI lint step (see below) is the enforceable gate. Pre-commit is for developer feedback speed, not security.

### CI lint gate (GitHub Actions)

Lint and type-check must run on every PR and block merge on failure. Developer tools are useless if they only run locally.

**The template below is for JS/TS projects.** For other stacks, substitute the appropriate setup action and commands:
- Python: `actions/setup-python` + `pip install -r requirements.txt` + `ruff check .` + `mypy .`
- Go: `actions/setup-go` + `golangci-lint run`
- Rust: `cargo clippy` + `cargo fmt --check`

**Prerequisite:** `package-lock.json` (or `yarn.lock` / `bun.lockb`) must be committed before CI runs. `npm ci` hard-fails without a lockfile in sync. Run `npm install` and commit the lockfile as part of the initial scaffold.

```yaml
# .github/workflows/ci.yml  (JS/TS projects)
name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck   # omit if JS (not TS) project
      - run: npm test -- --passWithNoTests
      - run: npm audit --audit-level=high   # blocks on known HIGH/CRITICAL CVEs
```

---

## Tier 2 — Ask Once

None. DX tooling is invisible plumbing (no felt UX or performance difference) — Claude chooses based on the stack. Not surfaced to the user.

---

## Tier 3 — DX Validation Block (injected into `validation.md`)

```markdown
## DX Baseline Checks

### D1 — Tooling presence [static]
- [ ] Linter config exists: `biome.json` OR `.eslintrc.*` (not both)
- [ ] Formatter configured (Biome handles both; or `.prettierrc` if ESLint path)
- [ ] `package.json` has: `lint`, `format:check`, `typecheck` (if TS) scripts

### D2 — Clean baseline [static]
- [ ] `npm run lint` exits 0 on current codebase (no suppressed errors, no lint-disable blanket comments)
- [ ] `npm run typecheck` exits 0 (if TypeScript project)

### D3 — Pre-commit hook [static]
- [ ] `.husky/pre-commit` exists and calls `lint-staged`
- [ ] `lint-staged` config covers `.ts`, `.tsx`, `.js`, `.jsx` files

### D4 — CI gate [static]
- [ ] `.github/workflows/ci.yml` exists and runs `lint` + `typecheck` + `test` on PR
```

---

## What this does NOT cover

- Secret scanning (CI security step) → `production-safety-baseline.md`
- Test strategy and coverage targets → covered by `/build-backend` validation scripts
- Deployment pipeline → platform-handled or post-PMF
- Code review automation → `repo-baseline.md` PR templates handle the human layer
