---
phase: 0
title: Standards + Schemas + MCP Spike
status: pending
effort: 1 week
---

# Phase 0: Standards + Schemas + MCP Spike

## ⓘ Red Team Changes (Applied 2026-04-14)

- **Finding #4 (Critical)**: Added MANDATORY MCP HTTP/SSE verification spike as §0 before schema work. If Claude Code doesn't support remote MCP transport in target version, entire architecture needs rework.
- **Finding #2/Finding #5 (scope)**: Cut 4 templates (`service`, `library`, `web-app`, `cli`) → ship 1 generic `code` template. Drop `templates/product/.gitkeep`, `templates/design/.gitkeep` placeholder dirs (YAGNI).
- **Finding (Assumption #8)**: `schema_version` uses `enum: ["1.0"]` instead of `const: "1.0"` to enable future evolution without retroactive breakage.

## <!-- Updated: Validation Session 1 — embedded templates + 1 starter agent -->

- **Validation #6**: NO separate `meta-template/` repo. All standards + schemas + template files live in `cli/templates/` and `cli/schemas/`, embedded via `go:embed` at compile time. Phase 0 deliverable is a directory structure inside the CLI repo, NOT a standalone git repo.
- **Validation #7**: Ship ONE starter agent: `standards/agents/code-reviewer/agent.md` + manifest. Proves the agent distribution pipeline works. User adds more via `.claude/local/`.
- **Validation #1**: Phase 0 MCP spike targets **latest Claude Code (1.x)** specifically. Document version tested. Fallback plan: if HTTP/SSE not supported, build local stdio proxy process in Phase 1.
- **Effort**: 1w → **3-5 days** (embedded templates = no separate repo setup; 1 agent = minimal content work).

## 0. MCP Transport Spike (MUST RUN FIRST)

**Goal**: Prove Claude Code (target version) can connect to `@modelcontextprotocol/sdk` HTTP/SSE server and invoke a tool. If stdio-only, redesign Phase 2 around local CLI-spawned proxy.

**Steps**:
1. Stand up minimal Node server with `@modelcontextprotocol/sdk` + HTTP/SSE transport + 1 dummy tool
2. Configure local Claude Code `~/.claude/settings.json` to point at it
3. Invoke tool from Claude Code session, verify response
4. Document: target Claude Code version, supported transport, any proxy required
5. **If spike fails**: pause plan, revisit Phase 2 design (local stdio proxy process vs remote HTTP)

**Owner**: tech lead.
**Duration**: 1 day.
**Blocker**: YES — do not start Phase 1 until this spike passes.

---

## Context Links

- Brainstorm §4.3 Meta-repo structure
- Brainstorm §12 PRD/spec templates
- Brainstorm §16.4 Meta-repo scaffold
- Brainstorm §17.12 teams.yaml schema

## Overview

**Priority**: P0 blocker for all subsequent phases.
**Status**: pending
**Description**: Define all machine contracts (JSON schemas), templates (PRD/spec/repo templates), meta-repo skeleton, and kind-namespaced template layout. Pure documentation work — no code, no infra.

## Key Insights

- Standards are the machine contracts Go CLI + Node server both validate against → single source of truth = JSON Schema files.
- Kind namespacing (`templates/code/`, `templates/product/`, `templates/design/`) enables Phase 5 extension without schema breakage.
- Required sections enforced by lint, not by freeform markdown parsing — schema-driven.
- Progressive enforcement: schemas support `advisory → block_new → block_edit → ci_gate` modes.

## Requirements

**Functional**:
- Meta-repo skeleton scaffoldable via `git init` + file copy (no CLI needed yet)
- JSON Schema files for: kanban, epic frontmatter, PRD frontmatter, spec frontmatter, teams.yaml, task frontmatter
- PRD + spec markdown templates with required sections defined
- Code project templates (`service`, `library`, `web-app`, `cli`) with template manifests
- Standards/schemas versioned (`schema_version: "1.0"`)
- Workspace config file schema (`.vibeflow/config.yaml`)
- Kanban stages defined per kind (`kanban-stages.yaml`)

**Non-functional**:
- All schemas Draft-07 JSON Schema compliant
- Templates pure markdown + Go template syntax only (no framework lock-in)
- Backward-compat path documented (schema_version bumps)

## Architecture

```
meta-template/                           ← new git repo (ships as starter)
├── .vibeflow/
│   ├── config.yaml                      (workspace config)
│   └── schema-version                   ("1.0")
├── standards/
│   ├── prd-template.md                  (required sections, frontmatter docs)
│   ├── spec-template.md
│   ├── epic-template.md
│   ├── task-template.yaml               (YAML, not markdown)
│   ├── review-checklist.md
│   ├── kanban-stages.yaml               (stages per kind)
│   ├── schemas/
│   │   ├── kanban.schema.json
│   │   ├── prd-frontmatter.schema.json
│   │   ├── spec-frontmatter.schema.json
│   │   ├── epic-frontmatter.schema.json
│   │   ├── task-frontmatter.schema.json
│   │   ├── teams.schema.json
│   │   ├── workspace-config.schema.json
│   │   ├── repo-vibeflow-yaml.schema.json
│   │   ├── agent-manifest.schema.json
│   │   └── skill-manifest.schema.json
│   ├── templates/
│   │   ├── code/
│   │   │   ├── service/
│   │   │   │   ├── template.yaml
│   │   │   │   ├── .claude/CLAUDE.md.tmpl
│   │   │   │   ├── docs/PRD.md.tmpl
│   │   │   │   ├── docs/specs/.gitkeep
│   │   │   │   ├── docs/adr/.gitkeep
│   │   │   │   ├── docs/wiki/.gitkeep
│   │   │   │   ├── plans/.gitkeep
│   │   │   │   ├── plans/tasks/.gitkeep
│   │   │   │   ├── .vibeflow.yaml.tmpl
│   │   │   │   ├── .gitignore.tmpl
│   │   │   │   ├── .gitattributes
│   │   │   │   └── README.md.tmpl
│   │   │   ├── library/
│   │   │   ├── web-app/
│   │   │   └── cli/
│   │   ├── product/.gitkeep             (Phase 5+ placeholder)
│   │   └── design/.gitkeep              (Phase 5+ placeholder)
│   └── agents/                          (starter agents, Phase 1 ships them)
│       └── .gitkeep
├── epics/
│   └── .gitkeep
├── wiki/
│   ├── architecture/README.md
│   ├── decisions/README.md
│   └── glossary.md
├── plugins/
│   └── .gitkeep
├── teams.yaml                           (empty skeleton)
├── README.md
└── .gitignore
```

## Related Files

**Create** (all in `meta-template/` repo):
- Schema files (10) in `standards/schemas/`
- Template files (4 code templates × ~14 files each = ~56 files)
- Standards docs (~7 files)
- Wiki starter files (3 files)
- Root files (teams.yaml, README, .gitignore, config.yaml)

**Reference** (brainstorm sections):
- §4.3, §4.4 (meta-repo + product repo structure)
- §12.3 (PRD schema), §12.5 (metrics schema), §12.7 (spec schema)
- §17.12 (teams.yaml full schema)
- §13.7 (template manifest)
- §18.4 (agent manifest)

## Implementation Steps

1. **Create `meta-template` repo** (separate git repo, ships as starter)
   - `git init meta-template && cd meta-template`
   - Root `.gitignore`, `README.md` explaining what this template is
   - Initial `.vibeflow/config.yaml` with schema_version

2. **Write JSON schemas** (10 files):
   - `kanban.schema.json` — kanban.yaml structure (index of tasks)
   - `task-frontmatter.schema.json` — per-task YAML schema
   - `epic-frontmatter.schema.json` — epic markdown frontmatter
   - `prd-frontmatter.schema.json` — PRD frontmatter (from brainstorm §12.3)
   - `spec-frontmatter.schema.json` — spec frontmatter (from §12.7)
   - `teams.schema.json` — teams.yaml full (from §17.12)
   - `workspace-config.schema.json` — `.vibeflow/config.yaml`
   - `repo-vibeflow-yaml.schema.json` — product repo `.vibeflow.yaml`
   - `agent-manifest.schema.json` — from §18.4
   - `skill-manifest.schema.json` — from §18.4
   - Each MUST include `$schema: http://json-schema.org/draft-07/schema#`
   - ⓘ **Red team (Assumption #8)**: Use `enum: ["1.0"]` on `schema_version` (NOT `const: "1.0"`) — allows future evolution without retroactive breakage of existing files

3. **Write markdown templates** (standards/):
   - `prd-template.md` with required sections (§12.2 content) + frontmatter docs in comments
   - `spec-template.md` (§12.6)
   - `epic-template.md`
   - `review-checklist.md`
   - `kanban-stages.yaml` — define stages for `code`/`product`/`design` (even if only code used in MVP)

4. **Write ONE generic code project template** (`standards/templates/code/default/`):
   - ⓘ **Red team #2/#5**: was 4 templates (service/library/web-app/cli), cut to 1. `--tech-stack` flag drives README.md content differences.
   - Has `template.yaml` manifest
   - Has `.claude/CLAUDE.md.tmpl`, `docs/PRD.md.tmpl`, `.vibeflow.yaml.tmpl`, `.gitignore.tmpl`, `README.md.tmpl`
   - Go template variables: `{{ .project_name }}`, `{{ .kind }}`, `{{ .team }}`, `{{ .epic }}`, `{{ .tech_stack }}`, `{{ .owner }}`, `{{ .workspace_path }}`, `{{ .date }}`, `{{ .quarter }}`
   - ⓘ **Security**: strict regex `^[a-zA-Z0-9_-]{1,64}$` on `project_name`, `team`, `owner` at CLI flag parse time (defense against template injection)

5. **Write workspace skeleton** (teams.yaml, wiki starters):
   - `teams.yaml` — empty skeleton with schema_version + comments pointing to docs
   - `wiki/architecture/README.md` — "Put architecture docs here"
   - `wiki/decisions/README.md` — "Put ADRs here"
   - `wiki/glossary.md` — empty

6. **Validate all schemas**:
   - Use `ajv` CLI or Go `github.com/santhosh-tekuri/jsonschema` to validate schemas compile
   - Cross-test: create sample valid + invalid YAML for each schema, confirm validation works
   - Commit test fixtures in `standards/schemas/tests/`

7. **Document progressive enforcement modes** (§12.9):
   - Add section to `standards/README.md` explaining `advisory → block_new → block_edit → ci_gate`
   - Config in `.vibeflow/config.yaml`

8. **Commit and tag**:
   - `git tag standards-v1.0`
   - This repo will be consumed by CLI `vibeflow workspace init`

## Todo List

- [ ] Create `meta-template/` repo
- [ ] Write 10 JSON schemas in `standards/schemas/`
- [ ] Write PRD template with required sections
- [ ] Write spec template with required sections
- [ ] Write epic template
- [ ] Write review checklist
- [ ] Write kanban-stages.yaml (3 kinds)
- [ ] Scaffold `templates/code/service/` (14 files)
- [ ] Scaffold `templates/code/library/` (12 files)
- [ ] Scaffold `templates/code/web-app/` (13 files)
- [ ] Scaffold `templates/code/cli/` (12 files)
- [ ] Write each template's `template.yaml` manifest
- [ ] Write teams.yaml skeleton
- [ ] Write wiki starter files
- [ ] Write workspace config schema + sample
- [ ] Validate all schemas compile (ajv or Go lib)
- [ ] Create test fixtures (valid/invalid samples per schema)
- [ ] Document progressive enforcement modes
- [ ] Tag `standards-v1.0`

## Success Criteria

- All 10 JSON schemas validate without errors
- Sample PRD passes `prd-frontmatter.schema.json` validation
- Sample invalid PRD fails with clear error messages
- `templates/code/service/` can be manually rendered with mock variables (Go text/template or cookiecutter)
- Meta-template repo cloneable and readable
- Schema test fixtures pass in CI (even if CI is just a Makefile locally for now)
- Brainstorm §12, §17.12, §18.4 specs match what's in `standards/schemas/`

## Risks

| Risk | Mitigation |
|---|---|
| Schema design flaws block Phase 1 | Validate with real example PRDs before freezing |
| Templates too rigid for real projects | Leave optional sections liberal, enforce only essentials |
| Go template syntax conflicts with markdown (e.g. `{{` inside prose) | Use `{{/* */}}` escape sequences, document |
| Missed required field → retroactive migration painful | Over-require initially, easier to relax than tighten |
| kind namespacing unused for MVP (code only) | Keep placeholder dirs anyway — costs nothing |

## Security Considerations

- No secrets in templates (no API keys, no credentials)
- `.gitignore.tmpl` excludes `.env`, `*.pem`, `token.json`, `secrets/`
- Schemas explicitly mark sensitive fields (e.g., `teams.yaml` members are PII)
- Document that meta-template is public/shareable (no org-specific data)

## Next Steps

- Phase 1 (Go CLI) consumes these templates as embedded resources or filesystem references
- CLI validates files against these schemas
- Server (Phase 2) parses YAML/markdown using same schemas
