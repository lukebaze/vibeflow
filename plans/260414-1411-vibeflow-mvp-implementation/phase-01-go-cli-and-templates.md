---
phase: 1
title: Go CLI (4 commands)
status: pending
effort: 1-1.5 weeks
depends_on: [phase-00]
---

# Phase 1: Go CLI (4 commands)

## ⓘ Red Team Changes (Applied 2026-04-14)

**Finding #2 (Critical)**: Massive scope cut. Was 12+ command groups; MVP ships only 4 essentials.

**KEEP (MVP)**:
- `vibeflow init <name>` — scaffold new repo
- `vibeflow link` — adopt existing repo into workspace
- `vibeflow lint [path]` — validate PRD/spec/kanban against schemas
- `vibeflow claude setup` — register MCP server in `~/.claude/settings.json`
- `vibeflow version`, `vibeflow help` (trivial cobra defaults)

**CUT (defer to post-MVP)**:
- `vibeflow workspace init` → manual: clone meta-template, edit teams.yaml (document in README)
- `vibeflow doctor` (nice-to-have)
- `vibeflow config get/set/list` (use env vars + `.vibeflow.yaml` directly)
- `vibeflow agents list/status/pin/unpin/exclude/sync` — keep only `agents sync` (simple file copy from local meta-repo clone); drop CRUD on pins
- `vibeflow login/logout/whoami` stub (Phase 4 adds real OIDC)
- `vibeflow completion <shell>` (polish)
- `--json` on every command → keep only on `lint` (for CI integration)
- Fuzzy-match typo suggestions (polish)
- Lipgloss styling + go-pretty tables → plain stderr + simple alignment
- Interactive prompts — pure flags first (remove survey lib dependency)
- Dry-run on every destructive command → only on `init` (which has side effects)

**Effort**: 2-3w → 1-1.5w (halved via scope cut).

## <!-- Updated: Validation Session 1 — manual sync + embedded templates -->

**5th MVP command added** (from validation):
- `vibeflow sync [repo]` — triggers server resync via `POST /admin/sync/:repo` API call. Replaces the polling daemon entirely. Users run manually after pushing to meta-repo or product repo. Optional: install as git post-commit hook.

**Templates embedded in CLI via `go:embed`** (validation #6):
- No separate `meta-template/` repo dependency
- `cli/templates/` + `cli/schemas/` + `cli/agents/` (1 starter) + `cli/workspace-skeleton/` (for `vibeflow workspace init`, if kept) all compiled into single binary
- `vibeflow init` extracts from binary
- CLI release = template release (versioned together)

**`vibeflow workspace init` RE-ADDED to MVP** (validation #6 dep):
- Was cut by red team, but with embedded templates it's trivial: scaffold workspace skeleton from `cli/workspace-skeleton/` into new directory
- No GitHub API calls in MVP version — user creates GitHub repo manually, then runs `vibeflow workspace init` locally + `git push`
- Removes manual `clone meta-template && edit teams.yaml` step entirely

**Final MVP CLI surface** (5 commands):
- `vibeflow init <name>`
- `vibeflow link`
- `vibeflow lint [path]`
- `vibeflow sync [repo]` ← NEW from validation
- `vibeflow claude setup`
- `vibeflow workspace init <name>` ← RE-ADDED from validation
- Plus `vibeflow version`, `vibeflow help`

**Effort**: 1-1.5w → **1w** (embedded templates simplify scaffolding; manual sync adds ~0.5d of CLI work but offsets against drop of daemon).

## Remaining Content (updated to reflect cuts)
---

## Context Links

- Brainstorm §13 CLI command surface
- Brainstorm §13.7 Template rendering
- Brainstorm §16.4 Meta-repo init flow
- Brainstorm §16.7 Repo link/adopt flow
- Phase 0: standards + schemas (consumed here)

## Overview

**Priority**: P0, first user-facing deliverable.
**Status**: pending
**Description**: Go CLI binary (`vibeflow`) providing scaffolding, linting, workspace init, repo link, and agent sync. Ships templates from Phase 0 embedded in binary. Works offline — no server dependency.

## Key Insights

- Go chosen for single static binary (§13.13 distribution). Zero install friction for devs.
- Templates embedded via `go:embed` directive → binary self-contained.
- CLI works offline for all local ops (init, lint, template render). Server only needed for MCP integration (set up via `claude setup`).
- Schema validation uses Go JSON Schema lib (`github.com/santhosh-tekuri/jsonschema/v5`).
- ⓘ **Red team**: No auth in Phase 1 at all (cut stub). Phase 4 adds real OIDC directly.
- Command pattern `vibeflow <noun> <verb>` for future extension.
- ⓘ **Security (merged finding)**: Template variable escaping — strict regex validation at CLI flag parse, no user input in shell commands (use `git commit -F` with stdin not inline).

## Requirements

**Functional**:
- `vibeflow init <name>` — scaffold repo with `--kind code` (MVP), template selection, team, epic
- `vibeflow workspace init <name>` — scaffold meta-repo from embedded skeleton
- `vibeflow link` — adopt existing repo into workspace
- `vibeflow lint [path]` — validate PRD, spec, task, kanban against schemas
- `vibeflow config get/set/list` — manage `~/.config/vibeflow/config.yaml`
- `vibeflow agents sync` — pull latest agents from meta-repo (local git ops only, no server)
- `vibeflow agents status/list/pin/unpin/exclude` — manage `.vibeflow.yaml` agent pins
- `vibeflow doctor` — health check (git, config, workspace detection)
- `vibeflow claude setup` — register MCP server in `~/.claude/settings.json`
- `vibeflow whoami/login/logout` — stub auth (real OIDC in Phase 4)
- `vibeflow version`, `vibeflow help`, `vibeflow completion <shell>`

**Non-functional**:
- <100ms startup (cold)
- Single static binary, cross-compile macOS/Linux/Windows, arm64/amd64
- Stable exit codes (§13.10)
- `--json` output mode on every command
- Clear error UX with hints + fuzzy matching (§13.11)
- Dry-run support on destructive commands

## Architecture

**Monorepo structure** (Vibeflow project itself):
```
vibeflow/                                ← main project repo
├── cli/                                 ← Go CLI
│   ├── cmd/
│   │   ├── root.go
│   │   ├── init.go
│   │   ├── workspace.go
│   │   ├── link.go
│   │   ├── lint.go
│   │   ├── config.go
│   │   ├── agents.go
│   │   ├── doctor.go
│   │   ├── claude.go
│   │   ├── auth.go
│   │   └── version.go
│   ├── internal/
│   │   ├── config/          (config precedence, loading)
│   │   ├── workspace/       (workspace detection, meta-repo ops)
│   │   ├── template/        (Go template rendering, embedded fs)
│   │   ├── schema/          (JSON schema validation)
│   │   ├── lint/            (PRD/spec lint rules)
│   │   ├── git/             (git ops via go-git or shellout)
│   │   ├── auth/            (token storage, OIDC stub)
│   │   ├── agents/          (agent pull, manifest, pins)
│   │   ├── prompt/          (interactive prompts, survey lib)
│   │   ├── output/          (text/json formatting, colors)
│   │   └── errors/          (error codes, hints, fuzzy match)
│   ├── templates/           ← embedded via go:embed
│   │   └── (copy from meta-template/standards/templates/code/)
│   ├── schemas/             ← embedded via go:embed
│   │   └── (copy from meta-template/standards/schemas/)
│   ├── go.mod
│   ├── go.sum
│   └── main.go
├── server/                              ← Phase 2+ (placeholder)
├── meta-template/                       ← from Phase 0 (reference or submodule)
└── README.md
```

**Key dependencies** (Go):
- `github.com/spf13/cobra` — command framework
- `github.com/spf13/viper` — config loading (precedence)
- `github.com/santhosh-tekuri/jsonschema/v5` — JSON schema validation
- `github.com/AlecAivazis/survey/v2` — interactive prompts
- `github.com/go-git/go-git/v5` — git ops in Go (or shellout to `git` binary for simplicity)
- `github.com/charmbracelet/lipgloss` — terminal styling
- `gopkg.in/yaml.v3` — YAML parsing
- `github.com/mitchellh/mapstructure` — config → struct

## Related Files

**Create**:
- `cli/main.go` (entry)
- `cli/cmd/*.go` (10+ command files)
- `cli/internal/**/*.go` (~15 packages)
- `cli/templates/` (embed dir, copied from Phase 0)
- `cli/schemas/` (embed dir, copied from Phase 0)
- `cli/go.mod`, `go.sum`
- `.github/workflows/cli-release.yml` (goreleaser cross-compile)

**Reference**:
- `meta-template/` from Phase 0 (sync templates/schemas via CI)

## Implementation Steps

1. **Project scaffold**:
   - `cd vibeflow && mkdir cli && cd cli`
   - `go mod init github.com/company/vibeflow/cli`
   - Set up `main.go` with `cobra` root command
   - Embed templates + schemas via `go:embed`

2. **Config system** (`internal/config/`):
   - Precedence: flag > env > project `.vibeflow.yaml` > user `~/.config/vibeflow/config.yaml` > system > baked default
   - Use viper with custom precedence chain
   - Schema-validated against `workspace-config.schema.json`

3. **Workspace detection** (`internal/workspace/`):
   - Walk up from CWD looking for meta-repo marker (`.vibeflow/config.yaml`)
   - Read teams.yaml to get team/repo list
   - Expose API: `Detect()`, `GetMeta()`, `GetTeams()`, `GetRepos()`

4. **Template rendering engine** (`internal/template/`):
   - Use `text/template` stdlib
   - Load from embedded FS
   - Variable substitution with custom funcs (`default`, `join`, `lower`, etc.)
   - Conditional file inclusion (e.g., `_skip_if_no_db/` dirs)
   - Post-render hooks (git init, git commit)

5. **Schema validation** (`internal/schema/`):
   - Load all schemas from embedded FS on startup
   - `Validate(schemaName, data)` returns structured errors with line numbers
   - Used by lint and template rendering sanity checks

6. **Lint engine** (`internal/lint/`):
   - Parse markdown frontmatter (YAML) + body sections
   - Extract heading structure for required-section check
   - Parse tables under known headings (Success Metrics)
   - Run rules from brainstorm §12.8
   - Severity levels: error/warn/info
   - Output formats: text (default), json, github-annotations (for CI)

7. **Init command** (`cmd/init.go`):
   - Flags: `--kind`, `--template`, `--team`, `--epic`, `--tech-stack`, `--description`, `--no-git`, `--no-workspace-pr`, `--yes`, `--dry-run`, `--json`
   - Interactive prompts via survey lib if missing flags (unless `--yes`)
   - Template selection from embedded `templates/code/`
   - Render files to target dir
   - Run post-render hooks (git init, commit)
   - Update meta-repo teams.yaml + push PR (optional)
   - Brainstorm §13.3 flow exactly

8. **Workspace init** (`cmd/workspace.go`):
   - `vibeflow workspace init <name>` scaffolds meta-repo from embedded skeleton
   - Prompts: department name, GitHub org, repo name, initial teams
   - Creates GitHub repo via API (requires PAT/token)
   - Scaffolds files from embedded `meta-template/` copy
   - Commits + pushes
   - Brainstorm §16.4 flow

9. **Link command** (`cmd/link.go`):
   - Adopt existing repo: detect workspace, create `.vibeflow.yaml`, migrate old docs if found
   - `--migrate` flag attempts PRD format migration (best-effort, user reviews)
   - Commit as "feat: adopt vibeflow workspace structure"
   - PR to meta-repo teams.yaml (optional via `--no-workspace-pr`)

10. **Lint command** (`cmd/lint.go`):
    - Default: lint all files in current project (`docs/PRD.md`, `docs/specs/*.md`, `plans/tasks/*.yaml`)
    - Path arg: lint specific file
    - `--fix` flag auto-fixes trivial issues (missing `schema_version`, sort frontmatter)
    - `--format github-annotations` for CI output
    - Exit code 7 on validation error

11. **Agents commands** (`cmd/agents.go`):
    - `list` — from meta-repo/standards/agents/ manifest
    - `status` — read `.claude/.managed-manifest.yaml` vs current meta-repo state
    - `sync` — copy latest agent files from local meta-repo clone, update manifest
    - `pin/unpin/exclude` — edit `.vibeflow.yaml`
    - No server involvement (git-based only, assumes meta-repo accessible locally)

12. **Doctor command** (`cmd/doctor.go`):
    - Check: git version, workspace detected, config valid, schema-validate `.vibeflow.yaml`, token present (stub auth), MCP registered
    - Warning on stale wiki cache (>N days)
    - `--fix` flag to auto-fix trivial issues

13. **Claude setup** (`cmd/claude.go`):
    - Read/write `~/.claude/settings.json`
    - Add/update `mcpServers.vibeflow` entry pointing to configured server URL
    - Handle missing file gracefully
    - Idempotent

14. **Auth stub** (`cmd/auth.go` + `internal/auth/`):
    - `login` — stub: accepts email, generates fake token, saves to `~/.config/vibeflow/token.json` (mode 600)
    - `whoami` — reads token file, shows email
    - `logout` — deletes token file
    - Document that real OIDC arrives in Phase 4
    - Structure code so OIDC swap is a single package change

15. **Output system** (`internal/output/`):
    - Text mode: colors via lipgloss, icons, tables (go-pretty)
    - JSON mode: stable schema, versioned
    - Exit codes centralized (§13.10)
    - Error format: `✗ Error: ... Hint: ... Docs: ...`

16. **Shell completion** (`cmd/completion.go`):
    - Cobra native support for bash/zsh/fish
    - Dynamic completion for team names (from local teams.yaml), epic IDs (from local meta-repo cache)

17. **Build + release pipeline**:
    - `.github/workflows/cli-release.yml` using goreleaser
    - Matrix: darwin-arm64, darwin-amd64, linux-arm64, linux-amd64, windows-amd64
    - Sign with cosign (optional, defer if complex)
    - Release artifacts to GitHub Releases
    - Homebrew tap (optional Phase 1.1)

## Todo List

- [ ] Scaffold `cli/` Go module + cobra root
- [ ] Embed templates + schemas via go:embed
- [ ] Config system with precedence chain
- [ ] Workspace detection (walk-up to meta-repo)
- [ ] Template rendering engine (text/template + custom funcs)
- [ ] Schema validation wrapper
- [ ] Lint engine (frontmatter + sections + rules)
- [ ] `init` command end-to-end
- [ ] `workspace init` command
- [ ] `link` command (adopt existing)
- [ ] `lint` command with `--fix`
- [ ] `agents` subcommands
- [ ] `doctor` command
- [ ] `claude setup` command
- [ ] `login/logout/whoami` stubs
- [ ] `config` subcommands
- [ ] Output system (text/json)
- [ ] Shell completion
- [ ] GoReleaser config + CI release pipeline
- [ ] Cross-platform smoke test (macOS, Linux, Windows)
- [ ] README + docs for each command

## Success Criteria

- `vibeflow init test-repo --kind code --template service --yes` produces a working repo in <2s
- `vibeflow lint docs/PRD.md` catches missing required sections with specific errors
- `vibeflow lint --fix` repairs trivial issues without breaking formatting
- `vibeflow workspace init my-dept` scaffolds a meta-repo skeleton
- `vibeflow agents sync` updates `.claude/agents/` from local meta-repo
- `vibeflow doctor` reports all green on a fresh setup
- Binary <20 MB, starts in <100ms
- CI builds + tests on 3 OS, releases pinned artifacts
- All commands support `--json` output

## Risks

| Risk | Mitigation |
|---|---|
| Template rendering edge cases (special chars, markdown vs Go template conflict) | Unit tests with adversarial inputs, use `{{/* */}}` escapes |
| go:embed binary bloat | Measure, strip unused templates, acceptable up to ~30MB |
| Schema validation error messages unclear | Test with sample invalid inputs, tune error formatting |
| Git ops via go-git buggy vs shellout | Start shellout (simpler), migrate later if needed |
| Interactive prompts conflict with `--yes` | Unit test both paths, CI runs non-interactive |
| Windows path handling bugs | CI matrix test from day 1 |
| Cobra fuzzy matching false positives | Use Levenshtein threshold, show top-3 suggestions |
| Auth stub leaks into production | Separate package boundary, Phase 4 swaps cleanly |

## Security Considerations

- `~/.config/vibeflow/token.json` mode 600
- No telemetry MVP (opt-in Phase 2+ if needed)
- `vibeflow init --dry-run` never writes or commits
- Schema validation before any file write (defensive)
- CODEOWNERS file generated in repo templates (Phase 4 activates enforcement)
- Signed release binaries (cosign, if time permits)

## Next Steps

- Phase 2 (MCP server) starts once CLI can produce valid product repos + meta-repos
- Phase 2 reuses schemas (copy or cross-ref)
- Phase 4 swaps stub auth with real OIDC
