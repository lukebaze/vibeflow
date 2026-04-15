---
id: PRD-vibeflow-mvp
title: Vibeflow — Enterprise Dev Team Workflow Super-Repo
status: draft
owner: hieuho
reviewers: []
target_quarter: Q2-2026
repo: vibeflow
schema_version: "1.0"
created: 2026-04-14
updated: 2026-04-15
tags: [mvp, super-repo, claude-code, enterprise-dev]
---

# Vibeflow — Enterprise Dev Team Workflow Super-Repo

> **Pivot note**: This PRD replaces the earlier Go CLI + MCP server plan (archived in `plans/archived/`). After discovering `cxzharry/avengers-team`, the team pivoted to a super-repo pattern that ships the same value in 1-2 days instead of 2 weeks. Inspiration credited in README.

## Problem

Enterprise dev teams using Claude Code lose productivity because each session starts without knowledge of team conventions, cross-repo patterns, or standardized workflows. Different devs write PRDs/specs/ADRs in different formats. Agents, skills, and slash commands drift per-repo. Code review depth varies. When a new project starts, there's no opinionated workflow — every team member freestyles.

**Core pain**: Teams lack a shared, opinionated, versioned source of truth for *how* to use Claude Code across their projects.

Existing attempts (internal wiki, shared Notion pages, copy-paste) fail because: (a) Claude Code can't read them natively, (b) they drift from actual practice, (c) no single command produces a consistent deliverable.

## Users

- **Primary**: Lead developer or engineering manager setting up workflow standards for a 3-10 person dev team in an enterprise org. They want one place to codify "how we use Claude Code here".
- **Secondary**: Individual developers on that team who want to start new projects with a known-good scaffolding and run opinionated slash commands instead of freestyling.

**Not in MVP**: Consumer AI teams (avengers-team serves this niche), non-dev roles (designer, PM, marketing, research). Vibeflow is dev-first.

## Success Metrics

| Metric | Baseline | Target | Measurement |
|---|---|---|---|
| time_to_first_deliverable | 30-60 min freestyle | < 10 min via `/vf-prd` | Self-reported: time from "git clone + init" to valid PRD written |
| prd_consistency | ~30% (freestyle) | 100% | % of PRDs passing `vf-lint.sh` after init |
| slash_command_adoption | 0 | ≥ 80% | % of Claude Code sessions in Vibeflow projects that invoke at least one `/vf-*` command |
| onboarding_time | 15-30 min | < 5 min | git clone → first successful slash command execution |
| teams_adopting | 0 | 3 | # of teams using Vibeflow after 2 weeks |

## Scope (Super-Repo MVP — ~2 days)

**This repo IS the product.** No binary, no server, no compile step. Users `git clone` Vibeflow, run `./scripts/vf-init.sh <project>`, and get a scaffolded project with Vibeflow's agents/skills/commands pre-loaded.

### Inclusions

1. **`.claude/` directory** (vendored, self-contained):
   - `agents/` — 7 curated Vibeflow-specific agents: `planner`, `code-reviewer`, `debugger`, `tester`, `researcher`, `docs-manager`, `fullstack-developer`
   - `skills/` — 6 skills: `karpathy-guidelines`, `plan`, `brainstorm`, `code-review`, `adversarial-dev`, `wiki`
   - `commands/` — 8 slash commands (all prefix `/vf-*`)

2. **Slash commands** (all markdown, read natively by Claude Code):
   - `/vf-prd <product>` — interview + fill PRD template
   - `/vf-spec <feature>` — interview + fill spec template
   - `/vf-adr <decision>` — record architecture decision
   - `/vf-plan <prd-path>` — break PRD into implementation phases
   - `/vf-review` — review pending code + docs changes
   - `/vf-wiki <query>` — search wiki (native grep)
   - `/vf-learn <source>` — ingest new knowledge into wiki
   - `/vf-init-project <name>` — scaffold new project (shells out to `vf-init.sh`)

3. **Bash scripts** (~150 LOC total):
   - `vf-init.sh` — `cp -r .claude/ templates/ CLAUDE.md <project>` + `mkdir -p` scaffolding + `git init`
   - `vf-sync.sh` — `diff -rq` report from master super-repo to local project, no overwrite
   - `vf-lint.sh` — bash + grep/sed frontmatter and required-section validation

4. **Templates** (plain markdown, copied by `vf-init.sh`):
   - `templates/prd.md` — 1-2 page enterprise PRD (Problem, Users, Success Metrics, Scope, Non-Goals, Dependencies, Open Questions)
   - `templates/spec.md` — technical spec (Overview, API Contracts, Data Models, Edge Cases, Testing Strategy, Security Considerations)
   - `templates/adr.md` — Michael Nygard-style ADR (Status, Context, Decision, Consequences)
   - `templates/project-starter/` — base project scaffold (README.md, CLAUDE.md, docs/, plans/)

5. **Wiki** (plain markdown, read via native grep or `/vf-wiki`):
   - `wiki/patterns/` — 5 seeded patterns: `1-page-prd-pattern.md`, `adr-pattern.md`, `cross-repo-convention.md`, `code-review-rubric.md`, `dogfood-ethos.md`
   - `wiki/domain/` — placeholder for enterprise dev domain knowledge
   - `wiki/examples/` — placeholder for worked examples
   - `wiki/retros/` — placeholder for project retrospectives

6. **`CLAUDE.md`** — workflow routing table auto-loaded by Claude Code in every session. Maps "I want to X" to the right `/vf-*` command.

7. **Reference docs in `docs/`**:
   - `docs/PRD-vibeflow-mvp.md` (this file)
   - `docs/specs/spec-standards.md` (schema + template reference, NOT enforced by tooling)
   - `docs/journals/` (session journals)

### Non-Goals

**Explicitly NOT in MVP** (many of these were in the earlier Go plan — now archived):

- ❌ Go CLI binary
- ❌ Node.js / TypeScript runtime
- ❌ Custom MCP server (Claude Code native `.claude/` is sufficient)
- ❌ JSON Schema validator (bash `grep`/`sed` is enough for MVP)
- ❌ Postgres / pgvector / embeddings / semantic search
- ❌ Docker Compose / Keycloak / OIDC / RBAC / audit log
- ❌ Central self-hosted server
- ❌ Git sync daemon (users run `vf-sync.sh` manually)
- ❌ Dashboard / web UI / write path via PR
- ❌ GitHub App / branch protection enforcement
- ❌ Agent Distribution Service
- ❌ Binary distribution (Homebrew, releases, cross-compile matrix)
- ❌ `go:embed` template system
- ❌ Multi-kind data model (only dev projects; no product/design tracks)
- ❌ Non-dev roles (PO workflow → avengers-team niche; designer/PM → defer)
- ❌ Cross-repo live query (self-contained per project model)
- ❌ Progressive lint enforcement modes (just advisory)

## User Flows

### 1. First-time setup (2 min)
```bash
git clone https://github.com/lukebaze/vibeflow.git ~/vibeflow
# That's it. No install, no binary, no runtime.
```

### 2. New project (1 min)
```bash
~/vibeflow/scripts/vf-init.sh my-service
cd my-service
# Claude Code auto-loads .claude/ on next session
```

### 3. Write a PRD (<10 min)
```bash
cd my-service
# In Claude Code:
/vf-prd my-service
# Interview-driven: 5 questions, Claude fills template, saves to docs/PRD.md
```

### 4. Break PRD into plan (<5 min)
```bash
/vf-plan docs/PRD.md
# planner agent reads PRD, produces plans/<slug>/plan.md + phase files
```

### 5. Record architecture decision (<5 min)
```bash
/vf-adr "choose Postgres over MongoDB"
# Interview + fill docs/adr/ADR-NNN-postgres-vs-mongo.md
```

### 6. Code review before commit (<5 min)
```bash
/vf-review
# code-reviewer agent reviews pending changes via git diff, applies rubric from wiki/patterns/
```

### 7. Sync updates from master Vibeflow (manual, weekly)
```bash
~/vibeflow/scripts/vf-sync.sh
# Diff-only report. User copies files they want, skips rest.
```

## Dependencies

- **Git** (for clone + init + sync)
- **Bash** (for scripts — macOS/Linux native, Windows via WSL or Git Bash)
- **Claude Code** (reads `.claude/` natively, executes slash commands)
- **ripgrep** (optional, speeds up `/vf-wiki` grep — falls back to native `grep`)

**No external services required**. No API keys. No database. No cloud deps. Works offline.

## Architecture Rationale

### Why super-repo over custom tooling?

After discovering `cxzharry/avengers-team` (2026-04-14), we realized Claude Code's native `.claude/` auto-loading + slash commands ALREADY solve the workflow standardization problem. Building a Go CLI + MCP server to do the same thing was ~20x more work for the same value.

### Why `cp -r` instead of template engine?

Template variable substitution adds complexity (Go templates, injection defense, schema validation) for little value. The only variable that matters is project name, which can be a simple `sed` one-liner in the init script. YAGNI.

### Why bash scripts instead of a compiled binary?

- Zero install: `git clone` is already standard dev workflow
- Zero build: no `go build`, no `npm install`, no `brew install`
- Zero runtime: bash is everywhere
- Easy to read/modify: team can contribute changes in their IDE
- Cross-platform: bash works on macOS/Linux/Windows-WSL

### Why no MCP server?

Claude Code already reads `.claude/` natively. A custom MCP server would provide `workspace_context` and `wiki_search` — but Claude Code's built-in file tools (Read, Grep, Glob) solve these problems without any server.

### Why enterprise dev (vs avengers-team's consumer AI)?

- avengers-team targets PO/Designer/Dev on Consumer AI team with Next.js + Cloudflare default
- Vibeflow targets dev teams in larger orgs with stack-agnostic, ADR-heavy, review-discipline workflows
- Different audience, same pattern. Not a competitor — a sibling project with different positioning.

## Open Questions

- [ ] Which 7 agents should we write fresh? Tentative: planner, code-reviewer, debugger, tester, researcher, docs-manager, fullstack-developer
- [ ] Vendor `karpathy-guidelines` skill or rewrite Vibeflow-specific principles?
- [ ] ADR template: Nygard-lite (4 fields) or enterprise (with risks, alternatives, rollback)?
- [ ] Should `/vf-review` include security-mindset auto-activation for auth/crypto/secrets code?
- [ ] Cross-repo wiki access pattern: document symlink approach in `wiki/patterns/cross-repo-convention.md`?
- [ ] Does "dev team" include QA/SRE/security? Start with Dev + Tester, defer others to Phase 2.
- [ ] Vendor skills in-repo or reference global `~/.claude/skills/`? Vendor for self-containment.

## Rollout

### Day 1 (~4-6 hours)
- [ ] Write `CLAUDE.md` routing table
- [ ] Write 8 slash commands (`.claude/commands/vf-*.md`)
- [ ] Write 3 bash scripts (`scripts/vf-init.sh`, `vf-sync.sh`, `vf-lint.sh`)
- [ ] Write 4 templates (`templates/prd.md`, `spec.md`, `adr.md`, `project-starter/`)
- [ ] Write 7 Vibeflow-specific agents (`.claude/agents/*.md`)
- [ ] Vendor 6 skills (`.claude/skills/*`)

### Day 2 (~4-6 hours)
- [ ] Seed 5 wiki patterns
- [ ] Write `README.md` with quickstart + credit to avengers-team
- [ ] Dogfood: run `./scripts/vf-init.sh test-project` end-to-end
- [ ] Fix anything that breaks
- [ ] Git tag `v0.1.0`
- [ ] Push to `github.com/lukebaze/vibeflow`

**Total**: **1-2 days solo**, ~half a day with a helper.

## Ship Criterion

**"If I can't use Vibeflow to develop Vibeflow, ship is blocked."**

Specifically:
- `./scripts/vf-init.sh test-project` produces a working scaffold
- `./scripts/vf-lint.sh docs/PRD.md` passes clean on this PRD
- `/vf-adr "pivot from Go to super-repo"` produces a valid ADR file
- `/vf-review` reviews a pending change via `code-reviewer` agent

## Trade-offs Accepted

- **No compiled binary** → users need `git` and `bash` (acceptable — dev tools users have these)
- **No schema validator** → bash lint gives weaker error messages than Go JSON Schema (acceptable — errors are still clear)
- **No MCP server** → no cross-repo live query tool (acceptable — Claude Code native file tools work)
- **No auto-update** → users run `vf-sync.sh` manually (acceptable — explicit is better than magic)
- **No Windows native** → requires WSL or Git Bash (acceptable for dev audience)

## What Survives from Archived Go Plan

Still valid as reference material:
- JSON schemas (moved to `docs/specs/spec-standards.md` as reference, not enforced)
- PRD/spec section requirements (enforced via bash lint now)
- Red-team findings (preserved in `plans/archived/.../reports/`)
- Validation log (preserved in archived plan)

What was wrong in the Go plan:
- Built a CLI + MCP server to solve a problem Claude Code already solves natively
- Over-engineered schema validation (bash is enough for MVP)
- Designed for cross-repo live query (problem that doesn't exist at POC scale)
- Estimated 2 weeks when 1-2 days does the job

## Credit

Pattern inspired by [`cxzharry/avengers-team`](https://github.com/cxzharry/avengers-team) — a super-repo for Product Consumer AI teams. Vibeflow adopts the same pattern for enterprise dev teams. Explicit credit in README.

## Red Team + Validation History

Preserved in `plans/archived/260414-1411-vibeflow-mvp-go-implementation/`:
- Original brainstorm (9 deep-dives)
- Red-team review (15 findings applied)
- Validation log (8 decisions)

Most findings are now moot because the scope they addressed has been deleted. Key findings that still apply to super-repo:
- Prompt injection defense → document in slash commands + wiki/patterns
- ADR mandatory → first-class in new plan
- Dogfood ethos → preserved as ship criterion

---

**Next**: Read `docs/specs/spec-standards.md` for PRD/spec/ADR format reference. `CLAUDE.md` for workflow routing.
