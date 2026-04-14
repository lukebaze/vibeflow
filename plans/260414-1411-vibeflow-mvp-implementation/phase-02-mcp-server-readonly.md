---
phase: 2
title: MCP Server (FTS-only, 5 tools)
status: pending
effort: 2-2.5 weeks
depends_on: [phase-00-spike, phase-01]
---

# Phase 2: MCP Server (FTS-only, 5 tools)

## ⓘ Red Team Changes (Applied 2026-04-14)

**Scope cuts**:
- **Finding #9 (High)**: 15+ MCP tools → **5 essentials**:
  - `workspace_context(repo)`
  - `wiki_search(query, scope)`
  - `epic_list(filter)`
  - `kanban_query(filters)`
  - `prd_get(repo)`
  - Plus internal: `health()`
  - **CUT**: `epic_get` (subsumed by `epic_list`), `wiki_resolve_link` (client-side util), `prd_validate` (CLI `lint` does this), `spec_get` (defer, low usage), `cross_repo_search` (duplicate of `wiki_search` scope=workspace), `team_members`/`team_contact` (embed in `workspace_context`), `agents_list`/`agent_get`/`agent_check_update` (not Claude-invoked), `force_sync` (admin API only, not MCP)
- **Finding #10 (High)**: Drop entire embeddings stack — no pgvector, no Gemini, no BullMQ, no hybrid search, no embedding worker, no DLQ. **FTS only (Postgres `tsvector`)**. Add semantic search post-MVP when real search quality complaints arrive.
- **Finding #15 (Medium)**: Shallow clone (`--depth 50`) + sparse checkout restricting to vibeflow paths (`docs/`, `wiki/`, `plans/`, `epics/`, `teams.yaml`, `.vibeflow.yaml`, `standards/`). Reject git-LFS (document "not supported MVP"). Add disk usage metric + alert threshold.
- **Scope #10**: Drop numeric SLOs, Prometheus metrics stub, 50-repo load test. Replace with: "syncs 5-repo test fixture in <5min".

**Security hardening** (merged findings):
- **Finding #13 (High)**: Mandate `websearch_to_tsquery` (safe parser) for ALL user FTS input. Mandate parameterized queries (`$1`, `$2`). Add CI lint rule banning template-literal SQL. Integration test suite with SQL injection probes for every tool accepting free-text.
- **Finding #12 (High)**: Wrap all user-authored content returned to Claude in provenance fences: `<untrusted-content from="wiki/foo.md" author="bob@...">...</untrusted-content>`. Strip HTML comments from returned markdown. Document trust hierarchy: meta-repo/standards = high trust; wiki/PRD/spec from product repos = untrusted.
- **Finding #7 (Critical)**: Admin API MUST refuse to bind to non-loopback unless Phase 4 auth configured. Startup check: if `OIDC_ISSUER` unset AND bind is `0.0.0.0`, hard-fail. Log the check.
- **Security #3 (merged)**: Git credential hygiene — use git credential helper or HTTP Basic header, NEVER embed tokens in clone URLs. Add pino redaction rules for `ghs_*`/`ghp_*`/`github_pat_*`/`Bearer *`/URL tokens. Wrap `simple-git`/`execa` stderr with scrubber. Unit test: inject fake token, assert never appears in logs.

**Effort**: 3-4w → 2-2.5w.

## <!-- Updated: Validation Session 1 — MANUAL SYNC, no daemon -->

**Validation #5 (MASSIVE cut)**: Drop entire sync daemon subsystem. At POC scale (1-3 repos, 2-5 users), polling is over-engineered.

**CUT** (not needed for MVP):
- Sync orchestrator (`sync/orchestrator.ts`)
- Worker pool
- Per-repo Redis locks (Redis still needed for BullMQ, kept for other reasons or dropped if no other queue)
- Polling scheduler
- Circuit breaker for sync failures
- Per-repo `poll_interval_s` configuration
- Scheduled reconciliation cron
- Prometheus metrics for sync lag

**REPLACED with**:
- `POST /admin/sync/:repo` endpoint — triggered by CLI `vibeflow sync [repo]` (§13.16.1 added in Phase 1)
- Sync flow: CLI hits endpoint → server runs one-shot `git fetch` + incremental reindex → returns stats
- Initial clone: `POST /admin/repos/register` clones fresh, reindexes fully
- Keep: classifier, parsers, incremental reindex logic, file handlers, `index_errors`

**Impact**: Drops ~30% of Phase 2 scope. Simpler to reason about, easier to debug (user triggers sync explicitly, knows exactly when it runs). Post-MVP can add polling/webhooks if needed.

**Validation #3 (POC scale)**: Drop disk monitoring alerts, drop large-scale load test, drop per-second performance budgets. Replace with "1-3 repo fixture syncs in <30s" acceptance test.

**Validation #8 (Keycloak bundled)**: Add `keycloak` service to `docker-compose.yml`:
```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin   # POC only, change for real deploy
    ports: ["8080:8080"]
```
Phase 2 server config points at `http://keycloak:8080/realms/vibeflow`. Realm auto-provisioned via Keycloak import file at startup.

**Effort**: 2-2.5w → **1.5-2w**.

## Remaining Content (updated to reflect cuts)
---

## Context Links

- Brainstorm §10 Git sync protocol
- Brainstorm §11 MCP tools API
- Brainstorm §16.3 Infrastructure deploy
- Phase 0: schemas (validate incoming files)
- Phase 1: CLI creates the repos this server will index

## Overview

**Priority**: P0, enables Claude Code integration.
**Status**: pending
**Description**: Node/TS server exposing MCP tools to Claude Code. Syncs meta-repo + product repos via git polling, indexes into Postgres, serves read-only tools over HTTP/SSE MCP transport. No write path in Phase 2 — reads only.

## Key Insights

- **Git is source of truth, Postgres is index**: on drift, reindex from git, never manually edit DB.
- **Incremental reindex** critical for performance — diff-based per commit (§10.2).
- **Embeddings async** — FTS works from day 1, embeddings catch up eventually. Never block sync on embedding.
- **No auth in Phase 2**: all reads assume local/dev context. Phase 4 adds OIDC.
- **Split-file design avoids conflicts**: tasks as individual files (§10.3.a).
- **Stateless role resolution deferred**: Phase 4. For now, hardcoded admin access.

## Requirements

**Functional**:
- Clone + incremental pull all registered repos (meta-repo + product repos)
- Poll every 30s (configurable per repo)
- Parse + index: epics, tasks, wiki chunks, PRD sections, spec sections, teams, agent manifests
- Postgres FTS on wiki + PRD + specs
- Gemini embeddings (async queue) for semantic wiki search
- MCP server exposing read tools (§11.2):
  - `workspace_context(repo)`
  - `epic_get(id)`, `epic_list(filter)`
  - `kanban_query(repo?, assignee?, status?)`
  - `wiki_search(query, scope, mode)`
  - `wiki_resolve_link(name)`
  - `prd_get(repo)`, `prd_validate(repo)`
  - `spec_get(repo, feature)`
  - `cross_repo_search(query)`
  - `team_members(repo?, team?)`, `team_contact(name)`
  - `agents_list(repo)`, `agent_get(repo, name)`, `agent_check_update(repo)`
  - `force_sync(repo)`, `health()`
- `index_errors` table + surfaced via health endpoint
- Circuit breaker for repeatedly failing repos (§10.4)

**Non-functional**:
- Git sync latency < 60s p99 at 50 repos
- FTS query < 200ms p95
- Semantic search (embedding) < 500ms p95 cached, 2s cold
- Server memory < 2GB at 50 repos
- Docker Compose deployable in < 10 min

## Architecture

**Monorepo addition** (Vibeflow project):
```
vibeflow/server/
├── src/
│   ├── index.ts                   (entry)
│   ├── config/
│   │   └── env.ts                 (env var parsing + validation)
│   ├── db/
│   │   ├── client.ts              (Postgres pool)
│   │   ├── schema.sql             (migrations)
│   │   └── migrations/
│   ├── sync/
│   │   ├── orchestrator.ts        (worker pool, scheduling)
│   │   ├── worker.ts              (per-repo sync worker)
│   │   ├── classifier.ts          (file → handler routing)
│   │   ├── parsers/
│   │   │   ├── epic-parser.ts
│   │   │   ├── task-parser.ts
│   │   │   ├── prd-parser.ts
│   │   │   ├── spec-parser.ts
│   │   │   ├── wiki-chunker.ts
│   │   │   ├── teams-parser.ts
│   │   │   └── agent-manifest-parser.ts
│   │   ├── git-ops.ts             (clone, fetch, diff via simple-git)
│   │   └── circuit-breaker.ts
│   ├── embed/
│   │   ├── queue.ts               (BullMQ)
│   │   ├── worker.ts              (Gemini calls)
│   │   └── gemini-client.ts
│   ├── search/
│   │   ├── fts.ts                 (Postgres FTS queries)
│   │   ├── semantic.ts            (pgvector queries)
│   │   └── hybrid.ts              (combine)
│   ├── mcp/
│   │   ├── server.ts              (@modelcontextprotocol/sdk)
│   │   ├── tools/
│   │   │   ├── workspace-context.ts
│   │   │   ├── epic.ts
│   │   │   ├── kanban.ts
│   │   │   ├── wiki.ts
│   │   │   ├── prd.ts
│   │   │   ├── spec.ts
│   │   │   ├── team.ts
│   │   │   ├── cross-repo-search.ts
│   │   │   ├── agents.ts
│   │   │   ├── force-sync.ts
│   │   │   └── health.ts
│   │   └── schema.ts              (tool input/output schemas, shared w/ tRPC)
│   ├── api/
│   │   ├── trpc.ts                (tRPC router for future dashboard)
│   │   ├── health.ts              (HTTP /health for probes)
│   │   └── admin.ts               (admin endpoints: register repo, etc.)
│   ├── schema/
│   │   └── validators.ts          (reuse JSON schemas via ajv)
│   └── lib/
│       ├── logger.ts              (pino)
│       └── metrics.ts             (Prometheus if wanted)
├── package.json
├── tsconfig.json
├── Dockerfile
├── docker-compose.yml             (dev stack: server + postgres + redis + minio)
└── README.md
```

**Key dependencies** (Node/TS):
- `@modelcontextprotocol/sdk` — MCP server
- `simple-git` — git ops (or shellout via execa)
- `pg` + `pgvector` — Postgres client
- `bullmq` — job queue
- `ioredis` — Redis client
- `@google/generative-ai` — Gemini embeddings
- `ajv` + `ajv-formats` — JSON schema validation (same schemas as CLI)
- `zod` — runtime type validation
- `fastify` or `hono` — HTTP server
- `pino` — logging
- `trpc` — RPC (for dashboard)
- `gray-matter` — frontmatter parsing
- `remark` — markdown AST

## Related Files

**Create**:
- `server/src/**/*.ts` (~30 files)
- `server/src/db/migrations/001_initial.sql` (Postgres schema)
- `server/Dockerfile`
- `server/docker-compose.yml` (dev + quick-start)
- `.github/workflows/server-ci.yml`
- `server/README.md`

**Reference**:
- Schemas from `meta-template/standards/schemas/` (copied or cross-ref)

## Implementation Steps

### 2.1 Bootstrap server

1. Scaffold `server/` with TypeScript strict mode
2. Setup Postgres + Redis via docker-compose
3. Write Postgres schema migrations (tables from brainstorm §10.5, §12.5, §18)
4. Health endpoint + Prometheus metrics stub

### 2.2 Git sync orchestrator

1. `orchestrator.ts` — reads registered repos from `repo_state` table
2. Worker pool (8 concurrent, configurable)
3. Per-repo lock (Redis SET NX with TTL)
4. Scheduler: next-poll-time per repo, min 30s interval
5. Error handling: circuit breaker after 5 consecutive failures

### 2.3 Git ops

1. `git-ops.ts` wrapper around `simple-git`
2. Initial clone to `$CLONE_STORAGE_PATH/<repo_id>/`
3. `fetch()` returns whether head moved
4. `diff_names(oldSha, newSha)` returns changed files with status
5. Handle force-push: detect non-ff, `reset --hard`, schedule full reindex

### 2.4 File classifier

Per brainstorm §10.2 file classifier table. Routes changed files to parsers:
- `epics/*.md` → epic-parser
- `wiki/**/*.md` → wiki-chunker
- `plans/tasks/*.yaml` → task-parser
- `plans/kanban.yaml` → ignored (regenerated, not indexed)
- `docs/PRD.md` → prd-parser
- `docs/specs/*.md` → spec-parser
- `teams.yaml` (meta-repo) → teams-parser
- `.vibeflow.yaml` → repo-config parser
- `standards/agents/*/manifest.yaml` → agent-manifest-parser
- `standards/skills/*/manifest.yaml` → skill-manifest-parser

### 2.5 Parsers (one per content type)

Each parser:
- Takes file content + path + repo context
- Validates frontmatter against JSON schema (ajv)
- Parses body (gray-matter → sections)
- Returns upsert ops for Postgres rows
- On parse error: write row to `index_errors` table, don't crash

**Wiki chunker**: split by headings (h1-h3), ~800 token target, overlap 100 tokens. Each chunk gets ID = hash(path + heading_path).

### 2.6 Postgres schema

Tables (brainstorm §10.5, §12, §18):
```sql
-- Tracking
CREATE TABLE repos (id UUID PK, name TEXT, clone_url, kind, team, workspace_id, ...);
CREATE TABLE repo_state (repo_id FK, last_synced_sha, last_sync_at, status, error_count, poll_interval_s);
CREATE TABLE sync_history (repo_id, from_sha, to_sha, files_changed, duration_ms, synced_at);
CREATE TABLE index_errors (repo_id, file_path, line, message, severity, created_at);

-- Content
CREATE TABLE epics (id TEXT PK, workspace_id, title, status, owner, repos TEXT[], quarter, body JSONB, file_path, last_modified);
CREATE TABLE tasks (id TEXT PK, repo_id, epic_id NULLABLE, title, status, assignee, phase, linked_prs INT[], created, updated, file_path);
CREATE TABLE wiki_chunks (chunk_id TEXT PK, repo_id, file_path, heading_path TEXT[], content TEXT, last_modified TIMESTAMPTZ, fts tsvector);
CREATE TABLE wiki_embeddings (chunk_id FK, embedding vector(768), created_at);
CREATE TABLE prds (repo_id FK, feature TEXT, frontmatter JSONB, file_path, last_modified, validation JSONB);
CREATE TABLE prd_sections (prd_id FK, section_name, content TEXT, metrics JSONB);
CREATE TABLE specs (repo_id, feature TEXT, frontmatter JSONB, body JSONB, api_contracts JSONB, file_path);
CREATE TABLE teams (team_slug PK, workspace_id, name, description, oidc_groups TEXT[], plugin, project_kinds TEXT[]);
CREATE TABLE team_members (team_slug FK, email, role, added_at);
CREATE TABLE agent_manifests (name, version, workspace_id, manifest JSONB, file_path, PRIMARY KEY (workspace_id, name, version));
```

Indexes:
- FTS: `GIN(fts)` on wiki_chunks, `GIN(to_tsvector(content))` on prd_sections
- pgvector: `HNSW(embedding)` on wiki_embeddings
- B-tree: repo_id, status, updated, epic_id

### 2.7 Embedding worker

1. BullMQ queue `embed-wiki-chunks`
2. Producer: after wiki parser upserts chunks, enqueue `embed(chunk_id)`
3. Worker: dequeue, call Gemini `text-embedding-004`, upsert to `wiki_embeddings`
4. Batch 100 chunks per API call
5. Error handling: retry 3x with backoff, DLQ on fail
6. Rate limit: 60 req/min (Gemini free tier default) — token bucket

### 2.8 Search

1. **FTS** (`search/fts.ts`): Postgres `tsquery` on `wiki_chunks.fts`, rank by ts_rank
2. **Semantic** (`search/semantic.ts`): Gemini-embed query, pgvector cosine similarity, top-K
3. **Hybrid** (`search/hybrid.ts`): reciprocal rank fusion of FTS + semantic, dedupe by chunk_id
4. Return chunks with `score, age_days, staleness_warning`

### 2.9 MCP server

1. `@modelcontextprotocol/sdk` setup with HTTP/SSE transport
2. Register each tool from §11 catalog
3. Each tool:
   - Validates input via zod schema
   - Queries Postgres
   - Returns envelope `{ ok, data, meta }` per §11.1.2
   - Logs to audit_log (deferred structured in Phase 4 — Phase 2 just basic logging)

### 2.10 Admin API

Minimal HTTP endpoints for operator actions:
- `POST /admin/repos/register` — register new repo to sync
- `DELETE /admin/repos/:id` — unregister
- `POST /admin/repos/:id/force-sync` — trigger sync immediately
- `GET /admin/health` — detailed status
- Protected by shared secret (env var) in Phase 2, proper auth Phase 4

### 2.11 Circuit breaker + failure handling

Per brainstorm §10.4:
- Network retries with exponential backoff
- `index_errors` on parse failure (don't crash)
- Force-push recovery: reset --hard + full reindex
- Postgres down: buffer events on disk, replay
- Stale lock release after timeout

### 2.12 Docker Compose (dev + quick start)

```yaml
services:
  server:
    build: ./server
    env_file: .env
    volumes:
      - clones:/var/vibeflow/clones
    depends_on: [postgres, redis, minio]
  postgres:
    image: postgres:16
    # pgvector extension installed via init script
  redis: image: redis:7
  minio: image: minio/minio
  caddy:
    # reverse proxy + TLS for prod; optional for dev
volumes:
  clones:
  pgdata:
  miniodata:
```

### 2.13 Testing

- Unit: parsers (all file types, valid + invalid inputs)
- Integration: spin up test Postgres, run full sync on fixture repo, verify Postgres state
- E2E: real Docker Compose + small fixture workspace, exercise MCP tools via test client
- Load test: 50 repo scenario, measure sync lag, FTS p95, memory

## Todo List

- [ ] Scaffold server/ TypeScript project
- [ ] Postgres migrations (001_initial.sql) with all tables + indexes
- [ ] Docker Compose dev stack (postgres + redis + minio)
- [ ] Config loader (env vars, validation)
- [ ] Logger + Prometheus metrics
- [ ] `git-ops.ts` (clone, fetch, diff, reset)
- [ ] Sync orchestrator + worker pool + per-repo locks
- [ ] File classifier routing
- [ ] Parser: epic, task, PRD, spec, wiki-chunker, teams, agent-manifest
- [ ] `index_errors` write path + surfacing
- [ ] Circuit breaker
- [ ] Embedding BullMQ queue + worker
- [ ] Gemini client with rate limit
- [ ] FTS search
- [ ] Semantic search (pgvector)
- [ ] Hybrid rank fusion
- [ ] MCP SDK setup + HTTP/SSE transport
- [ ] MCP tool: workspace_context
- [ ] MCP tool: epic_get, epic_list
- [ ] MCP tool: kanban_query
- [ ] MCP tool: wiki_search, wiki_resolve_link
- [ ] MCP tool: prd_get, prd_validate
- [ ] MCP tool: spec_get
- [ ] MCP tool: cross_repo_search
- [ ] MCP tool: team_members, team_contact
- [ ] MCP tool: agents_list, agent_get, agent_check_update
- [ ] MCP tool: force_sync, health
- [ ] Admin HTTP endpoints (register repo, force sync, health)
- [ ] Unit tests for parsers
- [ ] Integration test: fixture repo full sync
- [ ] E2E test: MCP tool exercise
- [ ] Load test: 50-repo scenario
- [ ] README + deployment docs

## Success Criteria

- 50-repo fixture syncs in <60s per cycle
- FTS query `wiki_search("oauth")` < 200ms p95
- Semantic query < 500ms p95 (cached embeddings)
- Parser handles 100% of valid fixture files, errors gracefully on invalid
- `force_sync(repo)` reindexes and returns in <5s for small repo
- MCP tool `workspace_context("auth-service")` returns full context per §11.2.1 schema
- `cross_repo_search("OAuth")` returns hits from 3+ repos
- `docker compose up` boots full stack + health green in <2 min
- Index error row appears on dashboard when fixture has malformed YAML
- Circuit breaker trips on repeated failures, logs visible

## Risks

| Risk | Mitigation |
|---|---|
| Gemini rate limits at high churn | Backoff, async queue, FTS fallback always works |
| MCP SDK breaking changes | Pin version, monitor changelog, integration test |
| Postgres FTS quality vs dedicated search engine | Hybrid with embeddings, document limitations |
| go-git vs simple-git vs shellout feature gaps | Start shellout (`execa("git", [...])`), migrate if slow |
| pgvector HNSW index build time | Build in background, tolerate cold-start slowness |
| Cross-repo large result sets in `cross_repo_search` | Strict limit (top 50), pagination |
| Memory blowup from loading large wiki files | Stream parsing, chunk incrementally |
| Stale lock breaks sync | Redis lock TTL + owner token, auto-release on timeout |
| Reindex storms on schema bumps | Version field in `repo_state`, gradual rollout |
| Git credentials leak in logs | Scrub credentials, never log full clone URLs |

## Security Considerations

- Server uses GitHub App or PAT with read-only scope per repo
- Credentials in env/vault, never in code or logs
- MCP server exposed over HTTPS only in production
- Admin endpoints behind shared secret (Phase 4 → proper auth)
- Postgres connection string in secrets manager
- Rate limiting on MCP tools (per-repo, per-connection)
- CORS: only allow configured origins (dashboard domain)

## Next Steps

- Phase 3 (dashboard) consumes same Postgres + tRPC layer
- Phase 4 adds write tools (PR-based), OIDC auth, audit log, RBAC filter
