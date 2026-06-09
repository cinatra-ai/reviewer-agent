# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- JavaScript (ESM) — `extension-kind-gate.mjs` (zero-dependency CI gate script, Node builtins only)
- JSON — `cinatra/oas.json` (agent flow definition), `package.json`
- Markdown — `skills/reviewer/SKILL.md` (LLM skill prompt)

**Secondary:**
- TypeScript — declared target via `tsconfig.json`; no `.ts`/`.tsx` source files are present in the repo (content-only extension). TypeScript config exists to support future source additions or monorepo compilation.

## Runtime

**Environment:**
- Node.js 24 (pinned in `.github/workflows/ci.yml` via `actions/setup-node@v4`)

**Package Manager:**
- pnpm (via Corepack — `corepack enable` called in CI)
- No lockfile committed (CI runs `pnpm install --no-frozen-lockfile` for standalone repos)
- `.npmrc` present — existence noted, contents not read

## Frameworks

**Core:**
- None — this is a content-only Cinatra extension. No application framework is used.

**Testing:**
- Not applicable — no test files present; CI runs `pnpm test --if-present` which is a no-op.

**Build/Dev:**
- `extension-kind-gate.mjs` — self-contained CI validation script (plain Node, no dependencies)

## Key Dependencies

**Critical:**
- None declared in `package.json` (`dependencies`, `devDependencies`, `optionalDependencies` are all absent). The package declares zero runtime or build dependencies.

**Infrastructure:**
- `@cinatra-ai/*` packages — consumed as optional peer dependencies provided by the cinatra monorepo workspace. They are NOT published to any public registry and are not installable standalone.

## Configuration

**Environment:**
- `CINATRA_BASE_URL` — runtime template variable used inside `cinatra/oas.json` as the LLM-bridge endpoint (`{{CINATRA_BASE_URL}}/api/llm-bridge`)
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — org-level GitHub secret used by the release workflow for marketplace submission (never read directly)

**Build:**
- `tsconfig.json` — TypeScript compiler config targeting ES2023/ESNext, `bundler` module resolution, `outDir: dist`, `rootDir: src`. No TypeScript sources currently exist.
- `package.json` — declares `cinatra.apiVersion: "cinatra.ai/v1"` and `cinatra.kind: "agent"`

## Platform Requirements

**Development:**
- Node.js 24+
- pnpm (via Corepack)
- Monorepo workspace (cinatra monorepo) for typecheck, test, and resolution of `@cinatra-ai/*` peers

**Production:**
- Cinatra Marketplace / Cinatra runtime platform
- Published as `@cinatra-ai/reviewer-agent` via `registry.cinatra.ai` (marketplace promotion saga, not direct Verdaccio publish)
- GitHub Actions for CI (`ci.yml`) and release (`release.yml` → reusable workflow at `cinatra-ai/.github`)

---

*Stack analysis: 2026-06-09*
