---
id: PRD-vibeflow-mvp
title: Vibeflow MVP — Minimal Cross-Repo Context for Claude Code
status: draft
owner: hieuho
reviewers: []
target_quarter: Q2-2026
repo: vibeflow
schema_version: "1.0"
created: 2026-04-14
updated: 2026-04-14
tags: [mvp, ai-workflow, claude-code, minimal]
---

# Vibeflow MVP — Minimal Cross-Repo Context for Claude Code

> **Schema note**: This PRD dogfoods Vibeflow's own standards (defined in `standards/schemas/prd-frontmatter.schema.json`). Required sections below enforced by `vibeflow lint`.

## Problem

Enterprise dev teams using Claude Code lose productivity because each session starts without knowledge of sibling repos, past decisions, or team conventions. Devs spend 30-60 min re-explaining context to Claude every morning. PRDs and specs drift across repos (different formats, different locations), so Claude can't learn them systematically. Cross-team features (auth used by mobile + web + API) force devs to manually juggle 3+ codebases to give Claude the full picture.

Existing tools (Confluence, Notion, Jira) store knowledge in places Claude can't read, forcing copy-paste. Existing Claude Code setups are per-repo isolates.

**Core pain** (from brainstorm §1): Claude Code at repo A doesn't know what repo B knows.

## Users

- **Primary**: Senior developers at a 3-10 person product team using Claude Code daily across 2-5 related repos (web app + API + shared libs). They want Claude to "know our stack" without re-briefing every session.
- **Secondary**: Team lead who wants devs to work from the same PRD/spec format so AI output is consistent.

**Not in MVP**: Designers, PMs, marketing, research roles. Non-dev track deferred to future Phase 5+.

## Success Metrics

| Metric | Baseline | Target | Measurement |
|---|---|---|---|
| time_to_claude_context | 15-30 min/session | < 30 sec | Self-reported: "time from `claude` to first productive prompt" |
| cross_repo_questions_answered | 0 | 80% | Count of user queries where Claude references sibling repo correctly (manual audit, 1 week sample) |
| prd_consistency | ~30% | 100% | % of PRDs passing `vibeflow lint` in adopted repos |
| devs_using_vibeflow | 0 | 3-5 | Active install count after 2 weeks |

## Scope (MINIMAL MVP — ~2 weeks solo)

> **Aggressive cut from original plan**: 5-7w plan → **~2w MVP**. Remove all infra. Zero server, zero database, zero auth, zero dashboard, zero write path.

**Inclusions**:

1. **Go CLI** (single binary, 3 commands):
   - `vibeflow init <name>` — scaffold new repo with `.vibeflow.yaml`, `.claude/CLAUDE.md`, `docs/PRD.md`, `docs/specs/`, `docs/wiki/` structure. 1 embedded template (generic `code` kind).
   - `vibeflow link` — register current repo into a workspace by appending to `<workspace>/teams.yaml`.
   - `vibeflow lint [path]` — validate PRD/spec frontmatter + required sections against embedded JSON schemas. Exit 1 on error for pre-commit hook.
   - Plus trivial: `version`, `help`.

2. **Standards + schemas** (shipped embedded in CLI binary via `go:embed`):
   - `prd-frontmatter.schema.json` (v1.0)
   - `spec-frontmatter.schema.json` (v1.0)
   - `teams.yaml.schema.json` (v1.0)
   - PRD template (required sections: Problem, Users, Success Metrics, Scope, Non-Goals)
   - Spec template (Overview, API, Data Models, Edge Cases)
   - Single code project template

3. **Local MCP stdio server** (Node/TS, single file ~300 LOC):
   - Spawned by Claude Code as subprocess via `~/.claude/settings.json` `mcpServers.vibeflow.command = "vibeflow-mcp"`
   - Reads directly from filesystem (no Postgres, no cache, no clone daemon)
   - Workspace discovered via env var `VIBEFLOW_WORKSPACE=/path/to/meta-repo`
   - 3 tools:
     - `workspace_context(repo)` — reads `.vibeflow.yaml`, resolves team + linked epics, returns project metadata
     - `wiki_search(query, scope)` — grep over `wiki/**/*.md` + PRD/spec files in all linked repos (no FTS, no embeddings — just ripgrep)
     - `list_projects()` — reads `workspace/teams.yaml`, returns all linked repos with owners
   - Stateless. Reads files on every call. At 5 repos × 100 wiki files grep is < 200ms.

4. **Meta-repo convention** (documented, not enforced by code):
   - `<workspace>/teams.yaml` (team → member → repo list)
   - `<workspace>/epics/*.md` (optional, flat markdown files)
   - `<workspace>/wiki/` (optional, markdown tree)
   - No `.vibeflow/config.yaml`, no standards overrides, no plugins
   - User creates workspace git repo manually, commits teams.yaml, shares the path via env var

## Non-Goals (defer to future phases)

- ❌ Central self-hosted server (MCP HTTP/SSE transport, Docker Compose, Postgres, Redis, Minio, Keycloak)
- ❌ Git sync daemon (polling, webhooks, circuit breakers, worker pool)
- ❌ Write path (no MCP write tools, no PR creation, no `write_queue`)
- ❌ RBAC / OIDC / teams.yaml enforcement / audit log
- ❌ Dashboard (no Next.js, no HTML admin pages — users read files in editor)
- ❌ Kanban board (`plans/kanban.yaml` optional, not rendered by tool)
- ❌ Semantic search / embeddings / Gemini / pgvector / BullMQ
- ❌ Agent Distribution Service / auto-PRs / branch protection enforcement
- ❌ 4 project templates (ship 1 generic)
- ❌ Multi-kind data model (only `kind: code` exists; `product`/`design` defer to Phase 5)
- ❌ Non-dev roles (design, product strategy, marketing, research)
- ❌ Web editor for non-devs
- ❌ Plugin/override model for standards
- ❌ Contractor / temp-access / impersonate / offboard flows
- ❌ Progressive lint enforcement modes (just advisory for MVP, block in git hook if user wants)
- ❌ Agent/skill distribution (users copy agents manually, or skip)
- ❌ Figma integration, binary asset handling, review workflow
- ❌ Two-tier kanban (workspace epic + per-repo task) — user writes flat markdown
- ❌ Shell completion, `vibeflow doctor`, `vibeflow config`, `vibeflow workspace init`
- ❌ `--json` mode everywhere (only on `lint`)
- ❌ Stub OIDC auth (CLI requires zero auth)
- ❌ Backup/restore, monitoring, Prometheus metrics

## User Flows

1. **First-time setup** (5 min):
   - `brew install lukebaze/tap/vibeflow`
   - `git clone <workspace-repo> ~/work/meta-repo` (or create fresh: `mkdir && git init && echo "teams: {}" > teams.yaml`)
   - `export VIBEFLOW_WORKSPACE=~/work/meta-repo`
   - `vibeflow claude setup` → writes to `~/.claude/settings.json`

2. **New repo** (30 sec):
   - `vibeflow init my-service` → scaffolds repo
   - `cd my-service && vibeflow link` → registers in `teams.yaml` locally (user commits + pushes manually)

3. **Daily work** (10 sec):
   - `cd my-service && claude`
   - Claude Code auto-calls `workspace_context("my-service")` via MCP
   - Returns: team, linked repos, epic (if any), PRD status
   - User asks cross-repo question → Claude calls `wiki_search("auth flow")` → grep returns chunks from sibling repos
   - Zero setup per session

4. **PRD lint** (2 sec):
   - `vibeflow lint docs/PRD.md`
   - Exit 0 + "✓ valid" or exit 1 + specific errors
   - User installs as pre-commit hook manually (documented, not auto-installed)

## Dependencies

- Go 1.22+ for CLI build
- Node 20+ for MCP stdio server (or: implement MCP server in Go to avoid Node dependency entirely — see Open Questions)
- `ripgrep` for wiki_search (fall back to Go native if not available)
- User must have Claude Code installed with MCP support
- GitHub / GitLab / Bitbucket — any git host (workspace is just a git repo)

**No external services required**. No API keys. No database. No cloud deps.

## Open Questions

- [ ] **CLI + MCP server both in Go, or CLI-Go + MCP-Node?** Go-only means 1 binary to ship + 0 Node dependency. Node has better MCP SDK. **Decision: try Go-only with minimal MCP impl (~200 LOC); fallback Node if SDK complexity blocks.**
- [ ] **Schema versioning strategy for v1.1+?** Accept `enum: ["1.0", "1.1"]` or single version per binary release?
- [ ] **Workspace discovery UX**: env var + CLI flag, or `.vibeflow` file upward search? Env var simpler for MVP.
- [ ] **`list_projects` needed or covered by `workspace_context`?** Probably drop `list_projects` — 2 tools total.
- [ ] **Linting severity config**: hardcode rules or read from `.vibeflow.yaml`? Hardcode for MVP.
- [ ] **How do users share workspace meta-repo path?** Env var / shell profile / `.envrc`. Not auto-detected. Document in README.
- [ ] **What about `plans/tasks/*.yaml`?** Optional in MVP. If user creates them, lint validates. No kanban rendering.
- [ ] **Does Claude Code spawn stdio MCP server correctly cross-platform (Windows especially)?** Needs spike validation before building.

## Rollout

- **Week 1**: Build Phase 0 (schemas) + CLI `init`/`lint` + stdio MCP server
- **Week 2**: Build `link`, `claude setup`, polish, 1 real repo dogfood test, write README
- **Week 3+ (post-MVP)**: Only if real users hit real pain → add caching, HTTP transport, write path, RBAC. NOT before.

**Ship criterion**: dogfood on `vibeflow` repo itself. If I can't use Vibeflow to develop Vibeflow, ship is blocked.

## Red Team Notes

This PRD replaces the 5-phase plan (5-7w) with a 2w minimal scope. The original plan remains in `plans/260414-1411-vibeflow-mvp-implementation/` as a reference "full vision" for Phase 3+ when real demand appears.

**Trade-offs accepted**:
- No server = no multi-user realtime state (OK: each dev runs their own stdio MCP locally)
- No Postgres = no FTS ranking, no semantic search (OK: ripgrep over 5-10 repos is fast enough)
- No RBAC = anyone with filesystem access reads everything (OK: git repo perms = workspace perms)
- No write path = Claude can read but not mutate state (OK: user commits manually, which is fine for a 3-5 person team)
- No dashboard = no lead visibility (OK: lead uses same Claude Code + CLI as devs; fancy UI isn't worth 2 weeks)

**What survives from original plan**:
- Standards (PRD/spec schemas) — core value
- CLI init/lint — core value
- MCP tool API contract — core value
- Git-as-truth principle — core value
- `kind: code` data model — keeps Phase 5 path open
- Dogfood ethos — core value

**What dies**:
- Phase 2 server architecture (replaced by stdio MCP)
- Phase 3 dashboard (deleted)
- Phase 4 write path + RBAC (deleted — user commits manually)
- Agent Distribution Service (deleted)
- Bundled Keycloak (deleted — no auth needed)

---

**Next**: Read `docs/specs/spec-cli.md`, `docs/specs/spec-mcp-stdio.md`, `docs/specs/spec-standards.md` for implementation details.
