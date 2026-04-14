---
phase: 3
title: Minimal HTML Admin Pages
status: pending
effort: 3-5 days
depends_on: [phase-02]
---

# Phase 3: Minimal HTML Admin Pages

## ⓘ Red Team Changes (Applied 2026-04-14)

**Finding #1 (Critical)**: Delete full Next.js dashboard. Replace with 3 server-rendered HTML pages.

**Was**: Next.js 15 App Router + tRPC + React + Tailwind + shadcn/ui + i18n (en+vi) + dark mode + 10 routers + 25 routes + 30 components + Playwright + Lighthouse + axe-core — 2-3 weeks work for "nice to have" visibility.

**Now**: 3 simple HTML pages served by Fastify/Hono on the existing Phase 2 server:
- `GET /admin/epics` — epic list with status + progress
- `GET /admin/kanban` — cross-repo task board
- `GET /admin/compliance` — lint score per repo

**Rendering**: Plain server-side HTML via template literals or `handlebars`/`ejs` (no React, no SSR framework). Tailwind CDN for minimal styling. Zero client JS except auto-refresh meta tag.

**Auth**: Same admin shared-secret as Phase 2 (Phase 4 swaps to OIDC cookie). Refuses non-loopback without auth.

**Cut to post-MVP** (Phase 5 web editor):
- Full Next.js App Router + tRPC + React
- Wiki browser (use CLI `grep` or Claude Code for now)
- PRD viewer (read files directly in repo)
- Universal search (use Claude Code `cross_repo_search`)
- Universal activity feed (use `git log` in meta-repo)
- Agent distribution view (use `vibeflow agents status` per repo)
- i18n, dark mode, themes
- Accessibility audit (add in Phase 5)
- E2E/Lighthouse/a11y tests

**Effort**: 2-3w → 3-5 days.

## Remaining Content (updated to reflect cuts)

Below sections reflect original scope — only §15.1-15.5 (3 HTML pages) apply to MVP. Rest is Phase 5+ reference.

---

## Key Insights (MVP)

- **Minimal HTML, not framework**: just enough for leads to see state, not a product
- **Same Node process** as MCP server — Fastify route handlers render HTML directly
- **Auto-refresh every 30s** via `<meta http-equiv="refresh" content="30">` — no WebSocket needed
- **Read-only**: no forms, no buttons, no mutations. Click-through to doc viewer (phase 5+).

## Requirements (MVP)

**Functional** (cut to essentials):
- `GET /admin/epics` — list epics with: id, title, status, progress bar (done tasks / total), linked repos
- `GET /admin/kanban` — cross-repo kanban: columns per status, cards showing task id/title/assignee/epic
- `GET /admin/compliance` — table per repo: lint errors, warnings, PRD valid?, staleness days, score 0-100

**Non-functional**:
- Response < 500ms (Postgres query + template render)
- Same auth as Phase 2 admin API
- Works in any modern browser, no JS required

## Architecture (MVP)

```
server/src/
├── admin/
│   ├── html-server.ts           (Fastify route handlers)
│   ├── templates/
│   │   ├── epics.html.ts        (template literal)
│   │   ├── kanban.html.ts
│   │   └── compliance.html.ts
│   └── style.css                (minimal, served inline)
└── (Phase 2 existing files)
```

No Next.js, no React, no tRPC, no Tailwind build, no i18n. Dependencies: existing Fastify/Hono + existing Postgres client.

## Implementation Steps (MVP)

1. Add 3 Fastify routes under `/admin/`
2. Write HTML template literal functions for each page (pure functions: data → string)
3. Query Postgres directly via existing Phase 2 DB client (same queries used by future tRPC routers)
4. Inline minimal CSS or reference Tailwind CDN
5. Add `<meta http-equiv="refresh" content="30">` for auto-refresh
6. Gate behind same shared-secret as admin API

## Todo List (MVP)

- [ ] Fastify route `/admin/epics` + HTML template
- [ ] Fastify route `/admin/kanban` + HTML template
- [ ] Fastify route `/admin/compliance` + HTML template + score calculation
- [ ] Minimal inline CSS (or Tailwind CDN)
- [ ] Auth middleware for admin routes
- [ ] Manual smoke test in browser

## Success Criteria (MVP)

- Open `https://vibeflow.domain/admin/epics` — sees live epic list from Postgres
- Kanban page renders 50 tasks without pagination fuss
- Compliance score computed correctly per repo
- Auto-refreshes every 30s
- No client JS required (works with JS disabled)

## Risks (MVP-specific)

| Risk | Mitigation |
|---|---|
| HTML template injection (XSS) | Escape all Postgres values via `escape-html` util |
| Auth cookie not yet implemented | Use same shared-secret header as Phase 2 admin |

---

## === Below: Original Phase 3 Scope (deferred to Phase 5+) ===

## Key Insights

- **Same Node process** as MCP server (§4.1): Next.js serves dashboard, Fastify/Hono serves MCP + admin. Shared Postgres client, shared schemas.
- **Dashboard is NOT web editor**: audience = leader watching, not creator. Phase 5 adds writing.
- **tRPC for API**: tight typing with zod schemas, shared with MCP tool schemas.
- **No auth in Phase 3**: wire structure now, enforce in Phase 4. Shared secret or localhost-only for dev.
- **Server-rendered heavy, client-side light**: SSR for fast FMP, minimal client JS for interactivity.
- **Multi-kind ready**: UI groups projects by `kind`, but MVP only renders `code`. Placeholder empty states for product/design.

## Requirements

**Functional**:
- Workspace home (`/`): active epics, recent activity, quick links
- Projects list (`/projects`): filter by kind/team/status
- Project detail (`/projects/:id`): PRD, specs, kanban, activity, agents
- Epic list + detail (`/epics`): progress bars, linked repos, linked tasks
- Wiki browser (`/wiki`): tree view + content renderer + search
- Workspace kanban (`/kanban`): cross-repo task board, filterable
- Compliance view (`/compliance`): lint scores per repo, staleness, missing standards
- Agent distribution view (`/agents`): version adoption across repos
- Universal search (`/search`): hits from wiki + PRD + specs + code
- Activity feed (`/activity`): recent edits, status changes, team actions
- Project settings view (`/projects/:id/settings`): read-only `.vibeflow.yaml` display

**Non-functional**:
- First meaningful paint < 1.5s
- Server-render for all routes, progressive hydration
- Responsive (desktop primary, tablet OK, mobile read-only minimal)
- Accessible (WCAG AA, keyboard nav)
- Dark/light theme (system preference default)
- i18n: en + vi (string extraction from day 1)

## Architecture

**Added to server/** directory:
```
server/src/
├── (Phase 2 files — unchanged)
├── api/
│   ├── trpc.ts                    (extended with dashboard router)
│   └── routers/
│       ├── workspace.ts
│       ├── project.ts
│       ├── epic.ts
│       ├── kanban.ts
│       ├── wiki.ts
│       ├── prd.ts
│       ├── spec.ts
│       ├── search.ts
│       ├── compliance.ts
│       ├── agents-distribution.ts
│       └── activity.ts
└── web/                           ← Next.js app
    ├── app/
    │   ├── layout.tsx             (root, theme, i18n)
    │   ├── page.tsx               (workspace home)
    │   ├── projects/
    │   │   ├── page.tsx           (list)
    │   │   └── [id]/
    │   │       ├── page.tsx       (project home)
    │   │       ├── docs/[path]/page.tsx  (doc viewer)
    │   │       ├── kanban/page.tsx
    │   │       ├── settings/page.tsx
    │   │       └── activity/page.tsx
    │   ├── epics/
    │   │   ├── page.tsx
    │   │   └── [id]/page.tsx
    │   ├── wiki/
    │   │   ├── page.tsx           (tree view)
    │   │   └── [...slug]/page.tsx (page viewer)
    │   ├── kanban/page.tsx        (workspace-wide)
    │   ├── compliance/page.tsx
    │   ├── agents/page.tsx
    │   ├── search/page.tsx
    │   └── activity/page.tsx
    ├── components/
    │   ├── layout/
    │   │   ├── app-shell.tsx
    │   │   ├── sidebar.tsx
    │   │   ├── header.tsx
    │   │   └── breadcrumbs.tsx
    │   ├── kanban/
    │   │   ├── kanban-board.tsx
    │   │   ├── task-card.tsx
    │   │   └── filters.tsx
    │   ├── markdown/
    │   │   ├── markdown-renderer.tsx  (mermaid, syntax highlight, wiki-links)
    │   │   └── toc.tsx
    │   ├── doc-viewer/
    │   │   ├── doc-viewer.tsx
    │   │   ├── frontmatter-panel.tsx
    │   │   └── lint-badge.tsx
    │   ├── epic-progress.tsx
    │   ├── agent-version-matrix.tsx
    │   ├── compliance-score.tsx
    │   └── search-results.tsx
    ├── lib/
    │   ├── trpc-client.ts
    │   ├── format.ts              (dates, durations, status colors)
    │   └── i18n/                  (en.json, vi.json)
    ├── styles/
    │   └── globals.css
    └── next.config.js
```

**Dependencies** (add to server/package.json):
- `next@15` (App Router)
- `react@18`
- `@trpc/server`, `@trpc/client`, `@trpc/react-query`
- `@tanstack/react-query`
- `zustand` (minimal client state)
- `tailwindcss` + `@tailwindcss/typography`
- `shadcn/ui` components (radix-based)
- `lucide-react` (icons)
- `react-markdown` + `remark-gfm` + `rehype-highlight` + `rehype-mermaid`
- `zod` (shared with tRPC)
- `next-intl` (i18n)
- `class-variance-authority`, `clsx`, `tailwind-merge`

## Related Files

**Create**:
- `server/src/api/routers/*.ts` (10 routers)
- `server/src/web/app/**/*.tsx` (~25 route files)
- `server/src/web/components/**/*.tsx` (~30 components)
- `server/src/web/lib/**/*.ts` (utilities, i18n)
- `server/src/web/styles/globals.css`
- `server/tailwind.config.js`
- `server/next.config.js`

**Modify**:
- `server/src/index.ts` (mount Next.js handler on same HTTP server)
- `server/package.json` (add deps)

## Implementation Steps

### 3.1 Next.js integration

1. Add Next.js to server package, use custom server pattern (`next({ dev }).getRequestHandler()`) so MCP SSE + tRPC + Next.js share one process
2. Next config: `output: "standalone"` for Docker
3. Tailwind + shadcn/ui setup
4. Root layout with theme provider, i18n provider, tRPC provider

### 3.2 tRPC setup

1. `trpc.ts` main router merges sub-routers
2. Each router uses zod schemas (reuse from MCP tool schemas where possible)
3. Context middleware reads shared-secret auth (Phase 4 swaps to OIDC)
4. Client: `trpc-client.ts` with React Query integration

### 3.3 Routers (thin wrappers over Postgres queries)

Each router mirrors the MCP tool surface but tailored for UI:
- `workspace.getContext(workspaceId)` — overview data
- `project.list({ kind?, team?, cursor? })`
- `project.get(id)` — full project data including PRD, specs list, kanban summary
- `epic.list({ status?, quarter? })`, `epic.get(id)`
- `kanban.query({ repo?, team?, assignee?, status? })`
- `wiki.tree(scope)`, `wiki.get(path)`, `wiki.search(query)`
- `prd.get(repo)`, `prd.getAll({ repo? })`
- `spec.list(repo)`, `spec.get(repo, feature)`
- `search.universal(query)` — fans out to wiki/prd/spec/code
- `compliance.scoreByRepo()` — aggregate lint passes, staleness, missing standards
- `agents.distribution()` — matrix: agents × repos × version
- `activity.feed({ limit, cursor })` — recent edits/status changes

### 3.4 Core layout + shell

1. `app-shell.tsx`: sidebar + header + main content + optional right panel
2. Sidebar nav: Home / Projects / Epics / Wiki / Kanban / Compliance / Agents / Search / Activity
3. Header: workspace switcher (stub), search bar (Cmd+K), notifications bell (placeholder), user menu
4. Breadcrumbs based on route
5. Theme toggle (light/dark/system)
6. Language switcher (en/vi)

### 3.5 Workspace home

1. Cards: active epics (top 5), recent edits (top 10), team activity (top 10), my inbox (reviews — placeholder in Phase 3)
2. Quick stats: total projects, lint compliance %, active tasks
3. Getting started panel for empty state

### 3.6 Projects list + detail

1. List: card grid, filter pills (kind, team, status), sort (updated/name), search
2. Detail page layout:
   - Header: name, team, kind, epic link
   - Tabs: Overview | Kanban | Docs | Activity | Settings
   - Overview: PRD summary, recent tasks, linked epic, tech stack, owners
   - Docs: tree view of PRD/specs/wiki (read-only)
   - Kanban: project-level board
   - Settings: render `.vibeflow.yaml` as structured view

### 3.7 Doc viewer

1. `doc-viewer.tsx`: left panel frontmatter (status, owner, reviewers, updated), center markdown render, right panel ToC
2. Markdown renderer: syntax highlight, mermaid diagrams, wiki-link resolution via `wiki.resolveLink` tRPC call
3. Lint badge: click shows errors/warnings inline
4. No edit UI in Phase 3 — "Edit in editor" button stub (disabled with tooltip "Phase 5")

### 3.8 Epic views

1. Epic list: progress bars per epic (done tasks / total), linked repos chips, quarter
2. Epic detail: body markdown, linked_tasks table grouped by repo, dependency list, status history (if sync_history has it)

### 3.9 Kanban board (workspace-wide + project-scoped)

1. Columns per stage (from `kanban-stages.yaml` for kind=code)
2. Task cards: title, assignee avatar, epic chip, owner_agent icon (🤖 claude), linked PR badge
3. Drag-drop disabled (read-only); hover shows "move" disabled with tooltip
4. Filters: team, assignee, status, epic, updated-since
5. Empty state: "No tasks match filters"

### 3.10 Wiki browser

1. Tree view left (folders/files)
2. Content right (reuse doc-viewer without frontmatter panel)
3. Search bar at top, fuzzy + FTS backed
4. Breadcrumbs
5. Staleness warning banner if doc > 90 days

### 3.11 Compliance view

1. Table: Repo | Kind | Lint errors | Lint warnings | PRD valid? | PRD age | Missing std | Score (0-100)
2. Score = weighted sum: lint errors -5, warnings -1, missing PRD -10, stale PRD -3, etc.
3. Sortable columns, filterable by team
4. Badges: green/yellow/red
5. "Show issues" row expand to details

### 3.12 Agent distribution view

Matrix view (§18.11.2 wireframe):
- Rows: agents
- Columns: repos
- Cells: version installed (or blocked/frozen/excluded status)
- Header: total adoption %, pending PRs, blocked count
- Click agent → detail: changelog, affected repos, update status

### 3.13 Universal search

1. Single search box, results grouped by kind (wiki, PRD, spec, code)
2. Backend fans out to Postgres FTS + pgvector hybrid
3. Snippet highlighting
4. Click → go to source doc viewer

### 3.14 Activity feed

1. Chronological list: commits on meta-repo/product-repos that touched vibeflow-managed files
2. Event types: doc created/updated, task status changed, epic status changed, agent updated
3. Author, timestamp, affected project, diff link (hidden in Phase 3)
4. Filter by type, team, user
5. Polling every 30s or SSE in Phase 3.1

### 3.15 Empty states + error states

- Every page has loading skeleton + empty state + error state
- Dashboard-wide error boundary

### 3.16 Testing

- Component tests via Vitest + React Testing Library
- E2E smoke via Playwright: main user flows (home → project → doc → kanban)
- Lighthouse CI for perf budgets
- a11y check via `@axe-core/react`

## Todo List

- [ ] Next.js 15 integration with existing server process
- [ ] Tailwind + shadcn/ui setup
- [ ] Theme + i18n providers
- [ ] tRPC main router + middleware
- [ ] Router: workspace
- [ ] Router: project
- [ ] Router: epic
- [ ] Router: kanban
- [ ] Router: wiki
- [ ] Router: prd
- [ ] Router: spec
- [ ] Router: search
- [ ] Router: compliance
- [ ] Router: agents-distribution
- [ ] Router: activity
- [ ] App shell (sidebar + header + breadcrumbs)
- [ ] Workspace home page
- [ ] Projects list page
- [ ] Project detail page (with tabs)
- [ ] Doc viewer component
- [ ] Markdown renderer with mermaid/wiki-links
- [ ] Epic list page
- [ ] Epic detail page
- [ ] Kanban board component (read-only)
- [ ] Workspace kanban page
- [ ] Project kanban page
- [ ] Wiki tree + browser pages
- [ ] Compliance view
- [ ] Agent distribution matrix view
- [ ] Universal search page
- [ ] Activity feed page
- [ ] Loading/empty/error states across all pages
- [ ] i18n strings (en + vi)
- [ ] Component unit tests
- [ ] E2E smoke tests (Playwright)
- [ ] Lighthouse perf budgets
- [ ] a11y audit

## Success Criteria

- Navigate home → projects → project → doc without reloads
- First meaningful paint < 1.5s on home page
- All routes have loading + empty + error states
- Universal search returns results from wiki + PRD + specs
- Kanban board renders 100+ tasks without lag
- Compliance view computes score correctly per repo
- Agent matrix shows version adoption accurately
- Dark/light theme toggles, persists
- en/vi switches UI language
- Lighthouse score > 90 perf, > 95 a11y on main pages
- E2E smoke suite passes in CI
- Dashboard accessible via `localhost:3000` after `docker compose up`

## Risks

| Risk | Mitigation |
|---|---|
| Next.js integration conflicts with Fastify/Hono | Use Next custom server pattern, share HTTP server instance |
| tRPC type bundle bloat | Tree-shake + split routers, lazy load on client |
| Mermaid rendering heavy on client | Lazy-load `rehype-mermaid`, SSR where possible |
| Large wiki pages slow | Stream rendering, virtualize long lists |
| i18n setup overhead | Use `next-intl` native App Router support, minimal complexity |
| Shadcn components drift over time | Pin version, copy into repo not npm |
| Compliance scoring formula unclear | Document rubric, adjustable via config |
| Agent matrix combinatorial explosion (many agents × many repos) | Pagination or virtualization, collapse unused rows |
| Read-only UX confuses users expecting edit | Clear "Read-only, edit in CLI or Phase 5" messaging |
| Activity feed loads all events at once | Cursor pagination from day 1 |

## Security Considerations

- CSP headers, XSS sanitization in markdown render (allow-list)
- No `dangerouslySetInnerHTML` except in markdown component with sanitizer
- tRPC input validation via zod on every endpoint
- Shared secret auth in Phase 3 (not production ready; Phase 4 swaps to OIDC)
- Rate limit per IP even in dev
- No PII in logs

## Next Steps

- Phase 4 adds auth, makes dashboard multi-user ready
- Phase 5 adds editing capability (web editor), reuses dashboard shell + components
