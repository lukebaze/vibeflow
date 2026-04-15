---
phase: 3
title: Dogfood + Release v0.1.0
status: pending
effort: 2 days
depends_on: [phase-02]
---

# Phase 3: Dogfood + Release v0.1.0

## Context Links

- PRD: [../../docs/PRD-vibeflow-mvp.md](../../docs/PRD-vibeflow-mvp.md)
- Phase 1 + 2: CLI + MCP server delivered
- Ship criterion from PRD: "if I can't use Vibeflow to develop Vibeflow, ship is blocked"

## Overview

**Priority**: P0 — MVP not shipped until dogfood passes.
**Status**: pending
**Description**: Use Vibeflow on the Vibeflow repo itself. Fix anything that breaks. Write README with quickstart. Cut v0.1.0 release. That's it.

## Key Insights

- Dogfood is the acceptance test. No fancy release ceremony.
- README is the only marketing MVP needs.
- v0.1.0 is intentional — signals "not stable yet, breaking changes possible"
- Homebrew tap can wait for v0.2.0 if time is tight — plain GitHub Release binary download works

## Requirements

**Functional**:
- `vibeflow` binary in PATH on dev machine
- `~/.claude/settings.json` has vibeflow MCP entry
- `VIBEFLOW_WORKSPACE` set to a workspace that contains the `vibeflow` repo linked as a project
- Real Claude Code session in vibeflow repo works: `workspace_context` returns data, `wiki_search` finds content
- `vibeflow lint docs/PRD-vibeflow-mvp.md` passes clean
- `vibeflow lint docs/specs/*.md` passes clean
- All CI tests green on 5 platforms
- Tagged release v0.1.0 on GitHub with binaries + checksums

**Non-functional**:
- README quickstart completes in <5 min for a new user
- Release artifacts signed (optional, cosign)
- No known P0 bugs at release tag

## Architecture

No new code. Pure integration + release work.

```
vibeflow/
├── README.md                          (rewritten: quickstart + example)
├── .github/workflows/cli-release.yml  (from Phase 1)
├── .goreleaser.yml                    (from Phase 1)
└── (all code from Phases 0-2)
```

## Related Files

**Create / Update**:
- `README.md` — quickstart + install + example
- `CHANGELOG.md` — v0.1.0 entry
- `LICENSE` — MIT (if not already)
- Release tag `v0.1.0` on GitHub
- GitHub Release notes

**Reference**:
- `docs/PRD-vibeflow-mvp.md` — dogfood target
- `docs/specs/*.md` — dogfood lint target

## Implementation Steps

1. **Local dogfood setup**:
   - Build CLI binary from source: `cd cli && go build -o ~/bin/vibeflow .`
   - Create a workspace dir: `mkdir -p ~/work/vibeflow-workspace && cd ~/work/vibeflow-workspace`
   - Init workspace: `git init && echo 'schema_version: "1.0"\nworkspace:\n  name: vibeflow-dev\nteams: {default: {repos: []}}' > teams.yaml`
   - Symlink or copy Vibeflow repo into workspace: `ln -s /path/to/vibeflow ./vibeflow`
   - Link it: `cd vibeflow && vibeflow link --workspace ~/work/vibeflow-workspace --team default`
   - Run setup: `vibeflow claude setup` (should set `VIBEFLOW_WORKSPACE` in MCP entry)
   - Restart Claude Code

2. **Dogfood test cases**:
   - Start Claude Code session in `/path/to/vibeflow`
   - Ask: "What's this project about?" → Claude should call `workspace_context("vibeflow")` and summarize
   - Ask: "How does the stdio MCP server handle errors?" → Claude should call `wiki_search("stdio error")` and reference `spec-mcp-stdio.md`
   - Ask: "What PRD sections are required?" → Claude should reference `spec-standards.md`
   - Verify provenance fences visible in Claude's context (should not affect its reading)
   - Fix any tool call failures before proceeding

3. **Lint dogfood**:
   - `cd /path/to/vibeflow && vibeflow lint docs/PRD-vibeflow-mvp.md`
   - Must exit 0 with no errors
   - `vibeflow lint docs/specs/spec-cli.md`
   - `vibeflow lint docs/specs/spec-mcp-stdio.md`
   - `vibeflow lint docs/specs/spec-standards.md`
   - Fix any failures (either in lint rules or in the docs themselves)

4. **Scaffold dogfood**:
   - `cd /tmp && vibeflow init demo-service --team default --yes`
   - `cd demo-service && vibeflow lint docs/PRD.md` → should pass (template is valid)
   - `cd demo-service && vibeflow link --workspace ~/work/vibeflow-workspace` → should register in teams.yaml
   - Cleanup: `rm -rf /tmp/demo-service && sed ...` (or don't, it's local only)

5. **README**:
   - Rewrite `README.md` top to bottom with:
     - Tagline: "Cross-repo context for Claude Code. Zero infra."
     - Problem statement (2 sentences)
     - Install: `brew install ...` or `curl -sSL ...` (or just "download from Releases" for v0.1.0)
     - 5-minute quickstart: init workspace + 2 repos + `vibeflow claude setup` + open Claude Code + ask a cross-repo question
     - Commands reference: `init`, `link`, `lint`, `claude setup`, `mcp-server`
     - Link to `docs/PRD-vibeflow-mvp.md` and `docs/specs/*.md` for details
     - Status badge: v0.1.0, CI build status
     - Known limitations: no write path, no server, no dashboard (point to PRD §Non-Goals)

6. **CHANGELOG**:
   - Create `CHANGELOG.md` with v0.1.0 entry
   - List all 5 commands
   - Link to PRD for context
   - Known issues section (empty or minimal)

7. **CI final check**:
   - Ensure `.github/workflows/cli-release.yml` runs on tag push
   - Ensure `.github/workflows/ci.yml` (if separate) runs on PR
   - Verify all tests pass on 5 platforms

8. **Release**:
   - Update version constant in `cli/cmd/version.go` to `0.1.0`
   - Commit: `chore: bump version to 0.1.0`
   - Tag: `git tag -a v0.1.0 -m "first MVP release"`
   - Push tag: `git push origin v0.1.0`
   - Verify GitHub Actions release workflow runs, produces 5 binaries + checksums
   - Edit GitHub Release notes on the created release: brief summary, install instructions, known limitations

9. **Post-release smoke test**:
   - Download binary from GitHub Releases
   - `./vibeflow --version` → shows v0.1.0
   - `./vibeflow init /tmp/release-test` → works
   - If smoke test fails, yank release (delete tag + release), fix, re-cut

10. **Share + feedback loop**:
    - Post release link to internal channel (if team exists)
    - Collect feedback for 1 week before planning v0.2.0

## Todo List

- [ ] Local dogfood workspace setup
- [ ] Dogfood: `workspace_context` works in real Claude Code
- [ ] Dogfood: `wiki_search` finds content across files
- [ ] Dogfood: `vibeflow lint docs/PRD-vibeflow-mvp.md` passes
- [ ] Dogfood: `vibeflow lint docs/specs/*.md` passes
- [ ] Dogfood: `vibeflow init` produces valid repo
- [ ] Dogfood: `vibeflow link` registers idempotently
- [ ] Fix any dogfood failures
- [ ] Rewrite README with quickstart
- [ ] Write CHANGELOG.md
- [ ] Add LICENSE (MIT)
- [ ] Bump version constant to 0.1.0
- [ ] Tag v0.1.0
- [ ] Verify release workflow produces 5 binaries
- [ ] Edit GitHub Release notes
- [ ] Smoke test downloaded binary
- [ ] Optional: submit Homebrew tap

## Success Criteria

- Claude Code session in vibeflow repo works end-to-end without errors
- All MVP PRDs and specs pass `vibeflow lint` with zero warnings
- README quickstart tested by running it fresh, takes <5 min
- v0.1.0 tag pushed and GitHub Actions produces all 5 platform binaries
- Smoke test downloads binary and verifies `--version` output
- Release visible at `https://github.com/lukebaze/vibeflow/releases/tag/v0.1.0`

## Risks

| Risk | Mitigation |
|---|---|
| Dogfood reveals architectural flaw | Fix in-scope, defer to v0.2.0 if out-of-scope |
| Claude Code 1.x behavior differs on Windows in real use | Document as known limitation, open issue |
| README quickstart instructions unclear | Pair with non-author to test |
| Release workflow fails on first run | Dry-run via manual workflow_dispatch before tag push |
| Binary signature issues on macOS Gatekeeper | Document workaround (`xattr -d com.apple.quarantine`), revisit signing post-v0.1.0 |

## Security Considerations

- Release binaries include embedded schemas + templates — verify integrity via checksum
- No secrets in release artifacts
- MIT license clear about no warranty
- Document: "Vibeflow reads all your repos' docs. Don't put secrets in PRDs/wiki."

## Next Steps

- v0.2.0 candidates based on feedback:
  - Cache wiki search results per session
  - Add `vibeflow agents sync` if users create shared agent libraries
  - Windows-specific polish
  - Homebrew tap automation
- Phase 5+ (from brainstorm "full vision"): central server, write path, non-dev extension — ONLY if demand materializes
