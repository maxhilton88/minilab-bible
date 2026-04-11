# Console Unit Right Panel — Data Completeness Audit
_Date: 2026-04-12 · Decision reference: D-0598_

---

## Overview

The Console Unit right panel lives in a single component (`components/bm/console/RightPanel.tsx`) and is fed by one primary API endpoint (`/api/bm/console/unit-details`). All 10 sections are rendered from the `UnitDetails` object returned by that endpoint.

---

## Section: Property

**Component:** `components/bm/console/RightPanel.tsx:214–233`
**API:** `GET /api/bm/console/unit-details`
**Table(s):** `units`, `blocks`

**Fields displayed:**
- `unit_number`
- `block_name` (resolved via secondary query to `blocks`)
- `unit_type`
- `size_sqft`
- `occupancy_status` (derived: Vacant / Occupied / Tenanted based on residents count + role)

**Fields in DB but NOT displayed:**
- `floor` — exists in `units`, not shown
- `strata_title_reference` — exists in `units`, not shown
- `share_units` — exists in `units`, not shown
- `is_active` — `units.is_active`, not shown

**+ button:** Missing — HubSection count is hardcoded to `1`, no `onAdd` prop passed.

**Issues:**
1. `floor` is not displayed anywhere on the panel despite being a key property attribute
2. `strata_title_reference` (strata title/parcel lot number) is hidden
3. No way to edit property details (floor, type, size) from the console — would need a slide-over
4. Occupancy status is derived on the fly, ignores `unit_occupancy_status` table

**Recommendations:**
- Show `floor` and `strata_title_reference` in the property card
- Add an edit slide-over for property metadata (floor, unit_type, size_sqft)

---

## Section: People

**Component:** `components/bm/console/RightPanel.tsx:236–304`
**API:** `GET /api/bm/console/unit-details` (read), `POST/PATCH /api/bm/residents` (write)
**Table(s):** `residents`, `users` (telegram check)

**Fields displayed:**
- `full_name`
- `role` (badge)
- `phone`
- Channel icons: `has_whatsapp` (derived: `!!phone`), `has_telegram` (via `users.telegram_id` check), email icon

**Fields in DB but NOT displayed:**
- `ic_number` — resident IC/passport, not shown
- `move_in_date` — not shown
- `move_out_date` — not shown
- `pdpa_consent` / `pdpa_consent_at` — not shown

**+ button:** Works → opens `resident` slide-over

**Resident slide-over fields collected:** name, phone, email, role (owner/tenant/family/occupant)

**Missing from form:**
- `ic_number` — not collected
- `move_in_date` — not collected
- `move_out_date` — not collected

**Issues:**
1. No "move out" flow — the X button hard-removes the resident (`is_active=false`) with no move_out_date recorded
2. No "transfer ownership" flow (owner → tenant or vice versa)
3. `ic_number` never collected through the console — only available via PWA onboarding
4. Email icon shown even when email is null (shows icon if `p.email` is truthy — correct, but worth auditing)
5. The `tenancies` table (`unit_id, tenant_resident_id, owner_resident_id, monthly_rent, start_date, end_date`) is entirely ignored — tenancy relationships and rental amounts not surfaced anywhere

**Recommendations:**
- Add `ic_number` field to the resident form
- Add "Move Out" action that sets `is_active=false`, records `move_out_date`, and optionally creates a `tenancies` end record
- Show `tenancies` data as a sub-section under People or as a separate "Tenancy" section

---

## Section: Payments

**Component:** `components/bm/console/RightPanel.tsx:307–322`
**API:** `GET /api/bm/console/unit-details`
**Table(s):** `collection_accounts`, `approvals`

**Fields displayed:**
- `outstanding_balance` (as "Outstanding" in red/green)
- `pending_approvals` count (as "Pending")

**Fields in DB but NOT displayed:**
- `months_overdue` — not shown
- `escalation_stage` — not shown
- `access_blocked` / `access_blocked_at` — access blocked status not shown
- `form11_served_at` — not shown
- `last_payment_amount` — not shown
- `last_payment_date` — not shown
- Full transaction history (`collection_transactions`) — never fetched

**+ button:** Missing — no `onAdd` prop. Cannot create a payment record from the panel.

**Issues:**
1. **BUG (data correctness):** The `approvalsCountRes` query at `unit-details/route.ts:107–110` filters only by `building_id`, NOT by `unit_id`. The "Pending" count shows building-wide pending approvals, not approvals for this specific unit.
2. Transaction history (`collection_transactions`) is never fetched — users cannot see payment history from the panel
3. `months_overdue` and `access_blocked` are operationally important but hidden
4. `last_payment_date` / `last_payment_amount` is fetched from `collection_accounts` but never returned in the JSON response

**Recommendations:**
- Fix approvals count to filter by `unit_id` (or remove if not per-unit)
- Show `last_payment_date`, `last_payment_amount`, `months_overdue`
- Show `access_blocked` warning badge if `access_blocked=true`
- Add a "View Transactions" link or expandable list pulling from `collection_transactions`

---

## Section: Cases

**Component:** `components/bm/console/RightPanel.tsx:325–348`
**API:** `GET /api/bm/console/unit-details` (read), `POST /api/bm/cases` (write)
**Table(s):** `cases`, `users` (for assignee names)

**Fields displayed:**
- `title`
- `status` (badge)
- `case_number`
- `assignee` (resolved from `users.full_name`)

**Fields in DB but NOT displayed:**
- `description` — not shown
- `category` — not shown in the list row
- `is_urgent` — not shown
- `priority` — not shown (exists in data, not rendered)
- `sla_due_at` — not shown
- `resolved_at` — not shown
- `assigned_contractor_org_id` — not shown

**+ button:** Works → opens `case` slide-over

**Case slide-over fields collected:** title, category, priority, assigned_to (staff only), description

**Missing from form:**
- `is_urgent` toggle — not available
- `sla_due_at` — not available
- Contractor assignment (`assigned_contractor_org_id`) — not available

**Issues:**
1. **No interfloor leak category** — categories available are: `resident_complaint`, `maintenance_request`, `internal_work`, `inspection`, `follow_up`, `other`. No dedicated `interfloor_leak` option. Cases schema uses a `case_category_type` enum that may or may not include it — BM would need to use `maintenance_request` + describe in title.
2. **No cross-unit linking** — `related_unit_id` does NOT exist in the `cases` table schema. Cross-unit case linking is not supported at the DB level.
3. Cases list capped at 10 most recent — no "show all" link
4. `priority` is in the data object (`c.priority`) but never rendered in the UI row
5. No click-through to open the full case detail from the panel

**Recommendations:**
- Add `interfloor_leak` to case category enum (DB migration needed)
- Render `priority` badge in case list row
- Add "show all" link for units with > 10 cases
- Consider adding `related_unit_id` to cases schema for cross-unit incidents

---

## Section: Applications

**Component:** `components/bm/console/RightPanel.tsx:350–369`
**API:** `GET /api/bm/console/unit-details` (read), `POST /api/bm/renovation` (write)
**Table(s):** `renovation_applications` **ONLY** — `form_templates` is completely ignored

**Fields displayed:**
- `type` (hardcoded string "Renovation" for all)
- `reference` (synthetic: `REN-{id.slice(0,6).toUpperCase()}`, not a DB column)
- `status` (badge)

**Fields in DB but NOT displayed:**
- `scope_of_work` (description) — not shown
- `start_date` / `end_date` — not shown
- `deposit_amount` / `deposit_paid` — not shown
- `contractor_org_id` — not shown
- `approved_by` — not shown
- `inspection_notes` — not shown

**+ button:** Works → opens `renovation` slide-over (hardcoded — does NOT show a dropdown of application types)

**Renovation slide-over fields collected:** description (scope_of_work), start_date, end_date, contractor_name

**Missing from form:**
- Application type dropdown from `form_templates` — absent entirely
- `contractor_org_id` (linked contractor org) — form has a plain text field for contractor name but it maps to nothing
- `deposit_amount` — not collected

**Issues:**
1. **CRITICAL: Not using form_templates.** The `form_templates` table contains a configurable Smart Forms engine with 10+ predefined application types per building (pet registration, move-in/move-out, renovation, parking, etc.). The console panel ignores all of this and hardcodes to renovation only.
2. **BUG: Contractor name is collected but silently dropped.** `renoContractor` state is populated from the form (`page.tsx:1711`) but the API call at `page.tsx:1293–1305` does NOT include it in the JSON payload. It is never saved to the database.
3. `renovation_applications` has no `reference_number` column — the UI fabricates one from the UUID prefix, which is misleading.
4. `SlideOverType` union type only includes `'renovation'` — there is no extension point for other application types.

**Recommendations:**
- Replace hardcoded renovation type with a dynamic dropdown fetched from `form_templates WHERE building_id = X AND is_active = true`
- Fix contractor name to either map to `contractor_org_id` (org select) or add a `contractor_name` text column to `renovation_applications`
- Show `start_date`, `end_date`, `status` in the applications list rows

---

## Section: Vehicles

**Component:** `components/bm/console/RightPanel.tsx:371–415`
**API:** `GET /api/bm/console/unit-details` (read), `POST /api/bm/console/add-vehicle` (write)
**Table(s):** `vehicle_logs` (filtered: `is_visitor_vehicle = false`)

**Fields displayed:**
- `plate_number`
- `created_at` (relative time only)

**Fields in DB but NOT displayed:**
- `resident_id` — which resident owns this vehicle (owner/tenant/family) — not shown
- `entry_time` / `exit_time` — latest entry/exit times — not shown
- `logged_by` — who registered it — not shown
- `gate_id` — gate used — not shown

**+ button:** Works → inline form (not slide-over)

**Vehicle inline form fields collected:** `plate_number` only

**Missing from form:**
- `resident_id` — no owner linkage (vehicle added to unit with no resident association)

**Issues:**
1. **No resident ownership linkage** — when a vehicle is registered from the console, it is linked to the `unit_id` but NOT to any `resident_id`. This means you cannot tell which resident (owner/tenant/family) owns which car.
2. No delete/deregister functionality — once added, a vehicle cannot be removed from the panel
3. `vehicle_logs` is a movement log table (entry/exit), not a vehicle registry. Using it as a registry is a misuse — a dedicated `unit_vehicles` table (or a boolean `is_registered` column) would be semantically cleaner.
4. No make/model/colour captured (not in DB schema either — would require schema change)
5. No display of which resident each vehicle belongs to

**Recommendations:**
- Add `resident_id` field to vehicle add form (dropdown of unit's residents)
- Add delete/deregister action per vehicle
- Show resident name next to plate in the list

---

## Section: Access Cards

**Component:** `components/bm/console/RightPanel.tsx:418–467`
**API:** `GET /api/bm/console/unit-details` (read), `PATCH /api/bm/console/access-cards` (write)
**Table(s):** `access_cards`

**DB schema:** `id, building_id, unit_id, resident_id, card_number, status, issued_date, deactivated_at, notes, created_at, updated_at`
**Status enum:** active / deactivated / lost / returned

**Fields displayed:**
- `card_number`
- `status` (badge)
- `resident_name` (resolved from `residents` join in route)
- `issued_date`

**Fields in DB but NOT displayed:**
- `notes` — card notes not shown
- `deactivated_at` — in the TypeScript type but not rendered in UI
- `lost` / `returned` status — only `active` and `deactivated` states handled by the API

**+ button:** Missing — no `onAdd` prop on the Access Cards `HubSection`. Cannot issue a new card from the panel.

**Issues:**
1. **No card issuance** — there is no way to add a new access card from the console panel. The `+` button is absent.
2. **Only deactivate supported** — the API (`access-cards/route.ts`) supports `deactivate` and `activate` actions but does NOT support `lost` or `returned` status transitions.
3. `deactivated_at` is in the TypeScript type but never rendered in the UI row
4. Card `notes` are stored but never displayed

**Recommendations:**
- Add `+` button → slide-over to issue a new card (fields: card_number, resident, issued_date, notes)
- Add "Mark as Lost" and "Mark as Returned" actions alongside Deactivate
- Show `notes` and `deactivated_at` in the card row

---

## Section: Visitors

**Component:** `components/bm/console/RightPanel.tsx:469–488`
**API:** `GET /api/bm/console/unit-details` (read), `POST /api/guard/visitors` (write)
**Table(s):** `visitors`

**Fields displayed:**
- `visitor_name`
- `visit_purpose` (as "type")
- `created_at` (relative time)

**Fields in DB but NOT displayed:**
- `visitor_phone` — not shown
- `visitor_ic` — not shown
- `vehicle_plate` — not shown
- `check_in_at` / `check_out_at` — not shown (checked_in/checked_out in type but not rendered)
- `photo_url` / `ic_photo_url` / `license_photo_url` — not shown
- `status` (active/checked_in/expired/cancelled) — not shown
- `visit_date` — not shown (using `created_at` instead)
- `is_preregistered` — not shown
- `gate_id` — not shown

**+ button:** Works → opens `visitor` slide-over

**Visitor slide-over fields collected:** name, phone, vehicle_plate, purpose

**Missing from form:**
- `visitor_ic` — not collected
- Photo capture — not available

**Issues:**
1. Check-in/check-out status not visible in the panel — BM cannot tell from the Console whether a visitor has arrived
2. `visit_date` should be shown instead of/alongside `created_at`
3. Only 5 most recent visitors shown, no "show all" link
4. Visitor status (`expired`, `cancelled`, `checked_in`) is tracked in DB but completely hidden from BM view

**Recommendations:**
- Show check-in status indicator (✓ / pending) per visitor row
- Show `visitor_phone` and `vehicle_plate` in row
- Use `visit_date` column for display where set

---

## Section: Fines

**Component:** `components/bm/console/RightPanel.tsx:490–512`
**API:** `GET /api/bm/console/unit-details` (read), `POST /api/app/fines` (write)
**Table(s):** `vehicle_fines`

**Fields displayed:**
- `violation_type` (as "description")
- `status` (badge)
- `fine_number`
- `vehicle_plate`
- `fine_amount` (as "RM X")

**Fields in DB but NOT displayed:**
- `vehicle_description` — not shown
- `location` — not shown
- `photos` — not shown
- `is_clamped` / `clamped_at` — not shown
- `payment_method` / `paid_at` — not shown
- `notes` — not shown
- `created_at` — not shown

**+ button:** Works → opens `fine` slide-over

**Fine slide-over fields collected:** violation_type (from fine_settings), amount, notes

**Missing from form:**
- `vehicle_plate` — **CRITICAL BUG:** hardcoded to `'N/A'` at `page.tsx:1332` when creating from unit context — `vehicle_plate` is a required field in `vehicle_fines` but the console form doesn't prompt for it
- `location` — not collected

**Issues:**
1. **BUG:** `vehicle_plate` is hardcoded to `'N/A'` in the fine creation payload. The `vehicle_fines.vehicle_plate` column is `NOT NULL` — this is a data quality issue. Should present a dropdown of unit's registered vehicles, or a free-text field.
2. `violation_type` stored in DB as snake_case enum (`illegal_parking`, `double_parking`, etc.) but displayed raw in the panel — should be formatted for display
3. `is_clamped` status not shown — BM cannot see clamp status from console
4. Only 5 most recent fines shown

**Recommendations:**
- Fix vehicle_plate to either default to unit's registered vehicles (dropdown) or provide a free-text field
- Show `is_clamped` badge and `location` in the fine row
- Display `violation_type` in a human-readable format (replace underscores)

---

## Section: Documents

**Component:** `components/bm/console/RightPanel.tsx:514–572`
**API:** `GET /api/bm/console/unit-details` (read), `POST /api/bm/documents` (write)
**Table(s):** `building_documents` (building-level), `messages` (chat-uploaded media)

**Fields displayed (building_documents):**
- `title`
- `category`
- `file_size` (in KB)
- `created_at` (relative)
- Link to `file_url`

**Fields in DB but NOT displayed:**
- `description` — not shown
- `file_name` — not shown
- `expires_at` — expiry date not shown (important for insurance/cert docs)
- `is_public` — not shown
- `tags` — not shown
- `uploaded_by` — not shown

**+ button:** Works → opens `document` slide-over

**Document slide-over fields collected:** file (PDF/TXT/MD only), title (docName), category (docCategory)

**Issues:**
1. **CRITICAL: Documents are building-level, not unit-scoped.** `building_documents` has no `unit_id` column. Every unit in the building shows the same document list. This is almost certainly not the intended behaviour — a unit's S&P agreement or tenancy agreement is unit-specific data, not building-wide.
2. **BUG: Form fields not sent to API.** The upload form at `page.tsx:1343–1352` only appends `file` to FormData. The `docName` (title) and `docCategory` fields are collected in the UI but silently dropped and never sent to `/api/bm/documents`.
3. Upload accepts only PDF/TXT/MD. No support for images (JPG/PNG for hand-signed docs etc.).
4. The "From Chat" sub-section correctly shows chat-uploaded media files (messages with `media_url`) which provides a useful unit-scoped document trail.
5. Document categories in the UI (SP Agreement, Tenancy Agreement, Renovation Permit, Insurance, Other) do not match the `building_documents` category enum (house_rules, bylaws, spa, strata_title, etc.) — likely causing data integrity issues.

**Recommendations:**
- Fix the upload FormData to include `title`, `category` fields
- Create a `unit_documents` table (or add `unit_id` nullable FK to `building_documents`) for unit-scoped documents
- Separate "Building Documents" (policies, house rules) from "Unit Documents" (S&P, tenancy) in the panel
- Show `expires_at` for documents where set

---

## Unit-scoped Tables with NO Representation on the Panel

The following tables have a `unit_id` column (or can be linked to a unit) but are **entirely absent** from the Console right panel:

| Table | unit_id link | What it contains | Priority |
|-------|-------------|-----------------|----------|
| `tenancies` | direct `unit_id` | Tenant–owner rental agreements, monthly rent, start/end dates | HIGH — critical for property management |
| `collection_transactions` | via `collection_accounts.unit_id` | Full payment/charge transaction history | HIGH — Payments section only shows balance |
| `legal_records` | optional `unit_id` | Tribunal cases, STR filings, legal disputes | HIGH — tribunal records are unit-scoped |
| `parcels` | direct `unit_id` | Parcel/courier deliveries logged for unit | MEDIUM |
| `facility_bookings` | via `resident_id` → `residents.unit_id` | Facility reservations by unit residents | LOW |
| `unit_occupancy_status` | direct `unit_id` | Formal occupancy status history | LOW — currently derived from residents |

---

## Summary of Critical Bugs

| # | Section | Bug | Severity |
|---|---------|-----|----------|
| 1 | Applications | Hardcoded to Renovation only — form_templates completely ignored | HIGH |
| 2 | Applications | `renoContractor` collected in UI but never sent to API — silently dropped | HIGH |
| 3 | Documents | Fetches building-level docs (no unit_id) — all units show same docs | HIGH |
| 4 | Documents | `docName` and `docCategory` not included in upload FormData — dropped | HIGH |
| 5 | Payments | Pending approvals count is building-wide, not unit-specific | MEDIUM |
| 6 | Fines | `vehicle_plate` hardcoded to `'N/A'` in fine creation payload | MEDIUM |
| 7 | Access Cards | No `+` button — cannot issue new access cards from the panel | MEDIUM |
| 8 | Vehicles | No resident linkage when registering vehicle — no way to track ownership | MEDIUM |
| 9 | People | No "move out" flow — hard removes resident with no date recorded | MEDIUM |

---

## Summary of Missing Data Not in DB (Schema Gaps)

| Gap | Affected Section | DB Change Needed |
|-----|-----------------|------------------|
| Interfloor leak case category | Cases | Add `interfloor_leak` to `case_category_type` enum |
| Cross-unit case linking | Cases | Add `related_unit_id` (uuid FK→units) to `cases` table |
| Vehicle ownership | Vehicles | `vehicle_logs.resident_id` exists but not used by add-vehicle route |
| Unit-scoped documents | Documents | Add `unit_id` (nullable) to `building_documents`, or create `unit_documents` |
| Contractor name on renovation | Applications | Add `contractor_name` (text) to `renovation_applications` or enforce `contractor_org_id` |
