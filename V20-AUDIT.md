# V20 Full Platform Audit

**Date:** 2026-04-10
**Scope:** 7 audit categories across 334 pages, 528+ API routes, 190+ DB tables
**Mode:** Read-only report. No fixes applied.

---

## Audit 1 — BM Staff Pages

| # | Category | Location | Issue | Severity | Suggested Fix |
|---|----------|----------|-------|----------|---------------|
| 1 | query | `app/api/bm/staff/route.ts` GET L38-48 | No role filter — fetches ALL roles (resident, pmc, etc.) but UI only renders bm/staff/committee. Wasted bandwidth. | P2 | Add `.in('role', ['bm', 'staff', 'committee'])` to the query |
| 2 | data-model | `app/bm/staff/page.tsx` L446 | Committee members (role='committee') appear in the staff page. Committee is NOT staff — they are elected owners/residents. | P2 | Filter committee out of staff page; manage via dedicated /bm/committee page or clearly label section |
| 3 | data-model | `app/api/bm/staff/all/route.ts` L59 | All Staff API includes committee role `.in('role', ['bm', 'staff', 'committee'])` mixed with contractor staff. BM could bulk-select committee for "Assign to company." | P2 | Exclude committee from All Staff query or prevent bulk-assign for committee rows |
| 4 | UX | `app/bm/staff/all/page.tsx` L282,488 | Role filter dropdown mixes display labels ("Building Manager", "Committee") with raw contractor roles ("guard", "cleaner"). Inconsistent. | P3 | Normalize all values to display labels |
| 5 | UX | `app/bm/staff/all/page.tsx` L595-606 | Checkbox shown on ALL rows including assigned + internal staff. Selecting ineligible rows → late error "No unassigned staff in selection." | P2 | Hide checkbox on non-eligible rows or disable "Assign" button when none eligible selected |
| 6 | data | `app/api/bm/staff/all/route.ts` L128 | Absent count hardcoded as `30 - present`. Incorrect for months with <30 working days. Misleading metric. | P3 | Calculate actual working days or remove "Absent" |
| 7 | bug | `app/bm/staff/all/page.tsx` L802 | FaceEnrollModal receives `source_id` (user_roles.id) as `userId` for internal staff. Should be `users.id`. Face enrollment will fail or corrupt data. | P1 | Pass the correct `user.id` instead of `user_roles.id` for source='user' |
| 8 | nav | `components/ui/MinilabShell.tsx` L50 + `app/bm/layout.tsx` L51-52 | Both "Staff" (`/bm/staff`) and "All Staff" (`/bm/staff/all`) highlight simultaneously — `startsWith('/bm/staff/')` matches both. | P2 | Use exact match for `/bm/staff` or exclude items where a more-specific match exists |
| 9 | responsive | `app/bm/staff/all/page.tsx` L552 | 12-column grid too wide for <1400px screens. Text truncation heavy. | P3 | Collapse low-priority columns behind expandable row or horizontal scroll |
| 10 | perf | `app/bm/staff/all/page.tsx` | No pagination — entire staff list rendered at once. Slow for 100+ staff. | P3 | Add client-side pagination (50/page) or virtual scroll |
| 11 | UX | `app/bm/staff/all/page.tsx` L374 | CSV export includes raw "source" column (`contractor_staff`/`user`) — meaningless to BM. | P3 | Rename to "Staff Type" with "Contractor"/"Internal (JMB)" |
| 12 | UX | `app/bm/staff/all/page.tsx` L838-843 | Edit modal for unassigned contractor staff has no empty/unassigned option in org dropdown. | P3 | Add "Unassigned" option |

---

## Audit 2 — Empty/Stub Pages

| # | Category | Portal | Page Path | Issue | Severity | Suggested Fix |
|---|----------|--------|-----------|-------|----------|---------------|
| 13 | stub | superadmin | `/superadmin/supplier-pricing` | TODO: API not built. Fake `setTimeout(500)` + `setItems([])`. No real data. | P2 | Build `/api/superadmin/supplier-pricing` or remove page from nav |
| 14 | stub | superadmin | `/superadmin/billing-settings` | TODO: API not built. Hardcoded pricing rows `[{PMC, 500}, {Contractor, 50}...]` + `setTotalMRR(12500)`. No real data. | P2 | Build `/api/superadmin/billing-settings` or remove page from nav |
| 15 | placeholder | app | `/app/onboard/placeholder` | "Coming soon" placeholder page for onboarding stages 3-8. | P3 | Expected — intentional placeholder for unreleased stages. Low priority. |
| 16 | no-data | marketing | `/for/contractor` | "Coming soon — register now to be first in line" marketing CTA. Factoring section. | P3 | Marketing — acceptable |
| 17 | no-data | marketing | `/features` | "Accounting Bridge" + "Hardware Control" marked "Coming soon." | P3 | Marketing — acceptable |
| 18 | no-data | bm | `/bm/settings/bank-accounts` | Two items marked "Coming Soon" badge (Billplz integration, FPX auto-reconcile). | P3 | Acceptable — clearly labeled as upcoming |
| 19 | no-data | bm | `/bm/procurement` | "Coming soon: Auto-tracking via Telegram hardware group" note. | P3 | Acceptable — clearly labeled |
| 20 | stub | opentender | `/opentender/post` L240 | "File upload coming soon" — photo attachment not implemented. | P3 | Build file upload for tender posts |

---

## Audit 3 — Dead/Disconnected Code

| # | Category | File | Status | Evidence | Suggested Fix |
|---|----------|------|--------|----------|---------------|
| 21 | dead-code | `components/bm/console/AIFeed.tsx` | DEAD | Defined but never imported by any .ts/.tsx file | Delete file |
| 22 | dead-code | `components/bm/console/ConsoleAnalytics.tsx` | DEAD | Defined but never imported by any .ts/.tsx file | Delete file |
| 23 | dead-code | `lib/retry.ts` (withRetry) | DEAD | Only file containing `withRetry` is its own definition. Zero imports. | Delete file |
| 24 | used | `lib/responsive/breakpoints.ts` | USED | Self-referencing only (imported in its own file). Suspicious but may be used via re-export. | Verify usage or delete |
| 25 | used | `app/(marketing)/_components/icon-box.tsx` | USED | Imported by `homepage-client.tsx` | Keep |
| 26 | used | `app/(marketing)/_components/particles-bg.tsx` | USED | Imported by `homepage-client.tsx` | Keep |
| 27 | used | `lib/utils/roles.ts` (getRoleLabel) | USED | Imported by `app/app/page.tsx`, `app/app/profile/page.tsx` | Keep |
| 28 | used | `components/marketing/StoreFinderMap.tsx` | USED | Imported by `homepage-client.tsx` | Keep |
| 29 | used | `components/marketing/TenderPreview.tsx` | USED | Imported by `homepage-client.tsx` | Keep |
| 30 | used | `lib/tender/category-image.ts` | USED | Imported by opentender pages | Keep |
| 31 | config | `jest.config.js` | USED | Test runner config — standalone by nature | Keep |
| 32 | config | `next.config.js` | USED | Framework config — standalone by nature | Keep |
| 33 | config | `postcss.config.js` | USED | Framework config | Keep |
| 34 | config | `tailwind.config.ts` | USED | Framework config | Keep |
| 35 | config | `sentry.client.config.ts` / `sentry.edge.config.ts` / `sentry.server.config.ts` | USED | Sentry auto-loads these | Keep |
| 36 | framework | `instrumentation.ts` | USED | Next.js auto-loads for telemetry | Keep |
| 37 | framework | `robots.ts` / `sitemap.ts` | USED | Next.js metadata convention files | Keep |
| 38 | framework | `global-error.tsx` | USED | Next.js error boundary — auto-loaded | Keep |
| 39 | framework | `sw.js` | USED | Service worker — loaded by browser, not imported | Keep |
| 40 | dead-doc | `docs/` isolated graph nodes | N/A | `V3 Build Requirements`, `V4 Build Requirements`, `Minilab Interface Contracts`, `Live Database Schema`, 25+ audit/task docs — graph sees them as disconnected because docs don't have code imports | Not dead — docs by nature. Graph limitation. |

---

## Audit 4 — God Node Risk

| # | Category | God Node | File(s) | Domains Touched | Missing Error Handling | Severity |
|---|----------|----------|---------|-----------------|----------------------|----------|
| 41 | god-node | `POST()` (WA webhook) | `app/api/webhooks/whatsapp/route.ts` | 8 domains: auth/HMAC, message parsing, sender profile upsert, media persist (R2), rate limiting (Redis), auto-registration, defaulter intercept, AI pipeline | Each message in `waitUntil` loop has try-catch with Sentry capture. Sender profile upsert is try-catch non-blocking. Redis rate-limit fail-open. Media persist fail-soft. | P3 — well-handled |
| 42 | god-node | `GET()` (WA webhook verify) | `app/api/webhooks/whatsapp/route.ts` | 1 domain: Meta challenge verification | Simple — no missing handling | P3 |
| 43 | god-node | `tgSend()` | `lib/telegram/notify.ts` L27-53 | 2 domains: Telegram API, user block detection | Missing: no try-catch around the entire function body. If `getCredential()` throws, caller gets unhandled rejection. Bot-blocked detection (403) correctly flags user. Non-ok responses logged but not thrown. | P2 — add top-level try-catch |
| 44 | god-node | `executeFormSubmission()` | `lib/ai/actions/form-executor.ts` | 13 form action types (task, maintenance, procurement, announcement, renovation, AGM, tender, insurance, document, petty-cash, asset, moveinout) | Main dispatcher validates action in registry. Each sub-executor validates required fields. BUT: no top-level try-catch wrapping the switch — if a sub-executor throws (e.g., Supabase network error), it propagates unhandled. | P2 — wrap dispatcher in try-catch |
| 45 | god-node | `processClawBotMessage()` | `lib/clawbot/engine.ts` | 10 onboarding steps: building details, resident data, accounting, telegram groups, structure, staff, documents, email forwarding, review | Each step handler function. Redis state machine. Modular design. `advanceStep()` appears to handle errors per-step. | P3 — acceptable |

---

## Audit 5 — Type Safety

| # | Category | Location | Issue | Severity | Suggested Fix |
|---|----------|----------|-------|----------|---------------|
| 46 | D-0345 | `lib/analytics/attendance.ts` L70,155,265,331 | 4x `.from('attendance_logs')` PostgREST reads. Violates D-0345 (RPCs only). | P1 | Replace with attendance RPCs |
| 47 | D-0345 | `lib/building-state.ts` L153 | `.from('attendance_logs')` PostgREST read | P1 | Replace with RPC |
| 48 | D-0345 | `lib/crons/attendance-sync.ts` L13,24,43 | 3x `.from('attendance_logs')` PostgREST reads | P1 | Replace with RPCs |
| 49 | D-0345 | `app/api/bm/dashboard/route.ts` L109 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 50 | D-0345 | `app/api/bm/reports/route.ts` L56 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 51 | D-0345 | `app/api/bm/reports/snapshot/route.ts` L45 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 52 | D-0345 | `app/api/bm/reports/daily/route.ts` L60 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 53 | D-0345 | `app/api/bm/contractors/[id]/route.ts` L39 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 54 | D-0345 | `app/api/bm/contractors/[id]/staff/[staffId]/attendance/route.ts` L51,128,147 | 3x `.from('attendance_logs')` reads | P1 | Replace with RPCs |
| 55 | D-0345 | `app/api/bm/staff/all/route.ts` L66 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 56 | D-0345 | `app/api/security/dashboard/route.ts` L51 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 57 | D-0345 | `app/api/security/attendance/route.ts` L41 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 58 | D-0345 | `app/api/security/guards/route.ts` L70 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 59 | D-0345 | `app/api/security/operations/route.ts` L43 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 60 | D-0345 | `app/api/cleaning/dashboard/route.ts` L34,46 | 2x `.from('attendance_logs')` reads | P1 | Replace with RPCs |
| 61 | D-0345 | `app/api/cleaning/attendance/route.ts` L35 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 62 | D-0345 | `app/api/cleaning/staff/route.ts` L42 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 63 | D-0345 | `app/api/cleaning/operations/route.ts` L40 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 64 | D-0345 | `app/api/kiosk/verify/route.ts` L105 | `.from('attendance_logs')` read (L142 is insert — acceptable) | P1 | Replace read with RPC |
| 65 | D-0345 | `app/api/kiosk/dashboard/route.ts` L77 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 66 | D-0345 | `app/api/org/route.ts` L51 | `.from('attendance_logs')` read | P1 | Replace with RPC |
| 67 | D-0345 | `app/api/attendance/clock/route.ts` L154 | `.from('attendance_logs')` read (L244 is insert — acceptable) | P2 | Replace read with RPC |
| 68 | FK-ambig | `lib/ai/assignment-engine.ts` L174,197 | `users!inner(full_name)` on `user_roles` — FK ambiguity (user_id + assigned_by both FK to users). PostgREST returns empty silently. | P1 | Change to `users!user_roles_user_id_fkey(full_name)` |
| 69 | FK-ambig | `lib/building-state.ts` L105 | `users!inner(telegram_id)` on `user_roles` — same FK ambiguity | P1 | Change to `users!user_roles_user_id_fkey(telegram_id)` |
| 70 | FK-ambig | `app/api/onboard/committee/route.ts` L19 | `users!inner(id, full_name, phone, ...)` on `user_roles` — same FK ambiguity | P1 | Change to `users!user_roles_user_id_fkey(...)` |
| 71 | FK-ambig | `lib/telegram/group-targets.ts` L62,81,100 | `users!inner(...)` — BUT already uses `users!user_roles_user_id_fkey` per D-0490 fix. VERIFY: grep shows these 3 lines still use `users!inner`. | P1 | Verify and fix if still using ambiguous hint |
| 72 | .single() | 267 files | 524 `.single()` calls across codebase. Many are appropriate (lookup by PK). Needs file-by-file triage for cases where 0-row is valid (should be `.maybeSingle()`). | P2 | Triage top-priority routes: auth, webhooks, public-facing |
| 73 | error-skip | 44+ API routes | `const { data } = await supabaseAdmin...` without destructuring `error`. Supabase errors silently ignored — `data` is null, no 500 returned. | P2 | Add `const { data, error }` + error check for all Supabase calls |
| 74 | build | `npx next build` | 0 type errors. 3 vendor warnings (punycode, text-encoding, critical dependency). 3 dynamic route notices. Build PASSES. | P3 | Vendor warnings are upstream — no action needed |

---

## Audit 6 — UI Consistency

| # | Category | Location | Issue | Severity | Suggested Fix |
|---|----------|----------|-------|----------|---------------|
| 75 | nav-overlap | `components/ui/MinilabShell.tsx` L50 | `isActive = pathname === href || pathname.startsWith(href + '/')` — causes double-highlight for any parent/child nav pairs. Confirmed: `/bm/staff` + `/bm/staff/all`. Likely also affects `/bm/reports` + `/bm/reports/*`, `/bm/settings` + `/bm/settings/*` etc. | P2 | Implement "most specific match wins" logic: if any nav item is an exact match or longer prefix, skip shorter prefix matches |
| 76 | nav-overlap | BM sidebar `/bm/staff` + `/bm/staff/all` | Both highlight when on `/bm/staff/all` — confirmed via code analysis | P2 | Same fix as #75 |
| 77 | nav-overlap | BM sidebar `/bm/settings/*` | All settings sub-pages (`/bm/settings/communications`, `/bm/settings/bank-accounts`, etc.) would highlight the parent "Settings" item too if it exists. | P3 | Verify and apply same fix |
| 78 | empty-state | Multiple API routes | 44+ routes destructure `const { data }` without `error` — on failure, `data` is null and pages show blank/empty state with no error message. | P2 | Propagate errors to UI for user-visible feedback |
| 79 | empty-state | `app/api/forms/[buildingSlug]/[templateSlug]/route.ts` L38-58 | Smart field data queries (`units`, `residents`, `facilities`, `contractors`, `blocks`) all use `const { data }` without checking error. Failed query → empty dropdown silently. | P2 | Add error checks |
| 80 | modals | General | Modal/Dialog components use shadcn/ui which handles `onOpenChange` automatically. No systematic issues found. | - | No action |

---

## Audit 7 — API Route Health

| # | Category | Route File | Issue | Severity | Suggested Fix |
|---|----------|-----------|-------|----------|---------------|
| 81 | no-error-check | `app/api/leave/approve/route.ts` L11,24 | `const { data } = await supabaseAdmin` — error not destructured in 2 helper functions (`buildingHasPmc`, `getOrgForUser`). Error silently ignored. | P2 | Add error handling |
| 82 | no-error-check | `app/api/guard/vehicles/route.ts` L18 | `const { data } = await supabaseAdmin` for plate autocomplete. Error not checked. | P2 | Add error handling |
| 83 | no-error-check | `app/api/forms/[buildingSlug]/[templateSlug]/route.ts` L9,38,42,46,50,58 | 6x `const { data }` without error — building resolve + all smart field queries. Public-facing endpoint. | P2 | Add error handling for public route |
| 84 | no-error-check | `app/api/billing/usage/route.ts` L22 | `const { data }` — credit usage query has no error check. Fails silently to empty array. | P3 | Low risk — returns empty gracefully |
| 85 | no-error-check | `app/api/superadmin/voice-prompt/route.ts` L12 | `const { data }` without error check | P3 | Add error handling |
| 86 | no-error-check | `app/api/developer/floor-plans/route.ts` L48 | `const { data }` without error check | P3 | Add error handling |
| 87 | no-error-check | `app/api/bm/contractors/[id]/route.ts` L38,78,99 | 3x `const { data }` without error in contractor detail GET | P2 | Add error handling |
| 88 | no-error-check | `app/api/cleaning/checklists/route.ts` L33 | `const { data }` without error | P3 | Add error handling |
| 89 | no-error-check | `app/api/auth/resident/request-otp/route.ts` L25 | `const { data }` in resolveBuilding helper — public OTP endpoint. Error silently ignored. | P2 | Public-facing — must handle errors |
| 90 | no-error-check | `app/api/bm/console/timeline/[id]/route.ts` L24,62,92,111,137,157 | 6x `const { data }` without error in timeline route | P2 | Add error handling |
| 91 | no-error-check | `app/api/bm/console/contacts/route.ts` L59 | `const { data }` without error | P2 | Add error handling |
| 92 | empty-catch | `app/api/pmc/reports/route.ts` L158,169 | `catch { /* ignore */ }` — 2 silent catch blocks in report generation | P3 | Log errors even if non-blocking |
| 93 | empty-catch | `app/api/pmc/dashboard/route.ts` L418 | `catch {}` — Redis cache set silently fails | P3 | Log error |
| 94 | empty-catch | `app/api/superadmin/registrations/approve-building/route.ts` L113 | `catch { /* ignore if table doesn't exist */ }` — swallows table access error | P3 | Log error for debugging |
| 95 | no-auth | `app/api/forms/[buildingSlug]/[templateSlug]/route.ts` | Public endpoint (no auth). No rate limiting. Could be used to enumerate building units, residents, facilities. | P2 | Add rate limiting or limit data exposure (e.g., only return unit_number, not full names for residents) |
| 96 | no-auth | `app/api/kiosk/verify/route.ts` | Auth via device_token only (no session). Accepts any image — potential for brute-force face matching. No rate limiting. | P2 | Add per-device rate limiting |
| 97 | D-0345 | `app/api/kiosk/verify/route.ts` L105 | `.from('attendance_logs')` read in a route that also correctly uses `.rpc('find_open_attendance')` at L93. Inconsistent. | P1 | Replace L105 with RPC |
| 98 | .single-risk | `app/api/kiosk/verify/route.ts` L30 | `.single()` on device lookup by token — if token not found, throws PGRST116 instead of returning clean 403. | P2 | Change to `.maybeSingle()` + null check |
| 99 | .single-risk | `app/api/forms/[buildingSlug]/[templateSlug]/route.ts` L73 | `.single()` on form template lookup — if template deleted/inactive, throws PGRST116 instead of clean 404. | P2 | Change to `.maybeSingle()` + null check |
| 100 | .single-risk | `app/api/superadmin/registrations/approve-building/route.ts` L34 | `.single()` on application lookup — if already processed, throws instead of clean error. Already guarded by `fetchError` check though. | P3 | OK — error is caught |

---

## Summary

| Category | P1 | P2 | P3 | Total |
|----------|-----|-----|-----|-------|
| Audit 1 — BM Staff | 1 | 5 | 6 | 12 |
| Audit 2 — Stub Pages | 0 | 2 | 6 | 8 |
| Audit 3 �� Dead Code | 0 | 1 | 2 | 3 (actionable) |
| Audit 4 — God Nodes | 0 | 2 | 3 | 5 |
| Audit 5 — Type Safety | 25 | 3 | 1 | 29 |
| Audit 6 — UI Consistency | 0 | 3 | 1 | 4 |
| Audit 7 — API Health | 1 | 12 | 7 | 20 |
| **TOTAL** | **27** | **28** | **26** | **81** |

### Critical Fixes (P1 — 27 issues)
- **22x D-0345 violations**: `.from('attendance_logs')` PostgREST reads across 14 files. Must migrate to RPCs.
- **3x FK ambiguity**: `users!inner(...)` on `user_roles` (2 FKs to users). Silent empty results.
- **1x Wrong ID**: FaceEnrollModal on All Staff page passes `user_roles.id` instead of `users.id`.
- **1x Mixed violation**: kiosk/verify uses both RPC and PostgREST for attendance_logs in same file.

### Top Priority Recommendations
1. **Batch fix D-0345 violations** — 22 reads in 14 files. Create RPCs for analytics/dashboard use cases.
2. **Fix 3 FK ambiguity bugs** — Change `users!inner(...)` to `users!user_roles_user_id_fkey(...)` in assignment-engine, building-state, onboard/committee.
3. **Fix FaceEnrollModal wrong ID** — `app/bm/staff/all/page.tsx` L802.
4. **Add error handling to 44+ API routes** — Destructure `error` from Supabase calls.
5. **Fix nav double-highlight** — Update MinilabShell.tsx isActive logic.
6. **Delete 3 dead files** — AIFeed.tsx, ConsoleAnalytics.tsx, retry.ts.
7. **Build stub API endpoints** — supplier-pricing, billing-settings (superadmin).
