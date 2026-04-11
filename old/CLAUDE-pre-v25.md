# Minilab — Master Rulebook for Claude Code

## BUILD STATUS
V1 (T001-T075), V2 (T076-T132), V3, and V4 Phases 37-41 are complete.
Foundation restructure (blocks hierarchy) is complete.
BM portal feature gaps (Tier 1-3) are fixed. 58 BM pages with full CRUD.
All registration flows use approval pattern (building + hardware store applications).
Website rewrite + Voice Sales Agent (Gemini Live) complete.
Resident portal with personal properties, marketplace, utilities, tenancy complete.
Hardening session complete: input sanitization, Zod validation, Redis caching, audit logging, dead code cleanup.
AI Agent: Phase 1-4 complete. Generic read (16 tables) + Generative UI (13 write handlers) + multi-channel (web/Telegram Mini Apps/WhatsApp) + async pipeline (waitUntil + Supabase Realtime) + multi-step collection + proactive pings.
Permissions: 54 granular permissions, 24 page gates, 8 API gates.
Telegram auto-groups: Telethon microservice on GCP e2-micro (free tier). 8 categories, invite-based flow, per-group bot mode settings. telegram_group_settings table (25 columns).
Shell portals: Supplier, Developer, Store, Org — all wired with real data. 5 mocks removed.
Face recognition: Browser-based face-api.js + pgvector (128-dim descriptors, zero cloud cost). FACE_PROVIDER env var.
Staff PWA: /app for field staff (guards, cleaners, BM staff). Face clock-in, incidents, tasks, console mini. Leave management + Fines + Cleaning checklist added (V13).
Console: Unified inbox (all channels) + CRM toggle. Sessions 15-17 complete.
Auth: Building service accounts (email+password) + Google OAuth for service provider registration added.
Dev login: /dev/login page with 13 demo accounts (all roles). DISABLE_DEV_LOGIN=true disables in prod.
Leave management: employer-based approval (PMC→PMC staff, BM→JMB staff, company admin→workers). Tables: leave_policies, leave_balances, leave_requests.
Fines & clamping: guard creates fine with photo evidence, BM/security_admin can waive. Tables: vehicle_fines, fine_settings.
Attendance: unlimited in/out pairs per day. ALL reads via Postgres RPCs. Zero JS date math for attendance. MYT timezone throughout.
Domain: minilab.my (live). 340+ pages, 525+ API routes, 189+ DB tables (migration 067+), 442+ decisions logged.
**V12 audit batch (6 sessions):** 28 security/error-handling fixes (P1), performance pagination + mobile responsiveness (P2), 22 notification hooks + API contract fixes (P3), resident complaint form + guard handover + cleaning checklists (P4), superadmin TOTP 2FA + health monitoring + building provisioning (P5), final polish (P6). ~95 total fixes from 7 domain audits + 7 technical audits.
**V14 audit + fix release (14 sessions, D-0347–D-0370):** 230+ fixes. PWA + BM portal fully audited (64 BM pages, 168 API routes). Task media upload, console WA/TG delivery, 6 resident notification flows. Inventory model consolidated. RPC caller rule + portal routing locked as permanent architecture decisions.
**April 6 session (D-0371–D-0387):** Committee analytics + PWA, AI material probe, org portal audit + cleaning shifts, attendance geofence full wiring, middleware routing unified (roleHome() deleted → getPortalPath()), superadmin service provider management + dedicated login page, org admin email+password auth. PMC full access portal — 15 building pages, verifyPmcBuildingAccess + getBuildingSession.
**V16 session (D-0388–D-0399):** contractor_staff.contractor_org_id nullable (mig 060), BM staff assign-to-company (mig 061), onboarding PWA — 8 stages live at /app/onboard (14+ API routes under /api/onboard/), staff form simplified (face-api removed), UX polish + design consistency.
**V17 session (D-0401–D-0421):** Staff permissions save fix. Onboarding PWA: delete staff, camera+gallery, multi-device sessions. Toggle nuclear fix + staff grouped MO/Security/Cleaning. MO roles simplified: Admin+Technician only. Service worker CACHE_NAME=minilab-v3. Preview auth bypass. Kiosk audited. Guard VMS: full audit, pre-reg form, QR gen→R2, html5-qrcode scanner, visit_date, BM visitor log. Guard VMS dark premium redesign: wizard walk-in, dark theme, IC/license snap. Superadmin user impersonation + amber banner. V17 verification sweep: 17/17 pass. Users list FK ambiguity fix. WA+TG messaging audit: 46 issues identified.
**V18 session (D-0422–D-0442):** Guard VMS flat redesign — 6 first-level buttons, single-step forms, lucide icons, no Walk-in/Parcel/Vehicle concept (D-0422–D-0427). Staff permissions: Admin/Receptionist + Accounts → 50 permissions each, `staff_role_permissions` table dropped (D-0424). Full WA+TG messaging audit — 46 issues found, 26 code fixes across 3 rounds: security (TG webhook secret, typed phone takeover removed, HMAC mandatory), core messaging (dedup, V3 killed, waitUntil, OTP user linkage, broadcast uncapped, portfolio fix, conversation scoping, Anthropic retry), notifications + quality (in-app notifications wired, TG/WA media → R2, 24h window + template fallback, voice note reply, DeepSeek URL standardized, credential vault standardized, bot-blocked detection, console read state). AI Console rewritten as unit-centric 3-panel command center — left: unit search + filters (All/My units/Unread/Outstanding), middle: unified cross-channel timeline with sender avatars + channel logos (WA/TG/Gmail/Minilab), compose bar with WA/TG/Email/Note channel picker, right: 10 operational sections (Property/People/Payments/Cases/Applications/Vehicles/Access Cards/Visitors/Fines/Documents) with slide-over create modals + inline forms (D-0434–D-0438). Guard VMS kiosk: multi-gate system with `building_gates` table, BM generates passcodes per gate, guard enters 6-digit code to activate, gate_id tracked on visitors + vehicle_logs (D-0439–D-0442).

### Key Architecture Changes (recent)
- **Cases renamed to Tasks** in UI. Backend stays `cases` table. `/bm/cases` is the Tasks page with List/Board (kanban) toggle. Old `/bm/tasks` redirects there. Create form uses "Type" (Resident Complaint, Maintenance Request, Internal Work, Inspection, Follow-up, Other) mapped to `category` column.
- **Procurement model changed**: No more PO/catalogue. Now uses delivery log + usage tracking + inventory (computed from deliveries IN minus usage OUT). Data in platform_settings KV. Future: Telegram hardware group auto-tracking.
- **Blocks hierarchy**: buildings -> blocks -> units (migration 033). Every unit has block_id NOT NULL. Single buildings get auto-generated default block. block_type: highrise/landed/commercial/villa/mixed.
- **Approval pattern**: building_applications + hardware_store_applications tables (migration 031). Superadmin reviews. On approval, real record created with trial status.
- **Multi-building sessions**: session.buildings[] array, building switcher dropdown in sidebar header, /bm/select-building page.
- **Onboarding**: Floating bottom-right widget with purple glow, first-login popup, building_onboarding table tracks 7 steps.
- **Phone format**: normalizePhone(), formatPhoneForDB(), phoneMatchQuery() utilities. Canonical format +60XXXXXXXXX.
- **Collection pipeline**: 7-stage escalation config in platform_settings KV. OpenClaw payment gateway integration card.
- **Sidebar**: 6 sections (Overview, Operations, People, Finance, Compliance, Settings). Operations includes Assets + Patrol.
- **V4 RAG + AI Console**: sender_profiles state machine, document_chunks with hybrid vector+keyword search, tickets table for BM Console feed, email_log for audit trail, staff_permissions RBAC with 54 granular permissions.
- **AI Generative UI**: Form card pattern for write operations. 13 write handlers (Phase 2: task/maintenance/procurement/announcement, Phase 3: renovation/agm/tender/insurance/document/petty-cash/asset, Phase 3b: moveinout). Dynamic dropdowns (staff/units/renovation apps/AGM sessions from DB). ai_form_mode KV setting (form_card/quick_text). Multi-channel sync (web form cards, Telegram Mini App buttons, WhatsApp text fallback). Multi-step data collection via Redis state machine for Telegram text channel. 33 ai_tools with Manglish+Malay keywords. Proactive pings: building-level toggle, rate-limited, wired into 5 crons.
- **Async AI pipeline**: Vercel waitUntil() background processing. Chat/send + Telegram webhook return 200 instantly. Client receives AI response via Supabase Realtime subscription on messages table. Message status state machine (pending→processing→complete/failed). Stuck-message cleanup cron every 5 min. Sync fallback preserved.
- **Telegram auto-groups**: Telethon microservice on GCP e2-micro (free tier). BM clicks Create → group appears on Telegram with bot as admin. 8 categories (bm_staff/contractors/security/cleaning/committee/residents/suppliers/custom). Invite flow: bot DMs invite links to category members. Per-group bot mode: listen/active/mention_only/notifications_only. telegram_group_settings table (25 columns). Env vars: TELEGRAM_GROUP_SERVICE_URL, TELEGRAM_GROUP_SERVICE_SECRET.
- **Face recognition**: Browser-based face-api.js + pgvector (D-0261). 128-dim float descriptors only — no images leave browser. Provider-agnostic interface at `lib/face/client.ts`. FACE_PROVIDER env var: `local` (default) or `rekognition`. AWS Rekognition retained as fallback.
- **Staff PWA**: `/app` mobile-first PWA for field staff. Role-adaptive bottom nav (guard/cleaner/BM/tech). Face clock-in, incident capture, cleaning before/after photos, task management, console mini. Sessions 16A-16C.
- **Console Unified Inbox**: Inbox/CRM mode toggle at `/bm/console`. Inbox aggregates all channels (WhatsApp, Telegram DMs, groups, web chat, cases) with filter tabs. Center: chat view. Right: contact profile edit + linked cases + collection balance + WhatsApp window status. Session 17B.
- **Telegram resident detection**: Webhook now checks residents table for user_id match — resident Telegram DMs routed to V4 AI pipeline with senderRole:'resident'. Console Inbox shows them under Telegram tab. D-0272.
- **Building service accounts**: `users.account_type` column (individual/building). Buildings can login with email+password (no phone required). `users.email` + `users.password_hash` + `users.google_email` + `users.google_id` added (migration 049). D-0260.
- **Google OAuth registration**: 7 service provider registration forms switched from phone to Google OAuth (NextAuth.js). Residents still use WhatsApp OTP. BM still uses Telegram QR. D-0259.
- **Dev login**: `/dev/login` page with 13 demo accounts (all roles). Uses email+password with bcrypt. API route at `/api/dev/login`. Set DISABLE_DEV_LOGIN=true in Vercel before going live.
- **Web chat async mode fix**: ChatBubble `useAsync` hardcoded to `false` — sync mode always. Realtime subscription stays active for proactive/Telegram-synced messages only. RLS blocks user_id=null AI inserts from client-side Realtime. D-0273.
- **Onboarding**: Floating widget + popup DELETED (replaced by ChatBubble). OnboardingWidget.tsx + OnboardingPopup.tsx removed in D-0232.
- **Attendance — RPCs only**: ALL attendance reads use Postgres RPCs (get_attendance_today, get_attendance_history, find_open_attendance, count_today_sessions, get_attendance_today_by_staff, get_attendance_history_by_staff). NEVER use PostgREST .from('attendance_logs').select() — causes schema cache errors after migration 057. D-0345.
- **RPC caller rule**: When ANY RPC signature changes (new params, renamed params, dropped overloads), you MUST grep for every .rpc('function_name') call across the entire codebase and update ALL callers in the SAME commit. Then run NOTIFY pgrst, 'reload schema'. No exceptions. Ref: D-0347/D-0348 incident.
- **Face clock-in principle**: Face IS the identity. No manual clock-in ever. No name dropdown. No fallback to typed name. If face not enrolled → show "No faces enrolled" and stop. If no match → "Try Again" only. D-0338.
- **Technician role**: Technician = role:staff + position:Technician in user_roles. NOT contractor_worker. Query: user_roles WHERE role='staff' AND position='Technician'. D-0330.
- **Image uploads**: Always compress client-side via lib/image-compress.ts before uploading (maxWidth=1200, quality=0.7). Applies to all photo evidence, cleaning logs, incident reports, face enrollment reference photos, petty cash receipts. D-0337.
- **Crons**: Single daily cron route for Vercel Hobby plan. All cron logic in lib/crons/. D-0328.
- **Leave management**: Employer-based approval routing. PMC role approves PMC staff. BM role approves JMB staff (role=staff). Company admin approves their contractor workers. Tables: leave_policies, leave_balances, leave_requests. D-0331.
- **Fines & clamping**: guard creates fine with photo evidence (vehicle_fines table). Payment required before unclamp. BM/security_admin can waive. fine_settings per building per violation type. D-0332.
- **Inventory model:** job_materials is single source of truth. case_id nullable (for non-task usage). procurement_item_id FK→procurement_items for stock tracking. Stock = procurement_order_lines IN minus job_materials OUT. inventory_usage table deprecated. D-0368.
- **Console delivery:** BM console messages now deliver via WhatsApp (sendWhatsAppDirect) and Telegram (tgSend). Delivery status tracked in messages.status column. D-0366.
- **Resident notifications:** notifyResident() helper at lib/notifications/resident.ts. WA preferred, TG fallback. Wired to: payment approval/rejection, resident registration approval/rejection, renovation approval/rejection. Best-effort, never blocks the action. D-0367.
- **Task media:** Upload via /api/app/tasks/upload to case-attachments bucket (R2 or Supabase Storage). Image compression via lib/image-compress.ts. Messages table stores media_url, media_type, media_mime_type. D-0365.
- **Portal routing — PERMANENT:** lib/auth/get-portal-path.ts is single source of truth for all login routing. Device detection via lib/auth/detect-mobile.ts. ALL login paths use getPortalPath(). NEVER modify without human approval. D-0370.
- **Auth gate rule:** PWA routes (/api/app/*) must use getSession(), never requirePermission(). requirePermission() is for BM portal only. D-0355/D-0356.
- **middleware.ts uses getPortalPath() directly:** roleHome() was deleted. middleware.ts imports getPortalPath(role, position, isMobile) from lib/auth/get-portal-path.ts — zero duplicate routing tables. position from session.activeContext.position (BuildingContext only). isMobile from User-Agent. D-0379.
- **PMC ownership model:** PMC org employs BM/admin/accounts staff and deploys them to buildings. JMB/MC can also hire staff directly. Both models coexist. PMC staff appear in user_roles with role='staff' linked to the building. D-0375.
- **Org admin auth:** email + password via /vendor-login (same as building service accounts). NOT Google OAuth. Create wizard Step 2 has Password + Confirm Password fields. bcrypt.hash(password, 10) → users.password_hash. D-0386.
- **Face enrollment:** Merged into Edit staff modal photo upload. handlePhotoUpload runs face-api.js detection + /api/face/enroll automatically after profile photo upload. No standalone Enroll button. Face status badge shown in edit modal. D-0382.
- **Attendance geofence:** building_geofences table (center_lat, center_lng, radius_meters, is_active). Haversine validation in lib/geo/haversine.ts. Enforced in /api/attendance/clock when is_active=true. If geofence active + no GPS → 403. If no geofence configured → allow freely (backwards compatible). D-0373.
- **Superadmin login:** /superadmin/login — dedicated page using Telegram QR flow. Escapes sidebar via (superadmin-auth) route group. Poll endpoint /api/superadmin/auth/poll checks role===superadmin. TOTP still enforced post-login. D-0384.
- **Service provider management:** /superadmin/providers — unified list of all 4 types. /superadmin/providers/create — 3-step wizard (company details → admin account → buildings). Security/cleaning/contractor → contractor_orgs + contractor_staff. PMC → organisations (org_type='management') + user_roles (role='pmc'). D-0383.
- **PMC full access portal (Option A):** PMC users get BM-level access to assigned buildings via x-building-id header. lib/pmc/verify-access.ts: verifyPmcBuildingAccess(userId, buildingId) — user_roles→building_org_assignments chain. lib/rbac/get-building-session.ts: getBuildingSession(req) — BM uses session.buildingId, PMC reads x-building-id + org verify. requirePermission() extended with optional req?: NextRequest (backwards compatible). OrgShell buildingNavSections prop + :buildingId substitution for building-level nav. 15 pages at /pmc/buildings/[buildingId]/{dashboard,cases,staff,units,residents,attendance,finance,maintenance,assets,documents,announcements,reports,settings,leave,fines}. D-0387.
- **Guard VMS kiosk — multi-gate:** `building_gates` table stores per-gate VMS passcodes. Guard enters 6-digit code on `/guard` → session scoped to building+gate. `visitors.gate_id` and `vehicle_logs.gate_id` track which gate. BM manages gates in /bm/settings/apps. Old single-passcode (buildings.vms_passcode) deprecated in favour of per-gate codes. D-0442.
- **AI Console — unit-centric:** `/bm/console` rewritten as 3-panel layout. Left: unit search + filters (All/My units/Unread/Outstanding). Middle: unified timeline across WA/TG/Email/Web with channel logos, sender avatars (staff/AI/system), internal notes (amber cards, messages.private=true, channel='internal_note'). Right: 10 operational sections with slide-over create modals. D-0434.
- **WA messaging hardened:** V3 pipeline removed (V4 only). waitUntil for async. Dedup via external_message_id unique index. 24h window check (sender_profiles.last_inbound_at) with template fallback. Media persisted to R2. Voice notes reply "not supported". D-0431/D-0432.
- **Security hardened:** TG webhook rejects missing secret header. handleTypedPhone removed (account takeover). WA HMAC mandatory (500 if META_APP_SECRET missing). D-0430.
- **Notifications:** In-app notifications wired via notifyBmInApp() alongside TG sends. TG bot-blocked detection (403 → users.telegram_blocked flag, filtered from future sends). D-0432.

**CRITICAL:** Every task commits and pushes to main. No branches. No worktrees.

## What This Is
Minilab is an AI-powered building management SaaS for Malaysian JMB/MC strata operations.
Stack: Next.js 14 App Router · TypeScript strict · Supabase PostgreSQL · Vercel · Tailwind CSS v3 + shadcn/ui
All specs live in the Project Bible (v21.0, 42 sections). This file is your session rulebook.

## Three Core Principles — Never Violate These
1. **Phone number = identity.** Every user identified by phone. Telegram Bot Login for service providers. WhatsApp Reverse OTP for residents. No passwords. No email login.
2. **Buildings are sovereign.** All building data belongs to the JMB/MC. Management orgs and contractor orgs access data via building assignments — they never own it. RLS enforces this at the database level on every table.
3. **No app downloads.** Residents use WhatsApp. Staff/BM/contractors use Telegram + web portals. PWAs: Attendance (contractor clock-in), Guard VMS, and /app (staff mobile PWA for guards/cleaners/BM field staff).

## Before Writing Any Code — Read First
- `docs/startup/quick-context.md` — current platform state
- `docs/startup/recent-decisions.md` — recent decisions. Never contradict a logged decision.
- `docs/startup/actual-schema-columns.md` — 165 tables. Read before touching any database code. Never guess a table name.
- `docs/startup/env-required.md` — all env var references
- `docs/startup/GRAPH_REPORT.md` — codebase knowledge graph (god nodes, communities, connections)

## Architecture Rules
- **AI routing:** DeepSeek V3 for ROUTINE (85%), Claude Sonnet for SENSITIVE/LEGAL (15%). Never swap.
- **Exact model strings:** ROUTINE = `deepseek-chat` (api.deepseek.com). SENSITIVE/LEGAL = `claude-sonnet-4-20250514` (Anthropic API). Never use aliases like `claude-sonnet-latest` — pin the exact version string.
- **Message processing:** minilab-process-message Edge Function is the single entry point for all AI logic. See `docs/bible-sections/s40-ai-brain.md`.
- **WhatsApp = residents only.** All service provider (BM/contractor/guard/supplier) comms go via Telegram.
- **RLS on everything.** Every table has Row Level Security. Never bypass RLS except in Supabase Edge Functions using service_role key.
- **Tool calling over data injection.** The AI calls tools to fetch data. Never pre-load all building data into a prompt.
- **Structured output always.** AI responses return JSON schema. Validate before executing any action.
- **Separate portals by contractor_type[].** Security → /security/*. Cleaning → /cleaning/*. Others → /contractor/*.

## File Ownership — Never Write Outside Your Scope
Read-only for all sessions: `packages/shared/types/` · `packages/shared/contracts/` · `components/ui/` (shadcn)

## Session Protocol — Do This Every Session
1. Read everything in `docs/startup/` folder (see docs/startup/SESSION-SOP.md for what to read)
2. Run: `git log --oneline -5` and `git status`
3. Do the work
4. Update docs (schema → actual-schema-columns.md, env vars → env-required.md, decisions → recent-decisions.md)
5. Commit each task separately, push to main ONCE at end of session (or when human says push)

## Hard Rules — Never Break
- Never guess a database column name. Check `docs/startup/actual-schema-columns.md` or query Supabase schema.
- RPC caller rule: When ANY RPC signature changes, grep every `.rpc('function_name')` call across entire codebase, update ALL callers in same commit, then run `NOTIFY pgrst, 'reload schema'`. Ref: D-0347/D-0348.
- Never commit `.env.local` or any file containing actual credential values.
- Never modify `packages/shared/types/database.types.ts` — regenerate it with Supabase CLI only.
- Never modify `packages/shared/contracts/interfaces.ts` without logging the change in `docs/startup/recent-decisions.md`.
- Never write code that bypasses RLS from the client side.
- Never hallucinate case numbers, balances, or dates in AI responses — validate against DB first.
- Never implement legal interpretation. Route to BM + Claude Sonnet LEGAL context only.
- NEVER read .env.local or any file containing key/secret/token/password in the filename. For env var names only, read docs/startup/env-required.md.
- NEVER modify any V1 code (T001–T075). V1 is frozen.
- ONE TASK = ONE GIT COMMIT. After every task completes with zero errors: run `git add .` → `git commit -m "[prefix]: brief description (D-XXXX)"`. Do NOT push after every commit. Push to main ONCE at end of session, or when the human explicitly says to push. Before pushing, run `npx next build` to verify zero type errors (warnings are OK). Reason: Vercel Hobby plan — every push triggers a full production build, so intermediate pushes waste build minutes and cause cascading deploy failures.
- If `git push` fails (conflict, auth error): stop, report the error via Remote Control, do NOT attempt force push. Wait for human instruction.
- If Vercel build fails after push: report the build error URL via Remote Control. Fix the error and push again. Do not mark the task [x] until the Vercel build passes.
- Before ANY database query: verify table and column names in docs/startup/actual-schema-columns.md. Never guess a column name.
- Never mark an Edge Function task [x] without first invoking it and verifying the output matches expected behaviour.
- **Portal routing — NEVER change without human approval.** Login routing is defined in `lib/auth/get-portal-path.ts`. The routing table is: building staff on phone → /app (PWA), building staff on PC → /bm (BM/admin/staff) or /app (guard/technician), resident → /resident always, ORG admins → their desktop portal always. See D-0370. Do NOT modify this function without explicit human approval.

## When to Stop and Ask (Telegram HITL)
Send a Telegram message to the human (MASTERAI_CHAT_ID) and wait for reply before continuing:
- Two valid architectural approaches with different trade-offs
- A feature scope that is not in the Bible
- An external service credential that needs human action
- An integration test that fails after 3 fix attempts
- Completing a build round (Round 1/2/3/.../7) — get approval before continuing to next


## Credentials — How to Set Up

You handle all credential files. The human pastes raw key=value pairs to you and you:
1. Create `.env.local` in the project root with the real values
2. Create `.gitignore` with `.env.local` listed (never commit real keys)
3. Create `.env.example` with placeholder names only (safe to commit)
4. Add all env vars to Vercel dashboard (Production + Preview environments)

**Format the human will paste:**
```
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
ANTHROPIC_API_KEY=sk-ant-...
```

**Never output the contents of `.env.local` in any response.**
**Never commit `.env.local` to git — ever.**

## Quick Reference
- Startup docs: `docs/startup/` (quick-context, recent-decisions, SESSION-SOP, actual-schema-columns, env-required)
- Full architecture: `docs/bible-sections/` (one .md per Bible section group)
- Revenue model: 10 streams, multi-building SaaS
- Database: 189+ tables (migrations 001-067+), pgvector enabled, Singapore region
- Decisions: 442+ logged (D-0001 through D-0442)
- Pages: ~340+ | API routes: ~525+
- Env vars: 46+ unique references (see `docs/startup/env-required.md`)
- Auth: Telegram Bot Login (service providers) + WhatsApp Reverse OTP (residents) + Google OAuth (registration) + email+password (building accounts) — ZERO SMS cost
- Face verification: face-api.js + pgvector (local, default). AWS Rekognition retained as fallback (FACE_PROVIDER=rekognition)
- Portals: 13 portals (/bm, /security, /cleaning, /contractor, /supplier, /developer, /pmc, /superadmin, /resident, /store, /committee, /guard, /org) + 5 micro-portals (attendance, kiosk, AGM, form, /app PWA)

## graphify

This project has a graphify knowledge graph at graphify-out/.

Rules:
- Before answering architecture or codebase questions, read graphify-out/GRAPH_REPORT.md for god nodes and community structure
- If graphify-out/wiki/index.md exists, navigate it instead of reading raw files
- After modifying code files in this session, run `python3 -c "from graphify.watch import _rebuild_code; from pathlib import Path; _rebuild_code(Path('.'))"` to keep the graph current
