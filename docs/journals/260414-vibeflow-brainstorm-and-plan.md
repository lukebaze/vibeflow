# Vibeflow MVP Brainstorm, Red-Team, Validation & Ship

**Date**: 2026-04-14 08:00–15:30
**Severity**: Foundational (MVP scope lock)
**Component**: Architecture, Product Design, Go CLI + MCP Server + Keycloak
**Status**: Shipped (public repo, plan + templates committed)

## What Happened

Started with a single request: design "vibeflow", an enterprise AI vibe-coding workflow optimizing Claude Code across multi-team product development. User works in a department with 3 teams (dev, design, product) and identified 4 pain points: context fragmentation across repos, PRD drift (specs not AI-readable), Claude Code visibility in team workflows, wiki not structured for LLM consumption.

### Brainstorm Phase (8am–2pm)
- Scoped via AskUserQuestion: generic design, GitHub Enterprise backbone, hybrid CLI+MCP+dashboard, all 4 pain points addressed
- Chose: git-native truth + MCP bridge + self-hosted central server per department
- Stack: Go CLI + Node.js MCP server + Postgres+pgvector + Gemini embeddings
- Wrote 18 deep-dive sections (~3500 lines) covering git sync protocol, MCP tools, PRD/spec standards, CLI surface, non-dev roles, web editor, bootstrap, RBAC
- **Key pivot**: mid-brainstorm, user flagged workflow missed Design + Product roles. Added Phase 5 non-dev track with `project.kind` field to unify dev/design/product without retrofit
- Output: `/Users/hieuho/vibeflow/plans/reports/brainstorm-260414-0841-vibeflow-workflow-design.md`

### Plan Phase (2pm)
- Created plans/260414-1411-vibeflow-mvp-implementation/ with plan.md + 5 phase files
- Original estimate: 11–16 weeks solo

### Red-Team Phase
- Spawned 4 hostile reviewers in parallel (Security Adversary, Failure Mode Analyst, Assumption Destroyer, Scope & Complexity Critic)
- Got 38 findings, deduped to 15, all accepted
- **Critical cuts**: Delete Phase 3 Next.js dashboard (→ 3 static HTML admin pages), CLI from 12+ to 4 commands, delete Agent Distribution Service, MCP tools from 15+ to 5, drop embeddings (Phase 5), add MCP HTTP/SSE transport verification spike in Phase 0
- **Security adds**: Branch protection enforcement, teams.yaml TOCTOU fix (SHA pinning + two-party approve rule), prompt injection defense (provenance fences), SQL injection defense (websearch_to_tsquery), write workdir race fix (git worktree per write)
- Timeline: 11–16w → **6–8w**

### Validation Phase
- Interview via AskUserQuestion with 8 critical questions
- **Critical decisions**: Keycloak bundled in docker-compose (POC-friendly IdP), POC scale (1–3 repos / 2–5 users), GitHub App auth over PAT, **manual sync via CLI only** (dropped entire git sync daemon — biggest scope cut), **templates embedded in Go binary via go:embed** (no separate meta-template repo), 1 starter agent
- Timeline: 6–8w → **5–7w**

### Ship Phase (3pm)
- Initialized new git repo in /Users/hieuho/vibeflow (separate from parent ~/.claude git)
- Updated .gitignore: track plans/, exclude local tooling (CLAUDE.md, AGENTS.md, .claude/)
- Committed 14 files: brainstorm report + plan.md + 5 phase files + 2 red-team reports + 4 templates + gitignore
- Created public repo: https://github.com/lukebaze/vibeflow via gh CLI
- Pushed successfully

## The Brutal Truth

**This was a full day of design work squeezed into 7 hours.** No implementation yet — just requirements, architecture, threat modeling, and scope negotiation. That's exactly right for MVP planning, but worth noting: user faced decision fatigue during validation phase (8 rapid questions in succession). Real product design is slow; compression via parallel red-teams buys speed but requires brutal cut discipline.

The biggest insight was the `project.kind` pivot. Mid-brainstorm realization that "vibe-coding workflow" only works if designers and product folks *see the work in Claude*. Without that, it's just a dev tool. That one field unifies 3 personas without 3 separate schemas.

Keycloak bundled decision was contentious in red-team (added 200MB Docker image) but necessary for POC: enterprise security theater, zero additional credentials, plays well with GitHub App. Tradeoff: POC scales to 2–5 users max. Acceptable.

Manual sync only (no daemon) was the hardest cut. Async git sync daemon was in the original design. Dropped it because: (a) POC doesn't need it, (b) adds 2–3 weeks, (c) real deployment will use GitHub Actions anyway, (d) manual CLI is "wrong" but works. Future: Phase 6 can add daemon if users scream.

## Technical Details

**Git Sync Protocol**: Workspace.yaml + per-repo .vibeflow/meta.yaml + SHA pinning in teams.yaml. No webhooks. CLI `vibeflow sync` reads git state, merges with canonical workspace, writes back. Two-party approval rule enforced via git branch protection.

**MCP Server**: 5 tools (workspace_get, project_create, project_read, kanban_update, template_render). Node.js, connects to Postgres. HTTP/SSE transport — **still unverified in Claude Code** (Phase 0 spike).

**Go CLI**: 4 commands (init, sync, agent, admin). ~3k SLOC target. go:embed templates (30–40 markdown files for PRD, spec, design brief, roadmap templates).

**Non-Dev Extension**: Design + Product roles use `project.kind: design | product | dev`. Same git tree, different .vibeflow/meta.yaml schema. Phase 5 adds web editor + design kanban + asset management.

**Security Model**: GitHub App (org-level write scope), Keycloak (OIDC), SQL injection defense via pgvector FTS, prompt injection via provenance fences (Claude metadata).

## What We Tried

1. **Semantic embeddings** → Dropped. FTS (PostgreSQL) sufficient for MVP. Gemini embeddings added 2w, $$ per query, 0 user demand.
2. **Async git sync daemon** → Dropped. Manual CLI "wrong" but 3w faster. Real deployment uses Actions anyway.
3. **Next.js dashboard** → 3 static HTML pages instead. Eliminates Node.js build complexity, ORM, API layer for POC.
4. **15 MCP tools** → 5 core tools. Scope creep detector worked.
5. **Separate meta-template repo** → Embedded in binary. One less moving part.

## Root Cause Analysis

Why did this project need a full 7-hour design cycle before code?

1. **Problem scope unclear**: "AI vibe-coding workflow" is a 10-word hand wave. User had to articulate it through Q&A.
2. **Multi-persona design**: Missed design + product roles in v1 brainstorm. Needed pivot.
3. **Stackless starting state**: No prior vibeflow artifact; had to invent from zero.
4. **Security + scale unknowns**: Unclear what "enterprise" means (1 team? 10 teams?). Red-team forced threat modeling early.
5. **Scope creep momentum**: Every section of brainstorm wanted feature X. Red-team was essential kill-switch.

Lesson: Hostile reviews at 2pm (not at end) saved 8 weeks. Validate assumptions *while* planning, not after.

## Lessons Learned

1. **Git as truth, MCP as bridge**: Don't own sync logic. Let git be source, MCP be query layer.
2. **`project.kind` abstraction pays for itself**: One schema, three UIs = clean extension path. Invest in generalization early if >2 personas.
3. **Manual is acceptable for POC**: Automation is nice. Correctness + velocity trump polish. User will demand daemon if real pain.
4. **Embedded templates > separate repo**: Fewer moving parts. Binary is easier to distribute than "clone meta repo first".
5. **Two-party approval rule**: Strongest security lever under git's control (short of signed commits). Worth enforcing now.
6. **Red-team at decision point, not end**: Parallelizing 4 hostile reviews saved 8 weeks of backtracking. Do this again.
7. **Keycloak bundled is pragmatic**: Adds image size but eliminates "how do we auth?" forever for POC.

## Next Steps

**Phase 0 (Spike)**: Verify MCP HTTP/SSE transport works in Claude Code (1–2 days). Blocker for Phase 1 start.

**Phase 1 (Go CLI + Schema)**: vibeflow init, workspace.yaml, teams.yaml, SHAs, git worktree per write. Target: 10 days.

**Phase 2 (Postgres + MCP)**: Keycloak + docker-compose, MCP server, workspace/project/kanban data model. Target: 12 days.

**Phase 3 (Integration)**: CLI ↔ MCP bridge, agent-readable PRDs, starter agent. Target: 8 days.

**Phase 4 (Hardening)**: Branch protection, two-party rule, SQL injection defense, prompt injection fences. Target: 5 days.

**Phase 5 (Non-Dev)**: Web editor, design kanban, `kind: design | product`. Target: 12 days (deferred if Phase 0 blocker lasts).

**Total estimate**: 5–7 weeks solo (48–56 days of 8h work).

---

**Repo**: https://github.com/lukebaze/vibeflow  
**Plan dir**: /Users/hieuho/vibeflow/plans/260414-1411-vibeflow-mvp-implementation/  
**Brainstorm report**: /Users/hieuho/vibeflow/plans/reports/brainstorm-260414-0841-vibeflow-workflow-design.md  
**Red-team reports**: /Users/hieuho/vibeflow/plans/reports/code-reviewer-260414-1134-hostile-design-review.md (full) + .deduped.md
