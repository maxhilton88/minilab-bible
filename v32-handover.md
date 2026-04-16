# V33 Session Handover — Pending Bugs & Tasks

**Generated:** End of V32 session (2026-04-17)
**Last clean state:** V32e, 0 TS errors, ignoreBuildErrors removed
**Decisions through:** D-0704
**Migrations through:** 097 (import_backfill_log)

---

## CRITICAL — Do First

### D-0702: 8 Unapplied Migrations in Production

Discovered during V32d TS audit. These migration files exist in the repo 
but were NEVER applied to the production Supabase DB.

| Migration | Description | Impact |
|-----------|-------------|--------|
| 055_d7_defect_portal_billing_onboarding.sql | Defect portal + billing tables | Feature incomplete |
| 063_visitors_doc_photos.sql | Visitor document photo columns | Guard VMS feature gap |
| 075_clean_html_email_messages.sql | Email message cleanup | Email rendering issues |
| 076_backfill_outbound_sender_profile.sql | Outbound sender profile | Email sender identity |
| 078_buildings_wa_app_secret.sql | WA app secret columns on buildings | WhatsApp config possibly broken |
| 085_resident_email_touch_backfill.sql | Resident email touch tracking | Data enrichment gap |
| 089_residents_phone_nullable_visitors_doc_photos.sql | Phone nullable + visitor photos | Schema drift |
| 090_visitors_pass_number.sql | Visitor pass numbering | Guard VMS feature gap |

**Migration naming drift (036-039 range):**
Repo files and DB entries share the same numeric prefix but have different names.
Content may or may not overlap — needs per-file audit before applying.

| Repo file | DB applied name |
|-----------|----------------|
| 036_cases_attachments.sql | 036_blocks_metadata |
| 037_petty_cash.sql | 037_kv_cleanup |
| 038_telegram_groups_extend.sql | 038_staff_position |
| 039_attendance_user_id.sql | 039_announcements_bank_accounts_insurance_building_documents |

**Approach:** 
  1. Audit each migration individually — read the SQL, check if tables/columns 
     already exist in prod (some may have been applied manually)
  2. Apply in numeric order, one at a time
  3. After each: verify via Supabase MCP that tables/columns exist
  4. Regenerate types after all applied
  5. Run tsc --noEmit to confirm no new errors introduced
  
**DO NOT bulk-apply. Per-migration audit required.**

---

## HIGH — V32 Follow-ups

### V32b: VMS Unit Search Floor-3 Blindspot

**Bug:** Guards at CHV report S4/S5/S6/S7 floor-3 units and S5/S6 high 
floor-2 units (02-09 to 02-12) as "missing" — but all 40 units exist in DB 
with is_active=true. 

**Suspected cause:** Dropdown scroll UX on guard phones. When guard types "S6", 
dropdown shows first ~16-20 matches alphabetically. Floor-3 units sort below 
floor-1/2 entries. Guard doesn't scroll, concludes units are missing.

**Blocked on:** Device screenshot or video from guard phone showing VMS unit 
search dropdown. Without seeing what the guard actually sees, fix is guesswork.

**Action:** Get Hoe Zee How or a CHV guard to screenshot the VMS search on 
their phone when typing "S6". Then audit the exact render code path.

**Files likely involved:**
  - components/shared/UnitSearchField.tsx (V32 merged component)
  - app/guard/page.tsx (VMS forms)

### V32c: GQ Ground-Floor Leading-Zero Notation

**Bug:** User types "G-05", DB stores "GQ/G-5" (no leading zero). 
Normalized: g05 vs gqg5 — not a substring match even after V32 normalization.

**Fix:** Small normalizer enhancement — collapse leading zeros in ground-floor 
patterns. Can fold into V32b session.

**Scope:** Affects GB, GQ, KV ground floors (any block with G-N format).

### D-0632: Unit Number Data Canonicalization

**Bug:** Multiple format variants stored for same block:
  - Greenscene: `GS/01-01` vs `GS - 06` vs `GS/01/07`
  - Greencoast: `GC/01-01` vs `GC 105`
  - Sawtelle B: `B601` (no dashes)
  - 6 duplicate normalized keys in DB (both dirty + clean versions exist)

**Impact:** Search works (V32 normalization absorbs variants), but exports, 
reports, invoices, WhatsApp messages show inconsistent unit numbers.

**Fix:** Write a one-time canonicalization migration that standardizes all 
unit_numbers to {PREFIX}/{FF}-{UU} format. Must check for FK references 
where unit_number is stored as denormalized string.

---

## MEDIUM — Production Quality

### D-0698: Telegram Webhook TypeScript Cleanup

**File:** app/api/auth/telegram/webhook/route.ts
**Issue:** 25+ `as any` casts throughout. Auth-critical, high-traffic endpoint 
flying blind on type safety. Only 1 actual TS error (fixed in V32d), but the 
file has zero meaningful type checking.

**Risk:** Medium — runtime works, but any refactor could introduce silent bugs 
that types would catch if they weren't suppressed.

**Approach:** Dedicated session. Carefully type supabaseAdmin calls, remove 
`as any` one at a time, verify behavior after each.

### BM Staff Page Dropdown Transparency Bug

**Bug:** Filter dropdowns on BM staff page have transparent background — 
text overlaps with page content behind.

**Fix:** CSS — likely missing `bg-white` or `bg-popover` on the dropdown container.

### DISABLE_DEV_LOGIN=true on Vercel

**Task:** Set environment variable on Vercel production to disable dev login bypass.
**Type:** Human task — Vercel dashboard, not code.

### Superadmin PDPA Deletion Workflow

**Status:** PDPA deletion cron exists (lib/crons/pdpa-deletion.ts). Types 
fixed in V32d. But the superadmin UI to initiate deletion requests may not 
be built. Needs audit.

---

## LOW — Deferred / Future Sessions

### MOA Phase 2 — Node.js + Playwright VPS Agent
Per MOA-SPEC.md (V28). Build the actual agent. Large scope — dedicated session(s).

### E2E Test with Seeteng at Lumi Residency
Lumi is test building. Seeteng is BM test user. Full walkthrough pending.

### CHV Onboarding Human Tasks
  - Create security + cleaning orgs via superadmin
  - Assign to CHV building
  - Set geofence for on-site attendance
  - Collection accounts setup
  - Access card + car plate vendor names (pending staff walkthrough)

### Resident History Card (Contact360.tsx)
Component exists but is orphaned — not wired into any page. Needs to be 
integrated into Console unit panel.

### P4 WhatsApp Resident Data Collection Blast
Designed, ready to build. Low priority until CHV go-live.

---

## Standing Rules Established in V32

1. **Building onboarding:** Every new building requires CSV→DB reconciliation 
   before go-live. Use V32a pattern (scripts/import-chv-gap-units.ts as template).

2. **TypeScript discipline:** ignoreBuildErrors stays false. @ts-expect-error 
   with reason only. Never @ts-ignore, never as any, never non-null !.

3. **Unit search:** All unit search goes through search_units_smart RPC. 
   Block selection = FILTER (uncapped by block_id). Text query = SEARCH (capped).
