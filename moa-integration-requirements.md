# MOA Integration Requirements Audit
**Date:** 2026-04-12 (V27)
**Purpose:** Map what Minilab needs from each legacy system, what vendor provides, and define MOA skill inventory for build planning.
**Status:** Analysis only. No code or schema changes in this session.

---

## Context

MOA-SPEC §10 lists three pending recon items:
- Access card (CHV): **vendor unknown** — need name, portal URL, credentials, workflows from Hoe Zee How
- Car plate (CHV): **vendor unknown** — need name, portal URL, credentials, workflows from Hoe Zee How
- Advelsoft workflows: **connection type proven** (Type 2 cloud RDP tested 2026-04-12) — need Hoe Zee How to confirm which reports to automate first

Access card and car plate skills cannot be built until recon completes. Advelsoft skills can proceed now.

---

## Table 1: Access Card System

**Minilab `access_cards` table** (Migration 070): id, building_id, unit_id, resident_id, card_number, status (active/deactivated/lost/returned), issued_date, deactivated_at, notes, created_at, updated_at.

**Critical gap:** No `access_card_logs` table exists. The access_cards table stores card metadata only — it has zero record of when cards were swiped, at which door, or whether entry was granted or denied. Every action involving entry/exit events requires a new table.

| # | Skill Name | Direction | Minilab Table / Field | What Minilab Needs | What Vendor Provides | MOA Skill Type | Priority |
|---|------------|-----------|----------------------|-------------------|---------------------|----------------|----------|
| 1 | `block_unit_access_cards` | WRITE (Minilab → vendor) | `collection_accounts.access_blocked = true` triggers MOA | When BM clicks "Block Access" in collection module, immediately disable ALL cards for that unit in the vendor system | Per-unit bulk card disable via web portal | write (no retry) | **P0** |
| 2 | `restore_unit_access_cards` | WRITE (Minilab → vendor) | `collection_accounts.access_blocked = false` triggers MOA | When payment clears and BM restores access, re-enable all cards for that unit | Per-unit bulk card enable via web portal | write (no retry) | **P0** |
| 3 | `deactivate_single_card` | WRITE (Minilab → vendor) | `access_cards.status = 'deactivated'` or `'lost'` | When resident moves out or reports card lost, immediately block that specific card number | Single card deactivate by card number | write (no retry) | **P0** |
| 4 | `issue_access_card` | WRITE (Minilab → vendor) | `access_cards` (new row insert) | When BM issues a new card and records it in Minilab, register it in the vendor system and grant the correct access levels for that unit type | Card programming: assign card number → unit, grant door access zones | write (no retry) | **P1** |
| 5 | `pull_swipe_logs` | READ (vendor → Minilab) | **NEW TABLE NEEDED:** `access_card_logs` | Hourly import of who swiped which door when — for security audit, breach detection, and collection evidence (prove access was actively used) | Entry/exit event log: card_number, door_name, event_type (granted/denied), timestamp | sync (hourly cron) | **P1** |
| 6 | `sync_cardholder_list` | SYNC (bidirectional) | `access_cards` ↔ vendor cardholder DB | Reconcile Minilab's card records against vendor's live cardholders — catch manual additions at vendor side, detect orphan cards for units with no active residents | Full cardholder export: card_number, unit_ref, holder_name, access_zones, status | sync (daily cron) | **P1** |
| 7 | `issue_visitor_temp_card` | WRITE (Minilab → vendor) | `visitors` (has qr_code field, no card integration) | When a visitor pre-registers for a specific date/time window, issue a temporary access code or card with that window baked in — no guard needed | Temporary card or PIN code registration with datetime expiry | write (no retry) | **P2** |
| 8 | `pull_contractor_access_events` | READ (vendor → Minilab) | **NEW TABLE:** `access_card_logs` with is_contractor flag | Know when contractor staff (whose cards are also registered) accessed the building — cross-check against approved `service_visits` schedule | Filtered event log by card group or access zone | read_only (daily cron) | **P2** |

### Access Card — New Tables/Columns Required

| Item | Type | Columns | Reason |
|------|------|---------|--------|
| `access_card_logs` | New table | id, building_id, access_card_id (FK), card_number, gate_id (FK→building_gates), door_name, event_type (granted/denied/unknown), event_at, vendor_event_id, is_contractor | No event log exists anywhere in Minilab |
| `access_cards.vendor_card_id` | New column | TEXT nullable | Map Minilab card row → vendor's internal card ID |
| `access_cards.vendor_synced_at` | New column | TIMESTAMPTZ nullable | Last time this card was confirmed synced with vendor |
| `access_cards.access_zones` | New column | TEXT[] default '{}' | Which doors/zones the card can access (varies by unit type, floor) |

---

## Table 2: Car Plate Recognition System

**Minilab `vehicle_logs` table**: id, building_id, plate_number, unit_id, resident_id, is_visitor_vehicle, entry_time, exit_time, logged_by (guard user), gate_id, relationship (owner/tenant/family), created_at.

**Critical gap:** `vehicle_logs` is a **manually-entered guard table** — `logged_by` is a guard who types the plate. An LPR system would auto-capture plates from cameras, check them against a vendor-side whitelist, and open the barrier gate automatically. Minilab currently has no plate registration workflow (no UI for residents to register plates), no LPR event source, and no parking allocation table.

`building_spaces` has `space_type = 'parking'` but stores physical space metadata (location, area_sqft) — it is not a plate→lot allocation table.

| # | Skill Name | Direction | Minilab Table / Field | What Minilab Needs | What Vendor Provides | MOA Skill Type | Priority |
|---|------------|-----------|----------------------|-------------------|---------------------|----------------|----------|
| 1 | `register_resident_plate` | WRITE (Minilab → vendor) | `vehicle_logs` (resident plate row with unit_id, relationship) | When resident registers a plate in Minilab, push it to the vendor's whitelist so the barrier opens automatically for that plate | Plate registration: plate_number → unit, holder name, valid-from date | write (no retry) | **P0** |
| 2 | `deregister_resident_plate` | WRITE (Minilab → vendor) | `vehicle_logs` (on soft-delete or resident move-out) | Remove plate from vendor whitelist when resident vacates or sells vehicle — prevents unauthorized access post move-out | Plate deletion or deactivation | write (no retry) | **P0** |
| 3 | `block_unit_plates` | WRITE (Minilab → vendor) | `collection_accounts.access_blocked = true` | When unit access is blocked for arrears, also block ALL registered plates for that unit from barrier entry — the physical counterpart to access card blocking | Per-unit bulk plate disable | write (no retry) | **P0** |
| 4 | `restore_unit_plates` | WRITE (Minilab → vendor) | `collection_accounts.access_blocked = false` | When payment clears and access is restored, re-enable all registered plates for that unit | Per-unit bulk plate enable | write (no retry) | **P0** |
| 5 | `pull_lpr_entry_exit_logs` | READ (vendor → Minilab) | `vehicle_logs` — needs new `source`, `confidence_score`, `snapshot_url`, `lpr_event_id` columns | Replace manual guard entry with automatic LPR event import: who entered/exited when, at which gate, confidence, camera snapshot. Vehicle_logs then becomes authoritative instead of a guard diary | Event stream: plate_number, gate_id, entry/exit, timestamp, confidence_pct, snapshot_image_url | sync (every 15 min cron, or webhook if vendor supports) | **P0** |
| 6 | `register_visitor_plate` | WRITE (Minilab → vendor) | `visitors.vehicle_plate` | When visitor pre-registers with a vehicle plate, add it to the vendor's temporary whitelist for the visit date/time window so barrier opens automatically | Temporary plate registration with datetime expiry | write (no retry) | **P1** |
| 7 | `remove_visitor_plate` | WRITE (Minilab → vendor) | `visitors.check_out_at` → trigger MOA | When visitor checks out (or visit date expires), remove their plate from the whitelist | Plate removal or expiry | write (no retry) | **P1** |
| 8 | `open_barrier_gate_remote` | WRITE (Minilab → vendor) | `moa_commands` (BM or guard triggers on-demand) | Guard can press "Open Gate" button in Minilab PWA for expected deliveries, visitor vehicles without pre-registration, or override events | Manual gate open command via vendor portal | write (dangerous — physical action, no retry) | **P1** |
| 9 | `sync_plate_whitelist` | SYNC (bidirectional) | `vehicle_logs` (resident plates) ↔ vendor whitelist | Daily reconciliation: detect plates in vendor system not in Minilab (manual additions by building admin), detect Minilab plates not in vendor (sync failures), detect status mismatches (blocked in one but not other) | Full registered plates export with status | sync (daily cron) | **P1** |
| 10 | `register_contractor_plate` | WRITE (Minilab → vendor) | **NEW COLUMN NEEDED:** `contractor_staff.vehicle_plate` | Contractor vehicles with approved active `service_visits` can access the building — register their plate for the project duration | Contractor plate group registration with project start/end dates | write (no retry) | **P2** |
| 11 | `pull_parking_occupancy` | READ (vendor → Minilab) | **NEW TABLE NEEDED:** `parking_allocations` | Know which physical car park lots are currently occupied, detect unauthorized vehicles in reserved lots (cross-check against `vehicle_fines` for double parking etc.) | Live occupancy per lot (requires per-lot LPR cameras at entries — only available if vendor tracks lot-level, not just gate-level) | read_only (hourly or on-demand) | **P2** |

### Car Plate — New Tables/Columns Required

| Item | Type | Columns | Reason |
|------|------|---------|--------|
| `vehicle_logs.source` | New column | TEXT DEFAULT 'manual' CHECK (manual/lpr) | Distinguish manual guard entries from LPR auto-events |
| `vehicle_logs.confidence_score` | New column | NUMERIC(5,2) nullable | LPR OCR confidence — needed to flag low-confidence reads for guard review |
| `vehicle_logs.snapshot_url` | New column | TEXT nullable | R2 URL of LPR camera snapshot at time of capture |
| `vehicle_logs.lpr_event_id` | New column | TEXT nullable | Vendor-side event ID for deduplication on re-imports |
| `contractor_staff.vehicle_plate` | New column | TEXT nullable | Needed before contractor plates can be registered with vendor |
| `parking_allocations` | New table | id, building_id, unit_id (FK), space_id (FK→building_spaces), plate_number, allocation_type (resident/visitor/contractor), is_active, valid_from, valid_until | No plate→parking lot mapping exists anywhere in Minilab |

---

## Table 3: Accounting System (Advelsoft + CSS + Condo Master)

**Minilab collection architecture:** `collection_accounts` (outstanding_balance, months_overdue, escalation_stage, access_blocked, last_payment_amount/date) + `collection_transactions` (charge/payment/waiver/penalty/adjustment) + `collection_settings` (monthly_maintenance_fee, sinking_fund_fee, thresholds) + `building_payment_settings` (payment modes, Billplz config).

**Critical tension:** Minilab's `collection_accounts.outstanding_balance` is **Minilab-computed** from its own transaction log. Advelsoft holds the **authoritative** financial records. For CHV launch, both will exist in parallel — Minilab's collection module is the operational layer (collection chase, escalaion, access blocking), but Advelsoft is the accounting system of record. The integration must bridge them without duplicating financial authority.

**Advelsoft Report Menu** (confirmed via §9): Statement · Reminders · Customer Transactions · Ageing Reports · Billing · Billing and Collection · Collection · Monthly Balance Analysis · Outstanding Balance Analysis · Credit Review · Deposits · Interest Charges Estimation · GL Listing · Audit List
**Output method:** Print → Universal Printer → browser PDF download (Type 2 RDP, proven ✅)
**Other modules:** AP7 (Accounts Payable) · CB7 (Cash Book) · GL7 (General Ledger) · TX7 (Tax)

| # | Skill Name | Direction | Minilab Table / Field | What Minilab Needs | What Vendor Provides | MOA Skill Type | Priority |
|---|------------|-----------|----------------------|-------------------|---------------------|----------------|----------|
| 1 | `pull_unit_account_mapping` | READ (vendor → Minilab) | **NEW COLUMN NEEDED:** `units.vendor_account_number` | **Prerequisite for all other skills.** Map Minilab unit IDs to Advelsoft's internal account/unit numbers. Without this mapping, no other skill can target the right unit. One-time setup, then reconcile monthly for new units | Full resident/unit listing with Advelsoft account numbers | read_only (one-time, then monthly) | **P0** |
| 2 | `pull_outstanding_balance_all_units` | READ (vendor → Minilab) | `collection_accounts` — needs `vendor_balance`, `vendor_last_synced_at` columns | Sync authoritative outstanding balance from Advelsoft into Minilab for BM dashboard display and resident portal "Your account" card. This IS the number residents and BMs see — Minilab-computed balance is an approximation | Outstanding Balance Analysis report — all units for a building | sync (daily cron minimum; 6-hourly for live feel) | **P0** |
| 3 | `pull_ageing_report` | READ (vendor → Minilab) | **NEW TABLE NEEDED:** `vendor_ageing_snapshots` | BM dashboard Ageing widget: how many units are current / 1-30 / 31-60 / 61-90 / 90+ days. Currently Minilab shows this from its own `months_overdue` (coarse-grained). Advelsoft has the precise ageing buckets | Ageing Reports → PDF download (parse bucket totals from PDF, or DATA output) | sync (daily cron) | **P0** |
| 4 | `export_unit_billing_statement` | READ (vendor → Minilab) | **NEW TABLE NEEDED:** `vendor_billing_statements` (unit_id, period, pdf_url, total_due, vendor_account_number, fetched_at) | When a resident views their account in the resident portal, generate/fetch their billing statement PDF and display it inline or send via WhatsApp. Currently the resident portal shows Minilab's computed balance only | Statement report for a specific unit + period → PDF via Universal Printer → R2 upload | read_only (on-demand when resident views, or monthly pre-generate for all units) | **P0** |
| 5 | `reconcile_collection_accounts` | SYNC (reconciliation) | `collection_accounts` (outstanding_balance vs vendor_balance) | Daily job that flags discrepancies between Minilab's collection_accounts.outstanding_balance and Advelsoft's authoritative balance — catches units that paid Advelsoft directly (bank transfer), units with credits/adjustments not in Minilab, data entry errors | Outstanding Balance Analysis for full building | sync (daily cron, alert BM on discrepancy) | **P0** |
| 6 | `pull_unit_transaction_history` | READ (vendor → Minilab) | `collection_transactions` — add `source TEXT DEFAULT 'minilab'` column (or new `vendor_transactions` table) | Full transaction history per unit (charges, payments, penalties, adjustments, interest) for resident portal statement view and collection escalation context. Minilab's own collection_transactions only has what was recorded in Minilab — misses direct Advelsoft entries | Customer Transactions report for a specific unit + date range | read_only (on-demand when resident/BM requests; or daily batch pull last 30 days) | **P1** |
| 7 | `record_payment_in_advelsoft` | WRITE (Minilab → vendor) | `collection_transactions` (when transaction_type='payment') | When a payment is received and approved in Minilab's collection workflow (bank transfer verified, Billplz webhook, cash collection), record it in Advelsoft so the accounting system of record stays current | Payment entry form in Transaction module: unit, amount, date, receipt reference | write (dangerous — financial record, no retry, requires manual review before triggering) | **P1** |
| 8 | `pull_monthly_billing_report` | READ (vendor → Minilab) | **NEW TABLE:** `vendor_billing_snapshots` or augment `reports_daily_snapshot` | Full monthly billing run output — what was charged to every unit this month (maintenance fee, sinking fund, special levies, etc.). Enables Minilab to show each unit's monthly charge breakdown without per-unit queries | Billing report → PDF or DATA output | sync (monthly cron, 1st of each month after Advelsoft closes billing) | **P1** |
| 9 | `pull_collection_report` | READ (vendor → Minilab) | `reports_daily_snapshot` (augment with vendor_collections_total) | Total collections received vs total billed — the BM and PMC dashboard "collection rate" KPI. Minilab currently has no data for this; it only tracks escalations | Billing and Collection report | sync (daily cron) | **P1** |
| 10 | `pull_unit_deposit_balance` | READ (vendor → Minilab) | **NEW COLUMN:** `units.maintenance_deposit_balance NUMERIC` or `collection_accounts.deposit_balance` | Maintenance deposit per unit — relevant for move-out refund processing, deposit usage approval. Minilab has no deposit tracking | Deposits report → PDF | read_only (monthly or on-demand) | **P2** |
| 11 | `pull_interest_charges` | READ (vendor → Minilab) | `collection_accounts` (augment with `vendor_interest_accrued`) | What interest has been charged on overdue units — provides accurate total-owed number for collection letters and Form 11 | Interest Charges Estimation report | read_only (monthly or on-demand) | **P2** |
| 12 | `pull_monthly_balance_analysis` | READ (vendor → Minilab) | `vendor_ageing_snapshots` or `reports_daily_snapshot` | Building-level financial health overview — opening balance, charges, receipts, closing balance per month. Needed for PMC financial reporting | Monthly Balance Analysis report → PDF | sync (monthly cron) | **P2** |

### Accounting — New Tables/Columns Required

| Item | Type | Columns | Reason |
|------|------|---------|--------|
| `units.vendor_account_number` | New column | TEXT nullable | Maps Minilab unit UUID → Advelsoft account number. Prerequisite for all accounting skills |
| `collection_accounts.vendor_balance` | New column | NUMERIC nullable | Advelsoft's authoritative outstanding balance — separate from Minilab-computed outstanding_balance |
| `collection_accounts.vendor_last_synced_at` | New column | TIMESTAMPTZ nullable | When vendor_balance was last pulled from Advelsoft |
| `collection_accounts.vendor_interest_accrued` | New column | NUMERIC nullable | Interest charged in Advelsoft but not yet in Minilab's transaction log |
| `collection_transactions.source` | New column | TEXT DEFAULT 'minilab' CHECK (minilab/advelsoft/import) | Distinguish Minilab-recorded transactions from Advelsoft-imported ones |
| `vendor_billing_statements` | New table | id, building_id, unit_id (FK), vendor_account_number, period (DATE), pdf_url, total_due, generated_at, fetched_at | No per-unit billing statement storage anywhere in Minilab |
| `vendor_ageing_snapshots` | New table | id, building_id, snapshot_date, pdf_url, current_count, overdue_30_count, overdue_60_count, overdue_90_plus_count, total_current_amount, total_overdue_amount, data JSONB, fetched_at | BM dashboard ageing widget needs this data |

---

## Summary

### Skills Count Per System

| System | Total Skills | P0 | P1 | P2 | Build Status |
|--------|-------------|----|----|----|----|
| Access Card | 8 | 3 | 3 | 2 | **BLOCKED** — vendor unknown (recon pending §10) |
| Car Plate LPR | 11 | 5 | 4 | 2 | **BLOCKED** — vendor unknown (recon pending §10) |
| Advelsoft Accounting | 12 | 5 | 4 | 3 | **READY** — connection type proven, session 2 in build order |
| **TOTAL** | **31** | **13** | **11** | **7** | |

### P0 Skills for CHV Launch

These are the minimum viable set — without these, CHV integration delivers no value.

**Access Card (3 P0 skills — requires recon first):**
- `block_unit_access_cards` — collections enforcement (physical access lock)
- `restore_unit_access_cards` — collections enforcement (access restore on payment)
- `deactivate_single_card` — move-out security

**Car Plate LPR (5 P0 skills — requires recon first):**
- `register_resident_plate` — residents can access building via barrier
- `deregister_resident_plate` — security on move-out
- `block_unit_plates` — collections enforcement (barrier lock mirrors access card block)
- `restore_unit_plates` — collections enforcement (barrier restore on payment)
- `pull_lpr_entry_exit_logs` — replaces manual guard entry, makes vehicle_logs authoritative

**Advelsoft Accounting (5 P0 skills — can start now):**
- `pull_unit_account_mapping` — prerequisite, must be first
- `pull_outstanding_balance_all_units` — resident portal shows real balance
- `pull_ageing_report` — BM dashboard ageing widget
- `export_unit_billing_statement` — resident gets their statement on demand
- `reconcile_collection_accounts` — prevents Minilab and Advelsoft diverging silently

### Schema Changes Needed (by priority)

**P0 schema (required before P0 skills can be built):**
- `units.vendor_account_number TEXT` — Advelsoft account number
- `collection_accounts.vendor_balance NUMERIC` — authoritative Advelsoft balance
- `collection_accounts.vendor_last_synced_at TIMESTAMPTZ`
- New table: `vendor_billing_statements` — statement PDF storage per unit/period
- New table: `vendor_ageing_snapshots` — parsed ageing report data per day

**P1 schema (required before P1 skills can be built):**
- `vehicle_logs.source`, `.confidence_score`, `.snapshot_url`, `.lpr_event_id` — LPR event fields
- `collection_transactions.source` — distinguish Minilab vs Advelsoft-imported transactions
- `access_cards.vendor_card_id`, `.vendor_synced_at` — vendor card reference

**P2 schema (deferred):**
- New table: `access_card_logs` — swipe event log
- New table: `parking_allocations` — plate→lot mapping
- `contractor_staff.vehicle_plate` — contractor vehicle registration
- `collection_accounts.vendor_interest_accrued` — interest from Advelsoft

### Recommended Sync Frequencies

| Data Type | Frequency | Rationale |
|-----------|-----------|-----------|
| Outstanding balance (all units) | Daily minimum, 6-hourly preferred | Resident portal shows this — must feel current |
| Ageing report | Daily | BM dashboard KPI |
| LPR entry/exit logs | Every 15 minutes | Near-real-time vehicle access visibility |
| Access card swipe logs | Hourly | Security audit, not real-time critical |
| Plate whitelist reconciliation | Daily | Catch manual vendor changes overnight |
| Cardholder list reconciliation | Daily | Catch orphan/unlisted cards |
| Unit billing statements | On-demand (when resident views) + monthly batch pre-generate | Avoid 700 PDF downloads per month unless needed |
| Transaction history per unit | On-demand or daily rolling 30-day batch | Heavy query — only pull when needed |
| Ageing/collection/billing reports (building-level) | Daily for operational metrics, monthly for full PDF archive | Dashboard data vs archival |
| Unit account mapping | One-time setup + monthly reconciliation for new units | Stable after initial onboarding |
| Payments recorded in Advelsoft | On-demand, triggered by BM payment approval | Financial write — always human-initiated |

### Build Order Recommendation

Given §10 pending recon and §15 build order from MOA-SPEC:

1. **Session 1:** Agent core (already planned)
2. **Session 2:** Advelsoft connector login
3. **Session 3:** Skill executor core actions
4. **Session 4:** `pull_unit_account_mapping` + `pull_outstanding_balance_all_units` (first Advelsoft P0 skills)
5. **Session 5:** `export_unit_billing_statement` + `pull_ageing_report` (remaining Advelsoft P0)
6. **Parallel:** Complete recon for access card + car plate vendors
7. **Session 6+:** Access card P0 skills (block/restore/deactivate)
8. **Session 7+:** Car plate P0 skills (register/deregister/pull_lpr_logs/block/restore)

### Cross-System Coordination — The Collections Enforcement Bundle

The most powerful CHV use case is: **Unit 3 months overdue → BM presses one button → Advelsoft updated + access cards blocked + barrier plates blocked + WhatsApp notice sent.**

This is not 3 separate MOA skills — it is 1 Minilab workflow that dispatches 3 MOA commands in parallel (one per system), each going to the appropriate connector. The `moa_commands` table needs `system_id` (per MOA-SPEC §11) to route correctly. The Minilab UI trigger already exists (collection escalation) — it just needs to fire moa_commands entries when `access_blocked` flips to true.

---

*Files created: docs/startup/moa-integration-requirements.md*
