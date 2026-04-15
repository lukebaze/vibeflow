---
title: Vibeflow Minimal MVP Implementation
status: pending
created: 2026-04-14
updated: 2026-04-14
slug: vibeflow-mvp-implementation
blockedBy: []
blocks: []
---

# Vibeflow Minimal MVP Implementation

2-week minimal MVP delivering cross-repo context for Claude Code. No server, no database, no auth.

## Context

- **PRD**: [../../docs/PRD-vibeflow-mvp.md](../../docs/PRD-vibeflow-mvp.md)
- **Specs**:
  - [spec-cli.md](../../docs/specs/spec-cli.md)
  - [spec-mcp-stdio.md](../../docs/specs/spec-mcp-stdio.md)
  - [spec-standards.md](../../docs/specs/spec-standards.md)
- **Brainstorm**: [../reports/brainstorm-260414-0841-vibeflow-workflow-design.md](../reports/brainstorm-260414-0841-vibeflow-workflow-design.md) — 9 deep-dives, "full vision" reference
- **Red team + validation history**: [reports/redteam-260414-vibeflow-mvp.md](reports/redteam-260414-vibeflow-mvp.md)

## Evolution of Scope

| Stage | Estimate | Scope |
|---|---|---|
| Original plan | 11-16w | Server + Postgres + Dashboard + Write path + RBAC + ADS + Embeddings |
| Post red-team | 6-8w | Cut 40% — drop dashboard Next.js, ADS, embeddings, simplify RBAC |
| Post validation | 5-7w | Manual sync CLI only, embedded templates, bundled Keycloak |
| **Post minimization** | **2w** | **No server at all. Go CLI + local stdio MCP. Stateless.** |

**User feedback driving the cut**: "i think the mvp is quite of overengineered, try to minimize the MVP"

## Core Philosophy (Minimal)

**Git is truth. MCP is a stdio subprocess. That's it.**

- No central server
- No Postgres / Redis / Minio / Keycloak
- No OIDC / RBAC / audit log
- No write path (user commits manually)
- No dashboard
- No sync daemon
- Each dev's Claude Code spawns its own local MCP subprocess that reads filesystem directly

## Tech Stack (Final)

| Component | Tech |
|---|---|
| CLI | Go (cobra, single static binary, ~15 MB) |
| MCP server | Go (stdio transport, JSON-RPC 2.0 minimal impl) |
| Distribution | GitHub Releases + Homebrew tap |
| Storage | Filesystem (user's git repos) |
| Search | ripgrep (with native Go fallback) |
| Auth | None (filesystem perms = workspace perms) |

**No runtime dependencies** except user has `git` and optionally `ripgrep` for faster search.

## Phases

| Phase | Name | Status | Est | File |
|---|---|---|---|---|
| 0 | Standards + schemas | pending | 2 days | [phase-00](phase-00-standards-and-schemas.md) |
| 1 | Go CLI (3 commands) | pending | 4 days | [phase-01-go-cli-and-templates.md](phase-01-go-cli-and-templates.md) |
| 2 | Local stdio MCP server | pending | 4 days | [phase-02-mcp-server-readonly.md](phase-02-mcp-server-readonly.md) |
| 3 | Dogfood + release | pending | 2 days | [phase-03-dashboard-readonly.md](phase-03-dashboard-readonly.md) |

**Total MVP**: **~2 weeks solo**. No parallelism opportunity — phases are sequential (standards → CLI → MCP → dogfood).

## Dependencies (phase order)

```
Phase 0 (schemas + templates) ─► Phase 1 (CLI embeds them) ─► Phase 2 (MCP reuses schema validator) ─► Phase 3 (dogfood)
```

## Success Criteria (MVP)

- [ ] `brew install lukebaze/tap/vibeflow` works on macOS
- [ ] `vibeflow init my-service` scaffolds a valid repo in <2s
- [ ] `vibeflow lint docs/PRD.md` catches missing required sections with specific errors + exit code 7
- [ ] `vibeflow link` registers repo in `teams.yaml` idempotently
- [ ] `vibeflow claude setup` writes valid MCP entry to `~/.claude/settings.json`
- [ ] Restart Claude Code → MCP server spawns → `workspace_context` returns project metadata in <50ms
- [ ] `wiki_search("auth")` over 3-repo fixture returns ranked results from sibling repos in <200ms
- [ ] `vibeflow lint` runs clean on `docs/PRD-vibeflow-mvp.md` (dogfood)
- [ ] Binary <20 MB, cross-compiled for darwin-arm64, darwin-amd64, linux-arm64, linux-amd64, windows-amd64
- [ ] README with 5-minute quickstart published to public repo

## Risks (plan-wide)

| Risk | Mitigation |
|---|---|
| Claude Code MCP stdio transport quirks | Phase 0 spike validates before Phase 1 starts |
| Go JSON-RPC 2.0 implementation complexity | Minimal impl (~100 LOC), fallback to Node MCP SDK if painful |
| ripgrep not installed on user machine | Native Go walker fallback included in Phase 2 |
| Windows path handling bugs | CI matrix test from day 1 |
| Template variable injection | Strict regex at flag parse, validated in Phase 1 tests |
| Scope creep back to server | User explicitly cut scope — if temptation arises, defer to Phase 5+ and honor PRD non-goals |

## Explicitly Deferred (NOT in MVP)

See PRD §Non-Goals for full list. Highlights:
- Central server, HTTP/SSE MCP transport
- Postgres, embeddings, semantic search
- Write path, PR creation, audit log
- OIDC, RBAC, teams.yaml enforcement
- Dashboard (any form)
- Agent Distribution Service
- Multi-kind data model (only `kind: code`)
- Non-dev roles, web editor, binary assets
- Kanban rendering, progressive lint enforcement

## Red Team + Validation History

Preserved for reference in [reports/redteam-260414-vibeflow-mvp.md](reports/redteam-260414-vibeflow-mvp.md).

Key prior decisions now MOOT (scope deleted before they mattered):
- Branch protection enforcement (no write path)
- teams.yaml TOCTOU (no dynamic RBAC)
- Audit log WORM (no write path)
- Agent Distribution Service (deleted)
- OIDC portability (no auth)
- Git sync daemon (deleted, replaced with manual file reads)

Key prior decisions that SURVIVE:
- `schema_version: enum: ["1.0"]` (not `const`)
- Prompt injection provenance fences in MCP output
- `kind: code` field for Phase 5 path
- 1 generic template (not 4)
- Dogfood ethos

## Next Steps

1. ✅ PRD + specs written and committed
2. Start Phase 0: write schemas + markdown templates
3. Phase 1: Go CLI scaffolding
4. Phase 2: MCP stdio server
5. Phase 3: Dogfood + release v0.1.0
