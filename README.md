# Contribution 73: MCP Tool: project_list

**Contribution Number:** 74

**Student:** Luis C. Martinez

**Issue:** https://github.com/orthogonalhq/nous-core/issues/73

**Status:** Phase II Complete

> **Note on issue type:** This is a **feature / enhancement** (add a new internal MCP capability tool), not a bug fix. Where the template assumes "reproduce the bug," I've reframed those sections as **baseline + gap verification**: proving the capability is genuinely missing, confirming the backing interface exists, and establishing a green test baseline to build against.

---

## Why I Chose This Issue

I chose this issue because it is a clear, bounded way to contribute to an AI-agent codebase. It is also a good opportunity to learn what MCP tools are, how they connect to existing project data, and how this project is organized.

---

## Understanding the Issue

### Problem Description

The Nous internal MCP catalog exposes capability tools to agents through a scoped, authorization-gated surface (`self/cortex/core/src/internal-mcp/`). Agents can already list *workflow runs* for a project via the `workflow_list` tool (issue #72, already shipped), but there is **no tool that lists the projects themselves** — even though the `IProjectStore.list()` method already exists. This issue adds a `project_list` leaf capability tool that surfaces that existing method through the MCP catalog.

### Expected Behavior

An authorized agent can invoke a `project_list` MCP tool and receive the list of projects returned by `IProjectStore.list()`.

### Current Behavior

No such tool exists. `grep -rn "project_list" self/cortex/core/src` returns zero matches. This is a missing capability (an enhancement), not a regression.

### Affected Components

- `self/cortex/core/src/internal-mcp/types.ts` — tool-name registry (`INTERNAL_MCP_TOOL_NAMES`)
- `self/cortex/core/src/internal-mcp/catalog.ts` — catalog entry + Zod input schema
- `self/cortex/core/src/internal-mcp/capability-handlers.ts` — the handler function
- `self/cortex/core/src/internal-mcp/authorization-matrix.ts` — which agent classes may call it
- `self/cortex/core/src/__tests__/internal-mcp/` — tests
- Backing interface (read-only, **not** modified per the issue's scope boundary): `IProjectStore.list(): Promise<ProjectConfig[]>` at `self/shared/src/interfaces/subcortex.ts:351`

---

## Reproduction Process

> For a feature, "reproduction" means establishing the **baseline** and **verifying the gap** rather than triggering a bug.

### Environment Setup

The project is a **pnpm 10 + Node.js 22+ monorepo** (TypeScript, Vitest, oxlint; no devcontainer).

Challenges and how I solved them:

- **Node version:** my default shell was on Node 20, but the repo requires Node 22+ (`engines.node >= 22`). Resolved with `nvm use 24` (v24.16.0 already installed).
- **Package manager:** the repo pins `pnpm@10.6.2` via the `packageManager` field. Resolved with `corepack enable` (Corepack auto-activates the pinned pnpm version inside the repo).
- **Build-before-test ordering (key finding):** running `pnpm test` *before* `pnpm build` produces **module-resolution errors**, not test failures — workspace packages import each other through their built `@nous/*` entry points (e.g. `@nous/subcortex-workflows`), which don't exist until the packages are built. After `pnpm build`, the suite passes. The correct order is `pnpm install` → `pnpm build` → `pnpm test` (this matches `CONTRIBUTING.md`).

Verified baseline:

| Step | Command | Result |
|---|---|---|
| Install | `pnpm install` | ✓ ~8s (warm store) |
| Build | `pnpm build` | ✓ exit 0 (all workspace packages incl. web + cli) |
| Tests (target area) | `pnpm exec vitest run self/cortex/core/src/__tests__/internal-mcp` | ✓ 11 files / **328 tests** passing |
| Lint | `pnpm lint` | ✓ 0 errors (149 pre-existing warnings) |

### Steps to Reproduce (Baseline & Gap Verification)

1. `nvm use 24 && corepack enable && pnpm install`
2. `pnpm build`
3. Confirm the gap: `grep -rn "project_list" self/cortex/core/src` → **no results** (the tool does not exist).
4. Confirm the seam exists: `IProjectStore.list(): Promise<ProjectConfig[]>` at `self/shared/src/interfaces/subcortex.ts:351`.
5. Confirm a green baseline in the area I'll touch: `pnpm exec vitest run self/cortex/core/src/__tests__/internal-mcp` → 328 passing.

### Reproduction Evidence

- **Working branch:** https://github.com/luiscmartinez/nous-core/tree/feat/project-list-mcp-tool (branched from `upstream/dev`; clean baseline — implementation pending, see WR-178 note below).
- **My findings:**
  - This is purely additive work inside existing contracts; the issue explicitly forbids modifying `IProjectStore`.
  - The direct precedent is the already-shipped `workflow_list` tool (#72), which follows the same four-file + tests pattern.
  - **`project_list` is broader in scope than `workflow_list`:** `workflow_list` is scoped to a single `projectId`, whereas `project_list` enumerates *all* projects. That cross-project surface is almost certainly why the maintainer opened an internal audit ticket (**WR-178**) before authorizing implementation.

---

## Solution Approach

### Analysis

There is no root-cause bug to fix — the capability is simply absent. The task is to add a new leaf handler that calls the existing `IProjectStore.list()` and register it across the MCP catalog, authorization matrix, and tool-name registry, with tests.

### Proposed Solution

Add a `project_list` capability tool that mirrors the shipped `workflow_list` tool: a handler calling `deps.projectStore.list()` with null-guarding, a catalog entry with a Zod input schema (likely empty — the tool takes no required arguments), an authorization-matrix entry, registration in `INTERNAL_MCP_TOOL_NAMES`, and tests following existing patterns.

### Implementation Plan

Using the UMPIRE framework (adapted for a feature):

**Understand:** Agents have no MCP capability to enumerate projects. Add a `project_list` internal MCP tool that surfaces the existing `IProjectStore.list()` through the scoped, authorization-gated catalog.

**Match:** The shipped `workflow_list` tool (#72) is the direct precedent — same registration pattern across four files:
- tool name → `types.ts:70` (`INTERNAL_MCP_TOOL_NAMES`)
- handler → `capability-handlers.ts:1261`
- authorization → `authorization-matrix.ts` (registered for multiple agent classes)
- catalog entry → `catalog.ts:480`

`project_list` is *simpler* than `workflow_list` (no `projectId` input) but *broader in scope* (returns all projects → authorization matters more).

**Plan:**
1. Add `'project_list'` to `INTERNAL_MCP_TOOL_NAMES` in `types.ts`.
2. Add a catalog entry + Zod input schema in `catalog.ts` (input schema likely empty/no-arg).
3. Add the handler in `capability-handlers.ts` calling `deps.projectStore.list()` with null-guarding.
4. Add authorization-matrix entries — **agent-class scope TBD pending WR-178** (which of `Cortex::System` / `Orchestrator` / `Worker` should see all projects).
5. Add tests in `__tests__/internal-mcp/` mirroring the `workflow_list` test cases.

**Implement:** Branch `feat/project-list-mcp-tool` (link above). **Gated on WR-178** — the maintainer is auditing the `project_list` surface and asked me to wait for the "safe implementation shape" before writing code. I will not implement until that guidance is posted; the plan above is provisional and the authorization scope (and any project-field redaction) may change based on the audit.

**Review:** Conventional commits — e.g. `feat(cortex-core): add project_list internal MCP capability tool`. Conventions per `CONTRIBUTING.md`: camelCase vars/functions, PascalCase types, kebab-case file names, `I`-prefixed interfaces, Zod schemas as the single source of truth for runtime validation. Lint via `pnpm lint` (oxlint, no eslint).

**Evaluate:** New vitest tests plus the full pre-submit gate from `CONTRIBUTING.md`: `pnpm typecheck && pnpm lint && pnpm test && pnpm build` all green.

---

## Testing Strategy

### Unit Tests

- [ ] `project_list` returns the projects from `IProjectStore.list()`
- [ ] Null-guarding: handled result when the store returns nothing / is undefined
- [ ] Authorization: only permitted agent classes can invoke it (per the matrix; scope to be confirmed via WR-178)

### Integration Tests

- [ ] Tool resolves through the scoped catalog surface for an authorized agent class
- [ ] Catalog / authorization-matrix / tool-name registrations are consistent (mirroring the `workflow_list` test patterns)

### Manual Testing

Baseline verified before implementation: `pnpm install` + `pnpm build` succeed; the internal-mcp test suite passes (328 tests) on Node 24. This establishes the green starting point against which the new tool's tests will be added.

---

## Implementation Notes

### Week 1 Progress (Phase II)

- Set up the local environment (Node 20 → 24, Corepack/pnpm, monorepo install/build).
- Created and pushed the working branch `feat/project-list-mcp-tool` from `upstream/dev`.
- Verified the gap (no existing `project_list`) and the backing seam (`IProjectStore.list()`).
- Established a green test/lint baseline.
- Drafted the UMPIRE solution plan and identified the `workflow_list` precedent.
- **Blocked on implementation pending the maintainer's WR-178 audit** — by design; engaging with the maintainer's review process rather than pre-empting it.

### Code Changes

- **Files to modify (planned):** `types.ts`, `catalog.ts`, `capability-handlers.ts`, `authorization-matrix.ts`, and a new test file under `__tests__/internal-mcp/`.
- **Key commits:** none yet (pre-implementation).
- **Approach decisions:** mirror the shipped `workflow_list` tool; keep the change additive and within existing contracts (do not modify `IProjectStore`).

---

## Pull Request

**PR Link:** Pending (Phase III — after the WR-178 audit guidance is received).

**PR Description:** Will adapt the analysis and plan above.

**Maintainer Feedback:**
- 2026-06-05: Maintainer (@atlamors) running an audit on the adapter surface; will report back in the thread.
- 2026-06-12: Maintainer opened internal ticket **WR-178** and is completing a detailed audit/review of the MCP `project_list` surface before asking me to proceed; will follow up with the "safe implementation shape."

**Status:** Awaiting maintainer's WR-178 audit before implementation.

---

## Learnings & Reflections

### Technical Skills Gained

- What the Nous internal MCP capability surface is and how a tool is registered (catalog + Zod schema + authorization matrix + tool-name registry + tests).
- Monorepo mechanics: pnpm workspaces, Corepack-pinned package managers, and why cross-package builds must precede tests.
- Reading an existing tool (`workflow_list`) as a precedent to scope a new, similar one.

### Challenges Overcome

- Node-version mismatch (20 vs 22+) and the build-before-test ordering gotcha that initially looked like test failures but was a module-resolution artifact.

### What I'd Do Differently Next Time

- Confirm the target branch and contribution conventions (`CONTRIBUTING.md`, `dev → staging → main` flow) up front, before touching anything.

---

## Resources Used

- [orthogonalhq/nous-core CONTRIBUTING.md](https://github.com/orthogonalhq/nous-core/blob/main/CONTRIBUTING.md) — tiers, conventions, PR process, MCP tool surface section
- [Issue #72 — workflow_list](https://github.com/orthogonalhq/nous-core/issues/72) (shipped) — the direct implementation precedent
- [Issue #73 — project_list](https://github.com/orthogonalhq/nous-core/issues/73) — this contribution
- Model Context Protocol documentation — background on MCP tools
