---
title: Vibeflow MVP Plan — Failure-Mode Red-Team Review
reviewer: code-reviewer (Failure Mode Analyst lens)
date: 2026-04-14
scope: plan.md + phase-00..04
verdict: 8 findings (3 Critical, 4 High, 1 Medium)
---

# Failure-Mode Red-Team Review — Vibeflow MVP

Hostile review under Murphy's Law. No praise. Only failures.

---

## Finding 1: Write PR path has no crash-recovery contract for in-flight writes

- **Severity:** Critical
- **Location:** Phase 4, §4.6 Write queue + §4.7 PR creator
- **Flaw:** The write pipeline (enqueue → clone mutate → branch → commit → push → PR → update row) has no transactional boundary. Each side effect (file write to shared clone, `git push`, GitHub API call) is outside the DB. Status transitions `pending → pr_open → merged` assume the process stays alive end-to-end, but nothing reconciles partial state on crash.
- **Failure scenario:** Server OOMs or is rolled during `task_create`. File has been written to the shared clone (`$CLONE_STORAGE_PATH`), branch created locally, **possibly pushed**, but `write_queue.status` still says `pending` and no `pr_number` recorded. On restart: (a) sync daemon sees a dangling local branch and/or a ghost remote branch, (b) retry handler re-runs the task, creating a second branch with duplicate task ID, (c) two PRs for the same op — both eligible for auto-merge — one overwrites the other or collides on path. The write queue has no idempotency key, no "claim" state, no reconciliation sweep.
- **Evidence:** §4.6 lists `pending → pr_open → merged | failed | rejected` but never defines an intermediate `in_progress` / `claimed_by_worker_x` state. No mention of a janitor that scans for rows stuck in `pending` past TTL. No mention of idempotency — step 4.9 `task_create` says "Generate task ID: `PLAN-YYYY-MM-DD-<slug>`" which is not deterministic (slug collisions on retry). Write queue table (§db migration 002) has no `lease_until`, no `worker_id`, no `attempt_count`, no `idempotency_key`.
- **Suggested fix:** Add `write_queue.lease_id`, `lease_expires_at`, `attempt_count`, `idempotency_key` (client-provided or derived from `(operation_type, normalized_payload_hash)`). Worker claims row with `UPDATE ... WHERE id = ? AND lease_expires_at < now() RETURNING *`. On startup, run reconciliation: for each `pr_open` row without `pr_number`, `git ls-remote` to detect dangling branches and either adopt or clean up. Make task ID generation deterministic within a write attempt.

---

## Finding 2: Sync daemon writes and write-path reads from the same shared clone with no locking protocol

- **Severity:** Critical
- **Location:** Phase 2 §2.3 Git ops + Phase 4 §4.7 PR creator ("Obtain local repo clone (from sync daemon shared storage)")
- **Flaw:** The plan explicitly says the PR creator reuses the sync daemon's clone at `$CLONE_STORAGE_PATH/<repo_id>/`. Phase 2 uses a Redis per-repo sync lock (§2.2 "Per-repo lock (Redis SET NX with TTL)"), but Phase 4 never mentions acquiring that lock when mutating the working tree. Sync worker and write worker will race on the same git workdir.
- **Failure scenario:** Sync worker is mid-`git fetch --prune` on `repo_id=X`. Concurrently, write worker `checkout -b vibeflow/auto/task-create-...` in the same directory. Outcomes: (a) `checkout` sees index contention, fails mid-way leaving detached HEAD; (b) sync worker's `reset --hard` (force-push recovery in §2.3) wipes the write worker's uncommitted changes; (c) write worker pushes a branch that is rooted on a stale base because sync hadn't finished fetching, PR opens against wrong base SHA → merge conflict spam; (d) two write ops on the same repo stomp on each other because there's no write-worker-to-write-worker lock either.
- **Evidence:** §4.7 step 1: "Obtain local repo clone (from sync daemon shared storage)". §2.2 step 3: "Per-repo lock (Redis SET NX with TTL)" — scoped only to sync worker. §2.3 step 5: "Handle force-push: ... `reset --hard`, schedule full reindex" — destructive, no coordination with writers. Nowhere in §4.7 is the sync lock mentioned.
- **Suggested fix:** Either (a) give the write-path its own dedicated worktree per op (`git worktree add`) — eliminates shared-state problem entirely, or (b) make the Redis lock a read/write lock owned by both subsystems with an explicit ordering protocol. Document `reset --hard` recovery as incompatible with in-flight writes — must drain write queue for that repo first.

---

## Finding 3: teams.yaml parse failure is implicit deny-all — bricks the entire workspace

- **Severity:** Critical
- **Location:** Phase 4 §4.3 RBAC resolver + §4.4 Rules engine ("default deny")
- **Flaw:** Principal resolution reads teams.yaml. Rules engine is "first match wins, default deny". But teams.yaml is a git-synced file that can arrive in a broken state (bad YAML, schema violation, partial write during sync, git merge conflict markers, empty file after a bad commit). The plan has no fallback behavior defined — the obvious implementation denies everything.
- **Failure scenario:** A well-intentioned admin opens a PR that reorders teams.yaml, introduces a tab char, breaks YAML parse. Admin merges (CI misses it because schema validation isn't enforced on the meta-repo itself — §4.5 validates `repo-vibeflow-yaml`, not `teams.yaml` on the server side at sync time). Sync daemon pulls, parser throws, resolver can't build Principal → every request returns `PERMISSION_DENIED`. Dashboard returns 403 universally. CLI users can't read. MCP tools can't be invoked. Meanwhile the admin can't even log in to roll back because their admin role evaluation also hits the deny path. Recovery requires shell access to the server box.
- **Evidence:** §4.3 "resolveRoles(principal, teamsYaml)" — no error-handling contract. §4.4 "Rules sorted by priority desc ... default deny". §2.11 "index_errors on parse failure (don't crash)" — only applies to parsers, not to RBAC consumption. No mention of "last known good teams.yaml" snapshot, no mention of safe-mode, no mention of bootstrap admin bypass for this case. §4.16 bootstrap flow explicitly requires "shell access to server" — so the planned recovery is a full outage.
- **Suggested fix:** (1) Validate teams.yaml against schema **at sync time** before promoting; if invalid, refuse to replace in-memory version, alert loudly, keep last-known-good. (2) Add a "break-glass" env var `VIBEFLOW_EMERGENCY_ADMIN=<email>` that grants admin regardless of teams.yaml state (logged prominently in audit). (3) Add a pre-merge CI hook that runs `vibeflow admin validate-teams` on the meta-repo PR. (4) Surface parse failures in the health endpoint.

---

## Finding 4: Audit log has no policy for insert-failure — silent bypass of compliance guarantee

- **Severity:** High
- **Location:** Phase 4 §4.5 "Every call (allow + deny) writes to audit log" + §4.12 Audit log
- **Flaw:** The plan claims the audit log is the compliance backbone ("append-only, retained indefinitely, supports compliance") but never specifies what happens if the audit insert itself fails (Postgres full, connection dropped, replica lag, insert trigger conflict). There are two bad options and the plan picks neither: (a) fail the request → availability loss but integrity preserved, (b) allow the request → compliance gap (decision made but not logged). With `§non-functional: audit log write < 20ms (async commit ok)`, the async path means B is the default.
- **Failure scenario:** Postgres tablespace fills overnight (nobody monitors disk — see Finding 7). At 03:12 all `INSERT INTO audit_log` start failing. Permission checks continue to evaluate (they read from memoized teams.yaml), writes go through, PRs auto-merge. For 4 hours, every sensitive action is executed without an audit trail. At 07:00 an admin notices. Compliance investigation cannot reconstruct who did what — the immutability trigger exists but there's nothing to make immutable. Trigger from §4.12 `BEFORE UPDATE OR DELETE ON audit_log FAIL` is the wrong protection: it prevents tampering, not absence.
- **Evidence:** §non-functional "Audit log write < 20ms (async commit ok)". §4.12 no mention of fail-closed behavior. §Risks "Audit log DB bloat" mentioned, but not "audit log insert failure". §Security "PII handling: audit logs contain emails" — doubles down on log-as-compliance without defining a failure contract.
- **Suggested fix:** Define the contract explicitly: sensitive write operations MUST write audit synchronously in the same DB transaction as the `write_queue` insert; reads MAY log async. For async path, add a local disk journal (WAL) that the logger spools to on DB failure, with a replay worker. Fail-closed for `impersonate`, `offboard`, `temp-access`, `task_create`, `epic_update_status` (compliance-sensitive). Monitor audit insert error rate as a P1 alert.

---

## Finding 5: Agent Distribution Service will create PR storms that overflow the write queue and GitHub rate limits

- **Severity:** High
- **Location:** Phase 4 §4.10 Agent Distribution Service + §4.7 PR creator
- **Flaw:** ADS is triggered by "Git sync daemon event on meta-repo push" and "Daily cron reconciliation". On a mass agent refactor (e.g., admin bumps all 12 agents from v1.x to v2.0 as a semver-major cleanup), the version detector emits 12 × N-repos updates. The `pr-batcher.ts` batches within a **30-minute window** (§4.10 step 4), but the window is per-repo and doesn't cap global PR creation rate. There's no backpressure from the write queue to the orchestrator.
- **Failure scenario:** Admin merges `meta-repo` PR titled "refactor: rename all claude agents to new naming". Sync detects it in 30s. Version detector: 12 agents × 80 repos = 960 eligible updates → 80 PRs (batched per repo). Auto-merge policy is "patch only", so 80 PRs sit open — but each triggers: (a) 80 branch creations in shared clones (file contention with Finding 2), (b) 80 GitHub API `POST /repos/:org/:repo/pulls` calls in under a minute (well under GitHub secondary-rate-limit of ~30 content-creating requests per minute → HTTP 403 `abuse`), (c) write_queue fills with 960 entries, consuming workers for hours, (d) ordinary `task_create` requests queue behind the avalanche, user sees write latency spike from 2s to 20min. If ADS is also run on daily cron, this repeats even after partial progress.
- **Evidence:** §4.10 step 4 "within 30min window, batch multiple updates into single PR per repo" — batches within repo, not globally. §Risks mentions "ADS storms on schema refactor (all agents change) → Rate limit per repo, batch aggressively, notify before mass update" — ack'd as a risk but mitigations are vague and not in the implementation steps. No concurrency cap on ADS workers. No queue depth check before enqueueing. GitHub App rate limits not modeled.
- **Suggested fix:** (1) Global ADS throttle: max N PRs created per minute across all repos, with respect to GitHub secondary rate limits. (2) "Dry-run + confirm" mode default when > K repos would be touched; admin must run `vibeflow admin distribute-agents --confirm` explicitly. (3) Write queue backpressure: ADS pauses when queue depth exceeds threshold. (4) Priority lane in write_queue so user-initiated writes aren't starved by ADS. (5) De-duplicate ADS triggers: if git-push-event and cron both fire within the batch window, only one runs.

---

## Finding 6: Embedding worker has no DLQ handler and no worker-restart state — chunks permanently miss their embeddings

- **Severity:** High
- **Location:** Phase 2 §2.7 Embedding worker
- **Flaw:** "Retry 3x with backoff, DLQ on fail" — the DLQ is named but nothing processes it. On a Gemini API outage or a rate-limit burst at high churn, hundreds of chunks land in the DLQ. There's no backfill path, no alert, no reprocessing command. Semantic search silently degrades because those chunks never make it into `wiki_embeddings`. FTS still works, so nobody notices for weeks.
- **Failure scenario:** Gemini free tier is 60 req/min (§2.7). At wiki re-shuffle time (bulk admin edits), parser emits 2000 chunks in 5 min. Token bucket refuses ~80% of requests. BullMQ retries 3x per job with backoff. Every retry fails because the bucket is dry. Jobs land in DLQ. A week later a dev asks Claude "how do we do OAuth" — `wiki_search(mode='semantic')` returns stale results because the new chunks have FTS entries but no embedding rows. Hybrid rank fusion silently drops them from semantic side. No alert fires because there's no "DLQ non-empty" metric, no "chunks missing embedding for >T" query.
- **Evidence:** §2.7 step 5 "DLQ on fail" — zero implementation steps for DLQ consumer. §2.7 step 6 "Rate limit: 60 req/min ... token bucket" — naïve bucket without adaptive backoff based on Gemini's own 429 responses. §2.8 "Hybrid ... reciprocal rank fusion" — degradation is invisible. No metric in §testing for "embedding completeness %". Success criteria §2 only checks cached semantic query latency, not coverage.
- **Suggested fix:** (1) DLQ consumer that retries with longer backoff + admin alert when depth > N. (2) Periodic reconciliation query: `SELECT count(*) FROM wiki_chunks WHERE chunk_id NOT IN (SELECT chunk_id FROM wiki_embeddings)` — if > 0 for more than M hours, warn. (3) Adaptive rate limiter that reads Gemini's 429 `retry-after` header. (4) `vibeflow admin reindex-embeddings` command. (5) Health endpoint reports embedding coverage ratio.

---

## Finding 7: No disk monitoring or cleanup for clone storage or Postgres — silent death by volume fill

- **Severity:** High
- **Location:** Phase 2 §2.12 Docker Compose + §2.11 Circuit breaker + Phase 4 §db migration 002
- **Flaw:** The plan allocates a `clones` volume (§2.12) that grows unboundedly — every clone, every branch created by the write path, every ADS batch, every rejected/stale `vibeflow/auto/*` branch. Postgres grows too: `audit_log` (append-only, grows forever — §Risks mentions "partition by month, archive" but no implementation step), `sync_history` (every commit, every repo), `index_errors` (no TTL), `wiki_embeddings` (large vectors). Nothing monitors disk. Nothing prunes.
- **Failure scenario:** Month 3 post-launch. 50 repos × ~500MB each + auto branches + Postgres pgvector growth fills the `pgdata` volume. Postgres goes read-only. Sync daemon can't write `repo_state` updates, circuit breaker trips on all repos, every MCP tool returns errors. Write queue fills because workers can't transition rows. Dashboard stops loading. Admin investigates — "it was fine yesterday" — discovers `audit_log` is 40GB. Emergency triage: `DELETE FROM audit_log WHERE ...` — but that fails because of the immutability trigger (§4.12). Recovery now requires dropping the trigger, which is itself an auditable event that cannot be audited.
- **Evidence:** §2.12 declares volumes but no reserves, no alerts. §Risks Phase 2 has no "disk full" entry. §Success §4 "Audit log cannot be updated/deleted" — hard trigger prevents emergency cleanup. §4 risks "Audit log DB bloat → Partition by month, archive old partitions to S3" acknowledged but no implementation step in §Todo list, no S3 archive pipeline in phase 4 todos. No `vibeflow admin disk-report`, no Prometheus disk alert in §2.1 metrics stub.
- **Suggested fix:** (1) Add `disk_usage` metric exported from server (check `df` on `$CLONE_STORAGE_PATH` and Postgres data dir). (2) Health endpoint fails if free < 10%. (3) Implement audit partitioning in Phase 4, not "defer to later" — day-1 partition by month with automated DETACH + S3 archive. (4) Clone GC: `git gc` daily, prune stale `vibeflow/auto/*` branches > 7 days. (5) `index_errors` TTL (30 days). (6) Emergency-delete procedure for audit log that is itself audited to an out-of-band log (file or syslog).

---

## Finding 8: Contractor expiry and token refresh both rely on server wall clock — clock skew causes silent authz drift

- **Severity:** Medium
- **Location:** Phase 4 §4.3 RBAC resolver (contractor check) + §4.1 OIDC + §security "OIDC tokens short-lived (~1h)"
- **Flaw:** Contractor expiry (§temp_access.expires_at), bootstrap token expiry, session expiry, JWT `exp` validation all use server `now()`. The plan has no NTP requirement, no skew tolerance, no "which clock is authoritative". Postgres `now()` may drift from Node `Date.now()` may drift from GitHub API clock may drift from OIDC IdP clock.
- **Failure scenario:** Server container runs in a K8s pod on a node with a broken NTP. Clock is 47 minutes behind reality. (a) An expired contractor's `temp_access.expires_at = 2026-04-14 09:00` — Postgres `now()` says `08:13`, resolver grants access for 47 more minutes after their contract ended. Audit log records 47 min of "allowed" events that should have been deny. (b) Inverse: OIDC IdP issues token with `exp = iat + 3600`, server's `jose` verification uses local clock which is ahead of IdP by 10min → valid tokens rejected as expired, users locked out. (c) `sessions.expires_at` doesn't match the cookie's client-visible expiry, users see session drop mid-action. (d) `write_queue` lease TTLs fire early, retrying in-flight writes → Finding 1 compounds.
- **Evidence:** §4.1 no leeway config on `jose` verify. §4.3 contractor check reads `expires_at` — no skew tolerance. §4.16 bootstrap token "time-limited" — no spec. §4.14 CLI "Auto-refresh on expiry" assumes monotonic agreement. No mention of NTP as operational requirement in §deployment.
- **Suggested fix:** (1) Use Postgres `now()` as single source of truth for all expiry reads — resolver fetches "now" in the same query it checks expiry. (2) Configure `jose` verify with `clockTolerance: '30s'`. (3) Document NTP as a required operational precondition in deployment docs. (4) `vibeflow admin health` shows server clock skew vs NTP pool. (5) Write_queue lease uses DB-side `now() + interval '5 min'`, not app-computed timestamps.

---

## Unresolved Questions

1. **Meta-repo force-push recovery at runtime**: Phase 2 §2.3 handles force-push by `reset --hard` + full reindex. But meta-repo holds `teams.yaml` — during the reindex window (could be minutes for a large workspace), is RBAC served from stale cache or is it unavailable? What if force-push removed admin entries?
2. **Write tool output determinism**: `task_create` returns `task_id` but the actual file merges asynchronously via PR. If the PR never merges (human review), does the returned `task_id` become a dangling promise? Can a subsequent `task_update_status` find the task before it's merged?
3. **Concurrent `task_update_status` on the same task**: Two users hit the same task. Both open PRs to the same file with different bodies. GitHub blocks the second merge with a conflict. What's the user-facing behavior? Phase 4 §4.8 doesn't specify.
4. **Dashboard SSR cache invalidation**: Phase 3 is "server-rendered heavy". On a write that auto-merges, how does the dashboard's SSR cache know to refresh? Polling interval not specified, no SSE invalidation path.
5. **OIDC group claim mapping drift**: `OIDC_GROUP_CLAIM` env var sets the claim key. If IdP renames the claim (or a user's groups change mid-session), how is the session re-evaluated? §4.1 has no "re-resolve on token refresh" step.
6. **Backup/restore**: The plan has no backup procedure for Postgres, clones, or the meta-repo mirror. A restore is never tested in success criteria. What does day-2 disaster recovery look like?
