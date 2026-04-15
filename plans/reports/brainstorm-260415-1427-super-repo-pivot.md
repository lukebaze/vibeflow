---
type: brainstorm
date: 2026-04-15
slug: super-repo-pivot
status: approved-design
supersedes: brainstorm-260414-0841-vibeflow-workflow-design
---

# Vibeflow Super-Repo Pivot

Brainstorm output documenting the pivot from "Go CLI + stdio MCP server" to "super-repo pattern" inspired by `cxzharry/avengers-team`.

## Problem Statement

Previous Vibeflow MVP (2-week Go implementation) was STILL over-engineered for the actual problem. User discovered reference repo `cxzharry/avengers-team` which solves the same workflow problem with ~100 LOC bash instead of ~2000 LOC Go.

User asked: "let's review the product, if the output like this: https://github.com/cxzharry/avengers-team"

## Scout Findings: avengers-team Pattern

**Repository**: `cxzharry/avengers-team` (created 2026-04-14, updated 2026-04-15, 222 KB)

**Description**: "Product Consumer AI team super-repo: agents, skills, slash commands, wiki for standardized PO workflow"

**Structure**:
```
avengers-team/
├── .claude/
│   ├── agents/           # 8 curated agents
│   ├── skills/           # 12 curated skills
│   └── commands/         # 7 slash commands (team-*)
├── scripts/
│   ├── team-init.sh      # ~50 LOC bash: cp -r .claude + templates + CLAUDE.md + git init
│   ├── team-sync.sh      # ~50 LOC bash: diff-only report from master
│   └── validate-frontmatter.sh  # ~30 LOC bash: grep/sed YAML validator
├── templates/            # PRD, research-report, vibe-app-starter
├── wiki/                 # domain, patterns, examples, retros, drafts
├── CLAUDE.md             # workflow routing table
└── README.md             # git clone → team-init.sh quickstart
```

**Core architectural truth**:
1. **It's just a git repo**. No binary, no server, no MCP server, no compile step.
2. **Shell scripts, not Go CLI**. `team-init.sh` = 50 lines of `cp -r` + `mkdir -p` + `git init`.
3. **Slash commands define the workflow**. `.claude/commands/team-*.md` files Claude Code reads natively.
4. **Self-contained projects**. After `team-init.sh`, new project has everything it needs. No cross-repo live query problem.
5. **Diff-only sync**. `team-sync.sh` shows differences, user manually pulls. No daemon, no polling.
6. **Bash validation**. `validate-frontmatter.sh` uses `grep`/`sed` — no JSON Schema validator needed.

## Comparison: Vibeflow Go Plan vs avengers-team

| Aspect | Vibeflow Go plan (2w) | avengers-team |
|---|---|---|
| Implementation language | Go CLI + Go MCP server | Bash + Markdown only |
| Lines of code | ~2000 LOC Go | ~100 LOC bash |
| Build step | `go build` → binary | None |
| Distribution | Homebrew + GitHub Releases + 5 platform binaries | `git clone` |
| Init command | Go `text/template` engine + schema validation + `go:embed` | `cp -r` + `mkdir -p` |
| MCP server | Custom stdio subprocess (~300 LOC Go) | **None** — Claude Code reads `.claude/` natively |
| Schema validation | Embedded JSON Schema lib + Go validator | Bash + `jq` |
| Templates | 1 generic, go:embed-ed | Multiple, plain files |
| Workflow | Undefined in MVP | **7 opinionated slash commands** |
| Cross-repo context | MCP tool `workspace_context` + `wiki_search` | **Side-stepped**: self-contained per project |
| Build time | 2 weeks solo | **1-2 days** |

## The Epiphany

The "cross-repo live query" problem that Vibeflow's PRD framed as **core** was the wrong framing. The real problem is:

1. ✅ **Shared agents/skills/commands across projects** → solved by `cp -r .claude/` at init
2. ✅ **Shared templates (PRD/spec)** → solved by `cp -r templates/`
3. ✅ **Updates from source of truth** → solved by `team-sync.sh` (diff + manual pull)
4. ⚠️ **Cross-repo wiki live search** → solved by symlink OR just reading files in Claude Code

**There is no cross-repo live query problem at POC scale.** Every attempt to solve it was over-engineering.

## Evaluated Approaches

| Approach | Pros | Cons | Verdict |
|---|---|---|---|
| **Keep Go CLI plan (2w)** | Compiled binary distribution, schema validator | 20x more code, 10x slower to ship, wrong architecture | ❌ Over-engineered |
| **Full pivot to super-repo** | Proven pattern (avengers-team running), 1-2d timeline, slash commands define workflow | Need content curation (agents, slash commands, wiki) | ✅ **Chosen** |
| **Fork avengers-team directly** | Fastest (4-6h), inherits everything | Loses Vibeflow's enterprise positioning, attribution concerns | ⚠️ Too aggressive |
| **Hybrid (super-repo + Go lint)** | Better lint error messages | Still 3-4d, unnecessary Go | ❌ Compromise |

## Final Decision: Full Pivot to Super-Repo

**Approved**: Rewrite Vibeflow as super-repo following avengers-team pattern. Custom content for enterprise dev teams (not consumer AI).

### Positioning

**Vibeflow = Enterprise dev team workflow super-repo.**
- Audience: dev teams in larger orgs (multi-repo, standards-heavy, cross-functional)
- Differentiator vs avengers-team: enterprise engineering focus, stack-agnostic, ADR-first, code review discipline

### Target Structure

```
vibeflow/
├── .claude/
│   ├── agents/           7 fresh Vibeflow-specific agents
│   ├── skills/           karpathy-guidelines + plan + brainstorm + code-review + adversarial-dev + wiki
│   └── commands/         8 slash commands (vf-*)
├── scripts/
│   ├── vf-init.sh
│   ├── vf-sync.sh
│   └── vf-lint.sh
├── templates/
│   ├── prd.md            Enterprise PRD (1-2 pages)
│   ├── spec.md           Technical spec
│   ├── adr.md            Architecture Decision Record (NEW vs avengers-team)
│   └── project-starter/
├── wiki/
│   ├── patterns/         1-page-prd, adr-pattern, cross-repo-convention, code-review-rubric
│   ├── domain/           Enterprise dev knowledge base
│   ├── examples/         Worked examples
│   └── retros/           Lessons learned
├── docs/
│   ├── PRD-vibeflow-mvp.md  Rewritten for super-repo shape
│   ├── journals/
│   └── archived/         Deprecated Go specs
├── plans/
│   ├── archived/         Archived Go plan + red-team + validation history
│   └── reports/
├── CLAUDE.md             Vibeflow workflow routing table
├── README.md             Quickstart: git clone + ./scripts/vf-init.sh
└── .gitignore
```

### Slash Commands

| Command | Purpose |
|---|---|
| `/vf-prd <product>` | Write enterprise PRD via interview |
| `/vf-spec <feature>` | Write technical spec |
| `/vf-adr <decision>` | Record architecture decision |
| `/vf-plan <prd-path>` | Break PRD into implementation phases |
| `/vf-review` | Review pending code + docs via code-reviewer agent |
| `/vf-wiki <query>` | Search wiki (native grep) |
| `/vf-learn <source>` | Ingest new knowledge → wiki |
| `/vf-init-project <name>` | Scaffold new project from template |

### Principles

1. Karpathy guidelines MANDATORY (borrowed from avengers-team)
2. YAGNI / KISS / DRY
3. Standards-heavy (PRD/spec/ADR templates enforced via bash lint)
4. ADR mandatory for architecture decisions
5. Review discipline (every change through `/vf-review`)
6. Stack-agnostic (user declares in PRD)
7. Multi-repo aware (wiki patterns document conventions)
8. Deliverable-driven (every session produces 1 of: PRD, spec, ADR, reviewed code)

## Implementation Timeline

**Day 1** (~4-6 hours):
- Write CLAUDE.md routing table
- Write 8 slash commands (vf-*)
- Write 3 bash scripts (init, sync, lint)
- Write 4 templates (prd, spec, adr, project-starter)
- Write 7 fresh Vibeflow-specific agents

**Day 2** (~4-6 hours):
- Seed wiki with 5 patterns
- Rewrite `docs/PRD-vibeflow-mvp.md`
- Archive old Go plan + specs
- Write README
- Dogfood: run `./scripts/vf-init.sh test-project` end-to-end
- Git tag v0.1.0

**Total**: **1-2 days** vs previous 2-week Go plan.

## Implementation Considerations

- **Vendor agents in-repo**: self-contained, works offline, version-controlled (avengers-team does this)
- **Agents**: write fresh for Vibeflow's enterprise focus (user chose this over copying from avengers-team)
- **ADR template**: Michael Nygard's lightweight format (Status, Context, Decision, Consequences)
- **Cross-repo wiki access**: Day 1 MVP = `cd ~/vibeflow && grep -r wiki/`. Add `/vf-wiki` command if needed.
- **Lint via bash**: `vf-lint.sh` uses `grep`/`sed` — no JSON schema validator
- **Credit avengers-team** in README as inspiration

## Success Metrics

- `./scripts/vf-init.sh my-project` produces working scaffold in <3s
- `./scripts/vf-lint.sh docs/PRD.md` catches missing required sections
- `/vf-prd <name>` in any new project produces valid PRD via interview
- Vibeflow dogfoods itself: `./scripts/vf-lint.sh` passes on Vibeflow's own docs
- Onboarding (git clone → first PRD) <5 minutes
- Ship v0.1.0 by end of 2 days

## Risks

| Risk | Mitigation |
|---|---|
| Perceived copy of avengers-team | Credit explicitly, differentiate positioning |
| Wiki too sparse at launch | Seed with 5 critical patterns, grow via `/vf-learn` |
| Agent curation time | User chose fresh writing; scope to 7 focused agents |
| Scope creep back to custom tooling | Re-read avengers-team README when tempted |
| Existing users confused by pivot | Preserve Go plan in archived/, document pivot in README |

## Archive Plan

- Move `plans/260414-1411-vibeflow-mvp-implementation/` → `plans/archived/260414-1411-vibeflow-mvp-go-implementation/`
- Move `docs/specs/spec-cli.md` → `docs/archived/spec-cli-go.md`
- Move `docs/specs/spec-mcp-stdio.md` → `docs/archived/spec-mcp-stdio-go.md`
- Update `docs/specs/spec-standards.md` — mark schemas as reference docs (not validator input)
- Rewrite `docs/PRD-vibeflow-mvp.md` — new super-repo shape
- Commit as `refactor: pivot to super-repo pattern, archive Go plan`

## Unresolved Questions

1. Which 7 agents to write? Tentative: `planner`, `code-reviewer`, `debugger`, `tester`, `researcher`, `docs-manager`, `fullstack-developer`.
2. `karpathy-guidelines` skill — rewrite or reference avengers-team's version?
3. ADR template detail level — lightweight (Nygard) or enterprise (with risks, alternatives, rollback)?
4. Cross-repo wiki symlink vs copy vs grep — which pattern to document first?
5. Does "dev team" audience include QA/SRE/security? Start with Dev + Tester, defer others.
6. Should `/vf-review` include security mindset auto-activation?
7. Vendor skills in-repo or reference global `~/.claude/skills/`? Vendor for self-containment.
