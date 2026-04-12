# Minilab — V27 Session Handover

# Created: 2026-04-12 | Covers: D-0601–D-0618+

---

## 1. V27 Summary — Everything Shipped

### AI Pipeline — Critical Fix + All 15 Audit Risks Resolved

**D-0601: AI Infrastructure Audit**
- Read-only audit of 30+ source files across the AI pipeline
- Found critical bug: `specialist-agent.ts` and `actions/executor.ts` were fully built but orphaned — never imported anywhere
- All 23 resident AI intents returned silence because the semantic router classified correctly but no code executed the actions
- Produced `docs/startup/v27-ai-action-audit.md`

**D-0602: Wired Resident AI Pipeline (CRITICAL FIX)**
- Imported `runSpecialistAgent()` + `executeActions()` + `fetchIntentData()` into `lib/ai/v4-pipeline.ts`
- Resident/contractor messages at Tier 1/2/3 now: classify intent → fetch data → run specialist agent → execute actions (create cases, bookings, visitor passes, notify BM) → send response
- Error fallback returns safe BM-referral message
- **This single change activated all 23 resident AI intents**

**D-0603: R-01 Telegram Webhook Fail-Closed**
- Webhook now returns 500 when `TELEGRAM_WEBHOOK_SECRET` is not configured
- Previously silently accepted all requests if env var was absent

**D-0604: R-03 Hallucination Guard Flags for Review**
- When Supabase credentials missing, returns `flagForReview: true` + logs error
- Previously silently passed everything unvalidated

**D-0605: R-07 Material Probe Redis After Send**
- Redis key now set AFTER successful message send, not before
- Prevents stale keys when Telegram/WhatsApp send fails

**D-0606: R-10 Anthropic Key to Credential Vault**
- `lib/ai/providers/anthropic.ts` now uses `getCredential()` like DeepSeek
- Lazy `resolveApiKey()` loads from vault on first `complete()` call
- Supports rotation without redeploy

**D-0607: R-06 contacts_temp Channel Columns**
- Added `channel` (default 'whatsapp') and `channel_id` columns
- Phone made nullable with CHECK constraint (phone OR channel_id must exist)
- Auto-registration insert now includes channel metadata

**D-0608: R-13 Public Holidays Configurable**
- Holidays loaded from `public_holidays` table with 1-hour cache
- Hardcoded 2026 list kept as fallback
- `checkWorkingHours()` accepts `dbHolidays` param

**D-0609: R-15 Routing Analytics Table**
- New `routing_analytics` table with indexes + RLS
- `logRoutingDecision()` does non-blocking insert alongside `console.log`
- `ai_view_routing_analytics` (30-day window) added to generic read allowlist

**All 15 risks from the AI infrastructure audit are now resolved.**

---

### Resident Data Dashboard — Full Feature

**D-0610: Data Quality Columns on Residents**
- 6 new columns: `data_status`, `last_contacted_at`, `data_source`, `phone_valid`, `verified_at`, `verified_by`
- `data_status` enum: 'verified' | 'touched' | 'imported' | 'unreached'
- RPC `update_resident_contact_status(p_resident_id, p_channel)` — never downgrades verified
- Wired into both WhatsApp and Telegram webhooks (fire-and-forget on inbound)
- AI view `ai_view_resident_data_quality` created

**D-0611: Data Dashboard Tab**
- New first tab "Data dashboard" on `/bm/residents`
- API: `GET /api/bm/residents/data-quality` — summary stats, per-block breakdown, pending approvals, monthly activity
- 5 sections: summary cards, data quality bars, pending verification + monthly activity, block breakdown table, action buttons
- CSV export (client-side from fetched data)

**D-0612: AI Resident Data Enrichment**
- New table: `resident_data_staging` (21 columns, RLS, indexes, ai_view)
- `enrich_resident` action handler in executor — inserts into staging with building context
- `ActionContext` extended with `channel` field, wired in v4-pipeline.ts
- AI prompt updated with 7 data collection rules:
  - Max 1 question per conversation
  - Help first, collect second
  - Never ask for IC
  - Skip if emergency/frustrated
  - Confidence levels: high/medium/low
- Staging API: `PATCH /api/bm/residents/data-quality/staging/[id]` — approve merges into residents table (handles existing + new + related person), reject with notes
- Dashboard updated: staging items shown with purple "AI update" badge + confidence level

**D-0616: Dashboard Design Polish + Backfill Fix**
- Migration 084: smarter backfill matching residents to sender_profiles by phone (handles +60/60 format differences)
- Redesigned quality bars: thick h-7 bars with white text inside, clean fintech aesthetic
- Fixed activity metrics: shows absolute count + "N last month" context instead of broken subtraction
- Premium design: dark approve buttons, uppercase tracking-wide headers, consistent gray-900/500 palette

**D-0617: Dashboard Drill-Down + Email Touch**
- Migration 085: email contacts now count as "touched" via sender_profiles email match
- Email processor (`lib/email/processor.ts`) wired to fire `update_resident_contact_status`
- New API: `GET /api/bm/residents/data-quality/block-detail` — 11 metrics with unit-level data
- All 8 block detail stats are clickable → opens `DrillDownPanel` slide-over
- Positive metrics show inverse view first ("Without" expanded, "With" collapsed)
- Unit numbers link to `/bm/console?unit_id=` for direct editing

**D-0618: Drill-Down Panel Polish**
- Panel widened to max-w-4xl, uniform text-xs typography
- Console deep-link via existing `?unit_id=` param
- Unreachable units (no phone/email) shown as non-clickable

---

### Console Fixes

**D-0613: Media Placeholder Rendering**
- `MessageMedia` now shows type-appropriate placeholder (Photo / Video / Document / Voice) when `media_url` is missing
- All 4 bubble types in `CenterTimeline.tsx` render `MessageMedia` when `media_type` present

**D-0614: WhatsApp Media Persistence Fix (Root Cause)**
- `persistWhatsAppMedia()` always used platform-level `META_ACCESS_TOKEN`
- Buildings with their own WABA (like CHV) store per-building `whatsapp_access_token`
- Fixed: now uses `getBuildingWhatsAppConfig(buildingId)` — same per-building token resolution as `sendTextMessage()`
- Added `console.error` logging at every failure point (6 paths were previously silent)

**D-0615: Console Media Lightbox**
- Click image thumbnail → full-screen lightbox (dark overlay, centered, max 90vw/90vh)
- Close via Escape / backdrop click / X button

---

### BM Dashboard Fix

**Dashboard Zero Data (CHV)**
- Root cause: URL-length crash — 712 unit UUIDs passed via `.in()` generated ~27KB query string, exceeding Nginx's 8KB URI limit → 414 error → catch block returned `{ warning }` with HTTP 200 → page crashed on `data.stats.openCases`
- Fix: replaced `.in(712 UUIDs)` with `units!inner` FK join, replaced ID pre-fetch with count-only query, catch now returns valid zero-shape `DashboardData`, client crash guard added

---

## 2. DB Migrations Applied in V27

| Migration | Summary |
|-----------|---------|
| 082 | 6 data quality columns on `residents` + `update_resident_contact_status` RPC + `ai_view_resident_data_quality` |
| 083 | `resident_data_staging` table (21 columns) + indexes + RLS + `ai_view_resident_data_staging` |
| 084 | Backfill `data_status='touched'` via phone matching on sender_profiles |
| 085 | Backfill `data_status='touched'` via email matching on sender_profiles |

Other schema changes (no migration number):
- `contacts_temp`: added `channel`, `channel_id` columns, phone made nullable (D-0607)
- `routing_analytics`: new table with indexes + RLS + ai_view (D-0609)

---

## 3. Current State

**Version:** V27 (2026-04-12)
**Database:** ~167 tables · migrations 001-085
**Pages:** ~288 · API routes: ~393 · Decisions: D-0618+ · Portals: 17+

---

## 4. Pending / Next Session

### P1 — Ready to Build
- **P4: WhatsApp Data Collection Blast** — batch queue, template send via cron, Redis state for data collection flow, bounce tracking, retry logic. Dashboard mockup and flow diagram already designed.
- **Resident History Card in Console** — audit confirmed NOT BUILT. `Contact360.tsx` exists but is orphaned (old V4 architecture). Needs new `HubSection` in `RightPanel.tsx`.
- **Delete orphaned `Contact360.tsx`** — confirmed never imported

### P1 — Human Tasks
- **CHV collection accounts setup** — dashboard collection section shows 0% / RM 0 because no collection_accounts exist per unit. Setup via `/bm/finance` or bulk setup.
- **DISABLE_DEV_LOGIN=true** in Vercel
- **E2E test with Seeteng** at Lumi Residency
- **CHV onboarding** — Hoe Zee How, 700+ units

### P2 — Important
- Verify AI enrichment is asking data questions in live conversations
- Verify routing_analytics table is collecting data
- Superadmin PDPA deletion
- BM staff page dropdown transparency bug

### P3 — Backlog
- MOA Phase 2
- AI cost analytics superadmin page
- AGM voting system
- Form 11 auto-generation
- Resident portal dashboard

---

## 5. Key Architecture Decisions from V27

| Rule | Decision | Never Break |
|------|----------|-------------|
| Resident AI pipeline: specialist-agent.ts + executor.ts wired into v4-pipeline.ts | D-0602 | All 23 intents route through this path |
| AI enrichment: max 1 data collection question per conversation, help first | D-0612 | Never interrogate residents |
| AI enrichment goes to staging table, BM approves before merge | D-0612 | Never write directly to residents from AI |
| data_status enum: verified > touched > imported (never downgrade) | D-0610 | RPC enforces this |
| Telegram webhook: fail-closed if TELEGRAM_WEBHOOK_SECRET absent | D-0603 | Security critical |
| WhatsApp media: use per-building WABA token via getBuildingWhatsAppConfig() | D-0614 | Platform token doesn't work for per-building WABA |
| Dashboard: use FK joins, never .in() with large UUID arrays | D-0619 | Nginx 8KB URI limit |
| All 3 channels (WA/TG/email) fire update_resident_contact_status on inbound | D-0610/D-0617 | Keeps data_status current |

---

*Generated: 2026-04-12 at end of V27 session (D-0618+)*
