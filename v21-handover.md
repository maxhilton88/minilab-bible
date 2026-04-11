# Minilab — V21 Session Handover
# Created: 2026-04-10 | Covers: V20 (D-0491–D-0507)

---

## 1. V20 Summary — Everything Shipped

### V20 Audit (pre-batch)
**Focus:** Full platform audit before CHV launch — 81 findings across 7 categories.
- Ran comprehensive audit across 334 pages, 528+ API routes, 190+ DB tables.
- Categories: BM Staff (12 findings), Stub Pages (8), Dead Code (3 actionable), God Nodes (5), Type Safety (29), UI Consistency (4), API Health (20).
- Findings logged in `docs/startup/V20-AUDIT.md`.
- **Graphify knowledge graph installed:** graphify-out/ — 1349 files, 1902 nodes, 2848 edges, 135 communities. God nodes: POST() (46 edges), GET() (38), tgSend() (14), executeFormSubmission() (14), processClawBotMessage() (14). Wired into session startup (SESSION-SOP.md step 5 reads docs/startup/GRAPH_REPORT.md).
- **Push rule changed:** ONE TASK = ONE GIT COMMIT. Push ONCE at end of session after `npx next build` passes. Prevents Vercel Hobby build quota waste.

### V20 Batch 1 — Type Safety Critical Fixes (unnumbered)
**Focus:** Clear P1 bugs found in audit before shipping feature fixes.
- **FK ambiguity:** `users!inner(…)` on `user_roles` changed to `users!user_roles_user_id_fkey(…)` in 4 locations: `lib/ai/assignment-engine.ts` (L174, L197), `lib/building-state.ts` (L105), `app/api/onboard/committee/route.ts` (L19), `lib/telegram/group-targets.ts` (L62, L81, L100). Root cause: `user_roles` has two FK columns to `users` (user_id + assigned_by) — PostgREST returns empty results silently when ambiguous.
- **FaceEnrollModal wrong ID fixed:** `app/bm/staff/all/page.tsx` L802 now passes `user.id` (users.id) instead of `user_roles.id` for internal staff enrollment. Was silently failing or writing descriptors to wrong face slot.
- **3 dead files deleted:** `components/bm/console/AIFeed.tsx`, `components/bm/console/ConsoleAnalytics.tsx`, `lib/retry.ts` (withRetry). None imported anywhere — confirmed by graph analysis.

### V20 Batch 2 — D-0345 Violations + BM Staff Page (migration 071)
**Focus:** Clear all 22 attendance_logs PostgREST violations, fix BM staff page issues.
- **Migration 071:** 6 new building-wide attendance Postgres RPCs — `get_attendance_today_by_building`, `get_attendance_history_by_building`, `get_attendance_stats_by_building`, `get_late_arrivals_by_building`, `get_absent_staff_by_building`, `get_attendance_by_shift`. Applied to Supabase. `NOTIFY pgrst, 'reload schema'` run.
- **22 PostgREST reads replaced:** `lib/analytics/attendance.ts` (4×), `lib/building-state.ts` (1×), `lib/crons/attendance-sync.ts` (3×), `app/api/bm/dashboard/route.ts` (1×), `app/api/bm/reports/route.ts` (1×), `app/api/bm/reports/snapshot/route.ts` (1×), `app/api/bm/reports/daily/route.ts` (1×), `app/api/bm/contractors/[id]/route.ts` (1×), `app/api/bm/contractors/[id]/staff/[staffId]/attendance/route.ts` (3×), `app/api/bm/staff/all/route.ts` (1×), `app/api/security/*` (3×), `app/api/cleaning/*` (3×), `app/api/kiosk/*` (2×), `app/api/org/route.ts` (1×), `app/api/attendance/clock/route.ts` (1× read). All D-0345 violations now cleared.
- **Committee excluded from staff queries (D-0498):** `app/bm/staff/page.tsx`, `app/api/bm/staff/route.ts`, `app/api/bm/staff/all/route.ts` — committee role removed from staff query filters. Committee members are elected owners, not building staff.
- **Nav double-highlight fixed (D-0499):** `components/ui/MinilabShell.tsx` — "most specific match wins" logic. When multiple nav items prefix-match the current pathname, only the longest match highlights. Eliminates `/bm/staff` + `/bm/staff/all` simultaneous highlight.
- **BM sidebar nav relabeled (D-0500):** "Staff" → "MO Staff". "Attendance" link added to nav section. `app/bm/layout.tsx` updated.
- **BM Attendance page built (D-0501):** `/bm/attendance` — full-page attendance view with date picker, category filter (All/MO/Security/Cleaning), per-staff daily in/out pairs via `get_attendance_history_by_staff` RPC. No PostgREST reads.

### V20 Batch 3 — Console + CHV Blocker Fixes (D-0491–D-0497)
**Focus:** Fix CHV launch blocker, regen types, wire console features.
- **resolveBuilding login_slug (D-0491):** Replaced `ilike(name, ...)` with `eq(login_slug, ...)` in 3 files. UUID fallback retained. CHV launch blocker resolved — building names with special chars or spaces were failing.
- **TypeScript types regenerated (D-0492):** `packages/shared/types/database.types.ts` regenerated from Supabase (was stale since 2026-03-25). Covers migrations 001–070.
- **TG resolveUser skipPrompt (D-0493):** `skipPrompt` param (default false) added to `resolveUser()`. When true, skips "Share Phone Number" prompt for unknown senders. Used for silent lookups.
- **Sidebar credit widget replaced (D-0494):** Old AI/WA credit widget removed. New per-building collection balance widget with ring chart + dark glassmorphism (`#1a1a2e→#16213e→#0f0f1a`). Shows collection rate %, total outstanding (RM), overdue/total units. `GET /api/bm/collection/summary`.
- **Console Documents from chat (D-0495):** Console Documents section auto-populates from `messages` table where `media_url` is set for unit residents. "From Chat" section shows sender name, file type icon, date. Below manual building_documents.
- **Access Cards wired (D-0496):** `access_cards` table (migration 070) fully wired to Console. Unit-details API returns card list with resident names. Hub section shows card_number/status/issued_date. Deactivate action via `PATCH /api/bm/console/access-cards`.
- **Case view for unitless cases (D-0497):** Cases without `unit_id` now load conversation history in console middle panel + note-only compose bar. Note API accepts `case_id` directly. Compose bar auto-switches to Note mode.

### V20 Batch 4 — Error Handling Sweep (D-0502–D-0505)
**Focus:** Fix 44+ silent error swallows across API routes.
- **Error destructuring sweep (D-0502):** 8 API routes fixed — all Supabase calls now destructure `{ data, error }`. Routes: `forms/[buildingSlug]` (resolveBuilding + 5 smart fields + template lookup), `request-otp` (resolveBuilding), `kiosk/verify` (device lookup), `console/timeline/[id]` (6 lookups), `console/contacts` (messages query), `contractors/[id]` (3 queries), `leave/approve` (2 helpers), `guard/vehicles` (autocomplete). Also converted `.single()` → `.maybeSingle()` for device lookup and template lookup.
- **God node try-catch (D-0503):** `tgSend()` (`lib/telegram/notify.ts`) wrapped in top-level try-catch — prevents unhandled rejection if `getCredential()` throws. `executeFormSubmission()` (`lib/ai/actions/form-executor.ts`) switch dispatcher wrapped — prevents propagation from sub-executor Supabase network errors.
- **Empty catch blocks (D-0504):** 3 routes — `pmc/reports` (2× AGM+tribunal queries now log errors), `pmc/dashboard` (Redis cache set logs error), `approve-building` (building_onboarding insert logs error).
- **Public route hardening (D-0505):** `forms/[buildingSlug]` rate-limited (30 GET/min, 10 POST/min per IP via `rateLimitByIP()`). Smart field `residents` query returns `unit_number` only — no full names on public endpoint (prevents resident enumeration). `kiosk/verify` rate-limited (30 face attempts/hour per device_token).

### V20 Batch 5 — Polish (D-0506–D-0507)
**Focus:** Final polish items before V20 close.
- **force-dynamic on billing routes (D-0506):** `export const dynamic = 'force-dynamic'` added to `app/api/credits/route.ts`, `app/api/billing/usage/route.ts`, `app/api/billing/purchases/route.ts`. Prevents stale session data from Next.js static cache.
- **Superadmin stub pages hidden (D-0507):** `app/superadmin/layout.tsx` — removed "Billing Settings" (`/superadmin/billing-settings`) and "Supplier Pricing" (`/superadmin/supplier-pricing`) from nav. Both were stubs with `setTimeout(500)` fake data and no real API. Page files kept on disk for future implementation. Unused `DollarSign` import removed.

---

## 2. DB Migrations Applied in V20

| Migration | Date | Summary |
|-----------|------|---------|
| 070 | 2026-04-10 | `access_cards` table: id, building_id, unit_id (nullable), resident_id (nullable), card_number (text), status (active/deactivated/lost/returned), issued_date (date), deactivated_at (timestamptz), notes (text), created_at, updated_at. RLS enabled. |
| 071 | 2026-04-10 | 6 building-wide attendance RPCs: `get_attendance_today_by_building(p_building_id, p_date)`, `get_attendance_history_by_building(p_building_id, p_start_date, p_end_date)`, `get_attendance_stats_by_building(p_building_id, p_date)`, `get_late_arrivals_by_building(p_building_id, p_date)`, `get_absent_staff_by_building(p_building_id, p_date)`, `get_attendance_by_shift(p_building_id, p_shift_id)`. No new columns. |

**Note:** Migrations 001–069 were applied in V1–V19 (see recent-decisions.md and v20-handover.md for history).

---

## 3. Pending Code Fixes (Priority Order)

### P1 — Critical (fix before wider rollout)
1. **`.single()` triage (507 calls, 259 files):** Many are safe (PK lookups inside try-catch). Triage order: auth routes, webhooks, public-facing routes first. For each: if 0-row is a valid state (not-found scenario), convert to `.maybeSingle()` + null check. See V20-AUDIT.md item #72.
2. **Superadmin stub APIs not built:** `/superadmin/billing-settings` and `/superadmin/supplier-pricing` — pages exist on disk, hidden from nav, but APIs (`GET/PATCH /api/superadmin/billing-settings`, `GET /api/superadmin/supplier-pricing`) not implemented. Build when billing infra is ready.

### P2 — Important (before wider rollout)
3. **Sidebar collection widget redesign:** Widget removed in D-0494 but no replacement designed. `/bm/layout.tsx` sidebar currently shows no widget. Needs new design decision.
4. **BM Attendance page polish:** Date range CSV export not yet implemented. No pagination for buildings with 100+ staff.
5. **Remaining `.single()` calls in POST handlers:** `forms/[buildingSlug]/[templateSlug]/route.ts` POST handler still uses `.single()` for building + template lookups (L115, L124, L135). Low risk (building was already resolved) but worth cleaning.

### P3 — Nice to have
6. **30 isolated graph nodes:** Most verified as OK (docs by nature, framework configs). `lib/responsive/breakpoints.ts` — only self-references, verify re-export or delete.
7. **Absent count hardcoded:** `app/api/bm/staff/all/route.ts` L128 — `30 - present` incorrect for months with <30 working days. Calculate actual working days or remove metric.
8. **BM staff All page UX:** 12-column grid too wide for <1400px, no pagination (slow for 100+ staff), CSV export shows raw "source" column.

---

## 4. Pending Human Tasks

| Task | Owner | Priority | Notes |
|------|-------|----------|-------|
| Set `DISABLE_DEV_LOGIN=true` in Vercel | Max | P1-CHV | Prevents demo accounts in production |
| CHV onboarding — Hoe Zee How | Max | P1 | Cyber Heights Villa, 700+ units. BM account creation + building setup + WA/TG channel config |
| E2E test with Seeteng at Lumi Residency | Max | P1-CHV | First real-world building walk-through |
| CHV UAT sign-off | Hoe Zee How | P1-CHV | Full flow: BM login → staff setup → resident onboard → WA AI test |
| Telethon keepalive monitoring | Max | P2 | Verify daily cron alert fires correctly. SSH: `ssh minilab@34.42.210.102` |
| Check WA minilab_notification template | Max | P2-CHV | Ensure template approved for CHV WABA account |
| Apply migration 071 `NOTIFY pgrst` | Max/Claude | P2 | Run `NOTIFY pgrst, 'reload schema'` in Supabase SQL editor if schema cache is stale |

---

## 5. Future Features (V21 Backlog)

### BM Portal
- BM Attendance page: CSV export, pagination, shift management view
- Sidebar collection widget redesign (removed in D-0494)
- `.single()` → `.maybeSingle()` triage — 507 calls, triage by priority
- Superadmin billing-settings + supplier-pricing APIs (hidden stubs)

### Console
- Console notifications panel — recent building events in right panel
- Multi-building AI context — AI knows all buildings a BM manages
- Console Documents: auto-populate from media files (D-0468 TODO)

### AI
- MOA Phase 2 — Electron/Python desktop agent for Advelsoft (Seeteng's current software)
- AI proactive pings refinement — smarter timing, per-unit rate limiting
- AI cost analytics in superadmin (stub page exists at /superadmin/ai-costs)

### Resident Portal
- Resident dashboard (balance, announcements, bookings, tasks)
- Resident mobile-optimised PWA
- Community feed

### Operations
- Procurement 2.0 — Telegram hardware group auto-tracking
- AGM voting system (migration exists, UI stub)
- Form 11 auto-generation from collection_accounts

### Infrastructure
- Superadmin billing-settings real API (platform_settings KV or dedicated table)
- Supplier pricing API (aggregate from purchase order history)
- WhatsApp template library management in BM portal

---

## 6. CHV Launch Readiness

| Category | Item | Status | Notes |
|----------|------|--------|-------|
| **Infrastructure** | Vercel hosting | ✅ Live | minilab.my |
| **Infrastructure** | Supabase DB | ✅ Live | nncbsumdmsyjjrgdjjea (Singapore) |
| **Infrastructure** | Cloudflare R2 | ✅ Live | media.minilab.my |
| **Infrastructure** | Telethon VM | ✅ Live | 34.42.210.102:8443 (keepalive cron active) |
| **Security** | DISABLE_DEV_LOGIN | ❌ Pending | Must set in Vercel before CHV |
| **Security** | TOTP 2FA superadmin | ✅ Done | D-0384 |
| **Security** | TG webhook secret | ✅ Done | D-0430 |
| **Security** | WA HMAC validation | ✅ Done | D-0430 |
| **Security** | Public route rate limiting | ✅ Done | D-0505 — forms + kiosk |
| **Auth** | BM login (Telegram QR) | ✅ Done | Per-building bot tokens |
| **Auth** | Guard VMS kiosk | ✅ Done | Multi-gate + passcode system (D-0442) |
| **Auth** | Dev login disabled | ❌ Pending | Human task — Vercel env var |
| **Building** | Building provisioning flow | ✅ Done | Superadmin 3-step wizard (D-0383) |
| **Building** | login_slug routing | ✅ Done | resolveBuilding uses eq(login_slug) — D-0491 |
| **Communication** | WhatsApp templates | ⚠️ Check | Verify minilab_notification approved for CHV WABA |
| **Communication** | Telegram groups | ✅ Done | Auto-create, auto-invite, keepalive, force-delete |
| **AI** | V4 pipeline (WA+TG) | ✅ Done | Async + dedup + 24h window |
| **AI** | Human takeover | ✅ Done | shouldSkipAI() + auto-escalation |
| **Console** | BM console all features | ✅ Done | 3-view, contact mgmt, 10 right-panel sections |
| **Console** | Error handling hardened | ✅ Done | D-0502–D-0504 sweep |
| **Onboarding** | BM 8-stage PWA journey | ✅ Done | /app/onboard — all 8 stages |
| **Attendance** | BM Attendance page | ✅ Done | D-0501 — /bm/attendance |
| **Reporting** | Basic analytics | ✅ Done | Committee analytics page |
| **E2E test** | Lumi Residency test | ❌ Pending | Seeteng walk-through |
| **E2E test** | CHV UAT | ❌ Pending | Hoe Zee How sign-off |

**CHV launch blockers remaining:** DISABLE_DEV_LOGIN (Vercel env var), WA template check, E2E tests, CHV building provisioning.

---

## 7. Key Architecture Decisions to Carry Forward

| Rule | Source | Never Break |
|------|--------|-------------|
| Portal routing: `lib/auth/get-portal-path.ts` is single source of truth | D-0370/D-0379 | Requires human approval to change |
| Auth gate: PWA routes use `getSession()`, BM portal uses `requirePermission()` | D-0355/D-0356 | Never swap |
| Attendance: ALL reads via Postgres RPCs — never PostgREST from `attendance_logs` | D-0345 | Schema cache errors on PostgREST |
| RPC caller rule: change signature → grep ALL callers → update ALL → NOTIFY pgrst | D-0347/D-0348 | No exceptions |
| AI routing: DeepSeek V3 (`deepseek-chat`) for ROUTINE, Claude Sonnet (`claude-sonnet-4-20250514`) for SENSITIVE | Architecture | Never alias model strings |
| AI human takeover: `shouldSkipAI()` checked before runV4Pipeline in ALL channels | D-0451 | Fail-open on errors |
| Face = identity: no manual clock-in fallback, no name dropdown | D-0338 | Non-negotiable |
| user_roles FK hint: `users!user_roles_user_id_fkey` when joining users from user_roles | D-0490 / V20-B1a | Two FKs on user_roles → ambiguous → empty results |
| TG group delete: Telethon confirmation required; DB-only delete via `?force=true` | D-0488 | Prevents orphaned TG groups |
| V4 pipeline only for WA: V3 removed D-0431 | D-0431 | No regression to V3 |
| WA 24h window: check last_inbound_at, use minilab_notification template outside window | D-0450 | Meta freeform messaging policy |
| Image uploads: compress client-side (lib/image-compress.ts) before R2 upload | D-0337 | All photos, not just some |
| Internal notes: channel='internal_note' (not 'internal') | D-0449 | 'internal' silently fails |
| Sender auto-link: lib/sender-profiles/auto-link.ts on WA inbound + resident creation | D-0476 | Keeps Contact view populated |
| Public endpoints must rate-limit: use `rateLimitByIP()` for GET, `rateLimitByIP()` for POST (lower limit) | D-0505 | forms/[buildingSlug] + kiosk/verify pattern |
| Supabase queries: always destructure `{ data, error }`, never `const { data }` alone | D-0502 | Silent null on error, no 500 returned |
| Push rule: ONE TASK = ONE COMMIT. Push ONCE at end of session after `npx next build` passes | V20 | Vercel Hobby: every push = full production build |
| Graphify: run `/graphify . --update --obsidian` after major architecture changes | V20-INF | Keep docs/startup/GRAPH_REPORT.md current |

---

## 8. Supabase Project Info

| Item | Value |
|------|-------|
| Project ID | `nncbsumdmsyjjrgdjjea` |
| Region | Singapore (ap-southeast-1) |
| URL | See `NEXT_PUBLIC_SUPABASE_URL` in .env.local |
| Edge Functions | `minilab-process-message` (AI pipeline entry point) |
| pgvector | Enabled — 128-dim face descriptors in `face_descriptors` table |
| Migrations applied | 001–071 |
| Types file | `packages/shared/types/database.types.ts` — regen with `npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea` |
| Last types regen | 2026-04-10 (V20, migration 070 — migration 071 is RPCs only, no columns, no regen needed) |

---

## 9. GCP VM (Telethon Microservice)

| Item | Value |
|------|-------|
| Instance | e2-micro (free tier) |
| IP | 34.42.210.102 |
| Port | 8443 |
| Health endpoint | `GET /keepalive` |
| Keepalive monitoring | Daily cron in lib/crons/ — alerts MASTERAI_CHAT_ID if down |
| Session re-auth | SSH into VM + `python3 authorize_session.py` (human task if session expires) |
| Env vars | `TELEGRAM_GROUP_SERVICE_URL`, `TELEGRAM_GROUP_SERVICE_SECRET` |

---

*Generated: 2026-04-10 at end of V20 session (D-0507)*
