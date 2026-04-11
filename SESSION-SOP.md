# Minilab — Session SOP
# Updated: 2026-04-08

## Session Startup (every time)
1. Read `CLAUDE.md` (project root) — master rulebook
2. Read `docs/startup/quick-context.md` — current state
3. Read `docs/startup/recent-decisions.md` — last ~50 decisions
4. Read `docs/startup/actual-schema-columns.md` — ONLY if doing DB/API work
5. Read `docs/startup/GRAPH_REPORT.md` — knowledge graph structure (god nodes, communities, connections)
6. Run: `git log --oneline -5` and `git status`

## After Every Task
- Schema changes → update `docs/startup/actual-schema-columns.md` in SAME commit
- New env vars → update `docs/startup/env-required.md` in SAME commit
- Architecture decisions → append to `docs/startup/recent-decisions.md` summary table + detail section with next D-XXXX
- New table → create ai_view_ + add to generic_read allowlist + add keywords to ai_tools
- Portal routing changes → REQUIRES HUMAN APPROVAL (do not modify `lib/auth/get-portal-path.ts` — D-0370)
- Auth gate changes → PWA routes use `getSession()`, BM portal routes use `requirePermission()` — never swap (D-0355/D-0356)
- UI fixes → verify with Preview tool (POST /api/dev/preview-session, navigate, screenshot) before committing
- Service worker changes → bump CACHE_NAME in public/sw.js
- Major architecture changes → regenerate graph: `/graphify . --update --obsidian`
- Commit: `git add . && git commit -m "[prefix]: description (D-XXXX)"` — do NOT push after every commit. Push ONCE at end of session (or when human says push). Before pushing, run `npx next build` to verify zero type errors (warnings OK). Vercel Hobby plan: every push = full production build.

## Prefixes
`feat:` | `fix:` | `docs:` | `refactor:` | `chore:`

## RPC Caller Rule
When changing ANY RPC signature: grep ALL `.rpc('fn_name')` callers across entire codebase, update ALL in same commit, then run `NOTIFY pgrst, 'reload schema'`. No exceptions.

## Preview Verification (after UI fixes)
1. Before committing any UI change, verify it visually:
   - Call POST `/api/dev/preview-session` with `{ "secret": "<PREVIEW_BYPASS_SECRET>" }` body to get an auth cookie
   - Navigate to the changed page using the Preview tool
   - Take a screenshot
   - Confirm the fix is visible in the screenshot
2. If the screenshot shows the fix did NOT take effect, investigate before committing.
3. This only works in local dev (`PREVIEW_BYPASS_SECRET` must be set in `.env.local`).

## For deep reference
- Full decisions history: docs/startup/old/decisions.log
- Full session history: docs/startup/old/session-context.md
- Original SOP: docs/startup/old/SESSION-SOP.md
