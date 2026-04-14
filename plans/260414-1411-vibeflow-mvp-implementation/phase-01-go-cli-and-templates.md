---
phase: 1
title: Go CLI (3 commands)
status: pending
effort: 4 days
depends_on: [phase-00]
---

# Phase 1: Go CLI (3 commands)

## Context Links

- Spec: [../../docs/specs/spec-cli.md](../../docs/specs/spec-cli.md) — authoritative contract
- PRD: [../../docs/PRD-vibeflow-mvp.md](../../docs/PRD-vibeflow-mvp.md)
- Phase 0: schemas + templates (embedded here)

## Overview

**Priority**: P0 — first user-facing deliverable.
**Status**: pending
**Description**: Single static Go binary `vibeflow` providing `init`, `link`, `lint` commands plus `claude setup` helper. Embeds Phase 0 schemas + templates via `go:embed`. Zero runtime dependencies except `git` (for init post-hook). Cross-compiles to 5 platforms.

## Key Insights

- Single binary distribution is the user-facing killer feature (zero install friction)
- All schemas + templates embedded at compile time — no network fetch, no external repo dependency
- Stub commands only where PRD explicitly includes them: `init`, `link`, `lint`, `claude setup`, `version`, `help`. No `doctor`, no `config`, no `agents`, no `workspace init`, no stub auth.
- Schema validator shared with Phase 2 MCP server via internal package
- Deliberately terse error messages + hints (per CLI spec §11)

## Requirements

**Functional**:
- `vibeflow init <name> [flags]` — scaffold repo from embedded template
- `vibeflow link [flags]` — register repo in workspace `teams.yaml`
- `vibeflow lint [path] [flags]` — validate PRD/spec/.vibeflow.yaml against schemas
- `vibeflow claude setup [flags]` — write MCP entry to `~/.claude/settings.json`
- `vibeflow version`, `vibeflow help` — trivial cobra defaults
- `--dry-run` on init and link
- `--format` on lint (text | json | github-annotations)
- Exit codes per spec (0, 1, 2, 3, 7)

**Non-functional**:
- Cold start < 100ms
- `init` end-to-end < 2s including `git init`
- Binary < 20 MB after strip
- Cross-compile for darwin-arm64, darwin-amd64, linux-arm64, linux-amd64, windows-amd64
- Pass CI tests on all 5 platforms

## Architecture

```
vibeflow/                              (monorepo root)
├── cli/
│   ├── main.go                        (entry, thin)
│   ├── cmd/
│   │   ├── root.go                    (cobra root, flags, version)
│   │   ├── init.go
│   │   ├── link.go
│   │   ├── lint.go
│   │   └── claude.go                  (claude setup subcommand)
│   ├── internal/
│   │   ├── config/
│   │   │   ├── load.go                (env + flag precedence)
│   │   │   └── types.go               (CliConfig struct)
│   │   ├── workspace/
│   │   │   ├── detect.go              (walk-up search for .vibeflow.yaml)
│   │   │   ├── teams-yaml.go          (parse + update teams.yaml)
│   │   │   └── repo.go                (read .vibeflow.yaml)
│   │   ├── template/
│   │   │   ├── render.go              (Go text/template with safe funcs)
│   │   │   ├── walker.go              (walk embedded FS, render tree)
│   │   │   └── funcs.go               (default, join, lower, date helpers)
│   │   ├── schema/
│   │   │   ├── validator.go           (jsonschema lib wrapper)
│   │   │   └── loader.go              (load embedded schemas at startup)
│   │   ├── lint/
│   │   │   ├── engine.go              (run rules, collect diagnostics)
│   │   │   ├── frontmatter.go         (parse YAML frontmatter)
│   │   │   ├── sections.go            (heading tree extraction)
│   │   │   ├── metrics-table.go       (table parser for Success Metrics)
│   │   │   └── rules.go               (rule definitions)
│   │   ├── gitops/
│   │   │   └── shellout.go            (git init, add, commit via exec.Command)
│   │   ├── errors/
│   │   │   └── typed.go               (error types with hints + exit codes)
│   │   └── output/
│   │       ├── text.go                (colored, iconed)
│   │       └── json.go                (stable schema)
│   ├── schemas/                       (from Phase 0, go:embed)
│   ├── templates/                     (from Phase 0, go:embed)
│   ├── go.mod
│   ├── go.sum
│   └── main_test.go
└── .github/workflows/cli-release.yml  (goreleaser matrix build)
```

**Dependencies** (minimal):
- `github.com/spf13/cobra` — command framework
- `github.com/santhosh-tekuri/jsonschema/v5` — JSON schema validation
- `gopkg.in/yaml.v3` — YAML parsing
- `github.com/goccy/go-yaml` — YAML edit-in-place preserving comments (for `link`)

Standard library for everything else. No survey (use plain prompts if needed), no viper (env + flag precedence by hand), no lipgloss (ANSI escape codes directly).

## Related Files

**Create**:
- `cli/main.go` + `cli/cmd/*.go` (5 files)
- `cli/internal/**/*.go` (~15 files)
- `cli/main_test.go` + integration tests (~5 files)
- `cli/go.mod`, `go.sum`
- `.github/workflows/cli-release.yml` (goreleaser)
- `.goreleaser.yml`
- `cli/README.md`

**Reference** (already exists):
- `cli/schemas/*.json` (from Phase 0)
- `cli/templates/code/default/*` (from Phase 0)

## Implementation Steps

1. **Bootstrap Go module**:
   - `cd cli && go mod init github.com/lukebaze/vibeflow/cli`
   - Add minimal deps (cobra, jsonschema, yaml.v3, go-yaml)
   - Setup `main.go` with cobra root command + version

2. **Embed assets**:
   - Add `//go:embed schemas/*.json` and `//go:embed templates/code/default/**/*` directives
   - Test: `go build` succeeds and binary includes files

3. **Schema validator wrapper**:
   - `internal/schema/loader.go` — parse all embedded schemas at startup
   - `internal/schema/validator.go` — `Validate(schemaName, data)` returns typed errors
   - Unit test against Phase 0 fixtures (valid + invalid)

4. **Workspace detection**:
   - `internal/workspace/detect.go` — walk up from cwd looking for `.vibeflow.yaml`
   - `internal/workspace/teams-yaml.go` — read/write preserving comments via go-yaml

5. **Template renderer**:
   - `internal/template/render.go` — Go text/template with safe funcs
   - `internal/template/walker.go` — walk embedded FS, render `.tmpl` files, copy others
   - Strict regex on template vars at input stage (defense against injection)

6. **Lint engine**:
   - `internal/lint/frontmatter.go` — parse YAML frontmatter between `---` delimiters
   - `internal/lint/sections.go` — walk markdown headings, build section tree
   - `internal/lint/metrics-table.go` — parse pipe-syntax tables under `## Success Metrics`
   - `internal/lint/rules.go` — define all rules from spec-standards.md §Lint rules
   - `internal/lint/engine.go` — run all rules, return sorted diagnostics

7. **`init` command**:
   - Parse flags (`name`, `--team`, `--description`, `--yes`, `--dry-run`)
   - Validate name regex `^[a-z0-9-]{1,64}$`
   - Check target dir doesn't exist
   - Walk embedded template tree, render each file with vars
   - Run `git init && git add -A && git commit -m "..."` via shellout
   - Print success + next-steps hint

8. **`link` command**:
   - Detect workspace from `VIBEFLOW_WORKSPACE` or `--workspace` flag
   - Read repo's `.vibeflow.yaml` (error if missing)
   - Read workspace's `teams.yaml` (error if missing or malformed)
   - Append repo name to `teams[team].repos` (create team if missing)
   - Write back preserving comments
   - Print reminder: "commit workspace changes manually"

9. **`lint` command**:
   - Default path: current working dir, find `docs/PRD.md`, `docs/specs/*.md`, `.vibeflow.yaml`
   - Explicit path: single file
   - Run lint engine, print diagnostics
   - Exit 7 on any error, 0 on pass
   - `--format json` outputs stable schema
   - `--format github-annotations` outputs `::error::` lines for GitHub Actions

10. **`claude setup` command**:
    - Read `~/.claude/settings.json` (create if missing)
    - Parse JSON preserving other entries
    - Add/update `mcpServers.vibeflow` entry with:
      - `command`: path to `vibeflow-mcp` binary (Phase 2, if separate) OR `command: "vibeflow", args: ["mcp-server"]`
      - `env: { VIBEFLOW_WORKSPACE: <current env value or prompt> }`
    - Write back with 2-space indent
    - Print reminder: "restart Claude Code to activate"

11. **Output system**:
    - Text mode with ANSI colors (`\033[...]`) gated on `isatty`
    - JSON mode stable schema
    - Typed errors with hints

12. **Testing**:
    - Unit: schema validator, lint engine, template renderer, frontmatter parser
    - Integration: end-to-end `init` → `link` → `lint` on temp dir
    - Integration: `claude setup` with existing + missing settings.json
    - CI matrix: 5 platforms

13. **Release pipeline**:
    - `.goreleaser.yml` with matrix build
    - `.github/workflows/cli-release.yml` triggers on git tag
    - Artifacts uploaded to GitHub Releases
    - Checksum file (sha256)
    - Homebrew tap update (optional, v0.1.0 can be manual)

14. **Dogfood check**:
    - Run `vibeflow lint docs/PRD-vibeflow-mvp.md` — must pass
    - Run `vibeflow init test-demo` — must produce valid repo
    - If either fails, fix before merging

## Todo List

- [ ] Go module bootstrap + deps
- [ ] Embed schemas + templates via go:embed
- [ ] Schema validator package
- [ ] Workspace detection package
- [ ] teams.yaml read/write (comment-preserving)
- [ ] Template renderer with safe funcs
- [ ] Lint engine: frontmatter parser
- [ ] Lint engine: section extractor
- [ ] Lint engine: metrics table parser
- [ ] Lint engine: rule runner
- [ ] `init` command end-to-end
- [ ] `link` command end-to-end
- [ ] `lint` command with all formats
- [ ] `claude setup` command (settings.json read/merge/write)
- [ ] Output system (text/JSON + exit codes)
- [ ] Unit tests for all internal packages
- [ ] Integration test: init → lint → link
- [ ] Integration test: claude setup
- [ ] `.goreleaser.yml` cross-compile config
- [ ] GitHub Actions release workflow
- [ ] CI matrix test on macOS + Linux + Windows
- [ ] Dogfood: `vibeflow lint docs/PRD-vibeflow-mvp.md` passes
- [ ] README with install + quickstart

## Success Criteria

- `vibeflow init demo --yes` produces a working repo in <2s
- `vibeflow lint docs/PRD.md` catches missing sections with specific errors, exit 7
- `vibeflow link` registers repo in `teams.yaml` idempotently
- `vibeflow claude setup` writes valid MCP entry to `~/.claude/settings.json` without clobbering other entries
- Binary <20 MB, starts <100ms on macOS M1
- All 5 platform builds pass CI
- `vibeflow lint` runs clean on `docs/PRD-vibeflow-mvp.md` (dogfood)
- Release v0.1.0 published to GitHub Releases

## Risks

| Risk | Mitigation |
|---|---|
| `go:embed` binary bloat | Measure, strip unused, target <20 MB |
| YAML comment preservation tricky | Use `goccy/go-yaml` (not yaml.v3), test thoroughly |
| Windows path handling bugs | CI matrix from day 1, use `filepath` package correctly |
| Cobra help formatting inconsistencies | Customize help template minimally |
| Template injection via `project_name` | Strict regex at flag parse, not at render |
| `claude setup` clobbers other MCP entries | Read-merge-write pattern with JSON preservation |
| Git shellout fails (git not installed) | Error with clear hint "install git from https://git-scm.com" |

## Security Considerations

- Template variables validated via strict regex BEFORE render
- Git commit message passed via `-F` with temp file, not inline
- `.gitignore` template excludes `.env*`, `*.pem`, `*.key`, `token.json`
- `~/.claude/settings.json` written with mode 600
- No network calls at runtime (all assets embedded)
- Path traversal defense: refuse `init` names containing `..`, `/`, `\`

## Next Steps

- Phase 2 (MCP stdio server) reuses `cli/internal/schema/` and `cli/internal/workspace/` packages
- Phase 2 is a separate binary OR subcommand `vibeflow mcp-server` — decision at Phase 2 start
