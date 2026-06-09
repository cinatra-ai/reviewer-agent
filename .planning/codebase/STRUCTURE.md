# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
reviewer-agent/
├── cinatra/
│   └── oas.json              # Cinatra Flow spec — node graph, edges, I/O contract
├── skills/
│   └── reviewer/
│       └── SKILL.md          # LLM skill prompt for the review ApiNode
├── .github/
│   └── workflows/
│       ├── ci.yml            # Standalone CI: build, typecheck, test, kind-gate
│       └── release.yml       # Release workflow
├── extension-kind-gate.mjs   # Self-contained CI gate for agent/workflow validation
├── package.json              # Package manifest with cinatra.kind = "agent"
├── tsconfig.json             # TypeScript config (targets src/ — no src/ exists today)
├── .npmrc                    # npm registry config
├── LICENSE                   # Apache-2.0
└── README.md                 # Usage and integration documentation
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Cinatra platform artefacts — the agent's machine-readable flow spec
- Contains: `oas.json` — the full Flow definition (nodes, edges, inputs, outputs, metadata)
- Key files: `cinatra/oas.json`

**`skills/reviewer/`:**
- Purpose: LLM skill definitions consumed by the flow's ApiNode
- Contains: `SKILL.md` — step-by-step LLM instructions, renderer vocabulary, output JSON schema
- Key files: `skills/reviewer/SKILL.md`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD pipelines
- Contains: `ci.yml` (build + kind-gate), `release.yml` (release automation)
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

## Key File Locations

**Entry Points:**
- `cinatra/oas.json`: Flow definition — `start_node` references the `start` StartNode; this is the runtime entry point
- `extension-kind-gate.mjs`: CI gate entry — `main()` runs when invoked as `node extension-kind-gate.mjs`

**Configuration:**
- `package.json`: Package identity (`@cinatra-ai/reviewer-agent`), `cinatra.kind = "agent"`, `cinatra.apiVersion = "cinatra.ai/v1"`
- `tsconfig.json`: TypeScript compiler config (targets a future `src/` directory; not active today as no `src/` exists)
- `.npmrc`: npm/pnpm registry settings

**Core Logic:**
- `cinatra/oas.json`: All flow behaviour — nodes `start`, `review`, `approval_gate`, `end`; control and data flow edges
- `skills/reviewer/SKILL.md`: All LLM reasoning rules — renderer selection, summary format, what NOT to do

**Testing:**
- Not applicable — no test files present. CI runs `pnpm test --if-present` (skipped). This repo is a source mirror; the Cinatra monorepo owns test execution.

## Naming Conventions

**Files:**
- Cinatra artefacts: lowercase, no hyphens — `oas.json`, `workflow.bpmn` (if workflow kind)
- Skill prompts: `SKILL.md` (uppercase, matches Cinatra skill loader convention)
- CI gate: `extension-kind-gate.mjs` (kebab-case, `.mjs` for ESM)
- Config files: standard ecosystem names (`package.json`, `tsconfig.json`, `.npmrc`)

**Directories:**
- Platform artefacts: `cinatra/` (reserved Cinatra namespace)
- Skill definitions: `skills/<skill-name>/` (kebab-case skill slug)
- GitHub Actions: `.github/workflows/` (standard)

**Node IDs in OAS:**
- Lowercase with underscores: `start`, `review`, `approval_gate`, `end`
- Edge names: `<from>_to_<to>_<field>` e.g. `start_to_review_draftBundleRef`

**Package naming:**
- Agent packages: `@<scope>/<slug>-agent` — e.g. `@cinatra-ai/reviewer-agent`
- Workflow packages: `@<scope>/<slug>-workflow` (enforced by `extension-kind-gate.mjs` regex)

## Where to Add New Code

**New renderer type:**
- Update `skills/reviewer/SKILL.md` — add to the "Available frontend renderers" closed vocabulary list
- Update `cinatra/oas.json` if the `approval_gate` metadata needs new renderer registration

**New flow input:**
- Add input to `cinatra/oas.json` → `$referenced_components.start.inputs` and to the top-level `inputs` array
- Add DataFlowEdge from `start` to the consuming node in `data_flow_connections`
- Add input to the consuming node's `inputs` array

**New flow node:**
- Add component definition to `cinatra/oas.json` → `$referenced_components`
- Add `$component_ref` entry to `nodes` array
- Wire ControlFlowEdges in `control_flow_connections`
- Wire DataFlowEdges in `data_flow_connections`

**New CI validation rule:**
- For banned primitive rules: add to `BANNED_PRIMITIVES` or `BANNED_TYPEHINTS` arrays in `extension-kind-gate.mjs`
- For structural rules: extend `validateAgent` or `validateWorkflow` functions in `extension-kind-gate.mjs`

**TypeScript source (future):**
- Create `src/` directory (tsconfig.json already targets `src/**/*.ts`, `src/**/*.tsx`)
- Output goes to `dist/` (configured in `tsconfig.json`)

## Special Directories

**`cinatra/`:**
- Purpose: Reserved namespace for Cinatra platform artefacts (`oas.json` for agents, `workflow.bpmn` for workflows)
- Generated: No (hand-authored / extraction-script-generated, then maintained)
- Committed: Yes

**`.planning/codebase/`:**
- Purpose: GSD codebase mapper output — architecture and structure analysis documents
- Generated: Yes (by GSD map-codebase tooling)
- Committed: Optional (project-specific)

---

*Structure analysis: 2026-06-09*
