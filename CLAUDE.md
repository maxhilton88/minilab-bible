# CLAUDE.md — Minilab Platform
<!-- Version: V36 · 2026-04-19 -->
<!-- THIS IS THE ONLY FILE READ AT STARTUP. Everything else is lazy-loaded via the Router. -->

---

## §1 · What Is Minilab

Minilab (minilab.my) — AI-powered ERP SaaS for the entire Malaysian strata property management industry.
Stack: Next.js 15 App Router · TypeScript strict · Supabase PostgreSQL · Vercel · Tailwind v3 + shadcn/ui
DB: Supabase project `nncbsumdmsyjjrgdjjea` (Singapore region)
Serves: JMB/MC buildings, PMC, security, cleaning, contractor, supplier/store, developer, residents across portfolio-scale operations. NOT a single-building tool.

---

## §2 · Session Protocol

1. Read THIS file (CLAUDE.md) — nothing else yet
2. Run: git log --oneline -10 && git status
3. Check §4 Router below — identify which docs (if any) your task needs
4. Extract only the relevant sections using awk (see §3)
5. Do the work
6. Run Closing Block (§6)

---

## §3 · How To Read Docs (awk, never cat)

Every lazy-loaded doc uses `## §Anchor-Name` headers as semantic markers.

Extract a section (up to the next anchor):

    awk '/^## §Face-Recognition/,/^## §/' docs/startup/recent-decisions.md | head -n -1

List all anchors in a file:

    grep '^## §' docs/startup/recent-decisions.md

Extract a table section from schema:

    awk '/^## §Table-attendance_logs/,/^## §/' docs/startup/actual-schema-columns.md | head -n -1

Rules:
- NEVER cat a full doc over 50 lines. Always awk the section you need.
- If you don't know which anchor to use, grep '^## §' first to see the index.
- If a doc has no anchors yet, read it fully ONCE, then add anchors as part of your closing block.
- Anchors are semantic — they never break when content is added/removed (unlike line numbers).

---

## §4 · Bible Router

Match your task to a row. Read only what's listed. If no row matches, you probably don't need extra docs.

### Schema & Database

| Task | File | Anchor |
|------|------|--------|
| New migration / new table | actual-schema-columns.md | §Table-{name} for related tables |
| Alter column on existing table | actual-schema-columns.md | §Table-{name} |
| Check if column/table exists | actual-schema-columns.md | §Table-{name} |
| New RPC function | actual-schema-columns.md | §RPCs |
| New env var needed | env-required.md | Full file OK (small, ~7KB) |

### Feature Areas → Decision History

| Task | File | Anchor |
|------|------|--------|
| Face recognition / enrollment | recent-decisions.md | §Face-Recognition |
| Attendance / clock-in-out | recent-decisions.md | §Attendance |
| AI agent / pipeline / routing | recent-decisions.md | §AI-Pipeline |
| Auth / login / portal routing | recent-decisions.md | §Auth-Routing |
| Billing / Stripe / credits | recent-decisions.md | §Billing |
| Guard VMS / visitor mgmt | recent-decisions.md | §Guard-VMS |
| Visitor pre-registration / invite-link flow | recent-decisions.md | §Guard-VMS |
| Multi-role context switcher / ContextSwitcher | recent-decisions.md | §Auth-Routing |
| AI observability (pipeline runs + actions log) | recent-decisions.md | §AI-Pipeline |
| PMC portal / multi-building | recent-decisions.md | §PMC-Portal |
| Onboarding PWA / staff reg | recent-decisions.md | §Onboarding |
| Console / inbox / messaging | recent-decisions.md | §Console |
| Unit search / normalization | recent-decisions.md | §Console |
| Telegram bot / groups / Telethon | recent-decisions.md | §Telegram |
| Committee portal | recent-decisions.md | §Committee |
| Procurement / hardware store | recent-decisions.md | §Procurement |
| Collection / chase engine | recent-decisions.md | §Collections |
| MOA / office agent / Advelsoft | recent-decisions.md | §MOA |
| MOA architecture / build spec / skills / connectors | MOA-SPEC.md | Full file OK |
| Resident portal / WhatsApp OTP | recent-decisions.md | §Resident |
| Supplier / store portal | recent-decisions.md | §Supplier-Store |
| Data backfill / re-import gaps | recent-decisions.md | §Data-Backfill |
| TypeScript errors / TS cleanup | recent-decisions.md | §TypeScript-Cleanup |
| Case / task detail sheet / inspection scheduling | recent-decisions.md | §Case-Management |
| Staff management / bulk actions / all-staff page | recent-decisions.md | §Staff-Management |

### Architecture & Code Navigation

| Task | File | Anchor |
|------|------|--------|
| Where is feature X in code? | GRAPH_REPORT.md | §Community-Index |
| What calls what? / dependencies | GRAPH_REPORT.md | §God-Nodes + §Connections |
| Permission / RLS questions | recent-decisions.md | §Permissions |
| Portal routing logic | NONE — see §5 Hard Rules below | — |

### Deep Reference (rarely needed)

| Need | File | When |
|------|------|------|
| Full V3-V12 audit history | docs/audit/*.md | Debugging regression from old code |
| AI agent architecture deep-dive | docs/minilab-ai-agent-architecture.md | MOA build or AI pipeline rewrite |
| Capabilities map | docs/capabilities-map.md | Product-level overview |
| Full raw decision history D-0001+ | docs/decisions.log | Only if recent-decisions.md doesn't cover it |

---

## §5 · Hard Rules (always in memory — never violate)

Billing: JMB/MC always free (no trial, no subscription). Service providers pay per-staff + unit-tier + credit packs. All pricing in billing_config table (superadmin editable, zero hardcoded values).

Hardware Store: NO inventory system. Telegram orders → invoice billing only.

Technician: A building staff member (role=staff, position=Technician), NOT a contractor_worker.

Face: face-api.js + pgvector, 128-dim descriptors, cosine similarity threshold 0.6, match_face() RPC. No manual clock-in ever. Face IS identity — no name dropdown. One photo = one enrollment (autoEnrollFace utility). No separate FaceEnrollModal.

Attendance: Unlimited in/out pairs per day. Late/early/OT calculated from building_working_hours.

Portal routing: Permanent, locked, via lib/auth/get-portal-path.ts. NEVER duplicate this logic anywhere.
- Building staff phone → /app · BM/admin PC → /bm/dashboard
- Guard/cleaner/tech PC → /app · Committee phone → /app PC → /committee/dashboard
- Resident → /resident always · Store = Supplier (same portal)
- ORG admins → their desktop portal always

Resident auth: Residents authenticate ONLY via WhatsApp Reverse OTP. Never Telegram, never Google OAuth, never email/password. Any code path that lets a resident in via another channel is a bible violation.

Google OAuth scope: Google OAuth is ORG-only — for service provider organizations (security, cleaning, developer, supplier, store admins). Never for residents or building staff. The Google callback MUST reject users without an org role.

Resident PWA device tracking: The resident PWA does NOT track devices. No last_active timestamps, no device_info, no session history. Device tracking is staff/guard/cleaner/kiosk only. `resident_pwa_sessions` was scaffolded but never wired — it is being dropped in V37.

PWA API routes: Use getSession() not requirePermission().

Types: NEVER modify database.types.ts manually — regenerate:
npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea

RPC Caller Rule: When ANY RPC signature changes → grep -r ".rpc('function_name')" --include="*.ts" --include="*.tsx" → update ALL callers in same commit → NOTIFY pgrst 'reload schema'.

TypeScript discipline: `typescript.ignoreBuildErrors` MUST stay false (or absent) in next.config.js. New TS errors must be fixed before commit. If genuinely unavoidable, use `@ts-expect-error` with a comment explaining why. NEVER `@ts-ignore`. NEVER `as any` (except `.from()` on tables not yet in generated types — always with `@ts-expect-error` noting which table and why). NEVER non-null `!` assertions without a prior null check in the same scope.
Before any commit: `npx tsc --noEmit 2>&1 | grep -c "error TS"` → must return 0. V32d removed ignoreBuildErrors after clearing 108 errors — ZERO errors is the permanent baseline.

Migration discipline: Every migration file MUST be applied to prod in the same session it is created. Verify immediately after apply:
  SELECT name FROM supabase_migrations.schema_migrations WHERE name LIKE '{number}%';
If migration contains CREATE POLICY without IF NOT EXISTS, note in closing block that re-execution is unsafe (register-only on re-apply). Closing block MUST confirm migration applied, not just file created.

PMC endpoint rule: All /api/bm/* handlers that call requirePermission() MUST pass req as second arg. check-api-permission.ts throws in dev if PMC session detected without req. Enforced since D-0730. No exceptions within /api/bm/* except /billing/* and /settings/danger/* (those don't use requirePermission at all).

Git: Push to main only. One task = one commit + push. No branches. If push fails: git pull --rebase origin main && git push origin main.

Infra:
- Telethon microservice: GCP e2-micro 34.42.210.102:8443
- R2 bucket: minilab-media · public URL: https://media.minilab.my · Account ID: 585f5b927632127fe9545602c82c0b09
- Sentry: gated on SENTRY_DSN env var
- Vercel Hobby: 1 daily cron only (/api/cron/daily) · Deploy via Deploy Hook (Vercel webhook broken)
- Face models: /api/models/face-api/
- Service provider registration: Google OAuth + email/password

Visitor pre-registration — dual path: Residents have two ways to pre-register a visitor:
1. Invite link (primary) — resident generates a link → shares to visitor → visitor self-registers IC + plate + photo → gets QR
2. Fill in details (legacy) — resident enters all visitor info → gets QR to share
Both paths coexist. Do not remove either. AI defaults to the invite-link path.

Visitor self-registration requirements: Form requires IC photo (mandatory, always), vehicle plate (mandatory unless "no vehicle" flag set), visitor name + IC number (text, mandatory). Unit_id is server-locked from the invite record — visitor cannot choose or change the unit.

Invite TTL: Per-building, configurable via BM Settings → Apps & Access (2–168 hours). Platform default 24h. Cron marks pending invites past `expires_at` as `status='expired'` daily.

Visitor status lifecycle:
- `visitor_invites.status`: `pending` → (`submitted` | `cancelled` | `expired`)
- `visitors.status`: `active` → (`checked_in` → `checked_out` | `expired`)
Guard check-in MUST set `status='checked_in'`; checkout MUST set `status='checked_out'`.

---

## §6 · Closing Block (every session end — no exceptions)

1. DECISIONS: Append to docs/startup/recent-decisions.md under the correct §Feature-Area anchor. Use next D-XXXX number. If no anchor exists for this feature, CREATE one.
2. SCHEMA: If any table/column changed → update docs/startup/actual-schema-columns.md. Update the relevant §Table-{name} section. If new table → add new §Table-{name} section.
3. ENV: If new env var → update docs/startup/env-required.md
4. NEW TABLE CHECKLIST: Create ai_view_ for it, add to generic_read allowlist, add keywords to ai_tools.
5. ROUTER: If this session introduced a new feature area or pattern → add a row to §4 Router in THIS file.
6. STATE: Update §7 Current State below (version, counts, pending items).
7. GRAPH: If major architectural change → update docs/startup/graph-report.md.
8. LIST files changed.
9. COMMIT with prefix (feat: | fix: | docs: | refactor: | chore:) and push to main.
10. SYNC BIBLE: automatic via .husky/pre-push on git push. Ensure bible §7 Current State is updated when counts change (version, tables, pages, routes, decisions).

---

## §7 · Current State

Version: V37 closed (2026-04-20) · V38 open
Database: 180 tables · migrations 001-110
Pages: 344 · API routes: 575 · Decisions: D-0769 latest · Portals: 17+

V37 shipped D-0755 through D-0768 (14 decisions) — full visitor pre-registration feature + major platform hardening cycle.

V37 accomplishments:
- D-0755/D-0756: Resident PWA audit + wide platform recon (identified 11 P0/P1 items)
- D-0757/D-0758: B3 instrumentation verified; logActions FK + ai_processed semantic fixed
- D-0759: Bible §5 hardened + 6 platform fixes (SW prefixes, middleware redirect, dashboardForRole removal, approvals catch, resident_pwa_sessions dropped)
- D-0760: Google OAuth ORG-only · handleLoginOTP resident-only + hard-scoped · ContextSwitcher in /app
- D-0761-D-0766: Visitor pre-registration feature — schema (migrations 109, 110), resident API + PWA UI, public /v/[token] page + visitor self-reg API + upload, cron expiry + BM TTL settings, AI tool swap to create_visitor_invite
- D-0767/D-0768: Nudge bugs recon + blocked investigation (compose/nudge target displayed contact — handed to V38-S1)

3 new permanent rules added to bible §5 this cycle (D-0759):
- Residents authenticate via WhatsApp Reverse OTP only (never Telegram, never Google, never email/password)
- Google OAuth is ORG-only (never residents or staff)
- Resident PWA does NOT track devices

Visitor pre-reg rules added (D-0765):
- Dual-path (invite link primary, legacy fallback — both preserved)
- Self-reg requires IC photo + vehicle plate (or no-vehicle flag)
- Unit server-locked from invite record
- Invite TTL per-building configurable 2-168h, default 24h
- Status lifecycle (pending→submitted/cancelled/expired for invites; active→checked_in→checked_out for visitors)

V38 active priorities:
- V38-S1: BM console compose bar + nudge target displayed contact (Opus product decision needed first — see D-0767/D-0768 + docs/audits/V37-NUDGE-BUGS-RECON.md)
- R1 observation accumulation continues (CHV AI ON since 2026-04-18)
- A1 AI audit rules engine (blocked on R1 traffic volume)
- PWA push infrastructure (web-push install → VAPID → SW handler → subscribe UI → notifyVisitorArrivalPwa)

Deferred (not P0):
- Haiku 4.5 model migration (pending CHV traffic evidence)
- WA/TG automated sender filter (B4 was email-only)
- MOA Phase 2 CHV walkthrough recon
- FIX-1B resolveUser silent 'bm' default hardening
- V32b VMS floor-3 blindspot (needs device screenshot)
- V32c GQ ground-floor notation (fold into V32b)
- Guard fraud detection (defaulters + guards pressing random numbers)
- building_payment_settings duplicate table audit
- Dead code cleanup (composeVisitorArrivalMessage, VISITOR_TYPE_PHRASE, PURPOSE_TO_CATEGORY in send.ts — orphaned after D-0754)
- Compose bar recipient selector UI polish (beyond V38-S1 fix)
- Committee login UX decision
- Expected-visitors-today view for guards
- Notification workstream (arrival notify + invite-submitted notify — blocked on PWA push)

Key contacts: Seeteng (Lumi test user), Hoe Zee How (CHV BM, 712 units, AI active)

---

## §8 · Doc Inventory

docs/startup/ — Working docs (lazy-loaded via §4 Router)
  actual-schema-columns.md    148KB  Anchored by §Table-{name}
  recent-decisions.md         351KB  Anchored by §Feature-Area
  env-required.md             6.4KB  Small — OK to read fully
  GRAPH_REPORT.md             9.8KB  Anchored by §Community / §God-Nodes

docs/ — Archive (touch only when Router points here)
  decisions.log               429KB  Raw D-0001+ (not anchored)
  session-context.md           35KB  Legacy (superseded by §7)
  SESSION-SOP.md               13KB  Legacy (superseded by §2+§6)
  minilab-ai-agent-architecture.md  34KB  AI deep-dive (for MOA)
  capabilities-map.md          22KB  Feature overview
  docs/audit/                 ~800KB  V3-V14 audit reports (moved here)
  docs/audits/V34-AI-STACK-AUDIT.md  AI+MOA stack audit for V35 brainstorm (D-0731)
  docs/audits/V35-BEHAVIORAL-AUDIT.md  8-bug behavioral root cause audit, CHV re-enable prerequisites (D-0732)
  docs/audits/V35-ARCHITECTURE-RECON.md  AI infrastructure dead code + RAG verdict (D-0733)
  docs/audits/V35-NIGHT-1-HANDOFF.md  Night 1 session narrative + B3/A1/R1 handoff (D-0740)
  docs/audits/V36-PDF-VISION-RECON.md  H1 Layer 2 PDF extraction prerequisites — table, Gemini, UI, blast radius (D-0744)
  docs/audits/V36-H1-LAYER2-RECON.md   H1 Layer 2 full recon — typed columns, expires_at, R2 retention, PDPA, encryption, cost (D-0745)
  docs/audits/V36-R1-BLOCKER-RECON.md  R1 pre-flip audit — email body parsing gaps + visitor notification quality, fix shapes (D-0752)
  docs/audits/V37-RESIDENT-PWA-AUDIT.md  Full resident PWA audit — push stack, auth, visitor flow, gaps P0-P2 (D-0755)
  docs/audits/V37-WIDE-RECON.md  Multi-role context + middleware + platform bug sweep — dashboardForRole P0, WhatsApp login P1, SW prefixes (D-0756)
  docs/audits/V37-B3-INSTRUMENTATION-VERIFY.md  B3 telemetry code trace — logActions FK P0 bug, ai_processed flag misuse, skip/run distribution from live data (D-0757)
  docs/audits/V37-VISITOR-PREREG-RECON.md  Visitor pre-reg full recon — legacy flow audit, invite-link gap analysis, schema options, 12 Opus open questions (D-0761)
  docs/audits/V37-NUDGE-BUGS-RECON.md  Nudge P0 bugs — Bug 1 (WA Web QR confusion) + Bug 2 (tenant→owner wrong phone, 286 units), root causes at file:line (D-0767)
  docs/audits/V37-CLOSE-SUMMARY.md  V37 14-decision changelog + V38 handover spec (D-0769)
