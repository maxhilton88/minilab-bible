# Minilab — Session Context (Quick Start)
# Last updated: 2026-04-05 by attendance status stale fix — Next.js Data Cache + Cache-Control headers (D-0350)
# Rule: Update this file at the END of every session

---

## BUILD STATUS

| Area | Status | Details |
|------|--------|---------|
| V1 (T001-T075) | ✅ Complete | Foundation |
| V2 (T076-T132) | ✅ Complete | Features |
| V3 | ✅ Complete | Full portal builds |
| V4 Phase 37 (RAG + Memory) | ✅ Schema done | sender_profiles, document_chunks, memory compiler |
| V4 Phase 38 (AI Console) | ✅ CRM rebuild | tickets, 3-panel CRM console, unit timeline, search, quick actions |
| V4 Phase 39 (Email) | ✅ Schema done | email_log, Cloudflare routing live, Resend sending |
| V4 Phase 40 (RBAC) | ✅ Complete | staff_permissions, PermissionGate, 24 pages gated, 54 permissions, 105 API routes gated |
| V4 Phase 41 (WhatsApp) | ✅ Schema done | Working hours, templates, channel config |
| AI Agent | ✅ Complete | 26 admin handlers + 13 write handlers, multi-step collection, proactive pings, 39 ai_tools (36 + 4 new V13-9, 1 deactivated), Manglish+Malay keywords, web chat bubble, Telegram admin, file import, scorecard, generative UI form cards, multi-channel sync, async pipeline (waitUntil + Realtime) |
| Telegram Auto-Groups | ✅ Complete | Telethon microservice on GCP e2-micro. 8 categories, invite-based flow, per-group bot mode settings. telegram_group_settings (25 cols). |
| Face Recognition | ✅ Complete | Browser-based face-api.js + pgvector (128-dim descriptors, zero cloud cost). Provider-agnostic interface at lib/face/client.ts. FACE_PROVIDER env var switches between local (default) and rekognition. |
| Staff PWA | ✅ Complete | /app PWA for field staff (guards, cleaners, BM staff). Role-adaptive bottom nav, face clock-in, incident capture, before/after cleaning logs, task management, console mini. Sessions 16A-16C. |
| Console Unified Inbox | ✅ Complete | Inbox/CRM mode toggle. Inbox aggregates WhatsApp, Telegram DMs, groups, web chat, cases. Center panel: chat view. Right panel: context-adaptive master edit (contact profile, case full CRUD, group bot mode). Collapsible sidebar. Paginated threads (20/batch). "+ New" quick-create (tasks, residents). Sessions 17B + Console redesign. |
| V4 Phase 42 (OpenClaw) | ❌ Not started | Accounting software integration |
| V4 Phase 43 (Hardware) | ❌ Not started | Access cards, barriers, lifts |
| V4 Phase 44 (Advanced) | ❌ Not started | Phone, BI, multi-language, native apps |
| Fines & Clamping | ✅ Complete | Guard PWA 3-step create + find/pay/unclamp. BM desktop with stats + waive + fine settings. Security portal. AI: fines/saman/clamp in generic_read. Migration 056. D-0332. |
| Leave Management | ✅ Complete | Employer-based approval (PMC→PMC staff, BM→JMB staff, company admin→workers). PWA + desktop views. Notifications. AI view. Migrations 054-055. D-0331. |
| Staff PWA Attendance | ✅ Complete | Unlimited in/out pairs per day. Face IS identity (no manual, no dropdown). All reads via Postgres RPCs. MYT timezone throughout. D-0338–D-0345. Migration 058: fixed NULL user_id bug in 4 RPCs. D-0347. Read path fixed: check/history/kiosk APIs now pass p_contractor_staff_id, /api/app/attendance rewritten from PostgREST to RPC, PostgREST schema cache reloaded. D-0348. Status stale fix: supabaseAdmin global fetch cache:'no-store' + check route Cache-Control headers + fetchCache segment config. D-0350. |
| BM Portal | ✅ ~97% | 58 pages, 24 sidebar items (added Fines), full CRUD. Import retry/clear UI. Staff management with card layout + face enrollment. |
| Security Portal | ✅ ~97% | Guard VMS: shift handover, plate autocomplete, batch parcels, attendance history, FOMEMA/WP expiry badges. Fines view added. |
| Cleaning Portal | ✅ ~93% | 10 pages, 9 nav items, areas + checklists + custom templates + expiry badges |
| Contractor Portal | ✅ ~90% | 15 pages, 12 nav items, job completion + real tenders |
| PMC Portal | ✅ ~87% | 9 pages, 8 nav items, Portfolio Command Center + compliance expiry alerts |
| Supplier Portal | ✅ ~85% | 10 pages, real data, settings save, tenders wired |
| Developer Portal | ✅ ~87% | 14 pages, real data, mock removed, contractors + tenders + DLP expiry warning |
| Resident Portal | ✅ ~88% | 10 pages, complaint form + photo listings + facility response + WA redirects |
| Superadmin | ✅ ~92% | TOTP 2FA, real health monitoring, building approval checklist |
| Store Portal | ✅ ~90% | 4 pages, real orders/revenue, server-side layout |
| Website | ✅ Complete | Landing page + voice sales agent + public property listings |
| Hardening | ✅ Complete | 7 fixes: middleware, sanitization, Zod, caching, audit, cleanup, e2e audit |

## PAGE / ROUTE COUNTS

| Portal | Pages | API Routes |
|--------|-------|------------|
| BM | 58 | 142 |
| Superadmin | 36 | 41 |
| Marketing/Website | 26 | — |
| Contractor | 15 | 17 |
| Developer | 14 | 7 |
| Security | 12 | 7 |
| Cleaning | 10 | 5 |
| Supplier | 10 | 6 |
| Register | 10 | 4 |
| Resident | 10 | 20 |
| PMC | 9 | 6 |
| Store | 4 | 6 |
| AGM | 4 | 1 |
| Auth | 4 | 16 |
| Opentender | 3 | 1 |
| Setup | 3 | 6 |
| Form | 2 | 2 |
| Guard | 1 | 9 |
| Attendance | 1 | 2 |
| Kiosk | 1 | 2 |
| Org | 1 | 1 |
| App PWA | ~37 | ~15 |
| Other (cron/webhooks/voice/etc.) | — | 52 |
| **Total** | **~287** | **~390** |

Note: App PWA (/app) grew significantly in V13 — added leave, fines, cleaning checklist+history, cleaning log, incident history, my-attendance. Page count via `find app -name "page.tsx" | wc -l` = 287 as of 2026-04-04.

## DATABASE

- **Tables:** 165 (live DB count as of 2026-04-04)
- **Migrations:** 57+ files applied to live DB. Latest applied: migration 057 (attendance_logs user_id + contractor_staff_id nullable)
- **Enums:** 66+ custom types (live DB)
- **Types file:** 8080 lines (regenerated 2026-03-25 — stale, regenerate before doing schema work)
- **New tables (V13 session):** leave_policies, leave_balances, leave_requests, vehicle_fines, fine_settings
- **New AI views (V13):** ai_view_leave, ai_view_fines
- **New Postgres RPCs (V13):** get_attendance_today, get_attendance_history, find_open_attendance, count_today_sessions, get_attendance_today_by_staff, get_attendance_history_by_staff, match_face
- **Enhanced (migration 057):** attendance_logs gained user_id (nullable), contractor_staff_id made nullable (was NOT NULL)
- **Enhanced (migration 057c):** face_enrollments gained reference_photo_url column
- **Earlier:** New tables (migration 041): personal_properties, property_listings, utility_bills, tenancy_documents. Enhanced (migration 046): audit_log gained building_id, source, old_data, new_data. Rebuilt (migration 047/048): telegram_group_settings (25 cols), messages gained status + Realtime publication.

## INFRASTRUCTURE

| Service | Status | Details |
|---------|--------|---------|
| Hosting | Vercel | Auto-deploy on push to main |
| DNS | Cloudflare | Active (was Vercel, moved Session 3) |
| SSL | Cloudflare | Full (strict) |
| Database | Supabase | nncbsumdmsyjjrgdjjea, Singapore |
| Email send | Resend | Domain verified, working |
| Email receive | Cloudflare | Catch-all *@minilab.my → gmail. Webhook endpoint ready at /api/webhooks/email. Per-building routing rules need manual Cloudflare dashboard setup. |
| Media storage | R2 + Supabase | R2 bucket minilab-media, media.minilab.my |
| Face recognition | Amazon Rekognition | Provider wired, code ready. Needs 3 env vars (ACCESS_KEY_ID, SECRET_ACCESS_KEY, REGION). Graceful degradation when unconfigured. Migration to browser-based face-api.js + pgvector planned. |
| Telethon microservice | GCP e2-micro (free tier) | Telegram group auto-creation/deletion. Called via HTTPS from Next.js. Env vars: TELEGRAM_GROUP_SERVICE_URL, TELEGRAM_GROUP_SERVICE_SECRET. |

## ENV VARS

46 unique process.env references across the codebase (including FACE_PROVIDER + DISABLE_DEV_LOGIN, added 2026-03-30). See `docs/env-required.md` for full list.

## BLOCKED ITEMS

| Item | Blocked on | Human action needed |
|------|-----------|-------------------|
| AWS Rekognition setup | AWS env vars | Human to add 3 env vars (ACCESS_KEY_ID, SECRET_ACCESS_KEY, REGION) — code is ready |
| Telethon microservice deploy | GCP VM setup | GCP e2-micro VM must be running with Telethon Python service. TELEGRAM_GROUP_SERVICE_URL + SECRET needed in Vercel. |

## DECISIONS LOG

346 decisions logged (D-0001 through D-0346). File: `docs/decisions.log`.

## LAST SESSION SUMMARY

### V12 Closing Summary (2026-04-01)

V12 was a 6-session audit-driven fix batch covering ~95 items:
- P1: 28 security + error handling fixes (data leaks, webhook try/catch, AI resilience)
- P2: 12 performance + mobile fixes (pagination, caching, overflow, breakpoints)
- P3: 22 notification hooks + API contract fixes (WhatsApp/Telegram wiring, dead buttons)
- P4: 14 resident + guard + cleaning portal fixes (complaint form, handover, checklists)
- P5: 8 PMC + developer + superadmin fixes (TOTP 2FA, health monitoring, provisioning)
- P6: Polish + docs sync (import retry/clear, guard expiry badges, docs sync)

Key infrastructure added:
- Superadmin TOTP 2FA (otplib, per-user DB secrets)
- Real platform health monitoring (DB/Redis/WhatsApp/AI)
- Building approval with BM auto-provisioning + checklist modal
- Shared EmptyState component
- 6 error.tsx + 4 loading.tsx error boundaries
- AI provider timeouts + retry + fallback
- Redis fail-open pattern
- Announcement WhatsApp broadcast (max 500)
- Guard shift handover API
- Custom cleaning checklist templates
- Import retry/clear UI for failed raw_imports
- Guard FOMEMA/work permit expiry badges (matching cleaning staff pattern)

Zero PlaceholderScreen stubs remaining in resident portal.
All webhook handlers wrapped in try/catch.
All cron jobs notify MASTERAI_CHAT_ID on failure.

### V12-P5: PMC + Developer + Superadmin (2026-04-01)

8 fixes in one commit (2 verified as already done, 1 not needed):
- Superadmin TOTP 2FA: per-user DB secrets (users.totp_secret/totp_enabled),
  enrollment QR modal in settings page, 4 API routes (/status /generate /enable
  /disable), verify-totp route updated to check DB first then env var fallback
- Real health monitoring: live DB/Redis/WhatsApp/AI checks (Promise.race
  timeout), richer ServiceHealth type (status/lastChecked/detail), 4-card grid
  replacing old 3-card boolean display
- Building approval: 4-item verification checklist modal gating Approve button,
  checklist passed in request body, audit_log entry on approval
- PMC: Form 11 verified live (P3 already done), compliance expiry alerts:
  complianceExpiringCount + complianceExpiringItems per building, amber badge
  on dashboard building cards
- Developer: contractor dropdown verified (P3 already done), sidebar stub pages
  not needed (all pages have real content), DLP expiry warning: dev_units query
  (dlp_end_date ≤90d + open defects), red alert card at dashboard top
- New DB columns: users.totp_secret, users.totp_enabled (migration applied)
- TS errors: 0 | Build: pass | D-0299

### V12-P3: Notifications + API Contracts (2026-04-01)

21 files changed in one commit (22 fixes — 2 already done):
- 8 notification hooks wired: case status WhatsApp to resident, visitor QR
  WhatsApp, booking result WhatsApp, incident auto-escalation to case + BM
  Telegram, BM case assignment Telegram to worker, renovation/movein/booking
  BM Telegram alerts, resident welcome WhatsApp, announcement WhatsApp
  broadcast (max 500, batched 20/100ms, fire-and-forget)
- ImportSection → 3-step import pipeline (upload→preview→confirm)
- AGM results already real DB (no change), parcels API already existed
- Security + Cleaning dashboard View Details buttons wired with useRouter
- /contractor/org/[id] already existed (confirmed functional)
- Cleaning tender bid submission wired (modal + /api/marketplace/bids)
- Developer defect contractor assignment dropdown added
- PMC Form 11 hardcoded 'none' → live legal_records query
- Contractor bid fee from platform_settings (fallback 10)
- Tenancy doc real file upload (both unit + property tenancy pages)
- Supplier revenue date: string startsWith → Date object comparison
- Contractor store mobile cart: Sheet drawer with full item list
- Contractor visit times: arrival/departure time inputs (default: now)
- TS errors: 0 | Build: pass | D-0297

### V12-P2: Performance + Mobile Responsiveness (2026-04-01)

12 fixes in one commit:
- CRITICAL: unit_occupancy_status filtered via buildingUnitIds pre-fetch (no building_id column)
- CRITICAL: getUnitsForBuilding() paginated (default limit 100) + specific column select list
- HIGH: staff/all Redis cache (3min) + PMC dashboard N+1 eliminated (batch BM users query)
- HIGH: Residents page limit 200→50 with Load More button (server-side search unchanged)
- Redis cache added to units (5min), residents (2min, skip on search), security/dashboard (3min), pmc/dashboard (5min)
- getBlocksForBuilding() select list narrowed
- MinilabShell p-6 → p-4 lg:p-6 (all portals)
- 5 overflow-hidden table wrappers → overflow-x-auto (B2)
- 11 table wrappers changed/added overflow-x-auto (B3: procurement, assignment, petty-cash, pmc/reports ×2, store/billing, superadmin: tenders/wallets/tender-settings/billing-settings/integration-requests/onboarding-queue)
- ~18 grid layouts → responsive breakpoints (dashboard, ai-training, agm pages, facilities, tenders, units, supplier/buildings, org/dashboard, voice-agent TabsList)
- ~7 form dialog grids → grid-cols-1 sm:grid-cols-2 (legal ×6, profile, documents ×3, working-hours, contractor/schedule)
- TS errors: 0 | Build: pass | D-0296

### Documentation Sync + ChatBubble Bug Fix (2026-03-30)

**Health check:** Full post-AI-Agent scan — zero TypeScript errors, build passes 250 pages, all Telegram/web chat/AI pipeline/auth components working. Report: `docs/health-check-post-ai-agent.md`.

**ChatBubble bug fixed:** `const useAsync = !!realtimeChannelRef.current` caused async mode once Realtime connected. Realtime event never arrived (RLS blocks `user_id=null` AI messages on client subscription). Fix: `const useAsync = false` — forces sync mode always. Sync path returns `{ userMessage, aiResponse }` correctly. Commit: `55ac6ce`.

**Docs synced:** decisions.log (D-0233 retroactive add, D-0273 added), session-context.md (counts updated), CLAUDE.md (counts + arch notes), env-required.md (FACE_PROVIDER + DISABLE_DEV_LOGIN added).

**Counts after sync:** 250 pages / 383 API routes / 48 migrations / 274 decisions / 46 env vars

---

### Sessions 15-17 Summary (2026-03-29 — multiple commits)

**Session 15 (face recognition + staff infrastructure):**
- 15C: Face recognition core — browser-based face-api.js + pgvector (D-0261). 128-dim descriptors only, zero image upload to server. Provider-agnostic interface at `lib/face/client.ts` (FACE_PROVIDER env var).
- 15D: Staff registration with face capture + document uploads across 5 portals (D-0262).
- 15E: Attendance PWA + kiosk face verification, migrated from AWS Rekognition to local (D-0263). Routes `/attendance` and `/kiosk` made public (no session required).
- 15F: BM All-Staff Dashboard — aggregates contractor + JMB staff, filters, doc expiry, face enrollment status (D-0264).
- 15G: Deep code audit — dead code, DB health, security, performance, consistency (D-0265). Report: `docs/deep-audit-session15.md`.
- 15H: OpenClaw/MOA Phase 1 — cloud infrastructure for Minilab Office Agent (D-0266). MOA settings page redesigned (D-0267).

**Session 15 also included (D-0259, D-0260):**
- D-0259: Google Auth (NextAuth.js) for service provider registration — 7 forms swapped from phone to Google OAuth.
- D-0260: Building service accounts — email+password login for buildings, `account_type` column added to users.

**Session 16 (staff PWA):**
- 16A: Premium face scanner UI + kiosk 4-screen redesign + guard VMS attendance/incident (D-0268).
- 16B: Staff PWA shell at `/app` — role-adaptive bottom nav, guards + cleaners features (D-0269).
- 16C: PWA tasks, chat, approvals, console mini for BM/Tech roles (D-0270).

**Session 17 (UI hardening + console upgrade):**
- 17A: Theme CSS purge (blue tint), cases kanban default, WhatsApp form redirect — form submissions create case + message + ticket + WhatsApp deep link (D-0271).
- 17B: Console unified inbox — Inbox/CRM mode toggle, threads from all channels, contact profile edit, WhatsApp window buttons, Telegram resident detection wired to V4 AI pipeline (D-0272).

**Post-session 17 hardening commits:**
- Split-screen dark login + register redesign
- Cases detail panel overflow + duplicate close + console link fix
- Dev login page (`/dev/login`) with 13 demo accounts for all roles (D-0000 not logged — added in hardening)
- Null-safe dashboard values for empty/demo accounts
- Building filter added to all supabaseAdmin routes (security: data leakage fix)

---

### Telegram Auto-Groups (2026-03-29 — post-closing commit cd97e20)

**D-0258 — Telegram group auto-creation via Telethon microservice:**
- Rebuilt telegram_group_settings table: was 10 columns, now 25 (added category, telegram_group_id bigint, invite_link, bot_mode, notifications_enabled, etc.)
- 8 group categories: bm_staff, contractors, security, cleaning, committee, residents, suppliers, custom
- lib/telegram/group-service.ts: Telethon microservice client (create/delete/getInviteLink)
- Redesigned Telegram settings page: collapsible category sections, Create/Delete/Settings per group
- POST creates group via Telethon then saves to DB. DELETE calls Telethon + removes from DB
- send-invites endpoint DMs invite links to category members via bot
- Per-group bot mode settings: listen / active / mention_only / notifications_only
- 2 new env vars: TELEGRAM_GROUP_SERVICE_URL, TELEGRAM_GROUP_SERVICE_SECRET

**New files:** lib/telegram/group-service.ts, lib/telegram/groups.ts
**Modified:** Telegram settings page, group-router.ts
**Decision:** D-0258

---

### Session 14 Final (2026-03-28/29 — massive multi-session build)

Session 14 spanned multiple Claude Code sessions. This is the complete summary of everything built.

**Hardening (7 fixes):**
- Middleware bypass revert (D-0237)
- Input sanitization + prompt injection guard (D-0238)
- Zod validation on 6 critical API routes (D-0239)
- Redis caching layer for 5 high-traffic endpoints (D-0240)
- Audit logging — enhanced audit_log table (migration 046) + 9 AI handler audit trails (D-0241)
- Dead code cleanup — 4 stale routes deleted (D-0242)
- E2E code path audit — 24/24 pass

**Telegram + ChatBubble fixes:**
- Web ↔ Telegram bidirectional message sync
- ChatBubble expand/maximize toggle button
- Markdown rendering in ChatBubble
- Working hours time parsing (day ranges, AM/PM)
- Numeric overflow fix (ai_confidence)
- Telegram /start welcome handler
- Confirmation results sync to Telegram
- Telegram sync helper rewrite (plain text, no parse_mode)

**Quick wins:**
- TS type declarations
- Capabilities-map.md refreshed
- Settings hub redesign (compact scorecard + grouped cards)

**Permissions overhaul:**
- Full audit + 5 issues resolved
- Staff permissions page fix + redesign (split panel, grouped toggles)
- System B deleted (redundant role permissions)
- 27→54 granular permissions, 24 page gates, 8 API gates
- 6 role templates updated

**AI Architecture — Phase 1 (Generic Read):**
- Structured query builder for 16 tables
- 15 performance indexes
- Routing fix for raw message passthrough

**AI Architecture — Phase 2 (Generative UI):**
- AIResponse types, channel adapter, form registry
- 4 write handlers (task, maintenance, procurement, announcement) + Telegram Mini App pages
- Dynamic dropdowns (staff + units from DB)
- Building-level ai_form_mode setting (form_card / quick_text)
- Form card sync to Telegram with Mini App inline keyboard buttons

**AI Architecture — Phase 2b (Keywords):**
- 24 ai_tools enriched with Manglish, Malay, informal English, synonyms (35-78 keywords each)

**AI Architecture — Phase 3 (8 more write handlers):**
- approve_renovation, create_agm_session, add_agm_motion, create_tender, manage_insurance, upload_document, log_petty_cash, register_asset

**Multi-step collection + Proactive pinging:**
- Redis state machine for Telegram text collection (lib/ai/multi-step/collector.ts)
- 35+ field question templates
- 5 crons wired for proactive Telegram pings
- Building-level toggle, rate limited (20hr TTL)

**Shell portals wired (Session 15):**
- Supplier (10pg): settings save, tenders wired
- Developer (14pg): mock removed, contractors + tenders wired
- Store (4pg): real orders/revenue, server-side layout
- Org (1pg): already fully wired

**Move-in/out handler (D-0256):**
- 13th write handler using tenancies table
- Form spec with 6 fields, 31 Manglish/Malay keywords
- Move-in inserts new tenancy, move-out ends existing

**Async AI pipeline (D-0257):**
- Vercel waitUntil() background processing on chat/send + Telegram webhook
- Client receives AI responses via Supabase Realtime subscription
- Message status state machine (pending → processing → complete / failed)
- Stuck-message cleanup cron every 5 minutes
- Sync fallback preserved for non-async clients
- Migration 048: status column + index + Realtime publication

**Totals for Session 14:** ~46 commits, D-0237 through D-0258
**New files:** ~25 | **Modified files:** ~60 | **Deleted:** 4 stale routes + 2 onboarding components
**TypeScript errors:** 0

### Session 14b (2026-03-29 — Phase 2b/3 keywords + write handlers + multi-step + proactive pings)

**Four features in three commits:**

1. **Phase 2b — Keyword enrichment:** All 24 active ai_tools enriched with Manglish, Malay, informal English, synonyms (35-78 keywords each, avg ~50). Database-only, no code changes.
2. **Phase 3 — 8 new write handlers:** approve_renovation, create_agm_session, add_agm_motion, create_tender, manage_insurance, upload_document, log_petty_cash, register_asset. Form specs, write handlers, form executors, ai_tools entries. Dynamic dropdowns for renovation applications + AGM sessions. Skipped moveinout (table doesn't exist). Total: 12 form specs, 12 write handlers, 32 ai_tools.
3. **Multi-step data collection:** Redis state machine (lib/ai/multi-step/collector.ts) for Telegram text channel. 35+ field question templates. Wired into v4-pipeline BEFORE admin routing. Handles skip/cancel/confirmation.
4. **Proactive ping controls:** lib/notifications/proactive-ping.ts with building-level enable/disable, rate limiting (20hr TTL). Wired into 5 crons (daily-ops, document-expiry, compliance, inventory-check, reminders). Toggle in Channels settings page. API at /api/bm/settings/proactive-pings.

**New files:** `lib/ai/multi-step/collector.ts`, `lib/notifications/proactive-ping.ts`, `app/api/bm/settings/proactive-pings/route.ts`
**Modified:** form-registry.ts, write-handlers.ts, form-executor.ts, admin-init.ts, v4-pipeline.ts, form-options route, channels page, redis.ts, 5 cron routes
**Decisions:** D-252 through D-254
**TypeScript errors:** 0

### Session 14 (2026-03-28 — AI Generative UI fixes)

**Three fixes in one commit:**

1. **Dynamic dropdowns:** `assigned_to_name` (text) → `assigned_to` (select with real staff UUIDs), `unit_number` (text) → `unit_id` (select with real unit UUIDs) in create_task + create_maintenance forms. AI resolves name/unit hints to IDs. New `/api/bm/chat/form-options` endpoint for Telegram Mini App.
2. **AI form mode setting:** `ai_form_mode:{buildingId}` KV in platform_settings. Values: `form_card` (default) | `quick_text`. All 4 write handlers check before returning. Toggle in Channels settings page. API at `/api/bm/settings/ai-form-mode`.
3. **Telegram form card sync:** `chat/send` route now detects form_card responses and uses `adaptResponse()` + Telegram Bot API with Mini App inline keyboard button instead of plain text.

**New files:** `app/api/bm/chat/form-options/route.ts`, `app/api/bm/settings/ai-form-mode/route.ts`
**Modified:** form-registry.ts, write-handlers.ts, form-executor.ts, telegram-forms page, channels settings page, chat/send route
**Decisions:** D-251
**TypeScript errors:** 0

### Hardening Session (2026-03-28 — 7 security + infrastructure fixes)

1. **Fix #1 (D-0237):** Reverted BM superuser middleware bypass. BM restricted to /bm/* + standalone apps. Guard VMS moved to ANY_AUTH_PREFIXES.
2. **Fix #2 (D-0238):** Input sanitization layer (lib/ai/sanitize-input.ts). 15 injection patterns blocked, message length limits, HTML/script stripping. Wired into v4-pipeline.ts + admin-router.ts. Guardrail prompts added to DeepSeek system messages.
3. **Fix #3 (D-0239):** Zod validation on 6 critical API routes (chat/send, chat/confirm, import/confirm, switch-context, staff POST, contractors POST). Shared validateBody helper in lib/api/validation.ts.
4. **Fix #4 (D-0240):** Redis caching layer (lib/cache.ts). Applied to 5 endpoints: setup-status (5min), dashboard (2min), collection-summary (5min), channels (1hr), building-details (10min). Cache invalidation on profile PATCH.
5. **Fix #5 (D-0241):** Enhanced audit_log table (migration 046). lib/audit.ts helper. Wired into 9 AI admin mutation handlers with source='ai_chat'.
6. **Fix #6 (D-0242):** Dead code cleanup. Deleted 4 routes (/api/bm/tasks/*, /api/test-face, /api/test-redis). Cleaned middleware matcher.
7. **Fix #7:** E2E code path audit — 24/24 tests passed (docs/e2e-audit-report.md).

**New files:** lib/ai/sanitize-input.ts, lib/api/validation.ts, lib/cache.ts, lib/audit.ts, migration 046
**New dependency:** zod
**Decisions:** D-0237 through D-0242

### AI Agent Phase D1 (2026-03-28 — Final session: scorecard, Form 11, Telegram files, cleanup)

**Phase D1 completed — AI Agent is feature-complete.**

- **Setup Scorecard:** GET /api/bm/settings/setup-status API returns live completion for 5 essential + 4 recommended items. Rendered at top of settings page. ChatBubble welcome message now contextual.
- **Form 11:** generate_form11 handler uses Claude Sonnet (SENSITIVE/LEGAL) to generate Notice of Demand. Queries unit, owner, building, balance.
- **Telegram File Uploads:** Webhook downloads documents via getFile API, parses with shared import functions, AI column mapping, inline keyboard confirmation.
- **Cleanup:** Deleted OnboardingWidget.tsx + OnboardingPopup.tsx. All 19 admin handlers have real logic — zero stubs.
- **Avatar:** ChatBubble uses Bot lucide icon consistently (no custom image available).

**All 19 handlers:** update_working_hours, update_building_profile, toggle_resident_portal, add_contractor, remove_contractor, add_staff, remove_staff, report_bug, get_collection_summary, get_unit_balance, get_dashboard_stats, search_unit, search_resident, draft_memo, send_announcement, draft_email_reply, platform_help, import_residents, generate_form11.

**Decision:** D-0232

### Session 11C (2026-03-27 — Personal properties + marketplace + utilities + tenancy)

**Migration 041:** 4 new tables (personal_properties, property_listings, utility_bills, tenancy_documents) + 3 enums (listing_type, listing_status, utility_type). Applied to live DB.

**10 new API routes:**
- `/api/resident/properties` + `/api/resident/properties/[id]` — personal property CRUD
- `/api/resident/listings` + `/api/resident/listings/[id]` — listing CRUD with auto-agent assignment
- `/api/resident/utilities` + `/api/resident/utilities/[id]` — utility bill CRUD
- `/api/resident/tenancy-docs` + `/api/resident/tenancy-docs/[id]` — tenancy document CRUD
- `/api/public/listings` + `/api/public/listings/[id]` — public property search (privacy-safe)

**7 new pages:**
- `/resident/property/[id]` — personal property detail (white design)
- `/resident/property/[id]/listing` — create listing for personal property
- `/resident/property/[id]/utilities` — utility bills for personal property
- `/resident/property/[id]/tenancy` — tenancy docs for personal property
- `/resident/unit/[unitId]/listing` — create listing for Minilab unit
- `/resident/unit/[unitId]/utilities` — utility bills for Minilab unit
- `/resident/unit/[unitId]/tenancy` — tenancy docs for Minilab unit

**2 new public pages:**
- `/properties` — public property listings (dark premium design, filters, no owner info)
- `/properties/[id]` — public listing detail (agent contact via WhatsApp only)

**Key features:**
- Agent auto-assignment via `partner_agencies.coverage_areas` containment query
- Public APIs never expose owner details (phone, user_id, contact_whatsapp)
- White card design for personal properties vs dark design for Minilab properties
- Reference number generation: `ML-LST-YYYYMMDD-XXXX`

**Modified:** Portfolio API (personal properties included), unit detail page (homeowner/tenant sections), marketing nav (Properties link).

**Decisions:** D-0214 through D-0219

### Session 12b (2026-03-27 — Resident auth + docs regen)

**Resident WhatsApp Reverse OTP auth flow built:** /resident/login page with 3 states (phone input → OTP display + WhatsApp send → verified redirect). New APIs: /api/auth/resident/request-otp, /api/auth/resident/check-otp.

**BM apps page updated:** Removed standalone Resident Portal card. Added toggle inside QR card. Redesigned print poster (black/white A4).

**Form listing page:** /form/[buildingSlug] shows Resident Portal button when toggle enabled.

**Docs regenerated:** capabilities-map.md, portal-inheritance.md.

**Decisions:** D-0208 through D-0213

### Session 10 (2026-03-26 — Console CRM rebuild)

Console rebuilt as full CRM command center. Unit-based 3-panel layout: search → timeline → profile. 4 new/modified API routes. Cases page linked with "Open in Console" button.

**Decisions:** D-0199 through D-0202

### Session 9 (2026-03-26 — Working hours + attendance)

Migration 040: building_working_hours, public_holidays, contractor_schedule_templates. Attendance status pipeline. Late detection in dashboard.

**Decisions:** D-0195 through D-0198

### Earlier sessions

See git log for Sessions 1-8.5 (foundation, infrastructure, portal builds, face recognition, guard VMS, routing fixes).

## WHAT TO DO NEXT (suggested priority)

1. E2E test with Seeteng at Lumi Residency — first real BM + staff face clock-in test (human task)
2. Set DISABLE_DEV_LOGIN=true in Vercel before real users (human task)
3. MOA Phase 2 — Electron/Python desktop agent for Advelsoft integration (cloud side done, D-0266)
4. Superadmin PDPA data deletion workflow
5. Invoice generation — Security/Cleaning/Contractor (needs Opus planning: service_contracts schema)
6. Committee portal — governance tabs in resident portal (needs Opus planning)
7. Payment gateway — FPX via BillPlz for resident service charge payments
8. Patrol PWA — port BLE scanning from /security/patrol to /app
9. Monthly management report PDF — PMC primary deliverable
10. Billing automation — superadmin subscription cycle + dunning

## V13 Session Summary (2026-04-03 — 2026-04-04, ~35 commits)

**Massive 2-day session.** All work on /app Staff PWA + supporting BM/security desktop + DB migrations.

### Features built:

**Cron consolidation (D-0328):** 24 separate cron routes → 1 daily master cron for Vercel Hobby plan (max 1 cron). Logic extracted to lib/crons/*.

**Leave management (D-0331):** Full leave system with employer-based approval routing (PMC approves PMC staff, BM approves JMB staff, company admin approves contractor workers). Tables: leave_policies, leave_balances, leave_requests. PWA flow (/app/leave), BM desktop (/bm/leave), Telegram notifications. AI view: ai_view_leave. 2 new ai_tools.

**Fines & clamping (D-0332):** Guard PWA 3-step create (photo evidence, plate, violation). Find/pay/unclamp flow. BM desktop with stats cards + waive modal + fine settings. Security portal view. Tables: vehicle_fines, fine_settings. AI view: ai_view_fines. 2 new ai_tools. fine-photos bucket added.

**Technician role fix (D-0330):** Technician = role:staff + position:Technician. NOT contractor_worker. All technician queries fixed to use user_roles with position filter.

**Staff page redesign (D-0333):** /bm/staff: card grid layout with profile photos, face enrollment status, masked phone with reveal, Telegram linked indicator. New staff photo upload endpoint.

**Face enrollment re-enroll (D-0334):** BM can view existing enrollment, see reference photo, re-enroll or remove. face_enrollments.reference_photo_url column added.

**Cleaning PWA overhaul (D-0335, D-0336):** Single-page cleaning log (no wizard). Checklist flow with area cards + progress bar. History page grouped by date. Before/after photo submission. Real cleaning areas from API.

**Login page redesign (D-0346):** Split layout — form left, hero image right. Premium dark aesthetic.

**Image compression (D-0337):** lib/image-compress.ts — client-side canvas compression (1200px max, 0.7 quality). Applied to fines, cleaning log, incident, petty cash, face enrollment photos.

**Face clock-in overhaul (D-0338):** Liveness = 3 consecutive frames (was blink detection). Manual clock-in removed entirely. Auto-clock on face match. face-status API checks building enrollment count (not individual). Face threshold lowered from 0.6 → 0.5.

**Multi-session attendance (D-0344):** Unlimited in/out pairs per day. Late detection on first session only. lib/attendance-calc.ts for MYT boundary + stats. Clock auto-detects action (in vs out).

**MYT timezone fix (D-0342, D-0343):** All attendance times display in Asia/Kuala_Lumpur. Date boundaries computed in Postgres (not JS). lib/format-time.ts shared utility.

**Postgres RPC migration (D-0345):** All 6 attendance RPCs created. Zero PostgREST .from('attendance_logs') queries remain.

**DB migrations applied:** 054 (leave), 055 (leave balances), 056 (fines), 057 (attendance_logs user_id + nullable contractor_staff_id), 057c (face_enrollments reference_photo_url)

### Key patterns established:
- All attendance reads: Postgres RPCs only
- All image uploads: compress via lib/image-compress.ts first
- Face clock-in: no manual fallback, no name dropdown, face IS identity
- Technician = role:staff + position:Technician

**TypeScript errors:** 0 at session end | **Decisions:** D-0328 through D-0346

---

## Session: Pre-launch readiness — Sentry integration (2026-04-01)
**Fix 1:** Import retry/clear UI verified complete (P6 work confirmed in place).
GET/DELETE /api/bm/import/upload and staleFailed banner in blocks/page.tsx all working.
**Fix 2:** Sentry integrated — @sentry/nextjs v10.47.0. 4 config files created
(sentry.client/server/edge.config.ts + instrumentation.ts). next.config.js wrapped with
withSentryConfig. global-error.tsx created. captureException added to 5 critical catch
blocks (BillPlz, Stripe, approve-payment, dashboard, WhatsApp). Fully optional — dormant
without SENTRY_DSN. TS: 0 errors. Build: pass. D-0304.
