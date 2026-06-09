# Coding Conventions

**Analysis Date:** 2026-06-09

## Overview

This is a content-only Cinatra agent extension. The primary deliverable is a skill prompt (`skills/reviewer/SKILL.md`) and an OAS manifest (`cinatra/oas.json`). The only JavaScript source file is the self-contained CI gate script `extension-kind-gate.mjs`. There is no `src/` directory and no application TypeScript to typecheck.

## Naming Patterns

**Files:**
- Skill prompts: `SKILL.md` (uppercase), located under `skills/<skill-name>/SKILL.md`
- Gate script: kebab-case `.mjs` (`extension-kind-gate.mjs`)
- Cinatra manifests: lowercase under `cinatra/` (`oas.json`)
- GitHub Actions: lowercase kebab-case YAML (`ci.yml`, `release.yml`)

**Functions (in `extension-kind-gate.mjs`):**
- Exported functions: camelCase verbs — `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `parseArgs`
- Internal helpers: camelCase — `walkLlmStrings`, `scanOasString`, `wordBoundary`
- Main entry: conventional `main()` function, called only when invoked directly

**Variables:**
- Constants: SCREAMING_SNAKE_CASE for module-level config sets — `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `LLM_VISIBLE_FIELDS`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`, `OBJECTS_LIST_CRM_RE`
- Local variables: camelCase — `packageRoot`, `findings`, `openTags`, `bpmnPrefixes`

**Types/Parameters:**
- No TypeScript in this repo's own sources (content-only extension). `tsconfig.json` is present as a baseline for `src/` if TypeScript is ever added, but no `.ts` files exist.

## Code Style

**Formatting:**
- Not detected (no `.prettierrc`, `.eslintrc`, or `biome.json` present)
- Style inferred from `extension-kind-gate.mjs`: 2-space indentation, double-quoted strings, trailing commas in multi-line arrays and objects

**Linting:**
- Not detected — no ESLint or Biome config

**Module system:**
- ES Modules (`"type": "module"` in `package.json`)
- Node built-ins imported via `node:` prefix — `import { readFileSync, existsSync, readdirSync } from "node:fs"`
- No third-party dependencies; zero-dependency is a hard constraint for this gate script

## Import Organization

**Order (observed in `extension-kind-gate.mjs`):**
1. Node built-in modules with `node:` protocol prefix (`node:fs`, `node:path`)
2. No third-party or internal imports (zero-dependency constraint)

**Path Aliases:**
- Not applicable — no module bundler or path mapping configured

## Error Handling

**Patterns:**
- Functions are pure: they return `string[]` error arrays rather than throwing — e.g., `validateAgent`, `validateWorkflow`, `validateBpmnSanity` all return `errors[]`
- `try/catch` wraps I/O operations (`readFileSync`, `readdirSync`); errors are pushed into the errors array and returned, not rethrown
- Early-return pattern: if a critical precondition fails (e.g., `oas.json` parse fails), populate errors and `return errors` immediately
- `main()` catches unexpected errors with a top-level `try/catch` and exits with code 1

**Exit codes:**
- `process.exit(0)` on success
- `process.exit(1)` on validation failures
- `exit 2` in CI shell scripts for first-party dependency-shape regressions

## Logging

**Framework:** `console.log` / `console.error` (no logging library)

**Patterns:**
- Success: `console.log(...)` with a `✓` prefix to stdout
- Failure: `console.error(...)` with a `✗` prefix and bullet `•` list of violations to stderr
- No structured logging

## Comments

**When to Comment:**
- Block comments at the top of each logical section delimited by dashed separators (`// ---`)
- Inline rationale for non-obvious decisions, especially zero-dependency and CI skip logic
- JSDoc-style `/** ... */` doc comments on exported functions (`validateAgent`, `validateBpmnSanity`, `findWorkflowSidecars`)

**Style:**
- Section headers use `// ----------- label -----------` ASCII borders
- Prose comments explain WHY (registry constraints, monorepo contract, Profile-1.0 authority), not just what

## Function Design

**Size:** Functions are focused and single-purpose; longest is `validateBpmnSanity` (~80 lines) which performs a complete XML sanity check

**Parameters:** Minimal — most functions take a single `packageRoot: string` or a parsed `pkg` object

**Return Values:** Pure functions return `string[]` (error list); `runGate` returns `{ kind, errors }`; `main()` has no return value (side-effectful)

## Module Design

**Exports:** Named exports only — all validation functions and `parseArgs`, `runGate` are exported. `main()` is NOT exported (entry-point guard pattern)

**Entry-point guard:**
```js
const invokedDirectly =
  process.argv[1] && resolve(process.argv[1]) === resolve(new URL(import.meta.url).pathname);
if (invokedDirectly) { main(); }
```
This allows the file to be both `node`-executed and imported in tests.

**Barrel Files:** Not applicable

## Skill Prompt Conventions (`skills/reviewer/SKILL.md`)

- Role declared at the top with a one-line description
- Inputs documented explicitly with bullet lists
- Steps numbered sequentially (`STEP 1`, `STEP 2`, …)
- Closed-vocabulary enum values listed explicitly in backtick code
- Output schema shown as a fenced JSON block
- Constraints (what NOT to do) stated as explicit steps (`STEP 4 — DO NOT modify the bundle`)
- MCP tool usage declared in a dedicated `## What I retrieve myself (MCP)` section

## TypeScript Config (baseline for future `src/`)

- Target: `ES2023`, module: `ESNext`, moduleResolution: `bundler`
- `strict: true` with `noImplicitAny: false`
- `verbatimModuleSyntax: true`
- `isolatedModules: true`
- Output: `dist/`, source: `src/`
- Config: `tsconfig.json`

---

*Convention analysis: 2026-06-09*
