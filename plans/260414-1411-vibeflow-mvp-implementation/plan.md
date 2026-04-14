---
title: Vibeflow MVP Implementation
status: pending
created: 2026-04-14
slug: vibeflow-mvp-implementation
blockedBy: []
blocks: []
---

# Vibeflow MVP Implementation

Enterprise AI vibe-coding workflow optimizing Claude Code for multi-team product development.

## Context

**Brainstorm**: [brainstorm-260414-0841-vibeflow-workflow-design.md](../reports/brainstorm-260414-0841-vibeflow-workflow-design.md) — 9 deep-dives covering architecture, git sync, MCP API, PRD/spec standards, CLI, non-dev extension, web editor, bootstrap, RBAC, agent distribution.

**Core philosophy**: Git as truth (markdown files in repos), MCP as bridge (Claude Code + dashboard read/write), self-hosted central server per department.

## Scope

**MVP (Phase 0-4)** — dev-only track, `kind: code` projects:
- Scaffolding + standards enforcement
- Claude Code integration via MCP
- Cross-repo context sharing (epics, wiki, PRD links)
- Lead dashboard for visibility
- Secure write path via PR

**Deferred to Phase 5+**: non-dev extension (design/product templates), web editor, binary assets, review workflows, Figma integration.

## Tech Stack

| Component | Tech |
|---|---|
| CLI | Go (cobra, single binary) |
| Server | Node/TS (Next.js 15, tRPC, @modelcontextprotocol/sdk) |
| DB | Postgres 16 + pgvector |
| Queue | Redis + BullMQ |
| Storage | S3-compatible (MinIO default) |
| Auth | OIDC (Better Auth/NextAuth) |
| Embeddings | Gemini text-embedding-004 |
| Git sync | Polling 30s, GitHub App (PAT fallback) |

## Phases (post red-team + validation trim)

| Phase | Name | Status | Est | File |
|---|---|---|---|---|
| 0 | Standards + Schemas + MCP Spike | pending | 3-5d | [phase-00](phase-00-standards-and-schemas.md) |
| 1 | Go CLI (5 commands, embedded templates) | pending | 1w | [phase-01](phase-01-go-cli-and-templates.md) |
| 2 | MCP Server (FTS, 5 tools, manual sync) | pending | 1.5-2w | [phase-02](phase-02-mcp-server-readonly.md) |
| 3 | Minimal HTML Admin Pages | pending | 3-5d | [phase-03](phase-03-dashboard-readonly.md) |
| 4 | Write Path + RBAC + Bundled Keycloak | pending | 2w | [phase-04](phase-04-write-path-and-rbac.md) |

**Total MVP**: **5-7 weeks solo**, 2.5-4 weeks with 2-3 devs.
Previous: 11-16w original → 6-8w post red-team → 5-7w post validation.

## Dependencies (phase order)

```
Phase 0 (standards) ──► Phase 1 (CLI) ──► Phase 2 (MCP server) ──┬─► Phase 4 (write path)
                                                                  │
                                                                  └─► Phase 3 (dashboard)
```

Phase 3 can start after Phase 2 Postgres schemas stable. Phase 4 requires Phase 2 + 3 core complete.

## Success Criteria (MVP)

- [ ] Admin can deploy server via Docker Compose in <30 min
- [ ] Admin can init meta-repo + link 3 product repos via CLI
- [ ] Dev can scaffold new repo via `vibeflow init` in <2 min
- [ ] Claude Code auto-loads workspace context on session start via MCP
- [ ] Claude can query epics, kanban, wiki, PRD, spec across repos
- [ ] `vibeflow lint` blocks PR on PRD schema violation
- [ ] Dashboard shows cross-repo kanban + epic progress for a department
- [ ] Agent update in meta-repo → auto-PR to all eligible repos
- [ ] teams.yaml RBAC enforces read/write permissions via MCP
- [ ] Audit log records every write + denied access

## Risks (plan-wide)

| Risk | Mitigation |
|---|---|
| Scope creep into Phase 5 territory | Freeze non-dev scope, defer all design/product features |
| Claude Code MCP spec changes | Pin `@modelcontextprotocol/sdk` version, monitor changelog |
| GitHub App setup friction blocks adoption | PAT fallback documented, both paths supported |
| Git sync polling scaling issues | Start with 50-repo target, profile before scaling |
| Gemini free tier rate limits | Batch embeddings, FTS fallback, document limits |
| Go CLI cross-platform bugs (Windows) | CI matrix test on 3 OS from day 1 |

## Next Steps

1. Review this plan with stakeholders
2. ✅ Red team review applied (see §Red Team Review below)
3. Run MCP HTTP/SSE verification spike FIRST (part of Phase 0) — if Claude Code doesn't support remote MCP, entire architecture needs rework
4. Start Phase 0 — standards + schemas + spike
5. Set up project repo for Vibeflow itself (monorepo: `cli/` + `server/` + `meta-template/`)

## Validation Log

### Session 1 — 2026-04-14 (post-red-team validation)

**4 critical decisions + 4 POC-scale refinements**:

| # | Question | Decision | Impact |
|---|---|---|---|
| 1 | Target Claude Code version | Latest (1.x) | Phase 0 spike targets current stable, fallback to stdio proxy if needed |
| 2 | OIDC IdP first | **Keycloak (bundled)** | docker-compose.yml ships Keycloak service, zero external IdP dep for POC |
| 3 | Day-1 scale | **POC: 1-3 repos, 2-5 users** | Drop pagination, perf tuning, disk monitoring alerts (over-engineered) |
| 4 | GitHub auth mode | **GitHub App** | Phase 4 uses App; requires org admin approval (document) |
| 5 | Git sync strategy | **Manual sync via CLI only** — NO DAEMON | **MASSIVE Phase 2 cut**: drop sync orchestrator, worker pool, circuit breaker, Redis per-repo locks, polling scheduler. Server exposes `POST /admin/sync/:repo`, CLI `vibeflow sync` triggers it. Users run manually or via git hook. |
| 6 | Template distribution | **Embedded in CLI via `go:embed`** | Cut separate `meta-template/` repo entirely. Single source in `cli/templates/`. Workspace scaffold extracts from binary. Offline-ready. |
| 7 | Starter agents | **1 minimal: `code-reviewer.md`** | Proves pipeline works, no maintenance burden |
| 8 | Keycloak deploy | **Bundled in docker-compose.yml** | `docker compose up` starts Vibeflow + Postgres + Redis + Minio + Keycloak. Zero external IdP config for POC. |

**Aggregate impact on timeline**: 6-8w → **5-7w solo** (manual sync cut eliminates ~3-5 days of Phase 2 work; embedded templates cut ~2 days of Phase 0; bundled Keycloak adds ~1 day to Phase 4 config). Net: **~1 week additional cut**.

**Updated phase estimates post-validation**:

| Phase | Est |
|---|---|
| 0 | 3-5 days (was 1w) |
| 1 | 1w (was 1-1.5w) |
| 2 | 1.5-2w (was 2-2.5w) |
| 3 | 3-5 days (unchanged) |
| 4 | 2w (was 2-2.5w) |

**New total MVP**: **5-7 weeks solo**, 2.5-4 weeks with 2-3 devs.

## Red Team Review

### Session — 2026-04-14
**Reviewers**: 4 hostile lenses (Security Adversary, Failure Mode Analyst, Assumption Destroyer, Scope & Complexity Critic)
**Findings**: 15 accepted, 5 rejected/merged, 18 consolidated into hardening todos
**Severity breakdown**: 8 Critical, 6 High, 1 Medium
**Impact**: Timeline 11-16w → **6-8w** solo. Scope cut ~40%. Security posture hardened.

| # | Finding | Severity | Disposition | Applied To |
|---|---|---|---|---|
| 1 | Delete Phase 3 full Next.js dashboard, replace with HTML admin pages | Critical | Accept | Phase 3 |
| 2 | Cut CLI to 4 essentials (init/link/lint/claude setup) | Critical | Accept | Phase 1 |
| 3 | Delete Agent Distribution Service (manual `agents sync` only) | Critical | Accept | Phase 4 |
| 4 | Claude Code MCP HTTP/SSE transport unverified → spike required | Critical | Accept | Phase 0 |
| 5 | Branch protection missing → direct git push bypasses RBAC | Critical | Accept | Phase 4 |
| 6 | teams.yaml TOCTOU → self-elevation race | Critical | Accept | Phase 4 |
| 7 | Phase 2/3 admin API production-reachable via shared secret | Critical | Accept | Phase 2, 4 |
| 8 | teams.yaml parse failure → universal deny → workspace brick | Critical | Accept | Phase 4 |
| 9 | MCP tools 15+ → cut to 5 essentials | High | Accept | Phase 2 |
| 10 | Drop embeddings (pgvector+Gemini+BullMQ) → FTS only | High | Accept | Phase 2 |
| 11 | Write PR + sync share workdir → use `git worktree` per write | High | Accept | Phase 4 |
| 12 | Markdown prompt injection into Claude via MCP tool output | High | Accept | Phase 2 |
| 13 | SQL/FTS injection via `wiki_search` → `websearch_to_tsquery` | High | Accept | Phase 2 |
| 14 | RBAC rules engine overbuilt → hardcoded 4-role switch | High | Accept | Phase 4 |
| 15 | Clone storage disk exhaustion → shallow + sparse + disk monitoring | Medium | Accept | Phase 2 |

**Rejected**:
- Multi-kind abstractions — keep `kind` field, costs zero, avoids Phase 5 retrofit

**Merged into hardening todos** (track in phase files, not standalone):
- Git credential scrubbing in logs
- Template variable escaping
- Audit log WORM sink (S3 object lock)
- Bootstrap token out-of-band flow
- Auto-merge pre-flight branch protection check

**Deferred to post-MVP**:
- Contractor/temp-access/impersonate/offboard flows
- Cross-team reviewer specialization
- Plugin model for standards overrides
- 4 project templates (ship 1 generic `code` template)
- Semantic search (pgvector + embeddings)
- Real-time activity feed (SSE)
- Dashboard multi-kind views
- Shell completion, doctor command, `--json` everywhere

**Full findings report**: `reports/redteam-260414-vibeflow-mvp.md` (reviewers' raw outputs)
