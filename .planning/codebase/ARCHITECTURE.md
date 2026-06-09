<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌──────────────────────────────────────────────────────────────┐
│                  Cinatra Flow Runtime                        │
│         (external orchestration platform)                    │
└────────┬──────────────────────────────────┬─────────────────┘
         │ draftBundleRef / followupBundleRef│
         ▼                                  │
┌────────────────────┐                      │
│   StartNode        │                      │
│   (Inputs)         │                      │
│  `cinatra/oas.json`│                      │
└────────┬───────────┘                      │
         │ DataFlowEdge                     │
         ▼                                  │
┌────────────────────────────────────────┐  │
│   ApiNode: "review"                    │  │
│   POST {{CINATRA_BASE_URL}}/api/llm-bridge│
│   LLM: openai / gpt-5.5               │  │
│   Skill prompt: `skills/reviewer/SKILL.md`
│   → fetches bundles via objects_get    │  │
│   → saves approved via objects_save    │  │
│   → returns approvedDraftBundleRef +   │  │
│     approvedFollowupBundleRef          │  │
└────────┬───────────────────────────────┘  │
         │ DataFlowEdge                     │
         ▼                                  │
┌────────────────────────────────────────┐  │
│   InputMessageNode: "approval_gate"    │  │
│   (Human-in-the-loop HITL interrupt)   │  │
│   renderer: @cinatra-ai/reviewer-agent:output
│   surface: reviewer:approval-gate:input│  │
│   riskClass: approval / requiresApproval: true
└────────┬───────────────────────────────┘  │
         │ ControlFlowEdge + DataFlowEdge   │
         ▼                                  │
┌────────────────────────────────────────┐  │
│   EndNode                              │  │
│   outputs: approvedDraftBundleRef,     │  │
│            approvedFollowupBundleRef,  │  │
│            userResponse               │  │
└────────────────────────────────────────┘  │
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| StartNode (Inputs) | Receives `agent_run_id`, `draftBundleRef`, `followupBundleRef` from runtime | `cinatra/oas.json` → `$referenced_components.start` |
| ApiNode (review) | Calls LLM bridge; LLM fetches bundles, applies quality review, saves approved bundles, returns refs | `cinatra/oas.json` → `$referenced_components.review` |
| Skill prompt | Defines LLM instructions: layout-advice vocabulary, step-by-step rules, NO bundle mutation | `skills/reviewer/SKILL.md` |
| InputMessageNode (approval_gate) | HITL pause — presents approved bundles to human reviewer via frontend renderer; emits `userResponse` | `cinatra/oas.json` → `$referenced_components.approval_gate` |
| EndNode | Collects and surfaces final outputs to parent flow | `cinatra/oas.json` → `$referenced_components.end` |
| extension-kind-gate | CI sanity gate: scans `cinatra/oas.json` for retired CRM primitives in LLM-visible strings | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Declarative Cinatra Flow — a JSON-defined node graph with control-flow and data-flow edges executed by the Cinatra platform runtime. The agent itself contains no imperative application code; all logic is expressed as a flow spec (`cinatra/oas.json`) and a skill prompt (`skills/reviewer/SKILL.md`).

**Key Characteristics:**
- Data-only configuration: the repo ships no `src/` TypeScript; behaviour is declared in JSON + Markdown
- Human-in-the-loop: the flow pauses at `approval_gate` (an `InputMessageNode`) before surfacing outputs
- LLM-mediated review: the `review` ApiNode delegates quality assessment entirely to an LLM (gpt-5.5) via the Cinatra LLM bridge
- Closed renderer vocabulary: the skill enforces exactly four renderer values (`email-drafts`, `email-followups`, `contacts-list`, `unknown`)

## Layers

**Flow Spec Layer:**
- Purpose: Declares the execution graph — nodes, control-flow edges, data-flow edges, and node metadata
- Location: `cinatra/oas.json`
- Contains: StartNode, ApiNode, InputMessageNode, EndNode, DataFlowEdge, ControlFlowEdge definitions
- Depends on: Cinatra platform runtime to execute
- Used by: Cinatra marketplace / Flow runtime

**Skill Prompt Layer:**
- Purpose: LLM instruction set for the `review` ApiNode; defines layout-advice rules, step sequence, output contract
- Location: `skills/reviewer/SKILL.md`
- Contains: Input definitions, renderer vocabulary, step-by-step instructions, JSON output schema
- Depends on: LLM bridge injecting `contentBundle` as inline input
- Used by: ApiNode system/user prompt fields (injected by the LLM bridge at runtime)

**CI Gate Layer:**
- Purpose: Pre-publish sanity check — validates `cinatra/oas.json` for retired CRM primitive usage in LLM-visible strings
- Location: `extension-kind-gate.mjs`
- Contains: `validateAgent`, `runGate`, `scanOasString`, `walkLlmStrings`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`
- Depends on: Node.js builtins only (zero external dependencies, zero `@cinatra-ai/*` deps)
- Used by: `.github/workflows/ci.yml` → `kind-gates` job

## Data Flow

### Primary Request Path

1. **Flow runtime invokes StartNode** — injects `agent_run_id`, `draftBundleRef` (UUID), `followupBundleRef` (UUID) (`cinatra/oas.json` → `start`)
2. **DataFlowEdges carry inputs to ApiNode** — three edges forward all three fields to `review` node (`cinatra/oas.json` → `data_flow_connections`)
3. **ApiNode POSTs to LLM bridge** — `{{CINATRA_BASE_URL}}/api/llm-bridge`; LLM fetches bundles via `objects_get`, reviews, saves via `objects_save`, returns `approvedDraftBundleRef` + `approvedFollowupBundleRef` (`cinatra/oas.json` → `review.data`)
4. **DataFlowEdges carry approved refs to EndNode** — `review_to_end_approvedDraftBundleRef`, `review_to_end_approvedFollowupBundleRef` (`cinatra/oas.json` → `data_flow_connections`)
5. **ControlFlowEdge advances to approval_gate** — human reviewer sees the approved bundles via the registered renderer (`approval_gate.metadata.cinatra.renderer`)
6. **InputMessageNode emits userResponse** — `approval_gate_to_end_userResponse` DataFlowEdge carries it to EndNode
7. **EndNode surfaces all three outputs** — `approvedDraftBundleRef`, `approvedFollowupBundleRef`, `userResponse`

### CI Validation Path

1. `ci.yml` `kind-gates` job runs after `build`
2. `node extension-kind-gate.mjs --package-root .` invoked
3. `runGate` reads `package.json`, detects `cinatra.kind = "agent"`, calls `validateAgent`
4. `validateAgent` parses `cinatra/oas.json`, walks LLM-visible fields (`system`, `user`, `description`) via `walkLlmStrings`
5. `scanOasString` checks each string against `BANNED_PRIMITIVES` + `BANNED_TYPEHINTS` + `OBJECTS_LIST_CRM_RE`
6. Exit 0 (clean) or exit 1 (violations printed)

**State Management:**
- No local state. Bundle persistence is the upstream leaf agent's responsibility (via `objects_save`/`objects_get` MCP primitives). The reviewer flow is stateless between invocations.

## Key Abstractions

**contentBundle:**
- Purpose: Opaque payload from an upstream leaf agent containing reviewable content (email drafts, follow-up sequences, or contact rows)
- Examples: Delivered inline to the LLM; shape is intentionally unspecified
- Pattern: Polymorphic input — the LLM infers renderer from bundle shape

**renderer vocabulary:**
- Purpose: Closed enum of four frontend layout identifiers the skill may emit
- Examples: `email-drafts`, `email-followups`, `contacts-list`, `unknown`
- Pattern: Enforced by SKILL.md prose rules; the `approval_gate` `InputMessageNode` reads `contentType` via DataFlowEdge

**BundleRef (UUID):**
- Purpose: Opaque object store reference (UUID format) pointing to a content bundle stored in the Cinatra object store
- Examples: `draftBundleRef`, `followupBundleRef`, `approvedDraftBundleRef`, `approvedFollowupBundleRef`
- Pattern: All inter-node bundle passing uses UUID refs, never inline bundle data

**extension-kind-gate:**
- Purpose: Self-contained, zero-dependency CI validator for extracted Cinatra extension repos
- Examples: `extension-kind-gate.mjs` — also validates workflow BPMN shape for workflow-kind repos
- Pattern: Pure functions returning `string[]` errors; main dispatches and exits with code 0 or 1

## Entry Points

**Flow execution:**
- Location: `cinatra/oas.json` → `start_node.$component_ref: "start"`
- Triggers: Cinatra Flow runtime invoking this agent node
- Responsibilities: Accepts `agent_run_id`, `draftBundleRef`, `followupBundleRef` inputs

**CI gate:**
- Location: `extension-kind-gate.mjs` → `main()` (invoked when file is run directly)
- Triggers: `node extension-kind-gate.mjs --package-root .` in `.github/workflows/ci.yml`
- Responsibilities: Reads `package.json`, dispatches kind-specific validation, exits 0/1

## Architectural Constraints

- **No imperative app code:** The repo ships no `src/` directory. Behaviour is entirely declarative (JSON flow spec + Markdown skill prompt).
- **LLM provider lock-in:** `cinatra/oas.json` specifies `preferredProvider: "openai"` and `preferredModel: "gpt-5.5"` — changing models requires editing the OAS.
- **Closed renderer enum:** The skill enforces exactly four renderer values; new frontend layouts require updating `skills/reviewer/SKILL.md` AND the frontend dispatcher.
- **Bundle immutability:** The skill explicitly forbids mutating the `contentBundle`; all approved outputs must be saved as new objects and returned as refs.
- **Zero-dep CI gate:** `extension-kind-gate.mjs` uses only Node.js builtins — no `npm install` required for the gate job.
- **Source mirror model:** This repo is extracted from the Cinatra monorepo. First-party `@cinatra-ai/*` packages are declared only as optional peerDependencies, never in `dependencies` or `devDependencies`. Standalone install/typecheck/test are skipped in CI for such repos.

## Anti-Patterns

### Inline bundle data passing

**What happens:** Passing full bundle content through DataFlowEdges instead of UUID refs
**Why it's wrong:** Bundles can be large; the flow spec and edge payload size limits would be exceeded, and persistence becomes the flow's responsibility
**Do this instead:** Upstream agents must save bundles via `objects_save` and pass UUID refs; reviewer fetches via `objects_get` (`cinatra/oas.json` → `review` node pattern)

### Inventing renderer names

**What happens:** SKILL.md step 2 violated — LLM emits a renderer string not in the closed vocabulary
**Why it's wrong:** The frontend dispatcher only handles the four declared values; unknown strings fall through silently or cause render failures
**Do this instead:** Always return one of `email-drafts`, `email-followups`, `contacts-list`, `unknown` as specified in `skills/reviewer/SKILL.md`

### Adding first-party deps to dependencies/devDependencies

**What happens:** `@cinatra-ai/*` packages placed in `dependencies` or `devDependencies` instead of optional peerDependencies
**Why it's wrong:** CI `build` job classifies the repo as standalone and attempts `pnpm install`, which fails because host-internal packages are not on any public registry
**Do this instead:** Declare host-internal packages only in `peerDependencies` with `peerDependenciesMeta.<pkg>.optional: true`

## Error Handling

**Strategy:** Delegated — the Cinatra platform runtime handles flow-level errors. The CI gate uses explicit exit codes (0 = pass, 1 = violations). The skill instructs the LLM to return `renderer: "unknown"` when the bundle shape is unrecognized.

**Patterns:**
- Gate violations: collected in `string[]` and printed to stderr before `process.exit(1)` (`extension-kind-gate.mjs` → `main`)
- Unrecognized bundle: skill returns `renderer: "unknown"` + descriptive summary; frontend falls back to JSON-Schema view
- OAS parse failure: `validateAgent` pushes parse error string and returns early (`extension-kind-gate.mjs` line 143)

## Cross-Cutting Concerns

**Logging:** None in the flow spec. `extension-kind-gate.mjs` uses `console.log` (pass) and `console.error` (violations).
**Validation:** Retired-primitive scan in `extension-kind-gate.mjs`; package-shape validation in `validateWorkflowPackageShape`; BPMN well-formedness in `validateBpmnSanity`.
**Authentication:** `agent_run_id` threaded through the flow as a runtime-injected hidden input; actual auth is handled by the Cinatra platform, not this repo.

---

*Architecture analysis: 2026-06-09*
