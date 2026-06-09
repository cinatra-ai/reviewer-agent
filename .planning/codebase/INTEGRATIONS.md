# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**Cinatra LLM Bridge:**
- Service: Cinatra internal LLM-bridge API
- What it does: Receives LLM prompts and returns structured JSON outputs for the `review` ApiNode
- Endpoint: `{{CINATRA_BASE_URL}}/api/llm-bridge` (POST)
- Auth: Provided by the Cinatra runtime (env var `CINATRA_BASE_URL` is injected at runtime)
- Defined in: `cinatra/oas.json` (the `review` node, `component_type: "ApiNode"`)

**Cinatra Marketplace:**
- Service: `registry.cinatra.ai` (Cinatra's internal package/extension registry)
- What it does: Hosts published extension packages; this agent is submitted via marketplace promotion saga
- Auth: `CINATRA_MARKETPLACE_VENDOR_TOKEN` org secret (GitHub Actions only)
- Defined in: `.github/workflows/release.yml`

**OpenAI:**
- Service: OpenAI API (accessed indirectly via Cinatra LLM bridge, not directly from this repo)
- Preferred model: `gpt-5.5` (`cinatra_llm.preferredModel` in `cinatra/oas.json`)
- Preferred provider: `openai` (`cinatra_llm.preferredProvider` in `cinatra/oas.json`)
- Auth: Managed by the Cinatra runtime; no API key is stored or referenced in this repo

## Data Storage

**Databases:**
- Not applicable — this extension does not directly connect to any database.

**Cinatra Objects Store (MCP primitives):**
- The `review` ApiNode system prompt calls `objects_get(draftBundleRef)` and `objects_get(followupBundleRef)` to fetch content bundles, and `objects_save` to persist approved bundles.
- These are Cinatra runtime MCP primitives, not a directly integrated database. Invoked via the LLM-bridge prompt, not from application code.
- Defined in: `cinatra/oas.json` (the `review` node `data.system` prompt string)

**File Storage:**
- Not applicable — no direct file storage integration.

**Caching:**
- Not applicable.

## Authentication & Identity

**Auth Provider:**
- Cinatra runtime — handles all authentication for API calls and MCP primitives. This extension repo contains no auth logic.

**Human-in-the-Loop Gate:**
- The `approval_gate` node (`component_type: "InputMessageNode"`) requires explicit human approval before the flow can proceed.
- Surface ID: `reviewer:approval-gate:input`
- Renderer: `@cinatra-ai/reviewer-agent:output`
- Defined in: `cinatra/oas.json` (`$referenced_components.approval_gate`)

## Monitoring & Observability

**Error Tracking:**
- Not detected — no error tracking SDK integrated at the extension level. Monitoring is handled by the Cinatra platform.

**Logs:**
- Not applicable at the extension level. The LLM-bridge and Cinatra runtime handle observability.

## CI/CD & Deployment

**Hosting:**
- Cinatra Marketplace (`registry.cinatra.ai`) — extension is submitted via GitHub Release trigger.

**CI Pipeline:**
- GitHub Actions
  - `ci.yml` — runs on push/PR to `main`; validates package shape, typechecks (skipped for source mirrors with `@cinatra-ai/*` peers), runs tests if present, dry-run packs.
  - `release.yml` — triggers on GitHub Release published or `workflow_dispatch`; delegates to reusable workflow `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main`.
  - `extension-kind-gate.mjs` — self-contained agent OAS validation gate run in `kind-gates` CI job; validates `cinatra/oas.json` and scans for retired CRM primitives.

## Environment Configuration

**Required env vars (runtime, injected by Cinatra platform):**
- `CINATRA_BASE_URL` — base URL for the LLM-bridge API endpoint

**Required secrets (GitHub Actions only):**
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — org-level secret for marketplace submission (referenced in `release.yml` via `secrets: inherit`)

**Secrets location:**
- GitHub org-level secrets (not stored in this repo)
- `.npmrc` file present — existence noted, contents not read

## Webhooks & Callbacks

**Incoming:**
- The `approval_gate` node (`InputMessageNode`) acts as a human-in-the-loop interrupt point — the frontend posts a user response back to the Cinatra runtime, which resumes the flow. This is mediated by the Cinatra platform, not a direct webhook to this extension.

**Outgoing:**
- The `review` ApiNode POSTs to `{{CINATRA_BASE_URL}}/api/llm-bridge` — this is the only outbound HTTP call in the flow definition (`cinatra/oas.json`).

---

*Integration audit: 2026-06-09*
