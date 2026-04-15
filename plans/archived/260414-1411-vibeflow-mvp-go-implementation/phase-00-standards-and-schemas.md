---
phase: 0
title: Standards + Schemas
status: pending
effort: 2 days
depends_on: []
---

# Phase 0: Standards + Schemas

## Context Links

- Spec: [../../docs/specs/spec-standards.md](../../docs/specs/spec-standards.md) — authoritative contract
- PRD: [../../docs/PRD-vibeflow-mvp.md](../../docs/PRD-vibeflow-mvp.md)

## Overview

**Priority**: P0 blocker for Phase 1 + 2.
**Status**: pending
**Description**: Write 4 JSON schemas + 2 markdown templates directly into `cli/schemas/` and `cli/templates/` directories of the Vibeflow monorepo. These files will be embedded into the Go CLI binary via `go:embed` in Phase 1. Pure content work — no Go code, no infra, no external repo.

## Key Insights

- Standards live **inside the CLI project**, not in a separate `meta-template/` repo. Embedded via `go:embed` at compile time.
- Schema version uses `enum: ["1.0"]` (NOT `const: "1.0"`) — allows future minor bumps without breaking old files.
- `kind` field exists in schema but only allows `"code"` in MVP — preserves Phase 5+ extension path.
- No progressive enforcement modes in MVP — lint is advisory, user decides when to install as pre-commit hook.

## Requirements

**Functional**:
- 4 JSON schemas (Draft-07):
  - `prd-frontmatter.schema.json`
  - `spec-frontmatter.schema.json`
  - `vibeflow-yaml.schema.json`
  - `teams-yaml.schema.json`
- 2 markdown templates with required sections:
  - `prd-template.md` (Problem, Users, Success Metrics table, Scope, Non-Goals)
  - `spec-template.md` (Overview, API Contracts, Data Models, Edge Cases)
- 1 project template directory: `templates/code/default/` with scaffold files
- Fixture sets for testing (valid + invalid samples per schema)

**Non-functional**:
- All schemas validate with `ajv` or Go `github.com/santhosh-tekuri/jsonschema/v5`
- All templates parseable Go `text/template` syntax
- Fixture tests runnable via `go test` in Phase 1

## Architecture

**Location** (inside Vibeflow monorepo, created in Phase 1 but scaffolded here):
```
cli/
├── schemas/                              (created in this phase)
│   ├── prd-frontmatter.schema.json
│   ├── spec-frontmatter.schema.json
│   ├── vibeflow-yaml.schema.json
│   ├── teams-yaml.schema.json
│   └── fixtures/
│       ├── valid/
│       │   ├── prd-minimal.md
│       │   ├── prd-full.md
│       │   ├── spec-minimal.md
│       │   ├── vibeflow-yaml-minimal.yaml
│       │   └── teams-yaml-minimal.yaml
│       └── invalid/
│           ├── prd-missing-required.md
│           ├── prd-bad-status.md
│           ├── prd-wrong-quarter.md
│           ├── spec-bad-prd-ref.md
│           └── teams-yaml-bad-team-slug.yaml
└── templates/                            (created in this phase)
    └── code/
        └── default/
            ├── template.yaml             (manifest, minimal)
            ├── .claude/CLAUDE.md.tmpl
            ├── docs/PRD.md.tmpl
            ├── docs/specs/.gitkeep
            ├── docs/wiki/.gitkeep
            ├── plans/.gitkeep
            ├── .vibeflow.yaml.tmpl
            ├── .gitignore.tmpl
            └── README.md.tmpl
```

Total files created: ~20.

## Related Files

**Create** (all in Vibeflow monorepo):
- 4 JSON schemas in `cli/schemas/`
- ~10 fixture files in `cli/schemas/fixtures/`
- 1 template manifest + ~8 template files in `cli/templates/code/default/`
- 2 canonical markdown template reference docs (copied to `docs/templates/prd-template.md` and `docs/templates/spec-template.md` as human-readable refs)

**Reference** (already written as source of truth):
- `docs/specs/spec-standards.md` — full schema definitions and lint rules

## Implementation Steps

1. **Scaffold Vibeflow monorepo** (if not done):
   - Create `cli/` directory
   - Create `cli/schemas/`, `cli/templates/code/default/`
   - Add `.gitignore` entries for Go artifacts

2. **Write 4 JSON schemas** — copy exactly from `docs/specs/spec-standards.md`:
   - `prd-frontmatter.schema.json`
   - `spec-frontmatter.schema.json`
   - `vibeflow-yaml.schema.json`
   - `teams-yaml.schema.json`
   - Each with `$schema: http://json-schema.org/draft-07/schema#`
   - Each with `schema_version` as `enum: ["1.0"]`
   - Strict `additionalProperties: false`

3. **Write fixture sets**:
   - For each schema, create minimal valid sample + maximal valid sample
   - For each schema, create 2-3 invalid samples covering: missing required, wrong type, wrong enum, bad regex
   - Commit in `cli/schemas/fixtures/valid/` and `invalid/`

4. **Write PRD markdown template**:
   - `cli/templates/code/default/docs/PRD.md.tmpl`
   - Required sections: Problem, Users, Success Metrics, Scope, Non-Goals
   - Go template vars: `{{ .project_name }}`, `{{ .team }}`, `{{ .owner }}`, `{{ .date }}`, `{{ .description }}`
   - Include example Success Metrics table skeleton

5. **Write spec markdown template** (canonical reference, not in init scaffolding):
   - `docs/templates/spec-template.md` — human-readable reference
   - Required sections: Overview, API Contracts, Data Models, Edge Cases

6. **Write code project template files**:
   - `template.yaml` manifest — minimal metadata
   - `.claude/CLAUDE.md.tmpl` — project instructions with MCP tool usage hint
   - `.vibeflow.yaml.tmpl` — conforms to vibeflow-yaml.schema.json
   - `.gitignore.tmpl` — standard ignores + `.env*`, `*.pem`, `token.json`, `secrets/`
   - `README.md.tmpl` — quick-start stub
   - `.gitkeep` files for empty dirs

7. **Validate**:
   - Manually run `ajv validate -s <schema> -d <fixture>` for each schema+fixture pair
   - Render template via `gomplate` or manual Go text/template eval to verify syntax

8. **Write canonical reference copies** in `docs/templates/`:
   - `docs/templates/prd-template.md` (plain, for users to read)
   - `docs/templates/spec-template.md`

## Todo List

- [ ] Scaffold `cli/schemas/` directory
- [ ] Scaffold `cli/templates/code/default/` directory
- [ ] Write `prd-frontmatter.schema.json`
- [ ] Write `spec-frontmatter.schema.json`
- [ ] Write `vibeflow-yaml.schema.json`
- [ ] Write `teams-yaml.schema.json`
- [ ] Write 10+ valid fixture files
- [ ] Write 8+ invalid fixture files
- [ ] Write `PRD.md.tmpl` (Go template)
- [ ] Write `CLAUDE.md.tmpl`
- [ ] Write `.vibeflow.yaml.tmpl`
- [ ] Write `.gitignore.tmpl`, `README.md.tmpl`
- [ ] Write `template.yaml` manifest
- [ ] Create `.gitkeep` for empty dirs (specs/, wiki/, plans/)
- [ ] Write canonical `docs/templates/prd-template.md`
- [ ] Write canonical `docs/templates/spec-template.md`
- [ ] Validate all schemas against fixtures (manual ajv or Go test prep)

## Success Criteria

- `ajv validate -s cli/schemas/prd-frontmatter.schema.json -d cli/schemas/fixtures/valid/prd-minimal.md` passes
- Each invalid fixture fails validation with clear error
- PRD template renders without Go template errors using sample variables
- This MVP PRD (`docs/PRD-vibeflow-mvp.md`) passes validation against the schema (dogfood check)
- All files committed and ready for Phase 1 to `go:embed`

## Risks

| Risk | Mitigation |
|---|---|
| Schema design mistakes found late | Dogfood immediately: validate this plan's own PRD first |
| Template syntax conflicts with markdown `{{` | Use `{{/* */}}` escape, document in template |
| Required fields too strict → real PRDs fail | Under-require initially, tighten post-MVP |
| `enum: ["1.0"]` vs `const: "1.0"` confusion | Document in spec-standards.md why enum (red team finding) |

## Security Considerations

- No secrets in any template file
- `.gitignore.tmpl` excludes `.env*`, `*.pem`, `token.json`, `secrets/`
- Schemas validate but do NOT execute user input
- Template vars quoted in YAML contexts to prevent injection

## Next Steps

- Phase 1 `go:embed`s these files into the CLI binary
- Phase 1 validates against these schemas at `lint` time
- Phase 2 MCP server reuses schema validator for `workspace_context` responses
