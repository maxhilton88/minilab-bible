# CLAUDE.md — Minilab Platform
<!-- Version: V42 in_flight · 2026-04-22 -->
<!-- THIS IS THE ONLY FILE READ AT STARTUP. Everything else is lazy-loaded via the Router. -->

---

## §1 · What Is Minilab

Minilab (minilab.my) — AI-powered ERP SaaS for the entire Malaysian strata property management industry.
Stack: Next.js 15 App Router · TypeScript strict · Supabase PostgreSQL · Vercel · Tailwind v3 + shadcn/ui
DB: Supabase project `nncbsumdmsyjjrgdjjea` (Singapore region)
Serves: JMB/MC buildings, PMC, security, cleaning, contractor, supplier/store, developer, residents across portfolio-scale operations. NOT a single-building tool.

---

## §2 · Session Protocol

FETCH LIVE STATE (every new session): GET https://minilab.my/api/bibble/state — returns current V-number, latest D, counts, all pending tasks with scenarios + remarks. This is the single source of truth for "what's going on." Report §7 version, latest D, top 3 pending tasks before first answer.

1. Fetch /api/bibble/state (above) — report version + top 3 pending tasks
2. Read THIS file (CLAUDE.md) — nothing else yet
3. Run: git log --oneline -10 && git status
4. Check §4 Router below — identify which docs (if any) your task needs
5. Extract only the relevant sections using awk (see §3)
6. Do the work
7. Run Closing Block (§6)

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
| MOA toggle (is_moa_active) / plate/card intents / approvals extension | recent-decisions.md | §MOA |
| MOA architecture / build spec / skills / connectors | MOA-SPEC.md | Full file OK |
| Resident portal / WhatsApp OTP | recent-decisions.md | §Resident |
| Supplier / store portal | recent-decisions.md | §Supplier-Store |
| Data backfill / re-import gaps | recent-decisions.md | §Data-Backfill |
| TypeScript errors / TS cleanup | recent-decisions.md | §TypeScript-Cleanup |
| Case / task detail sheet / inspection scheduling | recent-decisions.md | §Case-Management |
| Staff management / bulk actions / all-staff page | recent-decisions.md | §Staff-Management |
| KB entries / chunking / indexing / BM knowledge base | recent-decisions.md | §KB-Infrastructure |
| Guard incidents / cleaning complaints / operational tasks / notify_resident / listing | recent-decisions.md | §AI-Pipeline |
| Incidents tab on BM VMS / guard PWA / cleaner PWA (guard_incidents, cleaning_complaints UI) | recent-decisions.md | §Guard-VMS |
| Adoptive learning / BM reply style / bm_reply_exemplars / §4.6 prompt injection | recent-decisions.md | §AI-Pipeline |

### Architecture & Code Navigation

| Task | File | Anchor |
|------|------|--------|
| Where is feature X in code? | GRAPH_REPORT.md | §Community-Index |
| What calls what? / dependencies | GRAPH_REPORT.md | §God-Nodes + §Connections |
| Permission / RLS questions | recent-decisions.md | §Permissions |
| Portal routing logic | NONE — see §5 Hard Rules below | — |
| Task board / bibble / live state | /api/bibble/state (GET) | JSON |
| Individual bibble task details | /api/bibble/tasks/[id] (GET) | JSON |

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

Identity-shaped UI fields default to building scope: Signatures, names, avatars, and other "whose voice is this reply?" fields default to building-scoped rows with personalization tokens (e.g. `{{staff_name}}`), not per-user rows — unless there's a hard compliance requirement for per-user. Per-user fields of this shape become targets for shared-session cross-writes (D-0828 shipped per-user `users.email_signature`; when Asha logged in on HZH's front-desk PC her edits overwrote HZH's row — V41-S7 / D-0832 reverted to `buildings.email_footer_template`).

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

Sidebar / route permission gates must reference keys defined in lib/auth/permissions.ts PERMISSIONS catalog. Default-deny on undefined keys silently hides UI without warning (origin: V41-S4 — staff sidebar had approvals.manage ghost key, default-denied, Approvals tab invisible). If you add a new gated surface, confirm the permission string exists in the catalog before shipping. Consider startup lint to fail build on undefined keys.

Silent-infra-failure detection: when a feature fails and shares infrastructure with other features, check ALL others using the same infra. A silent failure in shared infra (embedder, classifier, vision, router, auth) is invisible until one feature makes it loud. Origin: V41-S3 — T9 capture crashed on dim mismatch; document_chunks had been 0 rows since V40-S3 across ALL KB usage, undetected because no user flow exercised retrieval until T9 tried to write.

Per-feature Save buttons live visually inside the section they save — never floating at page footer across multiple unrelated sections. A settings page with email config + SMS config + branding should have 3 Save buttons, each inside its own card. Origin: V41 — Asha's signature confusion: one footer Save saved all sections at once, edits to wrong field were ambiguously attributed to the visible save action.

DashScope text-embedding-v3 native dim = 1024. Compatible-mode supports {512, 768, 1024} only. NEVER use 1536 (that's OpenAI text-embedding-3-small, different provider). All pgvector columns on embedding-producing tables are vector(1024). All HNSW indexes are ops=vector_cosine_ops. Origin: V41-S3g — document_chunks + bm_reply_exemplars + media_search_index migrated from 1536→1024 in migration 118.

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
11. BIBBLE SYNC (MANDATORY — fails silently, so verify): For every bibble task touched this session, call upsertBibbleRemark(taskId, body) from lib/bibble/log.ts. For any new task surfaced, call createBibbleTask(...). Session close MUST update bibble_meta rows: version, version_status, latest_d, open_priorities. Use supabaseAdmin directly — no API hop. If task IDs are unclear, query: SELECT id, title, session FROM bibble_tasks WHERE status IN ('READY','IN_PROGRESS','RECON_NEEDED') ORDER BY session, display_order;

---

## §7 · Current State

Version: V42 in_flight · V41 closed 2026-04-21
Database: 186 tables · migrations 001-121
Pages: 347 · API routes: 602 · Decisions: D-0849 latest · Portals: 17+
Live task board: /bibble · State API: /api/bibble/state · Raw docs: /bibble/raw/*.md · Doc manifest: /api/bibble/docs
V42 in flight — ALL priorities, scenarios, remarks live at /bibble. This §7 no longer tracks pending items (moved to bibble_tasks table per D-0834). Historical session summaries preserved below for context.

V41-S7 shipped D-0832, D-0833 (2 decisions) — per-user email signature (D-0828) replaced with building-level email footer template. Migration 120 drops `users.email_signature*`, adds `buildings.email_footer_template` + `buildings.email_footer_enabled`. Console prefill interpolates `{{staff_name}}` from session full_name — fixes cross-user write bug (Asha writing into HZH's row on shared front-desk PC). UI: new "Email Footer" card at /bm/settings/email with live preview of interpolated output.

V41-S4 shipped D-0827 through D-0829 (3 decisions) — BM console email reply unblocked at CHV: channel-aware send guard (D-0827) + per-user email signature with auto-prefill into console compose (D-0828, migration 119, **superseded by V41-S7 / migration 120**) + /bm/settings/email layout left-aligned full-width (D-0829).

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

V38 shipped D-0771 through D-0780 (7 decisions + 1 recon + 1 audit) — BM console correctness, WA delivery observability, VMS checkout unblock, AI message status normalization.

V38 accomplishments:
- D-0771 (V38-S2): VMS checkout CHECK constraint extended to 'checked_out' + UI error surfacing — 935 stuck visitors unblocked
- D-0772 (V38-S1): Console send/nudge/compose target displayed contact in contact view (resolveTarget helper introduced)
- D-0774 (V38-S3): WA delivery webhook wired — statuses[] now processed, new columns delivered_at/read_at/failed_at/delivery_error/delivery_error_code, wa_delivered/wa_read/delivery_failed lifecycle, avg 1.56s delivery signal
- D-0775 (V38-S1b): Nudge internal log card persists against correct thread in contact view; resident_id surfaced on ContactListItem
- D-0778 (V38-S6): delivery_failed surfaced in console timeline — window_closed reuses 24h nudge banner, other failures get expandable dev-details card with copy-to-clipboard, one-click retry via new /api/bm/console/retry endpoint (classifyDeliveryFailure helper)
- D-0779 (V38-S5): Silent delivery failures fixed — deliveryError now captured in all failure branches across direct-message + send + retry routes (deliveryFailedPatch helper)
- D-0780 (V38-S7): AI message status normalized from 'complete' to 'delivered'; messages.status DB default dropped; 1,694 historical rows backfilled; 8 other insert sites made explicit

Three new class-killing helpers this cycle:
- lib/messaging/resolveTarget (or equivalent per page.tsx) — view-aware recipient resolution
- lib/messaging/classify-delivery-failure — failure category single source of truth
- lib/messaging/delivery-failed-patch — DB update shape for failed sends

V39 shipped D-0781 through D-0788 (8 decisions) — WhatsApp AI audit + Jarvis infrastructure recon + strategy handover. Discovery session, no code changes.

V39 accomplishments:
- D-0781: WhatsApp /ai-activity audit — 22 issues across 9 categories identified (silent drops, unimplemented handlers, enum hallucination, latency, DeepSeek bypass)
- D-0782: Jarvis infrastructure recon — RAG framework fully built but dormant (document_chunks 0 rows across all buildings)
- D-0783: Framing rule locked — Jarvis for Building is north star; deletion of tool surfaces as a fix is forbidden
- D-0784: /ai-activity page confirmed public by design (intentional operational trade-off, no auth gate)
- D-0785: KB architecture decided — user-defined free-form sections + entries (text or doc upload); schema choice pending V40 Opus validation
- D-0786: create_legal_signal_case orphan handler to be deleted in V40 hygiene batch
- D-0787: 11 write-action handlers deferred to V40+; get_payment_instructions (thin) brought to V39 execution
- D-0788: Session bootstrap system established — MINILAB-BOOTSTRAP.md + V39-HANDOVER.md (saved locally by Max)

V40-S1 shipped D-0789 (1 decision) — WA webhook + admin pipeline silent-drop instrumentation.
V40-S3 shipped D-0790 through D-0793 (4 decisions) — KB infrastructure + MOA schema + OpenAI purge.
V40-S5 shipped D-0796 through D-0799 (4 decisions) — MOA toggle UI + plate/card intents + approvals extension.
V40-S6 shipped D-0800 through D-0806 (7 decisions) — hygiene batch: enum validation, thin handlers, credential vault, orphan delete, synthetic intent doc, console guard.
V40-S7 shipped D-0807 through D-0808 (2 decisions) — building_working_hours in §3 + KB pre-fetcher for 3 intents.
V40-S8 shipped D-0809 through D-0814 (6 decisions) — guard/cleaner/task/resident/timeline/listing handlers + 8 dead handlers removed.
V40-S7b shipped D-0815 through D-0818 (4 decisions) — incidents + cleaning complaints UI on VMS + Guard PWA + Cleaner PWA.

V40-S3 accomplishments:
- D-0793: OpenAI embedder purged — lib/documents/embedder.ts fully rewritten to DashScope text-embedding-v3; platform single-vendor
- D-0792: unit_vehicles + unit_access_cards tables + buildings.is_moa_active (migration 114)
- D-0791: MOA task queue architecture decision — reuses approvals table, approval_type extends in S5
- D-0790: building_kb_entries table + lib/kb/indexer.ts + BM KB CRUD API (5 routes) + ai_view_kb + document_chunks.kb_entry_id link-back

V40-S6 accomplishments:
- D-0800: createCase enum validation + 28-entry coercion map; unknown values fall to 'other'; DB never rejects silently
- D-0801: get_payment_instructions thin handler reads building_payment_settings (actual columns); closes 1 weekly CHV failure
- D-0802: find_open_case thin handler; lookup by case_number or resident_id; closes 2 weekly COMPLAINT_FOLLOWUP failures
- D-0803: DEEPSEEK_API_KEY moved from process.env to credential vault in column-mapper.ts + auto-register.ts
- D-0804: create_legal_signal_case orphan handler deleted (confirmed zero callers)
- D-0805: ADMIN_COMMAND synthetic intent documented in INTENT_DEFINITIONS for audit completeness
- D-0806: resolveTarget null-phone warn + handleSend early-return toast; silent null-send drops closed

V40-S8 shipped D-0809 through D-0814 (6 decisions) — guard/cleaner/task/resident/timeline/listing handlers + 8 dead handlers removed.
V40-T9 shipped D-0819 through D-0822 (4 decisions) — adoptive learning: bm_reply_exemplars capture + retrieval + §4.6 injection.
V41-S3d shipped D-0823 (1 decision) — DashScope endpoint → intl region (Singapore API key accepted).
V41-S3f shipped D-0824 (1 decision, reverted) — T9-TRACE diagnostic logs (removed in S3g).
V41-S3g shipped D-0825 (1 decision) — DashScope native dim 1024 fix: migration 118 (3 columns + 5 indexes + 2 RPCs), embedder.ts 1536→1024, T9 traces stripped, bm.ts dead generateEmbedding deleted.

V40 EXECUTION PENDING (remaining after T9):

- T8: Seed CHV KB manually via new UI (Max + BM Hoe Zee How — payment details, house rules, facility rules, emergency contacts, committee decisions).
- T12: End-to-end verification — resident asks KB-answerable question, confirm retrieval fires and answer is grounded. Confirm BM reply captures into bm_reply_exemplars.

Pre-existing pending (carried forward):
- WA UI tick truth + rename 'delivered' → 'sent' (V38-S4 deferred — needs more webhook data)
- 24h window pre-send block (prevent 131047 instead of surfacing it post-fail)
- 887 stale 'active' visitors at CHV — daily cron flips active → expired after 24h
- A1 AI audit rules engine (blocked on R1 traffic volume)
- FIX-1B resolveUser silent 'bm' default hardening
- V32b VMS floor-3 blindspot (S4-S7 guards) — blocked on device screenshot
- V32c GQ ground-floor notation normalizer (fold into V32b)
- Guard fraud detection (defaulters + random-number guards)
- building_payment_settings duplicate table audit
- Haiku 4.5 model migration — may now have enough CHV traffic for evidence
- WA/TG automated sender filter (B4 was email-only)
- Dead code cleanup (composeVisitorArrivalMessage, VISITOR_TYPE_PHRASE, PURPOSE_TO_CATEGORY in send.ts — orphaned after D-0754)
- Compose bar recipient selector UI polish (beyond V38-S1 fix)
- Committee login UX decision
- Expected-visitors-today view for guards
- arrival notify + invite-submitted notify (blocked on PWA push infra)
- PWA push infrastructure — web-push install → VAPID → SW handler → subscribe UI → notifyVisitorArrivalPwa

Deferred to V40+:
- 9 missing write-action handlers (send_emergency_broadcast, notify_supplier, notify_contractors, notify_agencies, request_photo, charge_listing_fee, assign_contractor, set_recurrence, notify_resident, get_case_timeline) — see D-0787; find_open_case + get_payment_instructions now done (D-0801/D-0802)
- Resident pipeline unification with ai_tools table (architectural, separate design session)
- media_search_index activation (ingestion pipeline for image-based KB)
- voice_knowledge_base redesign (currently platform-wide, needs building_id scope)

Blocked on Max:
- CHV staff walkthrough with BM Hoe Zee How — record vendor workflows, identify access card / car plate vendor names
- DISABLE_DEV_LOGIN=true on Vercel
- E2E test with Seeteng at Lumi Residency
- MOA Phase 2 build (MOA-SPEC.md complete, Node.js + Playwright VPS agent targeting Advelsoft HTML5 RDP)
- Superadmin PDPA deletion workflow
- BM staff page dropdown transparency bug
- CHV collection accounts setup

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
  docs/audits/V38-S2-VMS-CHECKOUT-RECON.md  VMS checkout CHECK constraint root cause — 935 stuck visitors, fix shape (D-0770)
  docs/audits/V38-S3-WA-DELIVERY-RECON.md  WA double-tick lies — delivery webhook not wired since D-0366, fix spec (D-0773)
  docs/audits/V38-S5-SILENT-FAILURES-RECON.md  Silent delivery_failed writes — root cause in send routes, Nursyafiqah vs Nas, 33-row blast radius, fix shape (D-0776)
