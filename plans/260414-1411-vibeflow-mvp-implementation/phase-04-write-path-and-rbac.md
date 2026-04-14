---
phase: 4
title: Write Path + Simple RBAC
status: pending
effort: 2-2.5 weeks
depends_on: [phase-02, phase-03]
---

# Phase 4: Write Path + Simple RBAC

## ⓘ Red Team Changes (Applied 2026-04-14)

**Scope cuts**:
- **Finding #3 (Critical)**: **Delete Agent Distribution Service entirely**. Rely on CLI `vibeflow agents sync` (Phase 1) for manual updates. No `ads/` directory, no version-detector, no pr-batcher, no rollback subsystem. Add ADS in a future phase when agent updates become a real pain point.
- **Finding #14 (High)**: **Replace rules engine with hardcoded 4-role switch**. No custom DSL, no priority sort, no `meta-repo/.vibeflow/rules.yaml` loader. 30-line function: `if (role === 'observer' && action === 'write') deny; ...`. Roles: `workspace_admin | team_owner | team_member | team_observer`.
- **Defer post-MVP**: contractor expiry, temp-access grants, impersonation, offboarding flows, cross-team reviewer specialization, bootstrap token flow (use simpler "first login = admin if in OIDC admin group" rule), audit export CSV, `test-perms`/`test-teams-diff` admin commands.

**Critical security additions**:
- **Finding #5 (Critical)**: **Branch protection enforcement**. New step: for every linked repo (including meta-repo), programmatically configure GitHub branch protection on `main`:
  - Require PR before merge
  - Require CODEOWNERS review
  - Restrict direct push to `vibeflow-bot` only
  - Require signed commits
  - New command: `vibeflow admin verify-branch-protection` (fails sync if protection missing)
  - Reject any repo that fails verification
- **Finding #6 (Critical)**: **teams.yaml TOCTOU fix**. RBAC resolver pins `teams.yaml` SHA at start of each request. SHA included in audit log. Forbid self-mutating teams.yaml edits in same sync cycle — require two-party approval on any `teams.yaml` PR (CODEOWNERS: `@workspace-admins` + 1 other admin).
- **Finding #8 (Critical)**: **teams.yaml brick recovery**. On parse failure:
  1. Server keeps last-known-good teams.yaml in memory
  2. Logs error + alerts
  3. Continues serving with LKG until fixed
  4. Dashboard banner: "teams.yaml broken, using last-known-good"
  5. Break-glass: `vibeflow admin emergency-reset` via shell access (existing flow)
- **Finding #11 (High)**: **Write path uses `git worktree add` per write**. Each MCP write tool creates isolated worktree, commits there, pushes, cleans up. No shared clone dir for writes. Sync daemon's clone is read-only for sync; writes use ephemeral worktrees. Eliminates race entirely.
- **Merged security findings**:
  - **Audit log WORM**: dedicated DB role `vibeflow_audit_writer` with INSERT-only grant. Server connects as this role for audit writes. Main app role: SELECT but no UPDATE/DELETE. Bonus: stream audit events to S3 with Object Lock in parallel (Phase 4.1 if time permits, else document post-MVP).
  - **Auto-merge pre-flight**: before enabling auto-merge on any PR, verify via GitHub API that branch protection is active with required checks. If protection missing, fail-closed (mark write as `requires_human`).
  - **Bootstrap token**: deliver out-of-band (printed to CLI running on server shell), entered via POST form (not URL query). OIDC-first + token POST verify atomically. `Referrer-Policy: no-referrer`. Time-box 10 min single-use.

**Write path simplification**:
- **Finding (Scope #9)**: **Sync PR creation in MCP handler** (not async queue). Returns `{ pr_url, merged: bool }` synchronously. 2-second latency acceptable per plan's own SLO. Keep `write_log` table as audit trail (rename from `write_queue`), but no state machine, no retry loop, no `write_status` MCP tool.

**Effort**: 3-4w → 2-2.5w.

## <!-- Updated: Validation Session 1 — Keycloak bundled + GitHub App -->

**Validation #2 + #8**: OIDC implementation targets **Keycloak bundled in `docker-compose.yml`**.
- Realm auto-imported on Keycloak startup via `keycloak-realm.json` export file
- Realm pre-configures: `vibeflow` client, users test1/test2/admin, groups `vibeflow-admins`/`eng-platform`/`eng-growth`
- `OIDC_ISSUER=http://keycloak:8080/realms/vibeflow`
- `OIDC_CLIENT_ID=vibeflow`
- Generic OIDC path — no vendor-specific code for MVP. Other IdPs (Okta/Azure/Google) documented as "works by swapping env vars".

**Validation #4**: **GitHub App** is the target auth mode.
- Phase 4 ships with GitHub App flow (device code for CLI, app-install-link for admin)
- PAT fallback documented in README but NOT built (post-MVP if users complain)
- Requires org admin approval — document in bootstrap guide as prereq
- Private key stored encrypted in `secrets.vibeflow_github_app` (env or Docker secret)

**Validation #3 (POC scale)**: Simplify RBAC rollout:
- With 1-3 repos + 2-5 users, teams.yaml parse failure LKG fallback can be "die loud + shell-in fix"
- Drop the in-memory LKG cache for MVP, document the brick risk
- Still enforce: branch protection, TOCTOU SHA pinning, audit log separation, git worktree per write

**Effort**: 2-2.5w → **2w**.

## Remaining Content (updated to reflect cuts)

Below sections reflect original scope; ignore ADS (§4.10), rules engine DSL (§4.4), contractor/temp-access/impersonate/offboard, `write_queue` state machine — all cut per above.

---

## Context Links

- Brainstorm §10.3 Merge conflict strategy (write = branch + PR)
- Brainstorm §11.3 Write MCP tools
- Brainstorm §17 RBAC + teams.yaml
- Brainstorm §18.9 Agent Distribution Service
- Brainstorm §16.5 OIDC + GitHub App

## Overview

**Priority**: P0 — closes the loop; without writes, users need CLI for every mutation.
**Status**: pending
**Description**: Add OIDC/RBAC, MCP write tools (via PR), Agent Distribution Service for auto-PRs, CODEOWNERS enforcement, structured audit log, and real teams.yaml-driven permission engine. Replaces Phase 1 stub auth and Phase 2/3 shared-secret auth.

## Key Insights

- **All writes via PR** (§10.3.c) — server never pushes directly to main. GitHub handles conflict detection.
- **CODEOWNERS is last line of defense** — even if Vibeflow bug allows wrong user, GitHub blocks unauthorized merge.
- **Stateless RBAC** — permissions computed per request from OIDC token + latest teams.yaml. No cached user→roles DB.
- **Rules engine** (§17.18) — declarative priority-based rules, default deny.
- **Audit log immutable** — append-only, retained indefinitely, supports compliance.
- **Write policy configurable** — auto-merge for low-risk (task status), human review for PRD edits.

## Requirements

**Functional**:
- OIDC login flow via Better Auth/NextAuth (device code for CLI, browser for dashboard)
- Token storage: secure cookie (web) + `~/.config/vibeflow/token.json` mode 600 (CLI)
- `vibeflow login/logout/whoami` real OIDC (replace Phase 1 stub)
- RBAC rules engine with priority + first-match-wins + default-deny
- Enforcement in MCP tools, tRPC endpoints, admin API
- MCP write tools (§11.3):
  - `epic_update_status`
  - `task_create`
  - `task_update_status`
  - `write_status` (poll pending writes)
- Write queue (`write_queue` table)
- PR creation via GitHub API (GitHub App preferred, PAT fallback)
- Auto-merge per write policy (config in `.vibeflow/config.yaml`)
- CODEOWNERS file auto-generated in product repo templates (reuse Phase 1 templates)
- Agent Distribution Service (ADS) worker:
  - Detects agent/skill version bumps in meta-repo
  - Queries affected repos per pins policy
  - Opens batched update PRs
  - Auto-merges patches, notifies on minor/major
  - Rollback command support
- Audit log table with immutable append-only writes
- Audit log queryable by admins
- Admin commands:
  - `vibeflow admin validate-teams`
  - `vibeflow admin test-perms`
  - `vibeflow admin test-teams-diff`
  - `vibeflow admin first-admin` (bootstrap token)
  - `vibeflow admin grant-temp-access`
  - `vibeflow admin offboard`
  - `vibeflow admin impersonate`
  - `vibeflow admin audit-export`

**Non-functional**:
- Auth resolution < 50ms p95 (memoized per request)
- Write PR creation < 2s p95
- Auto-merge decision < 500ms
- Audit log write < 20ms (async commit ok)
- Zero privilege escalation via race conditions

## Architecture

**Server additions**:
```
server/src/
├── (Phase 2/3 files)
├── auth/
│   ├── oidc-client.ts             (OIDC discovery, token validation)
│   ├── session.ts                 (cookie sessions, refresh)
│   ├── device-code.ts             (CLI OAuth device code flow)
│   └── bootstrap-token.ts         (first-admin single-use)
├── rbac/
│   ├── resolver.ts                (principal → roles from teams.yaml)
│   ├── rules-engine.ts            (evaluate rules against action/resource)
│   ├── rules.ts                   (built-in rules + custom loader)
│   ├── contractor-check.ts
│   ├── cross-team-reviewer.ts
│   └── types.ts                   (Principal, Resource, Action, Decision)
├── write/
│   ├── queue.ts                   (write_queue table ops)
│   ├── pr-creator.ts              (GitHub API)
│   ├── policy.ts                  (auto-merge decision)
│   ├── branch-manager.ts          (branch naming, cleanup)
│   └── github-client.ts           (App or PAT)
├── ads/                           ← Agent Distribution Service
│   ├── orchestrator.ts            (trigger on meta-repo change)
│   ├── version-detector.ts        (diff manifests, detect bumps)
│   ├── repo-matcher.ts            (pin policy resolver)
│   ├── pr-batcher.ts              (combine updates in single PR)
│   └── rollback.ts
├── audit/
│   ├── logger.ts                  (append-only writes)
│   ├── querier.ts                 (admin queries)
│   └── exporter.ts                (CSV export)
├── mcp/tools/
│   ├── epic-update-status.ts
│   ├── task-create.ts
│   ├── task-update-status.ts
│   └── write-status.ts
├── api/routers/
│   ├── auth.ts                    (login/logout/whoami)
│   ├── admin.ts                   (extended: perms test, impersonation, etc.)
│   └── audit.ts                   (admin-only)
└── middleware/
    ├── auth-middleware.ts         (extract principal from token)
    ├── rbac-middleware.ts         (enforce permissions)
    └── audit-middleware.ts        (log every request)
```

**CLI additions**:
```
cli/cmd/
├── login.go           (replace stub with real OIDC device code)
├── admin.go           (admin subcommands)
└── (existing commands updated to use real auth)
```

**Database additions** (`server/src/db/migrations/002_rbac_audit.sql`):
```sql
-- Write queue
CREATE TABLE write_queue (
  id UUID PK,
  repo_id UUID FK,
  operation_type TEXT,              -- epic_update_status, task_create, etc.
  payload_json JSONB,
  branch TEXT,
  pr_number INTEGER,
  status TEXT,                      -- pending|pr_open|merged|failed|rejected
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ,
  created_by TEXT,
  error TEXT
);

-- Audit log (immutable)
CREATE TABLE audit_log (
  id UUID PK,
  timestamp TIMESTAMPTZ,
  principal_type TEXT,              -- user|service|bot|bootstrap
  principal_id TEXT,
  action TEXT,
  resource_urn TEXT,
  decision TEXT,                    -- allow|deny
  reason TEXT,
  request_id UUID,
  ip_address INET,
  user_agent TEXT,
  extra JSONB
);
-- Prevent updates/deletes via trigger or policy

-- Agent updates tracking
CREATE TABLE agent_updates (
  id UUID PK,
  agent_name TEXT,
  from_version TEXT,
  to_version TEXT,
  workspace_id UUID,
  repo_id UUID,
  pr_number INTEGER,
  status TEXT,                      -- pending|pr_open|merged|failed|rolled_back
  change_type TEXT,                 -- patch|minor|major
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ
);

-- Temporary access grants
CREATE TABLE temp_access (
  id UUID PK,
  email TEXT,
  team_slug TEXT,
  role TEXT,
  reason TEXT,
  granted_by TEXT,
  granted_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  revoked_at TIMESTAMPTZ
);

-- Sessions (encrypted tokens)
CREATE TABLE sessions (
  id UUID PK,
  user_id TEXT,
  token_hash TEXT,
  created_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  revoked_at TIMESTAMPTZ,
  last_used_at TIMESTAMPTZ,
  ip_address INET,
  user_agent TEXT
);
```

**New dependencies**:
- `@octokit/app` — GitHub App auth
- `@octokit/rest` — GitHub REST API
- `openid-client` — OIDC
- `iron-session` or Better Auth — session management
- `jose` — JWT verification

## Related Files

**Create**:
- `server/src/auth/**/*.ts` (5 files)
- `server/src/rbac/**/*.ts` (6 files)
- `server/src/write/**/*.ts` (5 files)
- `server/src/ads/**/*.ts` (5 files)
- `server/src/audit/**/*.ts` (3 files)
- `server/src/middleware/**/*.ts` (3 files)
- `server/src/mcp/tools/[write tools]` (4 files)
- `server/src/api/routers/{auth,admin,audit}.ts` (3 files)
- `server/src/db/migrations/002_rbac_audit.sql`
- `cli/cmd/admin.go` + `cli/internal/admin/*.go`

**Modify**:
- `cli/internal/auth/*.go` (replace stub with real OIDC)
- `server/src/mcp/server.ts` (add middleware chain)
- `server/src/api/trpc.ts` (add auth/rbac/audit middleware)
- All Phase 2 MCP read tools (add RBAC filter)
- All Phase 3 tRPC routers (add RBAC filter)
- Phase 1 templates: add `.github/CODEOWNERS` to product repo templates

## Implementation Steps

### 4.1 OIDC integration

1. `oidc-client.ts`: discovery, JWKS fetch, token verification (using `jose`)
2. Configure via env: `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_GROUP_CLAIM`
3. Login flow for dashboard: standard auth code + PKCE, redirect to `/auth/callback`
4. Login flow for CLI: OAuth device code, CLI polls for completion
5. Session storage: encrypted cookie (web) + `sessions` table for revocation
6. Refresh token handling (if IdP supports)

### 4.2 Principal resolution

1. `auth-middleware.ts`: extract token → verify → load claims → build `Principal` object
2. Principal fields: `type, email, groups, session_id, request_id`
3. Memoize within request lifetime (store on request context)

### 4.3 RBAC resolver

1. `resolver.ts`: `resolveRoles(principal, teamsYaml)` returns all applicable roles
2. Steps (§17.13):
   - Check `ex_employees` → return deny-all
   - Check workspace_admin (explicit or sso_mappings)
   - Iterate teams: explicit members → sso_mappings → skip
   - Apply contractor entries (expiry, repo allowlist)
   - Apply cross_team_reviewer grants
3. Returns `ResolvedPrincipal: { type, roles, teams, expires?, scope? }`

### 4.4 Rules engine

1. `rules-engine.ts`: load rules from built-in + workspace custom (`meta-repo/.vibeflow/rules.yaml`)
2. Rules sorted by priority desc
3. `evaluate(action, resource, resolvedPrincipal)` returns `Decision { allow: bool, reason, matched_rule }`
4. First match wins, default deny
5. Rule matching via simple DSL (match fields with `${}` interpolation)

### 4.5 Permission enforcement wrapper

1. `checkPermission(principal, action, resource) → Decision` (§17.15)
2. Called from:
   - `rbac-middleware.ts` for tRPC + Fastify routes
   - MCP tool handlers at entry point
   - Admin CLI subcommands (remote calls go through same API)
3. On deny: throw typed error, middleware returns `PERMISSION_DENIED` envelope
4. Every call (allow + deny) writes to audit log

### 4.6 Write queue

1. `write/queue.ts`: CRUD on `write_queue` table
2. Status transitions: `pending → pr_open → merged | failed | rejected`
3. Failure handling: retry transient (network), mark failed on logical (conflict)

### 4.7 PR creator

1. `pr-creator.ts`:
   - Obtain local repo clone (from sync daemon shared storage)
   - Create branch: `vibeflow/auto/<op-type>-<id>-<timestamp>`
   - Apply file changes (write via templating or direct file edit)
   - Commit as `vibeflow-bot` (GPG signed if configured)
   - Push branch
   - Open PR via GitHub API with structured body (op-type, user, payload summary)
   - Update `write_queue` row with `pr_number, status=pr_open`

### 4.8 Auto-merge policy

1. `policy.ts`: reads `.vibeflow/config.yaml` write_policy
2. Decides per operation: auto-merge or require_human
3. Auto-merge flow: GitHub API `PUT /pulls/:n/merge` with merge method `squash`
4. Auto-merge blockers: CI status required, CODEOWNERS approval required
5. If blocked: leave PR open, notify via webhook (Slack/etc — defer integration)

### 4.9 Write MCP tools

1. `epic_update_status`:
   - Validate input
   - Check permission `update` on `vbf://epics/<id>`
   - Validate transition (e.g., can't done → proposed)
   - Enqueue write: modify `meta-repo/epics/EPIC-XXX.md` frontmatter
   - Apply auto-merge policy
   - Return `write_id, status, pr_url?, requires_approval?`

2. `task_create`:
   - Validate input
   - Check permission `create` on `vbf://repos/<repo>/tasks`
   - Generate task ID: `PLAN-YYYY-MM-DD-<slug>`
   - Create file `plans/tasks/<id>.yaml` in product repo
   - PR flow
   - Return `task_id, write_id, status`

3. `task_update_status`:
   - Read existing task, mutate status field, commit
   - Usually auto-merges (low risk)

4. `write_status`: polls `write_queue` table

### 4.10 Agent Distribution Service

1. `ads/orchestrator.ts`: triggered by:
   - Git sync daemon event on meta-repo push
   - Daily cron reconciliation
   - Manual `vibeflow admin distribute-agents` command
2. `version-detector.ts`: reads current + last-known agent manifests, detects semver bumps
3. `repo-matcher.ts`: for each changed agent:
   - Query repos using this agent (from `.managed-manifest.yaml` + `.vibeflow.yaml` pins)
   - Apply pin rules (exact, tilde, caret, latest)
   - Emit list of eligible repos
4. `pr-batcher.ts`: within 30min window, batch multiple updates into single PR per repo
5. Create PRs via `write/pr-creator.ts` with structured body + changelog
6. Auto-merge on patch only (configurable)
7. `rollback.ts`: `agents rollback <name> --to <ver>` opens rollback PRs, marks version quarantined

### 4.11 CODEOWNERS generation

1. Update Phase 1 templates to include `.github/CODEOWNERS`:
```
# Managed by Vibeflow
plans/tasks/*.yaml      @vibeflow-bot
plans/kanban.yaml       @vibeflow-bot
.claude/agents/         @vibeflow-bot
.claude/skills/         @vibeflow-bot
.claude/.managed-manifest.yaml  @vibeflow-bot
docs/PRD.md             @team-owners
docs/specs/*.md         @team-members @tech-leads
```
2. Team-specific CODEOWNERS rendered based on team config in `.vibeflow.yaml`
3. `vibeflow lint` verifies CODEOWNERS matches expected

### 4.12 Audit log

1. `audit/logger.ts`: append-only inserts
2. Trigger on DB: `BEFORE UPDATE OR DELETE ON audit_log FAIL`
3. Every permission decision logged (allow + deny)
4. Every write_queue transition logged
5. Login/logout/token events logged
6. Impersonation events prominently flagged
7. Query API: `audit/querier.ts` — filter by principal, action, resource, time range
8. Export: `audit/exporter.ts` — CSV with cursor pagination

### 4.13 Admin endpoints + CLI commands

**Server**:
- `POST /admin/perms/test` (input: user, action, resource; output: decision)
- `POST /admin/teams/diff` (input: proposed teams.yaml; output: effect per user)
- `POST /admin/temp-access` (input: user, team, role, expires, reason)
- `POST /admin/offboard` (input: email; performs revoke + teams.yaml PR + task reassignment prompt)
- `POST /admin/impersonate` (admin only, time-boxed)
- `GET /admin/audit/query`
- `POST /admin/audit/export`
- `POST /admin/first-admin` (bootstrap token generator, requires shell access verification)
- `POST /admin/distribute-agents` (ADS manual trigger)

**CLI**:
- `vibeflow admin validate-teams` (local YAML parse + schema check)
- `vibeflow admin test-perms --user X --action Y --resource Z` (calls server)
- `vibeflow admin test-teams-diff --file teams.new.yaml`
- `vibeflow admin first-admin --email X` (requires local server shell access or remote admin secret)
- `vibeflow admin grant-temp-access <user> --team X --role Y --expires N --reason "..."`
- `vibeflow admin offboard <email>`
- `vibeflow admin impersonate <email> --duration 15m --reason "..."`
- `vibeflow admin audit-export --from X --to Y --format csv`

### 4.14 CLI real auth

1. Replace Phase 1 stub in `cli/internal/auth/`
2. Device code flow: show URL + code, poll endpoint, receive token
3. Store encrypted in `~/.config/vibeflow/token.json` mode 600
4. Auto-refresh on expiry
5. `whoami` hits server `/auth/whoami` endpoint

### 4.15 Phase 2 + 3 RBAC integration

1. All MCP read tools + tRPC routers get `rbac-middleware` added
2. Response filtering: omit fields user can't see (per role, per-team visibility)
3. Cross-team visibility rules applied (§17.9): draft vs published, team vs workspace
4. Ex-employees → deny-all consistently

### 4.16 Bootstrap flow

1. `vibeflow admin first-admin --email alice@company.com` (requires shell access to server)
2. Generates `bootstrap` entry in `teams.yaml` + token
3. Alice opens `https://vibeflow.domain/auth/bootstrap?token=X`
4. OIDC login → verify email matches → grant admin role → clear bootstrap flag
5. Server auto-creates PR to `teams.yaml` removing bootstrap flag, merges automatically

### 4.17 Testing

- Unit: rules engine (all role/action/resource combinations from §17.19 examples)
- Unit: teams.yaml resolver edge cases (contractor expiry, ex-employee, sso_mappings precedence)
- Integration: permission middleware denies correctly on MCP/tRPC
- Integration: write flow end-to-end (create task → PR → auto-merge → reindex → query reflects)
- Integration: ADS flow (bump agent → detect → open PR → auto-merge)
- Security: tests for privilege escalation attempts (manipulating teams.yaml mid-request, expired tokens, replay attacks)
- Audit log integrity: cannot be updated/deleted
- Load: 100 concurrent requests, measure auth resolution latency

## Todo List

- [ ] OIDC client setup + config
- [ ] Device code flow (CLI)
- [ ] Cookie sessions (web)
- [ ] Session storage + revocation
- [ ] Auth middleware (principal extraction)
- [ ] RBAC resolver (principal → roles)
- [ ] Rules engine (priority-based)
- [ ] Built-in rules from §17.18
- [ ] Custom rules loader (from meta-repo)
- [ ] RBAC middleware for tRPC
- [ ] RBAC check integration in MCP tool handlers
- [ ] Cross-team visibility filter
- [ ] Write queue table + CRUD
- [ ] PR creator (GitHub API, branch + commit + push + PR)
- [ ] Auto-merge policy
- [ ] MCP tool: epic_update_status
- [ ] MCP tool: task_create
- [ ] MCP tool: task_update_status
- [ ] MCP tool: write_status
- [ ] Agent Distribution Service orchestrator
- [ ] ADS version detector
- [ ] ADS repo matcher (pin policy)
- [ ] ADS PR batcher
- [ ] ADS rollback
- [ ] CODEOWNERS template updates (Phase 1 templates)
- [ ] Audit log table + trigger (immutable)
- [ ] Audit logger (append-only)
- [ ] Audit querier
- [ ] Audit exporter (CSV)
- [ ] Admin API endpoints (7+)
- [ ] CLI admin commands
- [ ] Bootstrap flow (first-admin token)
- [ ] Temp access grants
- [ ] Offboarding flow
- [ ] Impersonation with audit flag
- [ ] CLI real OIDC auth (replace stub)
- [ ] Phase 2 RBAC retrofit (all MCP read tools)
- [ ] Phase 3 RBAC retrofit (all tRPC routers)
- [ ] Unit tests: rules engine (all §17.19 examples)
- [ ] Integration tests: write PR flow
- [ ] Integration tests: ADS flow
- [ ] Security tests: privilege escalation
- [ ] Load tests: 100 concurrent, measure auth latency

## Success Criteria

- User can log in via OIDC from dashboard and CLI
- User's MCP tool access filtered by team/role (observer can read, cannot write)
- `task_create` MCP tool creates file + opens PR + auto-merges + reindexes, task appears in kanban within 60s
- `epic_update_status` with user lacking permission returns `PERMISSION_DENIED` + audit log entry
- Agent version bump in meta-repo triggers auto-PRs to eligible repos within 1 minute
- Patch updates auto-merge; minor updates stay open for review
- `vibeflow admin test-perms` correctly predicts allow/deny for §17.19 scenarios
- Bootstrap flow works on fresh server (one admin locked-out → emergency-reset → re-login)
- Offboarded user access revoked within 1 sync cycle (30s)
- Contractor access auto-denies after expiry
- Audit log cannot be updated/deleted (DB-level enforced)
- Admin can export 30 days of audit as CSV
- CODEOWNERS prevents vibeflow-bot from touching PRD files in test scenario

## Risks

| Risk | Mitigation |
|---|---|
| OIDC integration quirks per IdP (Okta vs Azure AD vs Google) | Test against multiple IdPs, document per-provider quirks, fallback to Keycloak POC |
| Stateless role resolution slow with large teams.yaml | Memoize per request, benchmark at 500+ members |
| GitHub API rate limit hit by PR storms | Batch updates, respect `X-RateLimit-Reset`, queue writes |
| Auto-merge races with manual PRs | Use GitHub's concurrent merge prevention, test race scenarios |
| CODEOWNERS syntax bugs break merging | Validate CODEOWNERS via `gh` before committing |
| Rules engine edge case: rule order dependency | Unit tests exhaustive, document evaluation order |
| Audit log DB bloat | Partition by month, archive old partitions to S3 |
| Impersonation misused | Prominent audit flag, admin notification, time-boxed |
| Write queue pile-up on outages | Dead letter queue, admin dashboard shows depth |
| Bootstrap token leak | Single-use, time-limited, email-bound, shell-access requirement |
| ADS storms on schema refactor (all agents change) | Rate limit per repo, batch aggressively, notify before mass update |
| Phase 2 tools RBAC retrofit misses edge cases | Audit all tool handlers, integration tests per tool per role |

## Security Considerations

- OIDC tokens short-lived (~1h), session tokens rotating (7d)
- All secrets in env/vault: OIDC client secret, GitHub App private key, Postgres password, JWT signing key
- Rate limit per user per token type
- CSRF protection on state-changing tRPC mutations (tokens in cookie require CSRF token)
- GitHub App private key rotation procedure documented
- Principle of least privilege enforced at 4 layers (§17.15): API, git, CI, audit
- PII handling: audit logs contain emails — document retention policy
- Emergency reset requires shell access to server — limits blast radius of token leak
- GDPR erasure: pseudonymize (hash) emails after retention, keep audit integrity

## Next Steps

- Phase 4 is MVP-complete. Report to stakeholders with metrics dashboard.
- Phase 5+: non-dev extension (web editor, design/product templates, binary assets)
- Phase 6+: observability/compliance hardening, cross-workspace federation, public docs sharing
