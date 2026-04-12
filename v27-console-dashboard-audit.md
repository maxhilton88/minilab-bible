# V27 Audit: Console Resident History Card + Dashboard Zero Data

**Date:** 2026-04-12  
**Decision refs:** D-0598 (prior console audit), D-0434, D-0457  
**Auditor:** Claude Code (session-driven)

---

## Files Read

| File | Purpose |
|------|---------|
| `components/bm/console/RightPanel.tsx` | Full right panel component — all sections |
| `components/bm/console/console-shared.tsx` | `UnitDetails` type definition |
| `components/bm/console/Contact360.tsx` | Legacy V4 contact history component |
| `app/bm/console/page.tsx` | Console page entry |
| `app/bm/dashboard/page.tsx` | Dashboard page — data fetching + render |
| `app/api/bm/dashboard/route.ts` | Dashboard API — all queries |
| `lib/rbac/get-building-session.ts` | Session/building resolution |
| `lib/cache.ts` | Redis cache implementation |
| `docs/startup/recent-decisions.md` | D-0434, D-0457, D-0598 decisions |

---

## Issue 1: Console Resident History Card

### Right Panel Section Inventory

The `RightPanel.tsx` `RightPanelInner` component has three rendering branches:

**Branch A — Loading:** Spinner only.

**Branch B — Unlinked Contact** (`isUnlinkedContact && selectedContact`):
| Section | Status |
|---------|--------|
| Sender Info card (name, phone, channel, first contact) | Renders |
| Action buttons (Link to Unit / Mark as External / Create Case) | Renders |

**Branch C — Full Operational Panel** (`details?.property`):
| # | Section | HubSection Label | Condition |
|---|---------|-----------------|-----------|
| 1 | Property | Property | Always |
| 2 | People | People | Always |
| 3 | Tenancy | Tenancy | Always (shows empty if none) |
| 4 | Payments + Collection History | Payments | Always |
| 5 | Cases | Cases | Always |
| 6 | Interfloor Issues | Interfloor | Always |
| 7 | Legal / Tribunal | Legal | Always |
| 8 | Applications | Applications | Always |
| 9 | Vehicles | Vehicles | Always + inline add form |
| 10 | Access Cards | Access Cards | Always |
| 11 | Visitors | Visitors | Always |
| 12 | Bookings | Bookings | Always |
| 13 | Fines | Fines | Always |
| 14 | Documents + Chat Files | Documents | Always |

**Branch D — Case without unit:** Case number + status card only.

**Branch E — Empty:** "Select a contact/case/unit" placeholder.

### Resident History Card Status: **NOT BUILT**

Evidence:
1. No "History", "Timeline", "Resident History", or "Interaction History" HubSection exists anywhere in `RightPanel.tsx`.
2. The `UnitDetails` interface in `console-shared.tsx` has **no `history` field** — there is no data contract for it.
3. `Contact360.tsx` exists (V4-T022) with a `recentHistory` array in its data model, but:
   - It is **not imported** anywhere in `app/bm/console/page.tsx` or `RightPanel.tsx`.
   - Its API endpoint is `/api/bm/console/contact-details` (a V4 endpoint — check if still active).
   - It was designed for the old conversation-centric inbox, not the current unit-centric console.
4. D-0434 (console redesign) explicitly lists right panel as "10 sections: property, people, payments, cases, applications, vehicles, access cards, visitors, fines, documents" — no history section.
5. D-0598 (prior console audit) lists 6 unit-scoped tables with no panel representation (tenancies, collection_transactions, legal_records, parcels, facility_bookings, unit_occupancy_status) — resident interaction history is not even on that list.

**Conclusion:** A "Resident History" card was never designed or built for the V27 console. `Contact360.tsx` is an orphaned V4 component.

---

## Issue 2: BM Dashboard Zero Data

### API Structure

`GET /api/bm/dashboard` → `getBuildingSession(req)` → `fetchDashboardStats(bid)`

**Session resolution** (`lib/rbac/get-building-session.ts`):
- BM/staff: uses `session.buildingId` from cookie — correct, no bug
- Superadmin: uses `session.buildingId ?? x-building-id header` — correct, but if neither set → returns null → 401 → page shows "Failed to load", NOT zeros
- If API returns 401: `dashRes.ok = false` → `setData` not called → `data = null` → "Failed to load dashboard data." message

**Error silencing risk (code bug):**
```typescript
} catch (err) {
  return NextResponse.json({ warning: 'Some dashboard data unavailable' }, { status: 200 });
}
```
If ANY query throws (e.g., bad RPC signature, column rename), the catch returns `{ warning: '...' }` with status 200. The dashboard page calls `setData({ warning: '...' })`. Then `data.stats` is `undefined` → runtime crash on `String(data.stats.openCases)`. This would show a **blank screen or error boundary**, not zeros.

Since the user reports "zeros" (not blank), the API IS succeeding. Zeros come from the data.

### DB Reality Check — Lumi Residency (`497c38b5-271d-4316-9580-93096d70038e`)

| Table | Count | Dashboard Impact |
|-------|-------|-----------------|
| `units` (active) | **0** | No collection_accounts possible; occupancy section hidden |
| `collection_accounts` | **0** | collectionRate=0%, totalOutstanding=RM0 |
| `cases` | 4 | 3 open/in_progress → openCases SHOULD show 3 |
| `maintenance_schedules` | 1 | maintenanceDue SHOULD show 1 |
| `approvals` (pending) | 1 | pendingApprovals SHOULD show 1 |
| `contractor_org_building_assignments` (active) | 3 | activeOrgs SHOULD show 3 |
| `visitors` | 0 | visitorsToday = 0 (correct) |
| `renovation_applications` | 0 | renovations = 0 (correct) |
| `facility_bookings` | 0 | facilityBookingsToday = 0 (correct) |
| `parcels` | 0 | uncollectedParcels = 0 (correct) |
| `legal_records` | 0 | legal deadlines = none (correct) |
| `agm_egm_meetings` | 1 | lastAgm shows correctly |

Case detail:
```
ML-20260403-6150  "Small fire"    status=open      category=security  (2026-04-03)
ML-20260325-1198  "Roof repair"   status=in_progress category=maintenance  (2026-03-25)
ML-20260325-4045  "Water leaking" status=open      category=maintenance  (2026-03-25)
ML-20260325-5026  "Leaking"       status=closed    category=maintenance  (2026-03-25)
```

**Urgent cases** (open > 48 hours): Both 2026-03-25 open cases — `overdueCases` SHOULD show 2.

### Root Cause: No Units Configured

**Primary cause: Lumi Residency has 0 active units in the `units` table.**

This creates a cascading data gap:
1. `collection_accounts` requires units to exist → 0 entries → `collectionRate = 0%`, `totalOutstanding = RM 0`
2. `unit_occupancy_status` is filtered via `unit_ids` → empty input → empty response → `data.occupancy.total = 0` → **entire occupancy section conditionally hidden** (`data.occupancy && data.occupancy.total > 0` guard)
3. Most resident-facing metrics (visitors, renovations, parcels, bookings) are legitimately zero — no data has been recorded

**Secondary issue: Non-zero metrics may be visible but unnoticed**

The following SHOULD be non-zero and should render correctly if the API succeeds:
- Section B: Open Tasks = 3, Pending Approvals = 1, Maintenance Due = 1
- Section D Contractors: activeOrgs = 3
- Urgent action bar: overdueCases = 2, pendingApprovals = 1

If those are also showing as 0, the issue is the session/buildingId not matching Lumi Residency in the logged-in user's cookie — meaning the user accessing the dashboard is scoped to a **different building** with truly no data.

### Attendance Note

The `get_attendance_buildings_today` RPC call returns contractor attendance. If no contractors have clocked in today, `attendanceToday = 0` — correct behavior, not a bug.

---

## Recommended Fixes — Priority Ordered

### P0 — Critical: Suppress dashboard crash on partial API failure
- **File:** `app/bm/dashboard/page.tsx` line 297
- **Problem:** If the API returns `{ warning: '...' }` (status 200 but non-DashboardData shape), `setData` accepts it and the render crashes
- **Fix:** Guard: `if (dashRes.ok && data?.stats) setData(data)`, otherwise show "Some data unavailable" toast without crashing

### P1 — Data setup: Lumi Residency needs units
- **Action:** Add units to Lumi Residency building (blocks + unit numbers via `/bm/settings/blocks`)
- **Impact:** Unlocks collection_accounts, occupancy tracking, and all resident-facing features
- This is Seeteng's primary onboarding blocker

### P2 — Build Resident History card (if wanted)
- No code exists. Would require:
  1. New `UnitDetails.history` field (array of `{ type, content, channel, created_at }`)
  2. New query in `/api/bm/console/unit-details` joining `messages`, `cases`, `collection_transactions` by `unit_id`
  3. New HubSection in `RightPanel.tsx` — "Activity History" or "Timeline"
- Recommend using D-0619+ for this feature

### P3 — Clean up Contact360.tsx (orphaned V4 component)
- `components/bm/console/Contact360.tsx` is unused dead code
- Check if `/api/bm/console/contact-details` still has a route; if so, both can be deleted
- Low urgency, but adds confusion to future audits

---

## Summary Table

| Issue | Status | Root Cause |
|-------|--------|-----------|
| Resident History card missing | **NOT BUILT** | Was never designed for V27 console; `Contact360.tsx` is orphaned V4 code |
| Dashboard shows zero data | **Data gap** | Lumi Residency has 0 units configured — collection, occupancy, resident metrics all zero by design |
| Dashboard crash risk | **Code bug** | Silenced catch returns `{ warning }` with 200; client `setData` crashes on render |
