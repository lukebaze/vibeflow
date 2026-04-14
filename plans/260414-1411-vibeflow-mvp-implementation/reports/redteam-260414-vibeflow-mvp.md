---
type: redteam
date: 2026-04-14
reviewers: 4
---

# Red Team Review — Vibeflow MVP Implementation

4 hostile reviewers, 38 raw findings, 15 accepted, 5 rejected, 18 merged.

## Reviewer Summary

| Reviewer | Lens | Findings | Key themes |
|---|---|---|---|
| Security Adversary | Attacker mindset | 10 | RBAC bypass via direct git push, TOCTOU self-elevation, credential leakage, prompt injection, audit log tamper, SQL injection |
| Failure Mode Analyst | Murphy's Law | 8 | Write crash-recovery, sync/write workdir race, teams.yaml brick, audit fail-closed, PR storms, embedding DLQ, disk exhaustion |
| Assumption Destroyer | Skeptic | 10 | Next.js custom server + MCP SSE coexist, Claude Code transport unverified, Gemini quota, OIDC group portability, clone storage, GitHub rate limits |
| Scope & Complexity Critic | YAGNI | 10 | Delete Phase 3, cut CLI to 4 commands, 5 MCP tools, drop embeddings, 1 template, delete ADS, simplify RBAC, drop multi-kind, sync writes |

## Accepted Findings (15)

See plan.md §Red Team Review for the full applied table.

### Critical (8)
1. Delete Phase 3 Next.js dashboard → 3 HTML admin pages (Scope)
2. Cut CLI to 4 commands (init/link/lint/claude setup) (Scope)
3. Delete Agent Distribution Service → manual `agents sync` (Scope)
4. Claude Code MCP HTTP/SSE transport unverified → Phase 0 spike (Assumption)
5. Branch protection missing → enforce via GitHub API (Security)
6. teams.yaml TOCTOU → pin SHA + two-party rule (Security)
7. Admin API production-reachable → bind-check + refuse non-loopback without OIDC (Security)
8. teams.yaml parse failure → last-known-good fallback + break-glass (Failure)

### High (6)
9. MCP tools 15+ → 5 essentials (Scope)
10. Drop embeddings → FTS only (Scope)
11. Write workdir race → `git worktree add` per write (Failure)
12. Prompt injection via MCP output → provenance fences (Security)
13. SQL/FTS injection → `websearch_to_tsquery` mandate (Security)
14. RBAC rules engine → hardcoded 4-role switch (Scope)

### Medium (1)
15. Clone storage exhaustion → shallow + sparse + disk monitoring (Assumption)

## Rejected (1 standalone)

- **Multi-kind abstraction (Scope #8)**: Reject. Keeping `kind` field costs zero, avoids Phase 5 retrofit pain. Accepted that `kind: code` is the only MVP value.

## Merged into hardening todo lists (18)

Not standalone findings but tracked in phase files:

**Security** (Phase 2 + 4):
- Git credential scrubbing in pino logs
- Template variable escaping with strict regex
- Audit log WORM sink (S3 Object Lock)
- Bootstrap token out-of-band flow + POST form
- Auto-merge pre-flight branch protection verification
- Audit role separation (INSERT-only role)

**Operational** (Phase 2 + 4):
- Disk monitoring + alert threshold
- Clock skew ops requirement (NTP)
- Embedding DLQ → moot after dropping embeddings
- ADS PR storms → moot after deleting ADS
- GitHub secondary rate limit handling
- OIDC group claim portability (IdP adapter pattern)

**Assumption** (Phase 0 spike):
- Next.js custom server + MCP SSE co-location → moot after deleting Phase 3
- Gemini free tier quota → moot after dropping embeddings
- schema_version `const` → `enum` fix inline

**Recovery / resilience** (Phase 4):
- Write PR crash-recovery idempotency key
- Concurrent write serialization
- Meta-repo force-push recovery

## Impact Analysis

**Timeline**: 11-16w solo → **6-8w solo**.

| Phase | Before | After | Cut |
|---|---|---|---|
| 0 | 1-2w | 1w | Add MCP spike, cut 3 templates |
| 1 | 2-3w | 1-1.5w | 12+ commands → 4 |
| 2 | 3-4w | 2-2.5w | 15+ tools → 5, drop embeddings |
| 3 | 2-3w | 3-5d | Delete Next.js, 3 HTML pages |
| 4 | 3-4w | 2-2.5w | Delete ADS, simplify RBAC, sync writes |

**Security hardening added** (4 layers):
1. Branch protection enforcement
2. TOCTOU fix (SHA pinning + two-party rule)
3. Prompt injection defense (provenance fences)
4. SQL injection defense (`websearch_to_tsquery`)

**Operational hardening added**:
1. `git worktree` per write (eliminates workdir race)
2. teams.yaml last-known-good fallback
3. Shallow clone + disk monitoring
4. Audit role separation (INSERT-only)

## Unresolved Questions (from reviewers)

1. Day-1 user count and repo count? Without concrete numbers, all scale planning is speculation.
2. Target Claude Code version for MCP transport?
3. Actual Gemini embedding budget at 50-repo scale? (moot after dropping embeddings, but re-emerges in Phase 5)
4. Meta-repo `--depth` optimal value for git sync?
5. How to handle GitHub org that forbids Apps? (PAT fallback documented but not tested)
6. Recovery procedure if meta-repo is force-pushed or deleted?
7. Backup + restore procedure tested? (none in success criteria)
8. GDPR retention policy for audit log?
