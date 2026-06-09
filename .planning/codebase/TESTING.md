# Testing Patterns

**Analysis Date:** 2026-06-09

## Overview

This is a content-only Cinatra agent extension. There are no test files (`*.test.*`, `*.spec.*`) in the repository and no test runner is configured. The `package.json` declares no `test` script and no test-framework dependency. The CI pipeline reflects this: it runs `pnpm test --if-present`, which is a no-op when no test script is present.

The only executable logic in the repo is `extension-kind-gate.mjs`, which exports pure validation functions suitable for unit testing but has no tests written yet.

## Test Framework

**Runner:** Not configured

**Assertion Library:** Not configured

**Run Commands:**
```bash
# CI equivalent (no-op — no test script defined):
pnpm test --if-present
```

## Test File Organization

**Location:** Not applicable — no test files exist

**Naming:** Not applicable

## Test Structure

Not applicable — no tests exist.

## Mocking

Not applicable — no test framework configured.

**Note for future tests:** All exported functions in `extension-kind-gate.mjs` are pure (string/object in → string[] out) and perform filesystem I/O only at the top call (`readFileSync`, `existsSync`). The recommended mocking approach when tests are added:
- Pass pre-parsed content to inner validators (`validateBpmnSanity(xml)`, `validateWorkflowPackageShape(pkg)`) — these require no mocking at all
- Use `vi.mock("node:fs")` or `jest.mock("node:fs")` for `validateAgent` and `validateWorkflow` which call `readFileSync`/`existsSync` internally

## Fixtures and Factories

**Test Data:** Not applicable

## Coverage

**Requirements:** None enforced

**View Coverage:** Not configured

## Test Types

**Unit Tests:** Not present. The exported gate functions (`validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `validateWorkflowPackageShape`, `findWorkflowSidecars`, `runGate`, `parseArgs`) are all pure or near-pure and are the natural targets for unit tests.

**Integration Tests:** Not present

**E2E Tests:** Not present

## CI Validation (Substitute for Tests)

The CI pipeline in `.github/workflows/ci.yml` provides structural validation that serves as the primary quality gate:

1. **First-party dependency shape check** — inline `node -e` script validates that no `@cinatra-ai/*` packages appear in `dependencies`/`devDependencies`/`optionalDependencies`; enforces optional peer declaration.

2. **Agent OAS gate** — `.github/workflows/ci.yml` runs `node extension-kind-gate.mjs --package-root .` in the `kind-gates` job, which:
   - Parses `cinatra/oas.json`
   - Scans all LLM-visible fields (`system`, `user`, `description`) for retired CRM primitives (`contacts_list`, `accounts_get`, etc.)
   - Scans for banned type hints (`@cinatra-ai/entity-accounts:account`, `@cinatra-ai/entity-contacts:contact`)
   - Returns violations as a non-zero exit code

3. **Pack dry-run** — `npm pack --dry-run` validates publishable package shape without resolving peers.

These CI steps are the current substitute for an automated test suite.

## Gaps and Recommendations

- `extension-kind-gate.mjs` has no tests despite containing non-trivial logic (XML tag-balance walker, namespace resolver, regex-based primitive scanner). Adding a test suite (e.g., Vitest) with fixtures for valid/invalid BPMN XML and OAS payloads would significantly increase confidence.
- No `test` script in `package.json` — adding one is a prerequisite for the CI `pnpm test --if-present` step to become meaningful.
- The `tsconfig.json` targets a `src/` directory that does not exist; if TypeScript sources are added, tests should live alongside them or under a top-level `test/` directory.

---

*Testing analysis: 2026-06-09*
