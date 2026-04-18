# Schema Reference — Minilab
<!-- Anchored by table name. Use: awk '/^## §Table-users/,/^## §/' docs/startup/actual-schema-columns.md | head -n -1 -->
<!-- List all anchors: grep '^## §' docs/startup/actual-schema-columns.md -->

## §Table-Index
<!-- All tables: pipe-separated list -->
access_card_logs | access_cards | agm_cards | agm_egm_meetings | agm_eligible | agm_motions | agm_proxies | agm_sessions | agm_votes | ai_tools | announcements | api_cost_alerts | approvals | asset_logs | assets | attendance_logs | audit_log | bank_accounts | bid_pricing_tiers | blocks | building_applications | building_documents | building_gaps | building_gates | building_geofences | building_onboarding | building_org_assignments | building_payment_settings | building_spaces | building_tenancies | building_tenancy_documents | building_working_hours | buildings | cases | chase_loops | cleaning_logs | collection_accounts | collection_escalations | collection_settings | collection_transactions | console_read_state | contact_channels | contacts_temp | contractor_org_building_assignments | contractor_orgs | contractor_schedule_templates | contractor_shifts | contractor_staff | contractor_staff_building_assignments | contractor_visits | daily_reports | data_imports | dead_letter_queue | dev_buildings | dev_defect_logs | dev_defects | dev_handover_checklists | dev_rectification_contractors | dev_unit_types | dev_units | developer_orgs | developments | document_chunks | email_log | face_enrollments | facilities | facility_bookings | fine_settings | form_templates | guard_shifts | guardhouse_incidents | guardhouses | hardware_store_applications | hardware_store_order_items | hardware_store_orders | hardware_store_products | hardware_store_subscriptions | in_unit_tender_bids | in_unit_tenders | insurance_policies | integration_requests | inventory_usage | investor_leads | invitations | job_materials | kiosk_devices | leave_balances | leave_policies | leave_requests | legal_records | maintenance_events | maintenance_schedules | media_search_index | messages | moa_agents | moa_commands | moa_skills | moa_supported_systems | monitoring_events | notification_preferences | notifications | openclaw_connections | organisations | parcels | partner_agencies | patrol_beacons | patrol_logs | patrol_schedules | pdpa_audit_log | pdpa_consents | pdpa_deletion_requests | pending_residents | permission_audit_log | personal_properties | petty_cash | phone_call_logs | phone_change_log | platform_issues | platform_settings | procurement_items | procurement_order_lines | procurement_orders | property_listings | public_holidays | push_notification_subscriptions | raw_imports | rental_listings | renovation_applications | reports_daily_snapshot | resident_pwa_sessions | security_flags | sender_profiles | service_visits | staff_permissions | supplier_building_relationships | supplier_credit_accounts | supplier_invoices | suppliers | telegram_group_settings | tenancies | tenancy_documents | tender_bids | tender_commissions | tenders | tickets | unit_occupancy_status | units | user_roles | users | utility_bills | vehicle_fines | vehicle_logs | vendor_ageing_snapshots | vendor_billing_statements | vendor_building_contracts | vendor_performance_history | vendor_subscriptions | vendor_wallets | vendors | visitors | voice_knowledge_base | voice_leads | voice_unanswered | wallet_transactions | whatsapp_templates | worker_activation_codes

---

# Minilab — Live Database Schema Audit

> **Last cleaned:** 2026-03-25 — 10 phantom tables removed (unit_residents, otp_codes, push_subscriptions, ai_memory, announcements, bank_accounts, insurance_policies, building_documents, committee_members, meeting_minutes). push_subscriptions corrected to push_notification_subscriptions. See Session 6A orphan audit.

> **Column updates applied (2026-04-10 Email Fix sessions):**
> - Migration 074: `messages.sender_profile_id` UUID FK→sender_profiles (nullable) + index `idx_messages_sender_profile_id`. Links message to sender profile for console contact-messages query. Backfill applied via email_log join.
> - Migration 072: `buildings.fallback_template_name` TEXT DEFAULT 'minilab_notification'. Per-building WhatsApp template name for 24h window fallback (D-0508).
>
> **Column updates applied (2026-04-10 Cron Session B):**
> - Migration 073: `attendance_logs.auto_clocked_out` BOOLEAN DEFAULT false (set by auto-clockout cron). `visitors.status` TEXT DEFAULT 'active' CHECK (active/checked_in/expired/cancelled) — set by visitor-cleanup cron.
>
> **Column updates applied (2026-04-10 V20):**
> - Migration 071: 6 new building-wide attendance Postgres RPCs — `get_attendance_today_by_building`, `get_attendance_history_by_building`, `get_attendance_stats_by_building`, `get_late_arrivals_by_building`, `get_absent_staff_by_building`, `get_attendance_by_shift`. No new columns. These replace all remaining D-0345 PostgREST violations across 14 files. Run `NOTIFY pgrst, 'reload schema'` after applying.
> - Migration 070: `access_cards` table (id, building_id, unit_id, resident_id, card_number, status, issued_date, deactivated_at, notes, created_at, updated_at). RLS enabled. Status: active/deactivated/lost/returned.
>
> **Column updates applied (2026-04-09 V19):**
> - Migration 067: `message_channel_type` enum + `internal_note` value. `sender_profiles.ai_paused` BOOLEAN DEFAULT false. `escalate_to_human` AI tool registered.
> - Migration 068: `sender_profiles.display_name` TEXT nullable. `sender_profiles.classification` TEXT DEFAULT 'unknown' (unknown/saved/external/resident).
> - Migration 069: `sender_profiles.company` TEXT nullable.
> - New table: `building_gates` (D-0442) — per-gate VMS passcodes. `visitors.gate_id` + `vehicle_logs.gate_id` FKs added.
> - `console_read_state` table (D-0432, migration 066) — (building_id, user_id, sender_profile_id, last_read_at).
> - All sender_profiles columns documented in V4 Tables section below.

> **Audited:** 2026-03-20 via API query against live Supabase database
> **Method:** Supabase REST API (table count + column discovery from database.types.ts)
> **Migration query:** Could not query `supabase_migrations.schema_migrations` via REST API (schema not accessible)
> **Note:** Table count (145) reflects 2026-03-20 audit. Actual count is higher due to V16–V19 migrations (building_gates, console_read_state, leave_policies, leave_balances, leave_requests added). Column definitions in sections below are current.

---

## 1. Table Count

**145 tables exist** in the live database (142 + 3 migration: leave_policies, leave_balances, leave_requests + 1 view: ai_view_leave).

### All 128 Tables (with live row counts)

| Table | Rows | Table | Rows |
|---|---|---|---|
| buildings | 5 | units | 150 |
| residents | 0 | ~~unit_residents~~ | PHANTOM |
| users | 11 | user_roles | 8 |
| cases | 0 | case_messages | 0 |
| case_history | 0 | collection_accounts | 0 |
| collection_transactions | 0 | collection_escalations | 0 |
| visitors | 0 | visitor_logs | 0 |
| facilities | 0 | facility_bookings | 0 |
| facility_maintenance | 0 | contractor_orgs | 0 |
| contractor_staff | 0 | shifts | 0* |
| attendance_records | 0* | parcels | 0 |
| hardware_stores | 0* | hardware_products | 0* |
| tenders | 0 | tender_bids | 0 |
| wallets | 0* | wallet_transactions | 0 |
| service_visits | 0 | renovation_applications | 0 |
| suppliers | 0 | purchase_orders | 0* |
| po_line_items | 0* | defects | 0* |
| pmc_orgs | 0* | developer_orgs | 0 |
| platform_settings | 1 | telegram_group_settings | 0 |
| documents | 0 | invoices | 0 |
| sender_profiles | 0 | chase_loops | 0 |
| daily_reports | 0 | conversations | 0 |
| messages | 0 | ~~ai_memory~~ | PHANTOM |
| building_ai_config | 0 | ~~committee_members~~ | PHANTOM |
| ~~meeting_minutes~~ | PHANTOM | ~~announcements~~ | PHANTOM |
| common_areas | 0 | maintenance_requests | 0 |
| vendor_contracts | 0 | vendor_invoices | 0 |
| payment_vouchers | 0 | petty_cash | 0 |
| ~~bank_accounts~~ | PHANTOM | sinking_fund | 0 |
| ~~insurance_policies~~ | PHANTOM | building_rules | 0 |
| agm_resolutions | 0 | complaints | 0 |
| utility_readings | 0 | emergency_contacts | 0 |
| ~~building_documents~~ | PHANTOM | notification_preferences | 0 |
| push_notification_subscriptions | 0 | ~~otp_codes~~ | PHANTOM |

**\* Tables marked with asterisk:** Exist in PostgreSQL (count query returns 0) but **NOT found in PostgREST schema cache** when attempting SELECT — meaning either:
- They have no RLS policies and aren't exposed via REST
- They were created directly in SQL but PostgREST wasn't reloaded
- They are stale/orphan tables from migrations that don't match database.types.ts

---

## 2. Actual Column Definitions (from database.types.ts — source of truth)

## §Table-buildings
### buildings
```
id, name, address, city, state, postcode, total_units, status, org_id,
strata_title_number, telegram_bot_token, timezone, total_share_units,
trial_ends_at, activated_at, whatsapp_access_token, whatsapp_display_name,
whatsapp_onboarding_method, whatsapp_phone_number_id, whatsapp_phone_number,
whatsapp_quality_rating, whatsapp_status, whatsapp_verified_at, whatsapp_waba_id,
fallback_template_name (text, default 'minilab_notification'),
minilab_email, email_forwarding_from, email_forwarding_verified, email_auto_reply_text,
building_image, login_slug (text, unique where not null),
building_user_id (uuid, FK → users),
vms_passcode (text), attendance_passcode (text),
wa_app_secret (text),
ai_fallback_user_id (uuid, FK → users),
ai_disabled (boolean, NOT NULL, default false),
created_at, updated_at
```
**NOT present:** latitude, longitude, latitude_lng (no geo columns at all)
**Migration 095:** Retroactive tracking of ai_disabled column (already live from D-0659 dashboard add). `ADD COLUMN IF NOT EXISTS` for idempotency. (D-0682)
**Migration 092 (dashboard):** Added ai_disabled (BOOLEAN NOT NULL DEFAULT false) — building-level AI kill switch. Checked by shouldSkipAI() before all per-sender checks. Also gates WhatsApp auto-registration for unknown senders (D-0684). (D-0659)
**Migration 079:** Added ai_fallback_user_id (UUID FK → users) — building-level AI fallback assignee for handoff cases (D-0589).
**Migration 078:** Added wa_app_secret (TEXT) — per-building Meta App Secret for HMAC webhook signature validation. Looked up by whatsapp_phone_number_id on inbound webhook.
**Migration 049:** Added login_slug, building_user_id for building service account auth.
**Migration 054:** Added whatsapp_phone_number (TEXT) — human-readable E.164 phone for wa.me links. Distinct from whatsapp_phone_number_id (Meta internal API ID). Populated by WA setup routes (Path A from Meta API response, Path B from OBO form input).
**Migration 062+:** Added vms_passcode (TEXT, 6-digit numeric) and attendance_passcode (TEXT, 6-digit numeric) for kiosk activation. BM sets via /bm/settings/apps. Guard VMS uses vms_passcode to activate kiosk session.

## §Table-units
### units
```
id, building_id, unit_number, floor, unit_type, share_units, size_sqft,
strata_title_reference, is_active, created_at, updated_at,
vendor_account_number (TEXT)   -- Migration 086: Advelsoft/legacy account number for this unit
```
**Note:** Column is `unit_type` NOT `type`
**Migration 096 indexes:** `idx_units_unit_number_normalized` (B-tree on `unit_number_normalized(unit_number)`), `idx_units_unit_number_trgm` (GIN on `unit_number_normalized(unit_number)` with `gin_trgm_ops`). Used by `search_units_smart` RPC.
**Extension:** `pg_trgm` enabled (Migration 096) for fuzzy unit search.

## §Table-residents
### residents
```
id, building_id, unit_id, full_name, phone (nullable), email, ic_number,
role (enum: resident_role_type), is_active, move_in_date, move_out_date,
pdpa_consent (boolean), pdpa_consent_at (timestamp), user_id, created_at, updated_at,
data_status (text, default 'imported', CHECK: verified/touched/imported/unreached),
last_contacted_at (timestamptz), data_source (text, default 'import', CHECK: import/whatsapp/telegram/email/manual/ai_enrichment/blast),
phone_valid (boolean, default true), verified_at (timestamptz), verified_by (uuid FK→users),
created_via (text, nullable)    -- Migration 097: backfill tag e.g. 'csv_backfill_v32a'
```
**NOT present:** `pdpa_consent_date` (correct name is `pdpa_consent_at`), `status` (use `is_active`)
**Migration 082:** Added data quality columns: data_status, last_contacted_at, data_source, phone_valid, verified_at, verified_by. Backfilled from messages (→touched) and approvals (→verified). RPC `update_resident_contact_status(p_resident_id, p_channel)` auto-updates on inbound messages.
**Migration 089:** `phone` changed from NOT NULL → nullable (D-0633). Email-only contacts linked from console have no phone.
**Migration 097:** Added `created_via` (nullable text) — backfill audit tag. CHV V32a rows tagged 'csv_backfill_v32a'. CHV total post-V32a: 1,081 residents (1,002 existing + 79 new).

## §Table-users
### users
```
id, phone (nullable for building accounts), full_name, auth_id, is_active, last_seen_at,
profile_photo_url, telegram_id, telegram_username, created_at, updated_at,
google_email, google_id, email (text, unique where not null), password_hash (text),
account_type (text, default 'individual', CHECK: 'individual'|'building'),
totp_secret (text, nullable), totp_enabled (boolean, default false)
```
**Migration 049:** Added email, password_hash, account_type. Made phone nullable with CHECK constraint (phone required unless account_type='building'). Building service accounts login with email+password, no phone needed.
**V12-P5:** Added totp_secret (TOTP 2FA secret, superadmin only) and totp_enabled (boolean). Per-user TOTP with env var fallback.
**Migration 066:** Added telegram_blocked (boolean, default false). Set to true when TG bot-blocked 403 detected.

## §Table-console_read_state
### console_read_state (D-0432, migration 066)
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings),
user_id (UUID NOT NULL FK→users),
sender_profile_id (UUID NOT NULL FK→sender_profiles),
last_read_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**Unique:** (building_id, user_id, sender_profile_id)
**Purpose:** Tracks last-read timestamp per conversation per BM user for console unread counts.

## §Table-user_roles
### user_roles
```
id, user_id, building_id, role (enum: user_role_type), position (text),
committee_position (text), is_primary_building, assigned_at, assigned_by, revoked_at,
created_at, updated_at,
last_template_applied_name (text), last_template_applied_at (timestamptz),
last_template_applied_by (uuid, no FK — allows sentinel)
```
**position:** Display job title for staff (Admin, Accounts, Assistant Manager, Technician, Staff). Auth/RLS uses `role` column; `position` is for display only. Added in migration 038.
**committee_position:** For role=committee only: Chairperson, Secretary, Treasurer, Member. Added in migration 039.
**last_template_applied_name/at/by:** Permission template audit trail (migration 107, D-0748). Written by `applyRoleTemplate` on every template apply. `last_template_applied_by` stores real actor UUID or sentinel `00000000-0000-0000-0000-000000000001` for system backfills. No FK constraint — allows sentinel. Staff with null `last_template_applied_at` AND zero permissions = drift incident. Partial index `idx_user_roles_no_template` on (role, position) WHERE role='staff' AND last_template_applied_at IS NULL.
**user_role_type enum values (22 total, as of migration 102a):** superadmin, bm, staff, committee, contractor_admin, contractor_supervisor, contractor_worker, guard, supplier, resident, owner, tenant, pmc, developer_admin, developer_staff, security_admin, security_supervisor, cleaning_admin, cleaning_supervisor, cleaning_worker *(added 102a, D-0727)*, store_admin, org_admin.
**Field-staff roles** (route to /app, not org dashboards): guard, cleaning_worker, contractor_worker.

## §Table-cleaning_logs
### cleaning_logs

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| contractor_staff_id | uuid | NO | | FK→contractor_staff |
| location_name | text | NO | | |
| location_beacon_id | uuid | YES | | FK→patrol_beacons |
| before_photo_url | text | YES | | |
| after_photo_url | text | YES | | |
| started_at | timestamptz | YES | now() | |
| completed_at | timestamptz | YES | | |
| notes | text | YES | | |
| status | text | YES | 'in_progress' | CHECK: in_progress, completed, verified |
| verified_by | uuid | YES | | FK→users |
| verified_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-cases
### cases
```
id, building_id, case_number (required string), title, description,
category (enum: case_category_type), status (enum: case_status_type),
priority, is_urgent, resident_id, unit_id, assigned_to,
assigned_contractor_org_id, ai_intent, ai_summary, source_message_id,
sender_profile_id (uuid, FK → sender_profiles),
sla_due_at, resolved_at, created_at, updated_at,
inspection_scheduled_at (TIMESTAMPTZ nullable),
related_unit_id (UUID FK → units nullable),
related_case_id (UUID FK → cases nullable)
```
**NOT present:** `reported_by` (correct name is `resident_id`), `case_number` is REQUIRED
**Migration 079:** Added sender_profile_id (UUID FK → sender_profiles) — links handoff cases to unregistered contacts. Added 'ai_handoff' to case_category_type enum (D-0590).
**Migration 081:** Added related_unit_id (UUID FK → units, indexed) — for interfloor leak cases linking affected/source unit. Added 'interfloor_leak' to case_category_type enum (D-0600).
**Migration 098:** Added inspection_scheduled_at (TIMESTAMPTZ nullable, partial index). D-0709.
**Migration 099:** Added related_case_id (UUID FK → cases ON DELETE SET NULL, partial index) — links secondary interfloor cases back to their primary case. D-0713.

## §Table-case_participants
### case_participants
```
id, case_id (FK → cases CASCADE), user_id (FK → users CASCADE),
role (text CHECK: participant|watcher, default participant),
added_by (FK → users nullable), created_at
UNIQUE(case_id, user_id)
```
**Migration 093:** New table (D-0673). RLS: bm/staff can manage. AI view: ai_view_case_participants (joins users + cases, exposes user_name + building_id).

## §Table-case_category_defaults
### case_category_defaults
```
id, building_id (FK → buildings CASCADE), category (text),
default_assignee_id (FK → users SET NULL nullable),
default_participant_ids (UUID[] default '{}'),
created_at, updated_at
UNIQUE(building_id, category)
```
**Migration 093:** New table (D-0673). RLS: bm only. Used by applyAutoParticipants helper on case creation.

## §Table-collection_accounts
### collection_accounts
```
id, building_id, unit_id, outstanding_balance, months_overdue,
escalation_stage, access_blocked, access_blocked_at, form11_served_at,
last_payment_amount, last_payment_date, created_at, updated_at,
vendor_balance (NUMERIC),              -- Migration 086: balance from legacy accounting system
vendor_last_synced_at (TIMESTAMPTZ)    -- Migration 086: when vendor balance was last fetched
```

## §Table-collection_transactions
### collection_transactions
```
id, building_id, account_id, transaction_type, amount, description,
processed_by, reference_number, transaction_date, created_at
```

## §Table-collection_escalations
### collection_escalations
```
id, building_id, account_id, from_stage, to_stage, triggered_by,
notes, created_at
```

## §Table-visitors
### visitors
```
id, building_id, visitor_name, visitor_phone, visitor_ic, unit_id,
hosting_resident_id, visit_purpose, is_preregistered, check_in_at,
check_out_at, logged_by, photo_url, qr_code, vehicle_plate, visit_date,
ic_photo_url, license_photo_url, gate_id,
checkout_gate_id (UUID FK→building_gates — mig 091, tracks exit gate),
status (TEXT DEFAULT 'active' CHECK active/checked_in/expired/cancelled — mig 073),
pass_number (TEXT, nullable — mig 090),
created_at
```
<!-- visit_date DATE added in migration 062 (D-0415) -->
<!-- ic_photo_url, license_photo_url TEXT added in migration 063 (D-0417) -->
<!-- gate_id uuid FK→building_gates added (D-0442) -->
<!-- checkout_gate_id uuid FK→building_gates added (D-0650, mig 091) -->

## §Table-facilities
### facilities
```
id, building_id, name, description, capacity, opening_time, closing_time,
advance_booking_days, max_booking_hours, deposit_amount, rules, is_active,
created_at, updated_at
```

## §Table-facility_bookings
### facility_bookings
```
id, building_id, facility_id, resident_id, booking_date, start_time,
end_time, status (enum: booking_status_type), purpose, guests_count,
deposit_paid, approved_by, cancelled_reason, created_at, updated_at
```

## §Table-contractor_orgs
### contractor_orgs
```
id, name, contact_phone, contact_email, registration_number,
contractor_type (string[]), service_categories (string[]),
admin_user_id, is_active, created_at, updated_at
```
**NOT present:** `company_name` (correct is `name`), `status` (correct is `is_active`)

## §Table-contractor_staff
### contractor_staff
```
id, contractor_org_id (nullable — migration 060), full_name, phone, role, ic_number, user_id,
is_active, joined_at, face_enrolled, compreface_subject_id,
ble_tracking_enabled, created_at, updated_at,
nationality, passport_number, work_permit_number, work_permit_expiry,
work_permit_file_url, fomema_status, fomema_expiry, fomema_file_url, photo_url
```
**NOT present:** `status` (correct is `is_active`)
**NOTE:** `contractor_org_id` is nullable (dropped NOT NULL in migration 060). Staff can exist without a company — BM creates staff first, links to company later via PUT /api/bm/staff/assign-company (D-0388/D-0389). FK constraint is preserved. Unassigned staff get a building assignment with null contractor_org_id (migration 061) so they appear on the All Staff page.
**Migration 100 (D-0719):** All contractor_staff with phone now have `user_id` set pointing to a users record. Guards with `role='guard'` also have a `user_roles` entry (role='guard') per building assignment, enabling Telegram login → /app routing. Staff creation API now upserts users record + links user_id on every new contractor_staff create.

## §Table-contractor_shifts
### contractor_shifts (NOT "shifts")
```
(Not directly queryable via REST — in database.types.ts)
contractor_staff_id, building_id, shift_date, start_time, end_time, shift_type
```

## §Table-attendance_logs
### attendance_logs (NOT "attendance_records")
```
id, building_id, contractor_staff_id (nullable — migration 057), user_id (uuid, nullable — migration 057),
shift_id, clock_in_at, clock_out_at, is_late, is_early_departure,
clock_in_lat, clock_in_lng, clock_out_lat, clock_out_lng,
compreface_match_score, face_match_score, verification_method,
selfie_url, offline_synced, notes,
auto_clocked_out (BOOLEAN DEFAULT false — mig 073, set by auto-clockout cron),
created_at
```
NOTE: contractor_staff_id is now nullable (dropped NOT NULL in migration 057).
user_id added for building staff who are not in contractor_staff table.
Either contractor_staff_id OR user_id must be set — not both required.
**Migration 093:** Unique partial indexes prevent duplicate open sessions (race condition fix):
- `idx_attendance_one_open_user` on (user_id, building_id) WHERE clock_out_at IS NULL AND user_id IS NOT NULL
- `idx_attendance_one_open_contractor` on (contractor_staff_id, building_id) WHERE clock_out_at IS NULL AND contractor_staff_id IS NOT NULL

## §Table-parcels
### parcels
```
id, building_id, unit_id, courier, tracking_number, description,
received_at, collected_at, collected_by, logged_by, photo_url,
notification_sent, resident_id, created_at
```

## §Table-tenancies
### tenancies
```
id (uuid), unit_id (uuid), building_id (uuid), tenant_resident_id (uuid),
owner_resident_id (uuid), monthly_rent (numeric), start_date (date),
end_date (date), is_active (boolean), created_at (timestamptz),
updated_at (timestamptz)
```
**Purpose:** Tracks tenant-owner rental relationships per unit. Links tenant and owner resident records. Used by /api/bm/tenancies and related routes.

## §Table-tenders
### tenders
```
id, building_id, title, description, category, tender_type (enum),
status (enum: tender_status_enum), budget_min, budget_max, deadline,
images (string[]), videos (string[]), location_detail, posted_by,
awarded_bid_id, completed_at, unit_id, created_at, updated_at
```

## §Table-tender_bids
### tender_bids
```
id, tender_id, contractor_org_id, quote_amount, quote_description,
estimated_days, attachments (string[]), contact_name, contact_phone,
token_cost, created_at
```

## §Table-vendor_wallets
### vendor_wallets (NOT "wallets")
```
id, org_id, balance, total_topped_up, total_spent
```

## §Table-wallet_transactions
### wallet_transactions
```
id, wallet_id, type (enum: wallet_tx_type_enum), amount, reference,
stripe_payment_id, tender_id, created_at
```

## §Table-service_visits
### service_visits
```
id, building_id, contractor_org_id, service_type, scheduled_date,
actual_arrival, actual_departure, work_done, findings, parts_replaced (string[]),
photos (string[]), is_compliance, status (enum), verified_by_bm,
verified_at, created_at, updated_at
```

## §Table-renovation_applications
### renovation_applications
```
id, building_id, unit_id, resident_id, scope_of_work, start_date,
end_date, deposit_amount, deposit_paid, status, contractor_org_id,
approved_by, inspection_notes, created_at, updated_at
```

## §Table-suppliers
### suppliers
```
id, company_name, contact_phone, contact_email, contact_name,
registration_number, service_categories (string[]), credit_terms_days,
is_active, approved_by, created_at, updated_at
```

## §Table-procurement_orders
### procurement_orders (NOT "purchase_orders")
```
id, building_id, supplier_id, order_number, status, total_amount, created_at
```

## §Table-procurement_order_lines
### procurement_order_lines (NOT "po_line_items")
```
id, order_id, item_name, quantity, unit_price, delivered_quantity
```

## §Table-developer_orgs
### developer_orgs
```
id, company_name, ssm_number, pic_name, pic_phone, pic_email,
address, logo, status (enum), admin_user_id, created_at, updated_at
```
**NOT present:** `registration_number` (correct is `ssm_number`), `contact_phone` (correct is `pic_phone`), `contact_email` (correct is `pic_email`), `logo_url` (correct is `logo`). **MISSING required field:** `pic_name`

## §Table-organisations
### organisations (used for PMC — NOT "pmc_orgs")
```
id, name, org_type, registration_number, contact_phone, contact_email, created_at, updated_at
```

## §Table-platform_settings
### platform_settings
```
id, key (unique), value (jsonb), description, updated_by, created_at, updated_at
```

---

## 3. Tables that DON'T EXIST in database.types.ts

These tables are referenced by the seeder or code but **do not exist** in the generated types:

| Seeder Uses | Actual Table Name | Status |
|---|---|---|
| `shifts` | `contractor_shifts` | **WRONG NAME** |
| `attendance_records` | `attendance_logs` | **WRONG NAME** |
| `wallets` | `vendor_wallets` | **WRONG NAME** |
| `purchase_orders` | `procurement_orders` | **WRONG NAME** |
| `po_line_items` | `procurement_order_lines` | **WRONG NAME** |
| `defects` | `dev_defects` | **WRONG NAME** |
| `pmc_orgs` | `organisations` (with org_type filter) | **WRONG NAME** |
| `hardware_stores` | *(does not exist)* | **TABLE MISSING** |
| `hardware_products` | *(does not exist)* | **TABLE MISSING** |
| ~~`unit_residents`~~ | *(does not exist — PHANTOM, removed 2026-03-25)* | **REMOVED** |

**Note:** `shifts`, `attendance_records`, `wallets`, `purchase_orders`, `po_line_items`, `defects`, `pmc_orgs`, `hardware_stores`, `hardware_products` show as "existing" in the PostgreSQL count query but are **NOT in PostgREST schema cache** and **NOT in database.types.ts**. These may be orphan tables from partially-applied migrations or views that were never properly exposed. `unit_residents` was confirmed PHANTOM (never existed in DB) in Session 6A audit.

---

## 4. Seeder Column Mismatches (demo-data.ts vs actual schema)

### CRITICAL ERRORS (will cause insert failures)

| # | Table | Seeder Column | Actual Column | Line(s) |
|---|---|---|---|---|
| 1 | `residents` | `pdpa_consent_date` | `pdpa_consent_at` | 168, 827 |
| 2 | `residents` | `status: 'verified'` | *(no status column — use is_active)* | 168, 826 |
| 3 | `contractor_orgs` | `company_name` | `name` | 178, 182, 186 |
| 4 | `contractor_orgs` | `status: 'active'` | `is_active: true` | 179, 183, 187 |
| 5 | `contractor_staff` | `status: 'active'` | `is_active: true` | 207, 219, 231 |
| 6 | `cases` | `reported_by` | `resident_id` | 281, 837 |
| 7 | `cases` | *(missing)* | `case_number` (required) | 278-284 |
| 8 | `developer_orgs` | `registration_number` | `ssm_number` | 573 |
| 9 | `developer_orgs` | `contact_phone` | `pic_phone` | 574 |
| 10 | `developer_orgs` | `contact_email` | `pic_email` | 574 |
| 11 | `developer_orgs` | `logo_url` | `logo` | 575 |
| 12 | `developer_orgs` | *(missing)* | `pic_name` (required) | 572-576 |
| 13 | `tender_bids` | *(missing)* | `token_cost` (required) | 467-474 |

### TABLE NAME ERRORS (seeder uses correct table names already for some)

The seeder at lines 262-268 already correctly uses `contractor_shifts` and `attendance_logs`.
The seeder at lines 422-433 already correctly uses `vendor_wallets`.
The seeder at lines 547-567 already correctly uses `procurement_orders` and `procurement_order_lines`.
The seeder at lines 632-643 already correctly uses `dev_defects`.
The seeder at line 646 already correctly uses `organisations` for PMC.

### Additional tables the seeder references that need verification:

| Table | Exists in types? | Seeder lines |
|---|---|---|
| `contractor_org_building_assignments` | Yes | 193-198 |
| `supplier_building_relationships` | Need to verify | 533 |
| `supplier_credit_accounts` | Need to verify | 537-541 |
| `developments` | Yes | 579-584 |
| `dev_buildings` | Yes | 587-592 |
| `dev_unit_types` | Yes | 595-598 |
| `dev_units` | Yes | 610 |
| `dev_rectification_contractors` | Yes | 637-643 |
| `building_org_assignments` | Yes | 653-656 |
| `maintenance_schedules` | Need to verify | 768-776 |

---

## 5. Schema-Summary.md vs Reality — Key Mismatches

| schema-summary.md says | Reality (database.types.ts) |
|---|---|
| `units`: "type" column | `unit_type` column |
| `residents`: "PDPA consent tracked" | `pdpa_consent` (bool) + `pdpa_consent_at` (timestamp) — no `pdpa_consent_date` |
| Section 15.4: `contractor_shifts` table | Correct in types, but seeder schema-cache query found `shifts` as separate table |
| Section 15.4: `attendance_logs` table | Correct in types |
| No `hardware_stores` table in any section | Seeder initially tried this — already handled via kvSet |
| No `maintenance_schedules` in types | Listed in schema-summary.md Section 15.11 — may exist but not in generated types |

---

## 6. Migration Status

**Could not query** `supabase_migrations.schema_migrations` — the schema is not accessible via Supabase REST API.

**19 migration files exist** in the codebase (per previous audit). Whether all 19 were applied to the live database cannot be confirmed via REST API. The Supabase Dashboard SQL Editor or `psql` direct connection would be needed.

**Orphan tables detected:** Tables like `shifts`, `attendance_records`, `wallets`, `purchase_orders`, `po_line_items`, `defects`, `pmc_orgs`, `hardware_stores`, `hardware_products` exist in PostgreSQL but NOT in `database.types.ts`. These are likely from migrations that were partially written or applied with different names than what the codebase expects. 10 phantom tables removed 2026-03-25 (Session 6A): `unit_residents`, `otp_codes`, `push_subscriptions` (corrected to `push_notification_subscriptions`), `ai_memory`, `announcements`, `bank_accounts`, `insurance_policies`, `building_documents`, `committee_members`, `meeting_minutes`.

---

## 7. CHECK Constraints (non-trivial, seed-relevant tables)

| Table | Constraint |
|---|---|
| `building_org_assignments` | `status IN ('active', 'inactive')` |
| `cases` | `priority IN ('low', 'normal', 'high', 'emergency')` |
| `collection_accounts` | `escalation_stage >= 0 AND escalation_stage <= 5` |
| `collection_escalations` | `triggered_by IN ('system', 'bm', 'payment', 'appeal')` |
| `collection_transactions` | `transaction_type IN ('charge', 'payment', 'waiver', 'penalty', 'adjustment')` |
| `contractor_orgs` | Security must be solo array; cleaning+landscape can combine; others cannot include security/cleaning/landscape |
| `contractor_staff` | `role IN ('admin', 'supervisor', 'worker', 'guard', 'cleaner')` |
| `renovation_applications` | `status IN ('pending', 'approved', 'rejected', 'in_progress', 'completed', 'overrun')` |
| `supplier_building_relationships` | `status IN ('approved', 'pending', 'blacklisted')` |

---

## 8. UNIQUE Constraints (seed-relevant)

| Table | Unique On |
|---|---|
| `cases` | `case_number` |
| `collection_accounts` | `unit_id` |
| `contractor_org_building_assignments` | `contractor_org_id, building_id` |
| `developer_orgs` | `ssm_number` |
| `platform_settings` | `key` |
| `procurement_orders` | `order_number` |
| `supplier_building_relationships` | `supplier_id, building_id` |
| `supplier_credit_accounts` | `supplier_id` |
| `units` | `unit_number, building_id` |
| `user_roles` | `user_id, building_id, role` |
| `users` | `phone` |
| `users` | `telegram_id` |
| `vendor_wallets` | `org_id` |
| `visitors` | `qr_code` |

---

## 9. Required Columns (NOT NULL, no default)

| Table | Must Provide |
|---|---|
| `attendance_logs` | building_id, contractor_staff_id |
| `building_org_assignments` | building_id, org_id |
| `buildings` | address, name |
| `cases` | building_id, case_number, title |
| `collection_accounts` | building_id, unit_id |
| `collection_escalations` | account_id, building_id, from_stage, to_stage, triggered_by |
| `collection_transactions` | account_id, amount, building_id, transaction_type |
| `contractor_org_building_assignments` | building_id, contractor_org_id |
| `contractor_orgs` | contact_phone, name |
| `contractor_shifts` | building_id, contractor_staff_id, end_time, shift_date, start_time |
| `contractor_staff_building_assignments` | building_id, contractor_staff_id (contractor_org_id nullable since mig 061) |
| `contractor_staff` | full_name, phone |
| `dev_buildings` | building_name, development_id, dlp_end_date, total_units, vp_date |
| `dev_defects` | category, description, dev_unit_id, pin_x, pin_y, submitted_by_phone, title |
| `dev_rectification_contractors` | contractor_name, contractor_phone, developer_org_id |
| `dev_unit_types` | development_id, type_name |
| `dev_units` | dev_building_id, unit_number, unit_type_id |
| `developer_orgs` | company_name, pic_name, pic_phone, ssm_number |
| `developments` | address, developer_org_id, development_name, dlp_end_date, total_units, vp_date |
| `facilities` | building_id, name |
| `facility_bookings` | booking_date, building_id, end_time, facility_id, resident_id, start_time |
| `maintenance_schedules` | building_id, frequency, next_due_date, title |
| `organisations` | name, org_type |
| `parcels` | building_id, unit_id |
| `platform_settings` | key, value |
| `procurement_order_lines` | item_name, order_id, quantity, unit_price |
| `procurement_orders` | building_id, order_number, supplier_id |
| `renovation_applications` | building_id, resident_id, scope_of_work, unit_id |
| `residents` | building_id, full_name, phone, unit_id |
| `service_visits` | building_id, contractor_org_id, scheduled_date, service_type |
| `supplier_building_relationships` | building_id, supplier_id |
| `supplier_credit_accounts` | supplier_id |
| `suppliers` | company_name, contact_phone |
| `tender_bids` | contractor_org_id, quote_amount, tender_id |
| `tenders` | building_id, posted_by, title |
| `units` | building_id, unit_number |
| `user_roles` | building_id, role, user_id |
| `users` | phone |
| `vendor_wallets` | org_id |
| `visitors` | building_id, visitor_name |
| `wallet_transactions` | amount, type, wallet_id |

---

## 10. All ENUM Types (complete)

| Enum | Values |
|---|---|
| `booking_status_type` | pending, confirmed, cancelled, completed |
| `building_status_type` | trial, active, suspended, churned |
| `case_category_type` | maintenance, complaint, enquiry, emergency, noise, renovation, collection, security, facility, other |
| `case_status_type` | open, assigned, in_progress, pending_approval, resolved, closed, escalated |
| `contract_type` | contracted, adhoc |
| `dev_building_status` | pre_vp, dlp_active, dlp_expired, handed_over |
| `dev_defect_category` | structural, plumbing, electrical, waterproofing, painting, carpentry, tiling, other |
| `dev_defect_status` | submitted, acknowledged, assigned, in_progress, rectified, verified, disputed, closed |
| `dev_development_status` | active, dlp_expired, handed_over |
| `dev_rectification_status` | active, inactive |
| `dev_unit_status` | pending_vp, occupied, defect_open, clear |
| `developer_org_status` | pending, approved, rejected |
| `org_type` | management, contractor, supplier |
| `procurement_status_type` | draft, submitted, acknowledged, delivered, invoiced, paid, cancelled |
| `resident_role_type` | owner, tenant, family, occupant |
| `service_visit_status_type` | scheduled, in_progress, completed, missed, rescheduled |
| `tender_status_enum` | draft, open, quoting, awarded, completed, cancelled |
| `tender_type_enum` | building, resident |
| `user_role_type` | superadmin, bm, staff, committee, contractor_admin, contractor_supervisor, contractor_worker, guard, supplier, resident, owner, tenant |
| `wallet_tx_type_enum` | topup, bid_deduct, refund, admin_adjust |
| `whatsapp_status_type` | not_connected, pending, active, suspended |

---

## V4 Tables (added by V4 migrations)

## §Table-sender_profiles
### sender_profiles (V4-T001, migration 024)
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings),
phone (TEXT), email (TEXT), telegram_id (TEXT),
sender_type (sender_type_enum NOT NULL DEFAULT 'unknown'),
user_id (UUID FK→users), resident_id (UUID FK→residents),
state_json (JSONB NOT NULL DEFAULT '{}'),
version (INTEGER NOT NULL DEFAULT 0),
last_inbound_at (TIMESTAMPTZ),
ai_paused (BOOLEAN NOT NULL DEFAULT false),  -- D-0453 migration 067 — manual pause (BM toggle, no TTL)
ai_paused_until (TIMESTAMPTZ),  -- D-0736 migration 104 — auto-pause TTL (set by BM outbound, clears after 4h)
display_name (TEXT),  -- D-0455 migration 068 — shown in console Contact view
classification (TEXT NOT NULL DEFAULT 'unknown'),  -- D-0455 migration 068 — unknown/saved/external/resident
company (TEXT),  -- D-0480 migration 069 — company/label for external contacts
created_at (TIMESTAMPTZ), updated_at (TIMESTAMPTZ)
```
**Unique:** (building_id, phone) WHERE phone IS NOT NULL; (phone, building_id) WHERE phone IS NOT NULL (D-0631 migration 088); (building_id, telegram_id) WHERE telegram_id IS NOT NULL; (building_id, email) WHERE email IS NOT NULL
**CHECK:** phone IS NOT NULL OR email IS NOT NULL OR telegram_id IS NOT NULL (no zombie profiles)
**Enum:** `sender_type_enum` = resident, contractor, supplier, committee, staff, unknown
**Classification values:** unknown (new/unnamed), saved (BM named but unclassified), external (external contact with optional company — D-0480), resident (linked to unit via resident_id)
**Purpose:** AI memory state machine for ALL message senders across all channels (D-101)
**state_json schema:** identity, behavioral_state, active_issues[], resolved_history_tags[], payment, preferences, interaction_stats, flags

## §Table-document_chunks
### document_chunks (V4-T008, migration 027)
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings),
document_id (UUID NOT NULL), chunk_index (INTEGER NOT NULL),
content (TEXT NOT NULL), embedding (vector(1536)),
content_tsv (tsvector GENERATED from content),
document_tier (INTEGER NOT NULL CHECK 0-5),
date_effective (TIMESTAMPTZ), date_superseded (TIMESTAMPTZ),
section_ref (TEXT), document_type (TEXT), document_title (TEXT),
page_number (INTEGER), is_active (BOOLEAN DEFAULT true),
model_version (TEXT DEFAULT 'text-embedding-3-small'),
created_at (TIMESTAMPTZ)
```
**Indexes:** HNSW on embedding (vector_cosine_ops), GIN on content_tsv, B-tree on building_id, document_id, document_tier, is_active, document_type
**RPC:** search_document_chunks(building_id, query_embedding, query_text, vector_weight, keyword_weight, limit, min_tier, max_tier) — D-113 hybrid search
**Purpose:** RAG pipeline embedded chunks with hybrid vector+full-text search (D-106)

## §Table-tickets
### tickets (V4-T029, migration 028)
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings),
channel (TEXT NOT NULL CHECK whatsapp/telegram/email/walk_in/phone_call/web),
conversation_id (UUID), resident_id (UUID FK→residents), unit_id (UUID FK→units),
sender_profile_id (UUID FK→sender_profiles),
category (TEXT), subject (TEXT), last_message_preview (TEXT),
status (TEXT NOT NULL DEFAULT 'open' CHECK open/in_progress/waiting/resolved/closed),
priority (TEXT NOT NULL DEFAULT 'normal' CHECK low/normal/high/critical),
assigned_to (UUID FK→users),
ai_confidence (FLOAT), ai_can_handle (BOOLEAN DEFAULT true),
ai_resolution (TEXT), ai_tier (INTEGER),
resolved_at (TIMESTAMPTZ), resolved_by (TEXT),
case_id (UUID FK→cases),
created_at (TIMESTAMPTZ), updated_at (TIMESTAMPTZ)
```
**Indexes:** building_id, status, assigned_to, resident_id, (building_id, status, created_at DESC) for feed
**Realtime:** Enabled for LIVE feed updates
**Purpose:** BM Console tickets — drives AI Feed (LIVE/NEEDS ME/DONE). D-109.

### buildings — V4 columns added (migration 029)
```
minilab_email (TEXT, UNIQUE), email_forwarding_from (TEXT),
email_forwarding_verified (BOOLEAN DEFAULT false), email_auto_reply_text (TEXT)
```

## §Table-email_log
### email_log (V4-T033, migration 029)
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings),
direction (TEXT CHECK inbound/outbound),
from_address (TEXT NOT NULL), to_address (TEXT NOT NULL),
subject (TEXT), reply_to (TEXT),
message_id (TEXT), in_reply_to (TEXT),
body_text (TEXT), body_html (TEXT),
attachments (JSONB DEFAULT []),
processed (BOOLEAN DEFAULT false),
ticket_id (UUID), sender_profile_id (UUID FK→sender_profiles),
resend_id (TEXT), resend_status (TEXT),
created_at (TIMESTAMPTZ)
```
**Purpose:** Email audit trail — inbound (Cloudflare) and outbound (Resend). D-108.

## §Table-staff_permissions
### staff_permissions (V4-T041, migration 030)
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings),
user_id (UUID NOT NULL FK→users), permission (TEXT NOT NULL),
allowed (BOOLEAN DEFAULT false), granted_by (UUID FK→users),
created_at (TIMESTAMPTZ), updated_at (TIMESTAMPTZ)
```
**Unique:** (building_id, user_id, permission)
**RPC:** check_permission(user_id, building_id, permission) → BOOLEAN
**Purpose:** Granular RBAC per staff per building. D-110.

## §Table-permission_audit_log
### permission_audit_log (V4-T048, migration 030)
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings),
actor_user_id (UUID NOT NULL FK→users), target_user_id (UUID NOT NULL FK→users),
action (TEXT CHECK grant/revoke/bulk_apply_template),
permission (TEXT), template_name (TEXT), details (JSONB),
created_at (TIMESTAMPTZ)
```
**Purpose:** Audit trail for all permission changes.

## §Table-blocks
### blocks (Foundation restructure, migration 033)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK->buildings ON DELETE CASCADE),
name (TEXT NOT NULL),
block_type (TEXT NOT NULL DEFAULT 'highrise' CHECK highrise/landed/commercial/villa/mixed),
total_floors (INTEGER DEFAULT 1),
total_units (INTEGER DEFAULT 0),
image_url (TEXT),
sort_order (INTEGER DEFAULT 0),
is_active (BOOLEAN DEFAULT true),
metadata (JSONB DEFAULT '{}'),
created_at (TIMESTAMPTZ DEFAULT now()),
updated_at (TIMESTAMPTZ DEFAULT now())
```
**Unique:** (building_id, name)
**Index:** idx_blocks_building_id on building_id
**metadata keys:** has_ground_floor (boolean), floor_labels (Record<string, string> — key is floor number/G, value is label)
**Purpose:** Strict hierarchy: building -> blocks -> units. Every unit belongs to a block. Single buildings get auto-generated default block. block_type distinguishes highrise vs landed vs commercial etc.

### units — column added (migration 033)
```
block_id (UUID NOT NULL FK->blocks)
```
**Note:** block_id is NOT NULL. All existing units were migrated to default blocks during migration. Dual-write: both building_id and block_id set on insert during transition period.

## §Table-raw_imports
### raw_imports (File import staging, migration 045)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK->buildings),
uploaded_by (UUID NOT NULL FK->users),
file_name (TEXT NOT NULL),
file_type (TEXT NOT NULL),           -- 'csv' | 'xlsx'
file_size_bytes (INTEGER),
row_count (INTEGER NOT NULL),
headers (TEXT[] NOT NULL),
sample_rows (JSONB NOT NULL),
raw_data (JSONB NOT NULL),
column_mapping (JSONB),
mapping_confidence (FLOAT),
mapping_confirmed (BOOLEAN DEFAULT false),
detected_blocks (JSONB),
block_mapping_confirmed (BOOLEAN DEFAULT false),
merge_group_id (UUID),
merge_priority (INTEGER DEFAULT 0),
merge_strategy (JSONB),
status (TEXT NOT NULL DEFAULT 'uploaded' CHECK uploaded/mapped/confirmed/importing/completed/failed/cancelled),
import_result (JSONB),
error_message (TEXT),
created_at (TIMESTAMPTZ DEFAULT now()),
updated_at (TIMESTAMPTZ DEFAULT now())
```
**Index:** idx_raw_imports_building on (building_id, status), idx_raw_imports_merge on merge_group_id WHERE NOT NULL
**RLS:** own building via user_roles + service_role policy
**Purpose:** Staging table for file imports. BM uploads file → code parses into staging → AI maps columns (headers + 5 rows only) → BM confirms → code executes bulk import to real tables. Multi-file merge via merge_group_id.

### buildings — columns added (migrations 029, 033)
```
building_image (TEXT) — URL to building photo in Supabase Storage building-images bucket
minilab_email (TEXT UNIQUE) — building-specific email for inbound email processing
email_forwarding_from (TEXT)
email_forwarding_verified (BOOLEAN DEFAULT false)
email_auto_reply_text (TEXT)
```

## §Table-building_applications
### building_applications (migration 031, updated migration 049)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_name (TEXT NOT NULL),
address (TEXT NOT NULL),
city (TEXT), state (TEXT), postcode (TEXT),
total_units (INTEGER),
chairman_name (TEXT NOT NULL),
chairman_phone (TEXT NOT NULL),
chairman_email (TEXT NOT NULL),
bm_name (TEXT NOT NULL),
bm_phone (TEXT NOT NULL),
bm_email (TEXT NOT NULL),
jmb_cert_url (TEXT),
status (TEXT NOT NULL DEFAULT 'pending' CHECK pending/approved/rejected),
rejection_reason (TEXT),
reviewed_by (UUID),
reviewed_at (TIMESTAMPTZ),
login_slug (TEXT), password_hash (TEXT),
created_at (TIMESTAMPTZ DEFAULT now()),
updated_at (TIMESTAMPTZ DEFAULT now())
```
**RLS:** Anyone can INSERT. Service role has full access.
**Purpose:** Pending building registrations. Superadmin approves/rejects. On approval, real building record created with trial status.

## §Table-hardware_store_applications
### hardware_store_applications (migration 031)
```
id (UUID PK DEFAULT gen_random_uuid()),
store_name (TEXT NOT NULL),
owner_name (TEXT NOT NULL),
owner_phone (TEXT NOT NULL),
owner_email (TEXT NOT NULL),
registration_number (TEXT),
address (TEXT NOT NULL),
city (TEXT), state (TEXT), postcode (TEXT),
territory_preference (TEXT),
business_cert_url (TEXT),
status (TEXT NOT NULL DEFAULT 'pending' CHECK pending/approved/rejected),
rejection_reason (TEXT),
reviewed_by (UUID),
reviewed_at (TIMESTAMPTZ),
created_at (TIMESTAMPTZ DEFAULT now()),
updated_at (TIMESTAMPTZ DEFAULT now())
```
**Index:** idx_store_applications_status on status
**RLS:** Anyone can INSERT. Service role has full access.
**Purpose:** Pending hardware store registrations. Same approval pattern as building_applications.

## §Table-building_onboarding
### building_onboarding (migration 031)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL UNIQUE FK->buildings ON DELETE CASCADE),
units_added (BOOLEAN DEFAULT false),
staff_added (BOOLEAN DEFAULT false),
vendors_connected (BOOLEAN DEFAULT false),
residents_added (BOOLEAN DEFAULT false),
documents_uploaded (BOOLEAN DEFAULT false),
communications_setup (BOOLEAN DEFAULT false),
payments_setup (BOOLEAN DEFAULT false),
completed_at (TIMESTAMPTZ),
created_at (TIMESTAMPTZ DEFAULT now()),
updated_at (TIMESTAMPTZ DEFAULT now())
```
**Purpose:** Tracks onboarding wizard progress per building. 7 boolean steps. completed_at set when all steps done.

## §Table-form_templates
### form_templates (migration 034)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK->buildings ON DELETE CASCADE),
name (TEXT NOT NULL),
slug (TEXT NOT NULL),
category (TEXT NOT NULL DEFAULT 'predefined' CHECK predefined/custom),
description (TEXT),
fields (JSONB NOT NULL DEFAULT '[]'),
on_approve (JSONB DEFAULT '{}'),
is_active (BOOLEAN DEFAULT true),
sort_order (INTEGER DEFAULT 0),
created_at (TIMESTAMPTZ DEFAULT now()),
updated_at (TIMESTAMPTZ DEFAULT now())
```
**Unique:** (building_id, slug)
**Indexes:** building_id, (building_id, slug)
**Purpose:** Smart Forms engine. 10 predefined templates installed per building. Fields JSONB stores form field definitions with smart types (unit_select, resident_select, etc.). on_approve JSONB configures what happens on approval (create_record, create_task, notify_resident, update_vms).

---

## §Table-petty_cash
### petty_cash (migration 037)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
type (TEXT NOT NULL CHECK: opening_balance|issue|expense|reimbursement|top_up),
amount (NUMERIC(10,2) NOT NULL DEFAULT 0),
description (TEXT),
category (TEXT — meals, transport, supplies, utilities, misc),
staff_id (UUID FK→users — who holds/spent the cash),
receipt_url (TEXT — photo evidence),
balance_after (NUMERIC(10,2) — running balance snapshot),
recorded_by (UUID FK→users — who entered it),
telegram_message_id (TEXT — if AI-captured from Telegram group),
created_at (TIMESTAMPTZ DEFAULT now())
```
**Indexes:** building_id, staff_id, created_at DESC
**RLS:** Enabled
**Purpose:** Petty cash ledger. Immutable-amount entries (only description/category/receipt editable after creation). Credit types (opening_balance, top_up, reimbursement) increase balance; debit types (issue, expense) decrease it. balance_after is computed at creation time.

---

## 11. Column Type Notes (data type pitfalls)

| Table | Column | DB Type | Notes |
|---|---|---|---|
| `contractor_shifts` | `start_time`, `end_time` | `time` | Use `'HH:MM:SS'` not full timestamp |
| `contractor_shifts` | `shift_date` | `date` | Use `'YYYY-MM-DD'` |
| `facility_bookings` | `start_time`, `end_time` | `time` | Use `'HH:MM:SS'` |
| `facility_bookings` | `booking_date` | `date` | Use `'YYYY-MM-DD'` |
| `developments` | `vp_date`, `dlp_end_date` | `date` | Use `'YYYY-MM-DD'` |
| `dev_buildings` | `vp_date`, `dlp_end_date` | `date` | Use `'YYYY-MM-DD'` |
| `service_visits` | `scheduled_date` | `date` | Use `'YYYY-MM-DD'` |
| `renovation_applications` | `start_date`, `end_date` | `date` | Use `'YYYY-MM-DD'` |
| `maintenance_schedules` | `next_due_date` | `date` | Use `'YYYY-MM-DD'` |

---

## 12. Previously Undocumented Tables (full column definitions)

## §Table-agm_cards
### agm_cards

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| session_id | uuid | NO | null | FK→agm_sessions |
| card_number | text | NO | | |
| card_color | USER-DEFINED | NO | | enum |
| qr_data | text | NO | | |
| is_assigned | boolean | YES | false | |
| assigned_to_eligible_id | uuid | YES | | FK→agm_eligible |
| created_at | timestamptz | YES | now() | |

## §Table-agm_egm_meetings
### agm_egm_meetings

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| meeting_type | text | YES | 'agm' | |
| meeting_date | date | NO | | |
| venue | text | YES | | |
| agenda | jsonb | YES | | |
| quorum_required | integer | YES | | |
| quorum_achieved | boolean | YES | | |
| resolutions | jsonb | YES | | |
| minutes_document_url | text | YES | | |
| created_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-agm_eligible
### agm_eligible

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| session_id | uuid | NO | | FK→agm_sessions |
| unit_number | text | NO | | |
| name | text | NO | | |
| ic_number | text | YES | | |
| entity | text | NO | | |
| vote_by_hand | integer | NO | 1 | |
| vote_by_poll | integer | NO | 1 | |
| card_id | uuid | YES | | FK→agm_cards |
| checked_in | boolean | YES | false | |
| checked_in_at | timestamptz | YES | | |
| checked_in_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |

## §Table-agm_motions
### agm_motions

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| session_id | uuid | NO | | FK→agm_sessions |
| motion_number | integer | NO | | |
| motion_text | text | NO | | |
| options | jsonb | YES | '["For","Against","Abstain"]' | |
| vote_mode | USER-DEFINED | NO | | enum |
| status | USER-DEFINED | YES | 'draft'::agm_motion_status | enum |
| opened_at | timestamptz | YES | | |
| closed_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-agm_proxies
### agm_proxies

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| meeting_id | uuid | NO | | FK→agm_egm_meetings |
| resident_id | uuid | NO | | FK→residents |
| unit_id | uuid | NO | | FK→units |
| proxy_holder_resident_id | uuid | YES | | FK→residents |
| proxy_form_url | text | YES | | |
| votes_granted | integer | YES | 1 | |
| submitted_at | timestamptz | YES | now() | |
| created_at | timestamptz | YES | now() | |

## §Table-agm_sessions
### agm_sessions

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| session_name | text | NO | | |
| session_type | USER-DEFINED | NO | | enum |
| date | date | NO | | |
| venue | text | YES | | |
| status | USER-DEFINED | YES | 'draft'::agm_session_status | enum |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-agm_votes
### agm_votes

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| motion_id | uuid | NO | | FK→agm_motions |
| card_id | uuid | NO | | FK→agm_cards |
| eligible_id | uuid | NO | | FK→agm_eligible |
| vote_value | text | NO | | |
| vote_weight | integer | NO | 1 | |
| scanned_by | uuid | NO | | FK→users |
| scanned_at | timestamptz | NO | now() | |
| unit_id | uuid | YES | | FK→units |

## §Table-ai_tools
### ai_tools (migration 043)

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| scope | text | NO | | 'resident' or 'admin' |
| name | text | NO | | UNIQUE |
| category | text | NO | | admin_settings, admin_people, etc. |
| description | text | NO | | |
| keywords | text[] | NO | '{}' | EN + Malay keywords |
| handler_name | text | NO | | maps to TypeScript handler |
| endpoint | text | YES | | documentation only |
| method | text | YES | | documentation only |
| parameters | jsonb | NO | '{}' | JSON Schema |
| response_format | text | YES | | |
| requires_confirm | boolean | YES | false | |
| min_role | text | NO | 'staff' | resident/staff/committee/bm/superadmin |
| is_active | boolean | YES | true | |
| examples | text[] | YES | '{}' | example trigger messages |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-platform_issues
### platform_issues (migration 044)

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| reported_by | uuid | NO | | FK→users |
| title | text | NO | | |
| description | text | YES | | |
| screenshot_url | text | YES | | |
| category | text | NO | 'bug' | bug/feature_request/question |
| status | text | NO | 'pending' | pending/in_progress/solved/rejected |
| priority | text | NO | 'normal' | low/normal/high/urgent |
| admin_response | text | YES | | superadmin reply |
| admin_responded_by | uuid | YES | | FK→users |
| admin_responded_at | timestamptz | YES | | |
| resolved_at | timestamptz | YES | | |
| source_message_id | uuid | YES | | link to chat message |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

**Indexes:** (building_id, status), status partial WHERE IN ('pending','in_progress'), reported_by
**RLS:** read_own_building (SELECT via user_roles), insert (building scope), service_role full access.
**Purpose:** Cross-building bug/feature tracking for superadmin. D-0227.

### messages — columns added (migration 044)
```
ai_tool_calls (JSONB), conversation_thread_id (UUID)
```
**Enum update:** message_channel_type now includes 'web_chat' (migration 044) and 'internal_note' (migration 067, D-0449).
**Index:** idx_messages_web_chat on (building_id, created_at DESC) WHERE channel = 'web_chat'.

### messages — columns added (migration 074)
```
sender_profile_id (UUID FK→sender_profiles, nullable) — links message to sender profile for console contact-messages query
```
**Index:** idx_messages_sender_profile_id on (sender_profile_id).
**D-0531:** Root cause of "No messages yet" in console — contact-messages API queried this column but it didn't exist.

### messages — columns added (migration 048)
```
status (TEXT, default 'complete') — message processing state: pending → processing → complete / failed
```
**Index:** idx_messages_status_pending on (status, created_at) WHERE status IN ('pending', 'processing').
**Realtime:** messages table added to supabase_realtime publication.

## §Table-api_cost_alerts
### api_cost_alerts

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| date | date | YES | CURRENT_DATE | |
| daily_cost_usd | numeric | YES | 0 | |
| seven_day_avg_usd | numeric | YES | 0 | |
| alert_fired | boolean | YES | false | |
| alert_fired_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-approvals
### approvals

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| case_id | uuid | YES | | FK→cases |
| approval_type | text | NO | | |
| requested_by | uuid | YES | | FK→users |
| assigned_to | uuid | YES | | FK→users |
| status | USER-DEFINED | YES | 'pending'::approval_status_type | enum |
| request_data | jsonb | YES | | |
| reviewed_by | uuid | YES | | FK→users |
| reviewed_at | timestamptz | YES | | |
| review_note | text | YES | | |
| telegram_message_id | bigint | YES | | |
| unit_id | uuid | YES | | FK→units — scope approvals per unit (Migration 080) |
| sender_profile_id | uuid | YES | | FK→sender_profiles — links payment approvals to chat sender (Migration 081) |
| expires_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |
| extracted_amount | numeric | YES | | D-0746: Gemini-extracted payment amount from receipt |
| extracted_date | date | YES | | D-0746: Gemini-extracted payment date (ISO) |
| extracted_reference | text | YES | | D-0746: Gemini-extracted transaction/reference number |
| extracted_bank | text | YES | | D-0746: Gemini-extracted originating bank name |
| extracted_account_number | text | YES | | D-0746: Gemini-extracted payer account number — PII, RLS-gated, never logged |
| extracted_account_holder | text | YES | | D-0746: Gemini-extracted payer name on receipt — PII, RLS-gated, never logged |
| extracted_payment_type | text | YES | | D-0746: duitnow \| ibg \| ibft \| fpx \| cash \| cheque \| transfer |
| extraction_confidence | numeric | YES | | D-0746: Gemini confidence 0.0–1.0 |
| extraction_status | text | NO | 'pending' | D-0746: pending \| done \| failed \| skipped |
| extraction_error | text | YES | | D-0746: Gemini error message if failed (max 500 chars) |
| extraction_completed_at | timestamptz | YES | | D-0746: When extraction finished (success or fail) |

## §Table-asset_logs
### asset_logs

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| asset_id | uuid | NO | | FK→assets |
| scanned_by | uuid | YES | | FK→users |
| action_type | USER-DEFINED | NO | | enum |
| condition | USER-DEFINED | NO | | enum |
| notes | text | YES | | |
| photos | ARRAY | YES | | |
| next_service_date | date | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-assets
### assets

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| asset_name | text | NO | | |
| category | text | NO | | |
| location | text | NO | | |
| qr_code | uuid | NO | gen_random_uuid() | |
| serial_number | text | YES | | |
| purchase_date | date | YES | | |
| warranty_expiry | date | YES | | |
| is_compliance | boolean | YES | false | |
| status | text | NO | 'active' | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-audit_log
### audit_log

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | YES | | FK→users |
| action | text | NO | | |
| resource_type | text | NO | | |
| resource_id | text | YES | | |
| details | jsonb | YES | '{}' | |
| ip_address | text | YES | | |
| building_id | uuid | YES | | FK→buildings (migration 046) |
| source | text | NO | 'web' | web/ai_chat/telegram/api/cron/system (migration 046) |
| old_data | jsonb | YES | | Before state (migration 046) |
| new_data | jsonb | YES | | After state (migration 046) |
| created_at | timestamptz | NO | now() | |

## §Table-bid_pricing_tiers
### bid_pricing_tiers

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| budget_min | numeric | NO | | |
| budget_max | numeric | YES | | |
| bid_cost | numeric | NO | | |
| created_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-building_gates
### building_gates

Multi-gate VMS kiosk system (D-0442). Each building can have multiple gates, each with its own VMS kiosk passcode. Guard enters passcode at /guard → looks up building_gates → activates VMS for that specific gate. Managed via BM Settings → Apps & Access.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| name | text | NO | | e.g. "Main Gate", "Back Gate" |
| gate_type | text | NO | 'vehicle' | vehicle/pedestrian/loading/emergency |
| vms_passcode | text | YES | | 6-digit code, unique per gate |
| is_active | boolean | NO | true | |
| created_at | timestamptz | NO | now() | |
| updated_at | timestamptz | NO | now() | |

## §Table-building_geofences
### building_geofences

Actively used for attendance clock-in/out validation (D-0373). Enforced when is_active=true — no GPS = 403 error. Allow freely when no record configured (backwards compatible). Haversine distance calc in lib/geo/haversine.ts. Configured via /bm/settings/profile (merged from standalone /bm/settings/location in D-0380).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| name | text | NO | | |
| center_lat | numeric | NO | | |
| center_lng | numeric | NO | | |
| radius_meters | integer | YES | 100 | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-building_org_assignments
### building_org_assignments

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| org_id | uuid | NO | | FK→organisations |
| status | text | YES | 'active' | |
| assigned_at | timestamptz | YES | now() | |
| ended_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-building_payment_settings
### building_payment_settings

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| payment_mode | text | NO | 'bank_transfer' | |
| bank_name | text | YES | | |
| bank_account_number | text | YES | | |
| bank_account_name | text | YES | | |
| billplz_api_key | text | YES | | |
| billplz_collection_id | text | YES | | |
| billplz_webhook_secret | text | YES | | |
| auto_restore_access | boolean | YES | false | |
| late_penalty_percent | numeric | YES | 0 | |
| grace_period_days | integer | YES | 14 | |
| receipt_logo_url | text | YES | | |
| receipt_footer_text | text | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-chase_loops
### chase_loops

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| case_id | uuid | YES | | FK→cases |
| building_id | uuid | NO | | FK→buildings |
| contractor_org_id | uuid | YES | | FK→contractor_orgs |
| status | text | NO | 'active' | |
| attempts | integer | NO | 0 | |
| last_followup | timestamptz | YES | | |
| next_followup | timestamptz | YES | | |
| started_at | timestamptz | NO | now() | |
| notes | text | YES | | |
| created_at | timestamptz | NO | now() | |
| updated_at | timestamptz | NO | now() | |

## §Table-collection_settings
### collection_settings

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| monthly_maintenance_fee | numeric | YES | 200 | |
| sinking_fund_fee | numeric | YES | 50 | |
| late_penalty_pct | numeric | YES | 10 | |
| reminder_day_1 | integer | YES | 14 | |
| reminder_day_2 | integer | YES | 30 | |
| form11_threshold_days | integer | YES | 90 | |
| access_block_threshold_months | integer | YES | 3 | |
| block_minimum_amount | numeric | YES | 0 | Migration 087: min outstanding before Stage 4/5 blocking fires; 0 = block at any amount |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-contact_channels
### contact_channels

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| resident_id | uuid | NO | | FK→residents |
| channel_type | USER-DEFINED | NO | | enum |
| channel_id | text | NO | | |
| is_primary | boolean | YES | false | |
| is_verified | boolean | YES | false | |
| preference_rank | integer | YES | 1 | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-contacts_temp
### contacts_temp

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| phone | text | YES | | D-0607: nullable for email-only reg |
| full_name | text | YES | | |
| message_preview | text | YES | | |
| channel | text | YES | 'whatsapp' | D-0607: whatsapp/telegram/email |
| channel_id | text | YES | | D-0607: phone/telegram_id/email |
| first_contact_at | timestamptz | YES | now() | |
| reviewed_by | uuid | YES | | FK→users |
| review_action | text | YES | 'pending' | |
| reviewed_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

CHECK: phone IS NOT NULL OR channel_id IS NOT NULL

## §Table-ai_pipeline_runs
### ai_pipeline_runs

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| created_at | timestamptz | NO | now() | |
| building_id | uuid | NO | | FK→buildings ON DELETE CASCADE |
| message_id | uuid | YES | | FK→messages ON DELETE SET NULL |
| sender_profile_id | uuid | YES | | FK→sender_profiles ON DELETE SET NULL |
| resident_id | uuid | YES | | FK→residents ON DELETE SET NULL |
| sender_ref | text | YES | | phone / telegram_id / email (pre-reg tracking) |
| sender_ref_type | text | YES | | 'phone' \| 'telegram_id' \| 'email' |
| channel | text | NO | | 'whatsapp' \| 'telegram' \| 'email' \| 'web_chat' |
| entry_point | text | NO | | 'wa_webhook' \| 'tg_webhook_path1' \| 'tg_webhook_path2' \| 'bm_chat_background' \| 'email_processor' \| 'auto_register' \| 'nudge_cron' |
| tier | int | YES | | 1-4 (null if skipped before pipeline) |
| handler | text | YES | | e.g. direct_balance_check |
| intent | text | YES | | e.g. MAINTENANCE_COMPLAINT |
| confidence | numeric(4,3) | YES | | 0.000-1.000 |
| had_response | boolean | YES | | true if AI sent reply |
| skip_reason | text | YES | | see SKIP_REASONS enum in pipeline-run-logger.ts |
| skip_reason_meta | jsonb | YES | | extra context for skip (PII may be present) |
| classifier_ms | int | YES | | classifier latency |
| rag_ms | int | YES | | RAG retrieval latency |
| specialist_ms | int | YES | | specialist agent latency |
| total_ms | int | YES | | end-to-end pipeline latency |
| prompt_tokens | int | YES | | total input tokens across all LLM calls |
| completion_tokens | int | YES | | total output tokens across all LLM calls |
| model_used | text | YES | | primary model (deepseek-chat / claude-sonnet-*) |
| is_auto_reg | boolean | NO | false | true if entry via auto-registration flow |
| auto_reg_step | text | YES | | 'initial' \| 'awaiting_unit_confirmation' \| 'confirmed' \| 'hallucination_retry' \| ... |
| error_text | text | YES | | error message if pipeline threw (PII may be present) |
| logger_version | int | NO | 1 | bump in pipeline-run-logger.ts when contract changes |
| extra_meta | jsonb | YES | | forward-compat catch-all |

RLS: enabled. Superadmin all, BM read (has_building_access), service insert (true).
Indexes: (building_id, created_at DESC), (message_id) partial, (building_id, skip_reason, created_at DESC) partial, (building_id, created_at DESC) where is_auto_reg=true, (sender_ref, sender_ref_type, created_at DESC) partial.
View: ai_view_ai_pipeline_runs (excludes sender_ref, skip_reason_meta, error_text, extra_meta).
Migration 105: created (V35-B3).

## §Table-ai_actions_log
### ai_actions_log

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| created_at | timestamptz | NO | now() | |
| pipeline_run_id | uuid | NO | | FK→ai_pipeline_runs ON DELETE CASCADE |
| building_id | uuid | NO | | FK→buildings ON DELETE CASCADE (denorm for RLS) |
| action_type | text | NO | | e.g. create_payment_approval, send_message |
| action_index | int | NO | | 0-based order within pipeline run |
| success | boolean | NO | | true if action succeeded |
| data | jsonb | YES | | action payload (may contain PII) |
| error | text | YES | | error message if action failed |
| extra_meta | jsonb | YES | | forward-compat catch-all |

RLS: enabled. Superadmin all, BM read (has_building_access), service insert (true).
Indexes: (pipeline_run_id), (building_id, action_type, created_at DESC), (building_id, created_at DESC) where success=false.
View: ai_view_ai_actions_log (excludes data, error, extra_meta).
Migration 105: created (V35-B3).

## §Table-routing_analytics
### routing_analytics

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | YES | | FK→buildings |
| channel | text | NO | | whatsapp/telegram/email/web |
| sender_type | text | YES | | resident/staff/bm/unknown |
| tier | smallint | NO | | 0-4 |
| handler | text | YES | | e.g. direct_balance_check |
| intent | text | YES | | e.g. MAINTENANCE_COMPLAINT |
| confidence | numeric(5,2) | YES | | 0.00-1.00 |
| processing_time_ms | integer | YES | | |
| model_used | text | YES | | deepseek-chat / claude-sonnet |
| actions_executed | text[] | YES | | action types array |
| had_response | boolean | YES | true | |
| error | text | YES | | |
| retrieved_chunk_count | integer | YES | | V35-W1: chunks from RAG retrieval (Tier 3 only) |
| retrieval_duration_ms | integer | YES | | V35-W1: RAG retrieval time inc. timeout |
| pipeline_run_id | uuid | YES | | FK→ai_pipeline_runs (B3 correlation ID) |
| created_at | timestamptz | YES | now() | |

RLS: enabled. Indexes: (building_id, created_at DESC), (intent, created_at DESC), (pipeline_run_id) partial.
View: ai_view_routing_analytics (30-day window).
Migration 103: added retrieved_chunk_count + retrieval_duration_ms (V35-W1).
Migration 105: added pipeline_run_id FK (V35-B3).

## §Table-contractor_org_building_assignments
### contractor_org_building_assignments

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| contractor_org_id | uuid | NO | | FK→contractor_orgs |
| building_id | uuid | NO | | FK→buildings |
| contract_start_date | date | YES | | |
| contract_end_date | date | YES | | |
| monthly_fee | numeric | YES | | |
| is_active | boolean | YES | true | |
| assigned_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-contractor_staff_building_assignments
### contractor_staff_building_assignments

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| contractor_staff_id | uuid | NO | | FK→contractor_staff |
| building_id | uuid | NO | | FK→buildings |
| contractor_org_id | uuid | YES | | FK→contractor_orgs (nullable — migration 061) |
| is_active | boolean | YES | true | |
| assigned_by | uuid | YES | | FK→users |
| assigned_at | timestamptz | YES | now() | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |
**NOTE:** `contractor_org_id` nullable since migration 061. Unassigned contractor staff get a row here with null contractor_org_id so they appear on the All Staff page. Set to the real org_id when BM assigns via PUT /api/bm/staff/assign-company (D-0389).

## §Table-contractor_visits
### contractor_visits

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| company_name | text | NO | | |
| staff_names | ARRAY | YES | | |
| purpose | text | YES | | |
| arrival_time | timestamptz | YES | now() | |
| departure_time | timestamptz | YES | | |
| linked_case_id | uuid | YES | | FK→cases |
| linked_tender_id | uuid | YES | | FK→tenders |
| completion_photo_url | text | YES | | |
| logged_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |

## §Table-daily_reports
### daily_reports

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| report_date | date | NO | | |
| ai_messages_processed | integer | YES | 0 | |
| ai_cost_rm | numeric | YES | 0 | |
| cases_opened | integer | YES | 0 | |
| cases_resolved | integer | YES | 0 | |
| attendance_rate | numeric | YES | 0 | |
| data | jsonb | YES | '{}' | |
| created_at | timestamptz | NO | now() | |
| updated_at | timestamptz | NO | now() | |

## §Table-data_imports
### data_imports

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| source_type | USER-DEFINED | NO | | enum |
| file_url | text | YES | | |
| status | USER-DEFINED | YES | 'pending'::import_status_type | enum |
| row_count | integer | YES | 0 | |
| success_count | integer | YES | 0 | |
| error_count | integer | YES | 0 | |
| error_log | jsonb | YES | | |
| initiated_by | uuid | YES | | FK→users |
| completed_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-dead_letter_queue
### dead_letter_queue

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| payload | jsonb | NO | '{}' | |
| error | text | YES | | |
| retry_count | integer | NO | 0 | |
| max_retries | integer | NO | 3 | |
| status | text | NO | 'pending' | |
| created_at | timestamptz | NO | now() | |

## §Table-dev_buildings
### dev_buildings

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| development_id | uuid | NO | | FK→developments |
| building_name | text | NO | | |
| total_units | integer | NO | | |
| vp_date | date | NO | | |
| dlp_end_date | date | NO | | |
| status | USER-DEFINED | NO | 'pre_vp'::dev_building_status | enum |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-dev_defect_logs
### dev_defect_logs

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| defect_id | uuid | NO | | FK→dev_defects |
| action | text | NO | | |
| actor_id | uuid | NO | | FK→users |
| notes | text | YES | | |
| photos | ARRAY | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-dev_defects
### dev_defects

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| dev_unit_id | uuid | NO | | FK→dev_units |
| submitted_by_phone | text | NO | | |
| pin_x | numeric | NO | | |
| pin_y | numeric | NO | | |
| room_area | text | YES | | |
| category | USER-DEFINED | NO | | enum |
| title | text | NO | | |
| description | text | NO | | |
| photos | ARRAY | YES | | |
| videos | ARRAY | YES | | |
| status | USER-DEFINED | YES | 'submitted'::dev_defect_status | enum |
| legal_timestamp | timestamptz | YES | now() | |
| immutable | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-dev_handover_checklists
### dev_handover_checklists

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| dev_unit_id | uuid | NO | | FK→dev_units |
| inspector_id | uuid | NO | | FK→users |
| checklist_items | jsonb | NO | | |
| overall_status | USER-DEFINED | NO | | enum |
| owner_signature | text | YES | | |
| completed_at | timestamptz | YES | | |
| pdf_url | text | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-dev_rectification_contractors
### dev_rectification_contractors

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| developer_org_id | uuid | NO | | FK→developer_orgs |
| contractor_name | text | NO | | |
| contractor_phone | text | NO | | |
| specialty | text | YES | | |
| status | USER-DEFINED | YES | 'active'::dev_rectification_status | enum |
| created_at | timestamptz | YES | now() | |

## §Table-dev_unit_types
### dev_unit_types

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| development_id | uuid | NO | | FK→developments |
| type_name | text | NO | | |
| floor_plan_image | text | YES | | |
| floor_plan_width | integer | YES | | |
| floor_plan_height | integer | YES | | |
| unit_count | integer | YES | 0 | |
| created_at | timestamptz | YES | now() | |

## §Table-dev_units
### dev_units

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| dev_building_id | uuid | NO | | FK→dev_buildings |
| unit_type_id | uuid | NO | | FK→dev_unit_types |
| unit_number | text | NO | | |
| owner_name | text | YES | | |
| owner_phone | text | YES | | |
| owner_ic | text | YES | | |
| vp_date | date | YES | | |
| dlp_end_date | date | YES | | |
| status | USER-DEFINED | YES | 'pending_vp'::dev_unit_status | enum |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-developments
### developments

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| developer_org_id | uuid | NO | | FK→developer_orgs |
| development_name | text | NO | | |
| address | text | NO | | |
| total_units | integer | NO | | |
| vp_date | date | NO | | |
| dlp_end_date | date | NO | | |
| building_id | uuid | YES | | FK→buildings |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |
| status | USER-DEFINED | YES | 'active'::dev_development_status | enum |
| buildings_count | integer | YES | 0 | |

## §Table-guard_shifts
### guard_shifts

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| guardhouse_id | uuid | YES | | FK→guardhouses |
| contractor_staff_id | uuid | YES | | FK→contractor_staff |
| shift_date | date | NO | | |
| start_time | time | NO | | |
| end_time | time | NO | | |
| clock_in_at | timestamptz | YES | | |
| clock_out_at | timestamptz | YES | | |
| handover_notes | text | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-guardhouse_incidents
### guardhouse_incidents

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| guardhouse_id | uuid | YES | | FK→guardhouses |
| incident_type | text | NO | | |
| description | text | NO | | |
| photo_url | text | YES | | |
| ai_caption | text | YES | | |
| severity | text | YES | 'low' | |
| reported_by | uuid | YES | | FK→users |
| media_urls | text[] | YES | | Array of media URLs (images/video) |
| voice_note_url | text | YES | | Voice recording URL |
| reported_by_staff_id | uuid | YES | | FK→contractor_staff |
| capture_location_lat | numeric | YES | | GPS latitude |
| capture_location_lng | numeric | YES | | GPS longitude |
| created_at | timestamptz | YES | now() | |

## §Table-guardhouses
### guardhouses

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| name | text | YES | 'Main Guardhouse' | |
| entry_points | ARRAY | YES | | |
| camera_urls | ARRAY | YES | | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-hardware_store_order_items
### hardware_store_order_items

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| order_id | uuid | NO | | FK→hardware_store_orders |
| product_id | uuid | NO | | FK→hardware_store_products |
| product_name | text | NO | | |
| quantity | integer | NO | 1 | |
| unit_price | numeric | NO | | |
| total_price | numeric | NO | | |

## §Table-hardware_store_orders
### hardware_store_orders

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| store_id | uuid | NO | | FK→hardware_store_applications |
| contractor_org_id | uuid | NO | | FK→contractor_orgs |
| ordered_by | uuid | NO | | FK→users |
| status | text | NO | 'pending' | |
| total_amount | numeric | NO | 0 | |
| credits_earned | integer | YES | 0 | |
| notes | text | YES | | |
| delivered_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-hardware_store_products
### hardware_store_products

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| store_id | uuid | NO | | FK→hardware_store_applications |
| name | text | NO | | |
| sku | text | YES | | |
| price | numeric | NO | | |
| category | text | NO | | |
| stock_status | text | YES | 'in_stock' | |
| image_url | text | YES | | |
| description | text | YES | | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-hardware_store_subscriptions
### hardware_store_subscriptions

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| store_id | uuid | NO | | FK→hardware_store_applications |
| amount | numeric | NO | | |
| billing_month | text | NO | | |
| status | text | YES | 'pending' | |
| paid_at | timestamptz | YES | | |
| invoice_number | text | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-in_unit_tender_bids
### in_unit_tender_bids

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| tender_id | uuid | NO | | FK→in_unit_tenders |
| contractor_org_id | uuid | NO | | FK→contractor_orgs |
| bid_amount | numeric | NO | | |
| estimated_hours | numeric | YES | | |
| notes | text | YES | | |
| bid_fee_paid | boolean | YES | false | |
| is_selected | boolean | YES | false | |
| created_at | timestamptz | YES | now() | |

## §Table-in_unit_tenders
### in_unit_tenders

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| unit_id | uuid | NO | | FK→units |
| resident_id | uuid | NO | | FK→residents |
| title | text | NO | | |
| description | text | YES | | |
| job_type | text | YES | | |
| asking_price | numeric | YES | | |
| listing_fee_paid | boolean | YES | false | |
| status | USER-DEFINED | YES | 'open'::tender_status_type | enum |
| expires_at | timestamptz | YES | | |
| awarded_to | uuid | YES | | FK→contractor_orgs |
| awarded_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-inventory_usage
### inventory_usage

**DEPRECATED** — No new writes. Use `job_materials` instead (migration 059 / D-0368). Table kept for historical data and FK integrity during building deletion.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| item_id | uuid | NO | | FK→procurement_items |
| quantity | integer | NO | | |
| location_type | text | NO | | |
| location_ref | text | NO | | |
| case_id | uuid | YES | | FK→cases |
| used_by | uuid | YES | | FK→users |
| telegram_message_id | text | YES | | |
| notes | text | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-investor_leads
### investor_leads

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| full_name | text | NO | | |
| email | text | YES | | |
| phone | text | YES | | |
| company | text | YES | | |
| message | text | YES | | |
| source | text | YES | 'website' | |
| status | text | YES | 'new' | |
| notes | text | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-invitations
### invitations

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| token | text | NO | | |
| invitee_phone | text | YES | | |
| invitee_email | text | YES | | |
| role | text | NO | | |
| invited_by | uuid | YES | | FK→users |
| status | text | NO | 'pending' | |
| expires_at | timestamptz | NO | | |
| accepted_at | timestamptz | YES | | |
| metadata | jsonb | YES | '{}' | |
| created_at | timestamptz | YES | now() | |

## §Table-job_materials
### job_materials

Single source of truth for all material usage (migration 059 / D-0368). Stock = procurement_order_lines IN minus job_materials OUT (WHERE procurement_item_id = item.id). case_id nullable for non-task usage. AI material probe (D-0373) writes here on case resolve — DeepSeek parses technician's Telegram reply, fuzzy-matches against procurement_items, and inserts rows via /api/bm/cases/[id]/materials.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| case_id | uuid | YES | | FK→cases (nullable — non-task usage allowed) |
| procurement_item_id | uuid | YES | | FK→procurement_items (null for custom/ad-hoc items) |
| staff_id | uuid | YES | | FK→contractor_staff |
| item_name | text | NO | | Denormalized for display |
| quantity | numeric | NO | | |
| unit_cost | numeric | NO | | |
| total_cost | numeric | YES | | Computed: quantity × unit_cost |
| supplier_name | text | YES | | |
| delivery_ref | text | YES | | |
| location_type | text | YES | | unit / area / common |
| location_ref | text | YES | | Where item was used |
| notes | text | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-kiosk_devices
### kiosk_devices

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| contractor_org_id | uuid | YES | | FK→contractor_orgs |
| device_name | text | NO | | |
| device_token | text | NO | | |
| qr_code | text | YES | | |
| is_active | boolean | YES | true | |
| last_seen_at | timestamptz | YES | | |
| registered_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-legal_records
### legal_records

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| record_type | text | NO | | |
| title | text | YES | | |
| description | text | YES | | |
| reference_number | text | YES | | |
| unit_id | uuid | YES | | FK→units |
| resident_id | uuid | YES | | FK→residents |
| opposing_party | text | YES | | |
| amount | numeric | YES | | |
| payout_amount | numeric | YES | | |
| current_stage | integer | NO | 0 | |
| stages | jsonb | NO | '[]' | |
| documents | jsonb | YES | '[]' | |
| metadata | jsonb | YES | '{}' | |
| is_closed | boolean | YES | false | |
| closed_at | timestamptz | YES | | |
| created_by | uuid | YES | | FK→users |
| updated_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-maintenance_events
### maintenance_events

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| schedule_id | uuid | NO | | FK→maintenance_schedules |
| building_id | uuid | NO | | FK→buildings |
| due_date | date | NO | | |
| completed_at | timestamptz | YES | | |
| completed_by | uuid | YES | | FK→users |
| photo_urls | ARRAY | YES | | |
| completion_notes | text | YES | | |
| next_due_date | date | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-maintenance_schedules
### maintenance_schedules

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| contractor_org_id | uuid | YES | | FK→contractor_orgs |
| title | text | NO | | |
| description | text | YES | | |
| frequency | text | NO | | |
| next_due_date | date | NO | | |
| assignee_user_id | uuid | YES | | FK→users |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-media_search_index
### media_search_index

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| message_id | uuid | YES | | FK→messages |
| source_type | text | NO | | |
| media_url | text | NO | | |
| ai_caption | text | YES | | |
| embedding | USER-DEFINED | YES | | vector |
| entity_tags | ARRAY | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-messages
### messages

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| case_id | uuid | YES | | FK→cases |
| user_id | uuid | YES | | FK→users |
| resident_id | uuid | YES | | FK→residents |
| channel | USER-DEFINED | NO | | enum |
| direction | USER-DEFINED | NO | | enum |
| content | text | YES | | |
| media_type | text | YES | | |
| media_url | text | YES | | |
| media_mime_type | text | YES | | |
| external_message_id | text | YES | | UNIQUE WHERE NOT NULL — dedup index (D-0431) |
| sender_profile_id | uuid | YES | | FK→sender_profiles |
| status | text | YES | | pending/processing/complete/failed (async pipeline) |
| private | boolean | YES | false | true = internal note (amber card in console, channel='internal_note') D-0434 |
| unit_id | uuid | YES | | FK→units — activity cards scoped to unit (D-0665). idx_messages_unit_id partial index WHERE unit_id IS NOT NULL |
| media_urls | jsonb | YES | | D-0677: multi-image activity cards — array of {url, mime_type, filename?} |
| ref_id | uuid | YES | | D-0677: polymorphic ref to source record (no FK). idx_messages_ref WHERE ref_id IS NOT NULL |
| ref_table | text | YES | | D-0677: names the table ref_id points to ('cases', 'renovation_applications', etc.) |
| metadata | jsonb | YES | | D-0677: rich card metadata (case_number, title, category, assigned_to_name, resolution_note) so Console renders without refetch |
| ai_processed | boolean | YES | false | |
| ai_intent | text | YES | | |
| ai_model_tier | USER-DEFINED | YES | | enum |
| ai_confidence | numeric | YES | | |
| is_flagged | boolean | YES | false | |
| created_at | timestamptz | YES | now() | |
<!-- idx_messages_external_message_id_unique: UNIQUE on external_message_id WHERE external_message_id IS NOT NULL (D-0431) -->
<!-- idx_messages_ref: partial btree on (ref_table, ref_id) WHERE ref_id IS NOT NULL (D-0677) -->
<!-- staff_role_permissions table DROPPED in migration 064 (D-0424) — was redundant, user_roles.staff_role_id column also dropped -->

## §Table-monitoring_events
### monitoring_events

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| level | text | NO | 'info' | |
| message | text | NO | | |
| source | text | YES | | |
| details | jsonb | YES | '{}' | |
| created_at | timestamptz | NO | now() | |

## §Table-notification_preferences
### notification_preferences

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK→users |
| notification_type | text | NO | | |
| channel | text | NO | 'both' | |
| created_at | timestamptz | NO | now() | |
| updated_at | timestamptz | NO | now() | |

## §Table-notifications
### notifications

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK→users |
| building_id | uuid | YES | | FK→buildings |
| type | text | NO | | |
| title | text | NO | | |
| message | text | YES | | |
| read | boolean | NO | false | |
| data | jsonb | YES | '{}' | |
| created_at | timestamptz | NO | now() | |

## §Table-openclaw_connections
### openclaw_connections

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| connection_status | USER-DEFINED | YES | 'disconnected'::openclaw_connection_status | enum |
| last_heartbeat | timestamptz | YES | | |
| accounting_type | text | YES | | |
| access_system_type | text | YES | | |
| tunnel_url | text | YES | | |
| api_key | text | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-partner_agencies
### partner_agencies

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| agency_name | text | NO | | |
| registration_number | text | YES | | |
| contact_name | text | YES | | |
| contact_phone | text | YES | | |
| contact_email | text | YES | | |
| coverage_areas | ARRAY | YES | | |
| commission_split_pct | numeric | YES | 30.00 | |
| is_active | boolean | YES | true | |
| approved_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-patrol_beacons
### patrol_beacons

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| beacon_uuid | text | NO | | |
| location_name | text | NO | | |
| floor | text | YES | | |
| coordinates | jsonb | YES | | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |

## §Table-patrol_logs
### patrol_logs

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| staff_id | uuid | NO | | FK→contractor_staff |
| beacon_id | uuid | NO | | FK→patrol_beacons |
| building_id | uuid | NO | | FK→buildings |
| detected_at | timestamptz | NO | | |
| signal_strength | integer | NO | | |
| created_at | timestamptz | YES | now() | |

## §Table-patrol_schedules
### patrol_schedules

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| staff_id | uuid | NO | | FK→contractor_staff |
| shift_id | uuid | YES | | FK→guard_shifts |
| required_beacon_ids | ARRAY | NO | | |
| time_windows | jsonb | NO | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-pdpa_audit_log
### pdpa_audit_log

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | YES | | FK→users |
| action | text | NO | | |
| details | jsonb | YES | '{}' | |
| ip_address | text | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-pdpa_consents
### pdpa_consents

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK→users |
| building_id | uuid | YES | | FK→buildings |
| consent_type | text | NO | | |
| granted | boolean | NO | false | |
| granted_at | timestamptz | YES | | |
| revoked_at | timestamptz | YES | | |
| ip_address | text | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-pdpa_deletion_requests
### pdpa_deletion_requests

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK→users |
| building_id | uuid | YES | | FK→buildings |
| requested_by | uuid | YES | | FK→users |
| requested_at | timestamptz | YES | now() | |
| scheduled_deletion_at | timestamptz | NO | | |
| status | text | NO | 'pending' | |
| grace_period_days | integer | YES | 30 | |
| executed_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-pending_residents
### pending_residents

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| unit_no | text | NO | | |
| name | text | NO | | |
| phone | text | NO | | |
| owner_or_tenant | text | NO | | |
| ic_last4 | text | YES | | |
| submitted_at | timestamptz | YES | now() | |
| status | USER-DEFINED | YES | 'pending'::pending_resident_status | enum |
| reviewed_by | uuid | YES | | FK→users |
| reviewed_at | timestamptz | YES | | |
| rejection_reason | text | YES | | |

## §Table-phone_call_logs
### phone_call_logs

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| resident_id | uuid | YES | | FK→residents |
| phone | text | NO | | |
| direction | USER-DEFINED | YES | 'inbound'::message_direction_type | enum |
| duration_seconds | integer | YES | | |
| transcript | text | YES | | |
| recording_url | text | YES | | |
| ai_summary | text | YES | | |
| outcome | text | YES | | |
| linked_case_id | uuid | YES | | FK→cases |
| pdpa_consent | boolean | YES | false | |
| created_at | timestamptz | YES | now() | |

## §Table-phone_change_log
### phone_change_log

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK→users |
| old_phone | text | NO | | |
| new_phone | text | NO | | |
| changed_by | uuid | YES | | FK→users |
| reason | text | YES | | |
| verified_by_bm | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |

## §Table-procurement_items
### procurement_items

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| item_name | text | NO | | |
| sku | text | YES | | |
| unit | text | YES | 'unit' | |
| reorder_threshold | integer | YES | 10 | |
| current_stock | integer | YES | 0 | |
| preferred_supplier_id | uuid | YES | | FK→suppliers |
| unit_price | numeric | YES | | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-push_notification_subscriptions
### push_notification_subscriptions

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK→users |
| building_id | uuid | YES | | FK→buildings |
| endpoint | text | NO | | |
| p256dh_key | text | NO | | |
| auth_key | text | NO | | |
| device_type | text | YES | 'web' | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |

## §Table-rental_listings
### rental_listings

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| unit_id | uuid | NO | | FK→units |
| resident_id | uuid | NO | | FK→residents |
| asking_rent | numeric | NO | | |
| bedroom_count | integer | YES | | |
| furnishing | text | YES | 'partially' | |
| available_from | date | YES | | |
| status | USER-DEFINED | YES | 'draft'::rental_listing_status_type | enum |
| description | text | YES | | |
| photos | ARRAY | YES | | |
| matched_agency_id | uuid | YES | | FK→partner_agencies |
| closed_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-reports_daily_snapshot
### reports_daily_snapshot

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| snapshot_date | date | NO | | |
| open_cases | integer | YES | 0 | |
| resolved_cases | integer | YES | 0 | |
| pending_approvals | integer | YES | 0 | |
| contractor_attendance_rate | numeric | YES | 0 | |
| collection_outstanding | numeric | YES | 0 | |
| new_residents | integer | YES | 0 | |
| ai_messages_processed | integer | YES | 0 | |
| api_cost_usd | numeric | YES | 0 | |
| created_at | timestamptz | YES | now() | |

## §Table-resident_pwa_sessions
### resident_pwa_sessions

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK→users |
| building_id | uuid | NO | | FK→buildings |
| resident_id | uuid | YES | | FK→residents |
| session_token | text | NO | | |
| device_info | jsonb | YES | | |
| last_active_at | timestamptz | YES | now() | |
| notification_preferences | jsonb | YES | | |
| expires_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-security_flags
### security_flags

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| message_id | uuid | YES | | FK→messages |
| flag_type | text | NO | | |
| matched_pattern | text | YES | | |
| raw_content | text | YES | | |
| sender_phone | text | YES | | |
| risk_score | numeric | YES | | |
| reviewed | boolean | YES | false | |
| reviewed_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |

## §Table-supplier_building_relationships
### supplier_building_relationships

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| supplier_id | uuid | NO | | FK→suppliers |
| building_id | uuid | NO | | FK→buildings |
| status | text | YES | 'pending' | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-supplier_credit_accounts
### supplier_credit_accounts

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| supplier_id | uuid | NO | | FK→suppliers |
| credit_limit | numeric | YES | 50000 | |
| current_balance | numeric | YES | 0 | |
| overdue_amount | numeric | YES | 0 | |
| last_payment_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-supplier_invoices
### supplier_invoices

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| supplier_id | uuid | NO | | FK→suppliers |
| building_id | uuid | NO | | FK→buildings |
| order_id | uuid | YES | | FK→procurement_orders |
| invoice_number | text | NO | | |
| amount | numeric | NO | | |
| due_date | date | NO | | |
| paid_at | timestamptz | YES | | |
| is_overdue | boolean | YES | false | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |
| status | text | YES | | |

## §Table-telegram_group_settings
### telegram_group_settings

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| group_name | text | NO | | |
| category | text | NO | 'custom' | bm_staff/contractors/security/cleaning/committee/residents/suppliers/custom |
| group_type | text | YES | | Legacy alias for category |
| group_purpose | text | YES | | Legacy |
| telegram_group_id | bigint | YES | | Telegram chat ID from Telethon service |
| group_id | text | YES | | Legacy string form of telegram chat ID |
| invite_link | text | YES | | Telegram invite link |
| bot_mode | text | NO | 'listen' | listen/active/mention_only/notifications_only |
| ai_mode | text | NO | 'silent' | silent/approval/auto |
| ai_respond | boolean | YES | false | Legacy — use ai_mode |
| ai_capture | boolean | YES | true | Legacy |
| notifications_enabled | boolean | YES | true | Proactive pings toggle |
| purpose_tags | ARRAY | YES | '{}' | e.g. ['security','patrol'] |
| respond_to_types | ARRAY | YES | '{}' | |
| notification_types | ARRAY | YES | '{}' | |
| contractor_org_building_assignment_id | uuid | YES | | FK→contractor_org_building_assignments |
| hardware_store_id | uuid | YES | | |
| ai_accuracy_rate | numeric | YES | 0 | |
| auto_mode_eligible_since | timestamptz | YES | | |
| is_active | boolean | YES | true | |
| created_by | uuid | YES | | FK→users |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-tender_commissions
### tender_commissions

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| tender_id | uuid | YES | | FK→tenders |
| commission_type | text | NO | | |
| amount | numeric | NO | | |
| payer_type | text | YES | | |
| payer_id | uuid | YES | | |
| transaction_ref | text | YES | | |
| paid_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-unit_occupancy_status
### unit_occupancy_status

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| unit_id | uuid | NO | | FK→units |
| status | USER-DEFINED | YES | 'vacant'::occupancy_status_type | enum |
| updated_by | uuid | YES | | FK→users |
| effective_date | date | YES | CURRENT_DATE | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-vehicle_logs
### vehicle_logs

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| plate_number | text | NO | | |
| unit_id | uuid | YES | | FK→units |
| resident_id | uuid | YES | | FK→residents |
| is_visitor_vehicle | boolean | YES | false | |
| entry_time | timestamptz | YES | now() | |
| exit_time | timestamptz | YES | | |
| logged_by | uuid | YES | | FK→users |
| gate_id | uuid | YES | | FK→building_gates (D-0442) |
| relationship | text | YES | NULL | owner/tenant/family — vehicle registrant relationship to unit (Migration 080) |
| source | text | YES | 'manual' | Migration 086: 'manual'\|'lpr'\|'moa_sync' |
| lpr_event_id | text | YES | | Migration 086: vendor LPR system event ID |
| confidence_score | numeric(5,2) | YES | | Migration 086: LPR plate-read confidence % |
| snapshot_url | text | YES | | Migration 086: LPR snapshot image URL |
| created_at | timestamptz | YES | now() | |

## §Table-vendor_building_contracts
### vendor_building_contracts

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| vendor_id | uuid | NO | | FK→vendors |
| building_id | uuid | NO | | FK→buildings |
| start_date | date | NO | | |
| end_date | date | YES | | |
| monthly_fee | numeric | YES | | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-vendor_performance_history
### vendor_performance_history

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| vendor_id | uuid | NO | | FK→vendors |
| building_id | uuid | NO | | FK→buildings |
| month | date | NO | | |
| attendance_rate | numeric | YES | | |
| completion_rate | numeric | YES | | |
| complaint_rate | numeric | YES | | |
| response_time_hours | numeric | YES | | |
| composite_score | numeric | YES | | |
| created_at | timestamptz | YES | now() | |

## §Table-vendor_subscriptions
### vendor_subscriptions

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| contractor_org_id | uuid | YES | | FK→contractor_orgs |
| building_id | uuid | NO | | FK→buildings |
| org_id | uuid | YES | | FK→organisations |
| subscription_type | text | NO | | |
| monthly_fee | numeric | NO | | |
| status | USER-DEFINED | YES | 'active'::vendor_subscription_status_type | enum |
| billing_day | integer | YES | 1 | |
| activated_at | timestamptz | YES | now() | |
| suspended_at | timestamptz | YES | | |
| cancelled_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |
| next_billing_date | date | YES | | |
| locked_at | timestamptz | YES | | |
| terminated_at | timestamptz | YES | | |

## §Table-vendors
### vendors

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK→buildings |
| company_name | text | NO | | |
| contact_name | text | YES | | |
| contact_phone | text | YES | | |
| contact_email | text | YES | | |
| service_type | text | YES | | |
| contract_start_date | date | YES | | |
| contract_end_date | date | YES | | |
| monthly_fee | numeric | YES | | |
| is_active | boolean | YES | true | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-voice_knowledge_base
### voice_knowledge_base

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| question | text | NO | | |
| answer | text | NO | | |
| created_at | timestamptz | NO | now() | |
| updated_at | timestamptz | NO | now() | |

## §Table-voice_leads
### voice_leads

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| name | text | YES | | |
| phone | text | YES | | |
| role | text | YES | | |
| status | text | NO | 'new' | |
| duration_seconds | integer | YES | 0 | |
| source | text | YES | 'voice_agent' | |
| transcript | text | YES | | |
| created_at | timestamptz | NO | now() | |
| updated_at | timestamptz | NO | now() | |

## §Table-voice_unanswered
### voice_unanswered

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| question | text | NO | | |
| session_id | text | YES | | |
| resolved | boolean | NO | false | |
| created_at | timestamptz | NO | now() | |

## §Table-whatsapp_templates
### whatsapp_templates

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | YES | | FK→buildings |
| template_name | text | NO | | |
| category | text | NO | | |
| language | text | YES | 'en' | |
| components | jsonb | YES | | |
| variables | ARRAY | YES | | |
| approval_status | text | YES | 'pending' | |
| meta_template_id | text | YES | | |
| submitted_at | timestamptz | YES | | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

## §Table-worker_activation_codes
### worker_activation_codes

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| code | text | NO | | |
| contractor_staff_id | uuid | NO | | FK→contractor_staff |
| phone | text | NO | | |
| user_id | uuid | YES | | FK→users |
| used_at | timestamptz | YES | | |
| expires_at | timestamptz | NO | | |
| created_at | timestamptz | YES | now() | |

---

## Migration 039 Tables (Session 6B)

## §Table-announcements
### announcements
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings ON DELETE CASCADE),
title (TEXT NOT NULL), content (TEXT NOT NULL),
priority (announcement_priority: normal, important, urgent — DEFAULT 'normal'),
status (announcement_status: draft, published, archived — DEFAULT 'draft'),
published_at (TIMESTAMPTZ), expires_at (TIMESTAMPTZ),
created_by (UUID FK→users),
notify_whatsapp (BOOLEAN NOT NULL DEFAULT false),
notify_telegram (BOOLEAN NOT NULL DEFAULT false),
attachment_urls (TEXT[]),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**Indexes:** idx_announcements_building (building_id), idx_announcements_status (building_id, status)
**RLS:** select=own_building, insert/update=bm+staff, delete=bm only

## §Table-bank_accounts
### bank_accounts
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings ON DELETE CASCADE),
payment_method (payment_method_type: duitnow, fpx, manual_transfer — DEFAULT 'duitnow'),
bank_name (TEXT), account_number (TEXT), account_holder_name (TEXT),
duitnow_proxy_type (TEXT), duitnow_proxy_id (TEXT),
platform_fee_cents (INTEGER NOT NULL DEFAULT 150),
platform_share_cents (INTEGER NOT NULL DEFAULT 50),
is_split_charge (BOOLEAN NOT NULL DEFAULT true),
status (bank_account_status: active, pending_verification, disabled — DEFAULT 'pending_verification'),
is_primary (BOOLEAN NOT NULL DEFAULT false),
verified_at (TIMESTAMPTZ), verified_by (UUID FK→users),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**Unique:** (building_id, payment_method, duitnow_proxy_id)
**RLS:** select=own_building, insert/update/delete=bm only

## §Table-insurance_policies
### insurance_policies
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings ON DELETE CASCADE),
insurance_type (insurance_type: fire, public_liability, lift, fidelity_bond, machinery_breakdown, other),
provider_name (TEXT NOT NULL), policy_number (TEXT),
coverage_amount (NUMERIC), premium_amount (NUMERIC),
start_date (DATE NOT NULL), end_date (DATE NOT NULL),
status (insurance_status: active, expiring_soon, expired, cancelled — DEFAULT 'active'),
document_url (TEXT), notes (TEXT),
reminder_days (INTEGER NOT NULL DEFAULT 30),
created_by (UUID FK→users),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**Indexes:** idx_insurance_building (building_id), idx_insurance_expiry (building_id, end_date)
**RLS:** select=own_building, insert/update=bm+staff, delete=bm only

## §Table-building_documents
### building_documents
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings ON DELETE CASCADE),
title (TEXT NOT NULL),
category (document_category: house_rules, bylaws, spa, strata_title, fire_cert, ccc, management_agreement, insurance, audit_report, agm_minutes, committee_minutes, circular, form, correspondence, legal, financial, other — DEFAULT 'other'),
description (TEXT), file_url (TEXT NOT NULL), file_name (TEXT), file_size (INTEGER),
uploaded_by (UUID FK→users),
unit_id (UUID FK→units — nullable, NULL=building-wide, set=unit-specific — added Migration 080),
expires_at (DATE), is_public (BOOLEAN NOT NULL DEFAULT false),
tags (TEXT[]),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**Indexes:** idx_building_docs_building (building_id), idx_building_docs_category (building_id, category), idx_building_docs_expiry (expires_at) WHERE expires_at IS NOT NULL, idx_building_documents_unit_id (unit_id) WHERE unit_id IS NOT NULL
**RLS:** select=own_building, insert/update=bm+staff, delete=bm only

---

## Migration 040: Working Hours (2026-03-26)

## §Table-building_working_hours
### building_working_hours
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings ON DELETE CASCADE),
day_of_week (INTEGER NOT NULL — 0=Sunday, 1=Monday ... 6=Saturday),
is_working_day (BOOLEAN NOT NULL DEFAULT true),
start_time (TIME — null if not working day),
end_time (TIME — null if not working day),
grace_minutes (INTEGER NOT NULL DEFAULT 15),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
UNIQUE(building_id, day_of_week)
```
**Indexes:** idx_working_hours_building (building_id)
**RLS:** select=own_building, insert/update=bm+staff, delete=bm only

## §Table-public_holidays
### public_holidays
```
id (UUID PK), building_id (UUID FK→buildings ON DELETE CASCADE — null = national),
name (TEXT NOT NULL), date (DATE NOT NULL),
is_recurring (BOOLEAN NOT NULL DEFAULT false),
source (TEXT NOT NULL DEFAULT 'manual' — 'manual', 'api', 'state'),
state (TEXT — null = national, 'selangor', 'johor' etc),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
UNIQUE(building_id, date)
```
**Indexes:** idx_holidays_date (date), idx_holidays_building (building_id) WHERE building_id IS NOT NULL
**RLS:** select=all, insert/update=superadmin(national)+bm/staff(building), delete=superadmin(national)+bm(building)
**Seeded:** 16 Malaysian national holidays for 2026

## §Table-contractor_schedule_templates
### contractor_schedule_templates
```
id (UUID PK), building_id (UUID NOT NULL FK→buildings ON DELETE CASCADE),
contractor_org_building_assignment_id (UUID NOT NULL FK→contractor_org_building_assignments ON DELETE CASCADE),
schedule_name (TEXT NOT NULL — 'Day Shift', 'Night Shift', etc),
day_of_week (INTEGER NOT NULL — 0=Sunday ... 6=Saturday),
is_working_day (BOOLEAN NOT NULL DEFAULT true),
start_time (TIME), end_time (TIME),
grace_minutes (INTEGER NOT NULL DEFAULT 15),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
UNIQUE(contractor_org_building_assignment_id, schedule_name, day_of_week)
```
**Indexes:** idx_contractor_schedule_building (building_id)
**RLS:** select=own_building, insert/update=bm+staff, delete=bm only

---

## Migration 041 — Resident Properties, Listings, Utilities, Tenancy (Session 11C)

## §Table-personal_properties
### personal_properties
```
id (UUID PK DEFAULT gen_random_uuid()),
user_id (UUID NOT NULL FK→users ON DELETE CASCADE),
property_name (TEXT NOT NULL),
address (TEXT),
city (TEXT),
state (TEXT),
postcode (TEXT),
image_url (TEXT),
ownership (TEXT NOT NULL DEFAULT 'owner'),
property_type (TEXT DEFAULT 'residential'),
notes (TEXT),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**Indexes:** idx_personal_properties_user (user_id)
**RLS:** select/insert/update/delete = own (user_id = auth.uid())

## §Table-property_listings
### property_listings
```
id (UUID PK DEFAULT gen_random_uuid()),
user_id (UUID NOT NULL FK→users),
unit_id (UUID FK→units),
personal_property_id (UUID FK→personal_properties),
building_id (UUID FK→buildings),
agent_agency_id (UUID FK→partner_agencies),
listing_type (listing_type ENUM: rent, sale),
reference_number (TEXT NOT NULL),
title (TEXT NOT NULL),
description (TEXT),
price (NUMERIC NOT NULL),
deposit_months (INTEGER DEFAULT 2),
furnishing (TEXT DEFAULT 'unfurnished'),
bedrooms (INTEGER),
bathrooms (INTEGER),
floor_area_sqft (INTEGER),
photos (TEXT[]),
contact_phone (TEXT),
contact_whatsapp (TEXT),
status (listing_status ENUM: active, pending, rented, sold, withdrawn — DEFAULT 'active'),
views_count (INTEGER NOT NULL DEFAULT 0),
listed_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
expires_at (TIMESTAMPTZ),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
CHECK (unit_id IS NOT NULL OR personal_property_id IS NOT NULL)
```
**Indexes:** idx_listings_status (status, listing_type), idx_listings_building (building_id) WHERE building_id IS NOT NULL, idx_listings_user (user_id)
**RLS:** select = active OR own, insert/update/delete = own (user_id = auth.uid())

## §Table-utility_bills
### utility_bills
```
id (UUID PK DEFAULT gen_random_uuid()),
user_id (UUID NOT NULL FK→users),
unit_id (UUID FK→units),
personal_property_id (UUID FK→personal_properties),
building_id (UUID FK→buildings),
utility_type (utility_type ENUM: electricity, water, gas, sewage, internet, assessment, quit_rent, other),
provider_name (TEXT),
account_number (TEXT),
bill_month (DATE NOT NULL),
amount (NUMERIC NOT NULL),
due_date (DATE),
paid (BOOLEAN NOT NULL DEFAULT false),
paid_date (DATE),
receipt_url (TEXT),
bill_url (TEXT),
notes (TEXT),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**Indexes:** idx_utility_bills_user (user_id), idx_utility_bills_unit (unit_id) WHERE unit_id IS NOT NULL
**RLS:** select/insert/update/delete = own (user_id = auth.uid())

## §Table-tenancy_documents
### tenancy_documents
```
id (UUID PK DEFAULT gen_random_uuid()),
unit_id (UUID FK→units),
personal_property_id (UUID FK→personal_properties),
building_id (UUID FK→buildings),
uploaded_by (UUID NOT NULL FK→users),
document_type (TEXT NOT NULL DEFAULT 'tenancy_agreement'),
title (TEXT NOT NULL),
file_url (TEXT NOT NULL),
start_date (DATE),
end_date (DATE),
tenant_user_id (UUID FK→users),
notes (TEXT),
created_at (TIMESTAMPTZ NOT NULL DEFAULT now()),
updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
```
**RLS:** select = uploaded_by OR tenant_user_id = auth.uid(), insert/update/delete = uploaded_by only

## §Table-access_cards
### access_cards (Migration 070)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
unit_id (UUID FK→units),
resident_id (UUID FK→residents),
card_number (TEXT NOT NULL),
status (TEXT CHECK IN ('active','deactivated','lost','returned')),
issued_date (DATE),
deactivated_at (TIMESTAMPTZ),
notes (TEXT),
created_at (TIMESTAMPTZ DEFAULT now()),
updated_at (TIMESTAMPTZ DEFAULT now()),
vendor_card_id (TEXT),         -- Migration 086: vendor system card ID
vendor_synced_at (TIMESTAMPTZ) -- Migration 086: last sync from vendor access system
```
**RLS:** enabled; building members

## §Table-access_card_logs
### access_card_logs (Migration 086)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
access_card_id (UUID FK→access_cards),
card_number (TEXT),
gate_id (UUID FK→building_gates),
door_name (TEXT),
event_type (TEXT CHECK IN ('granted','denied','unknown')),
event_at (TIMESTAMPTZ NOT NULL),
vendor_event_id (TEXT),
is_contractor (BOOLEAN DEFAULT false),
created_at (TIMESTAMPTZ DEFAULT now())
```
**RLS:** SELECT = building_id IN user_roles
**ai_view:** ai_view_access_card_logs (omits raw card details)

## §Table-vendor_billing_statements
### vendor_billing_statements (Migration 086)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
unit_id (UUID NOT NULL FK→units),
vendor_account_number (TEXT),
period (DATE NOT NULL),
pdf_url (TEXT NOT NULL),
total_due (NUMERIC),
generated_at (TIMESTAMPTZ),
fetched_at (TIMESTAMPTZ DEFAULT now()),
created_at (TIMESTAMPTZ DEFAULT now())
```
**RLS:** SELECT = building_id IN user_roles
**ai_view:** ai_view_vendor_billing_statements (omits pdf_url)

## §Table-vendor_ageing_snapshots
### vendor_ageing_snapshots (Migration 086)
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
snapshot_date (DATE NOT NULL),
pdf_url (TEXT),
current_count (INTEGER DEFAULT 0),
overdue_30_count (INTEGER DEFAULT 0),
overdue_60_count (INTEGER DEFAULT 0),
overdue_90_plus_count (INTEGER DEFAULT 0),
total_current_amount (NUMERIC DEFAULT 0),
total_overdue_amount (NUMERIC DEFAULT 0),
data (JSONB DEFAULT '{}'),
fetched_at (TIMESTAMPTZ DEFAULT now()),
created_at (TIMESTAMPTZ DEFAULT now())
```
**RLS:** SELECT = building_id IN user_roles
**ai_view:** ai_view_vendor_ageing_snapshots (omits data blob and pdf_url)

## §Table-moa_skills
### moa_skills
```
id (UUID PK DEFAULT gen_random_uuid()),
name (TEXT UNIQUE NOT NULL),
category (TEXT NOT NULL CHECK IN ('accounting','access_card','barrier','lift','custom')),
description (TEXT),
input_schema (JSONB NOT NULL),
output_schema (JSONB NOT NULL),
requires_rdp (BOOLEAN DEFAULT false),
version (TEXT DEFAULT '1.0'),
is_active (BOOLEAN DEFAULT true),
system_id (TEXT),          -- Migration 086: which vendor system this skill targets
connection_type (TEXT)     -- Migration 086: e.g. 'rdp','api','scraper'
```
**RLS:** select = public read

## §Table-moa_agents
### moa_agents
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
agent_name (TEXT NOT NULL),
api_key_hash (TEXT NOT NULL),
last_heartbeat (TIMESTAMPTZ),
installed_skills (TEXT[] DEFAULT '{}'),
version (TEXT),
status (TEXT DEFAULT 'offline' CHECK IN ('online','offline','busy')),
config (JSONB DEFAULT '{}'),
created_at (TIMESTAMPTZ DEFAULT now())
```
**RLS:** all = building_id IN user_roles

## §Table-moa_commands
### moa_commands
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
skill_name (TEXT NOT NULL),
input_data (JSONB NOT NULL),
status (TEXT DEFAULT 'pending' CHECK IN ('pending','processing','completed','failed','timeout')),
result_data (JSONB),
error_message (TEXT),
requested_by (UUID FK→users),
requested_at (TIMESTAMPTZ DEFAULT now()),
picked_up_at (TIMESTAMPTZ),
completed_at (TIMESTAMPTZ),
timeout_seconds (INT DEFAULT 30),
system_id (TEXT)           -- Migration 086: which vendor system to target
```
**RLS:** all = building_id IN user_roles

## §Table-moa_supported_systems
### moa_supported_systems
```
id (UUID PK DEFAULT gen_random_uuid()),
category (TEXT NOT NULL CHECK IN ('accounting','access_card','barrier_gate','car_plate')),
name (TEXT NOT NULL),
vendor (TEXT),
description (TEXT),
is_active (BOOLEAN DEFAULT true),
created_at (TIMESTAMPTZ DEFAULT now())
```
**RLS:** select = public, all = superadmin only

## §Table-integration_requests
### integration_requests
```
id (UUID PK DEFAULT gen_random_uuid()),
building_id (UUID NOT NULL FK→buildings),
requested_by (UUID NOT NULL FK→users),
category (TEXT NOT NULL CHECK IN ('accounting','access_card','barrier_gate','car_plate')),
system_name (TEXT NOT NULL),
vendor_name (TEXT),
notes (TEXT),
status (TEXT DEFAULT 'pending' CHECK IN ('pending','approved','rejected','implemented')),
reviewed_by (UUID FK→users),
reviewed_at (TIMESTAMPTZ),
created_at (TIMESTAMPTZ DEFAULT now())
```
**RLS:** all = building_id IN user_roles OR superadmin

---

## §Table-face_enrollments
## face_enrollments

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK → buildings |
| user_id | uuid | YES | | FK → users (building staff) |
| contractor_staff_id | uuid | YES | | FK → contractor_staff |
| descriptor | vector(128) | NO | | Face descriptor from face-api.js |
| is_active | boolean | NO | true | Deactivated on re-enroll |
| liveness_passed | boolean | YES | | Whether liveness check passed at enrollment |
| enrolled_at | timestamptz | NO | NOW() | |
| reference_photo_url | text | YES | | URL of reference JPEG captured at enrollment (migration 057c) |

**Migration 050:** Initial table with HNSW cosine index on descriptor.
**Migration 057c:** Added reference_photo_url column.
**RPC:** match_face(p_building_id, p_descriptor, p_threshold) → TABLE(user_id, contractor_staff_id, similarity) — cosine distance search, returns best match above threshold.
Either user_id OR contractor_staff_id is set per row (not both). is_active=false rows are ignored by match_face.

---

## §RPCs
## Postgres RPCs (attendance system — D-0345)

All attendance reads use these RPCs exclusively. Zero PostgREST .from('attendance_logs') queries in codebase.
All RPCs compute MYT date boundaries in Postgres using AT TIME ZONE 'Asia/Kuala_Lumpur'.

| RPC | Parameters | Returns | Purpose |
|-----|-----------|---------|---------|
| get_attendance_today | p_building_id uuid, p_user_id uuid | TABLE(sessions, status, today_summary) | Today's sessions + status for building staff |
| get_attendance_history | p_building_id uuid, p_user_id uuid, p_limit int | TABLE(date, sessions[]) | Paginated history grouped by date |
| find_open_attendance | p_building_id uuid, p_user_id uuid | attendance_logs row | Open session (clock_out_at IS NULL) for clock-out |
| count_today_sessions | p_building_id uuid, p_user_id uuid | integer | Count of sessions today |
| get_attendance_today_by_staff | p_building_id uuid, p_contractor_staff_id uuid | same as get_attendance_today | For contractor staff (by contractor_staff_id) |
| get_attendance_history_by_staff | p_building_id uuid, p_contractor_staff_id uuid, p_limit int | same as get_attendance_history | For contractor staff |
| get_attendance_buildings_today | p_building_ids uuid[] | TABLE(all attendance_logs columns) | Today's attendance across multiple buildings (MYT boundaries in Postgres) |
| get_attendance_buildings_range | p_building_ids uuid[], p_start_date date, p_end_date date, p_limit int DEFAULT 500 | TABLE(all attendance_logs columns) | Attendance logs for date range across buildings |
| get_attendance_staff_building_range | p_building_id uuid, p_contractor_staff_id uuid, p_start_date date, p_end_date date | TABLE(all attendance_logs columns) | Single staff member attendance in a building for date range |
| get_attendance_log_verify | p_log_id uuid, p_contractor_staff_id uuid, p_building_id uuid | TABLE(all attendance_logs columns) | Verify a single log belongs to staff+building (for BM overrides) |
| get_attendance_unsynced | p_limit int DEFAULT 100 | TABLE(all attendance_logs columns) | Unsynced logs (synced_to_reports=false, has clock_out_at) for cron |
| get_attendance_today_stats | (none) | TABLE(all attendance_logs columns) | All attendance today across all buildings (for cron summary) |
| match_face | p_building_id uuid, p_descriptor vector(128), p_threshold float | TABLE(user_id, contractor_staff_id, similarity) | Face recognition match against face_enrollments |
| unit_number_normalized | p text | text | Immutable — canonicalize unit string: lowercase + strip non-alphanumeric. "GB/01-05" → "gb0105" |
| search_units_smart | p_building_id uuid, p_query text DEFAULT '', p_block_id uuid DEFAULT NULL, p_limit int DEFAULT 200 | SETOF units | 3-path waterfall: (A) block filter → all units uncapped, (B) no query → first N, (C) exact normalized ILIKE → fuzzy pg_trgm fallback. SECURITY DEFINER. Migration 096. |

---

## AI Database Views (created in migration v13_6_ai_database_views)

All views: no PII (no phone, IC, email, telegram_id, password_hash, totp_secret, auth_id).
All views: include building_id for RLS-style filtering via generic_read.

### ai_view_staff
Columns: building_id, user_id, full_name, role, position, is_active, has_telegram (bool), has_face (bool), last_seen_at, assigned_at, created_at
Source: user_roles JOIN users (WHERE revoked_at IS NULL)

### ai_view_residents
Columns: resident_id, building_id, full_name, unit_number, block_name, role, move_in_date, move_out_date, is_active, created_at
Source: residents JOIN units JOIN blocks

### ai_view_contractors
Columns: building_id, org_id, org_name, contractor_type, service_categories, is_active, assignment_status, contract_start_date, contract_end_date, active_staff_count, created_at
Source: building_org_assignments JOIN contractor_orgs

### ai_view_tasks
Columns: id, building_id, case_number, title, category, status, priority, is_urgent, assigned_to_name, resolved_at, sla_due_at, is_overdue (computed), created_at, updated_at
Source: cases LEFT JOIN users (assigned_to)

### ai_view_facilities
Columns: id, building_id, name, description, capacity, is_active, created_at, booking_count_this_month (subquery)
Source: facilities

### ai_view_announcements
Columns: id, building_id, title, content_preview (LEFT 200 chars), priority, status, published_at, expires_at, created_by_name, created_at
Source: announcements LEFT JOIN users (created_by)

### ai_view_patrol
Columns: log_id, building_id, guard_name, beacon_name, beacon_floor, scanned_at, created_at
Source: patrol_logs LEFT JOIN users LEFT JOIN patrol_beacons (WHERE detected_at >= NOW() - 30 days)

### ai_view_visitors
Columns: id, building_id, visitor_name, unit_number, visit_purpose, vehicle_plate, is_preregistered, check_in_at, check_out_at, created_at
Source: visitors LEFT JOIN units (WHERE created_at >= CURRENT_DATE — today only)

### ai_view_collection
Columns: id, building_id, unit_number, block_name, outstanding_balance, months_overdue, escalation_stage, last_payment_date, last_payment_amount, access_blocked, created_at, updated_at
Source: collection_accounts JOIN units LEFT JOIN blocks

### ai_view_petty_cash
Columns: id, building_id, type, amount, description, category, receipt_url, balance_after, recorded_by_name, created_at
Source: petty_cash LEFT JOIN users (WHERE created_at >= NOW() - 90 days)

### ai_view_credits
Columns: building_id (=entity_id), credit_type, balance, free_monthly_allowance, free_remaining, purchased_remaining, last_reset_at, usage_this_month (subquery)
Source: credit_balances WHERE entity_type='building'

### ai_view_documents
Columns: id, building_id, title, document_type, description, file_name, expires_at, is_expired (computed), days_until_expiry (computed), uploaded_by_name, is_public, created_at
Source: building_documents LEFT JOIN users (uploaded_by)

### ai_view_insurance
Columns: id, building_id, policy_type, provider_name, policy_number, start_date, end_date, premium_amount, status, is_expired (computed), days_until_expiry (computed), created_at
Source: insurance_policies

### ai_view_renovation
Columns: id, building_id, unit_number, applicant_name, work_description, status, start_date, end_date, deposit_amount, deposit_paid, created_at
Source: renovation_applications JOIN units LEFT JOIN residents

### ai_view_telegram_groups
Columns: id, building_id, group_name, category, group_type, bot_mode, ai_mode, ai_respond, notifications_enabled, is_active, created_at
Source: telegram_group_settings (no invite_link, no telegram_group_id)

### ai_view_attendance
Columns: id, building_id, staff_name, staff_role, clock_in_at, clock_out_at, is_late, is_early_departure, verification_method, created_at
Source: attendance_logs LEFT JOIN contractor_staff (WHERE created_at >= NOW() - 7 days)

### ai_view_bookings
Columns: id, building_id, facility_name, booked_by_name, booking_date, start_time, end_time, status, guests_count, purpose, deposit_paid, created_at
Source: facility_bookings LEFT JOIN facilities LEFT JOIN residents (WHERE booking_date >= CURRENT_DATE - 30)

### ai_view_tenders
Columns: id, building_id, title, category, tender_type, status, budget_min, budget_max, deadline, area, bid_count (subquery), created_at
Source: tenders

## §Table-building_gaps
### building_gaps
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK → buildings |
| gap_type | text | NO | | e.g. people_no_telegram, ops_stale_tasks |
| severity | text | NO | | CHECK IN ('red', 'amber', 'green') |
| category | text | NO | | CHECK IN ('people', 'operations', 'finance', 'compliance', 'setup') |
| entity_id | uuid | YES | | Optional: linked record (e.g. specific unit) |
| entity_type | text | YES | | Optional: table name of entity |
| title | text | NO | | Short human-readable gap title |
| description | text | NO | | Detailed gap description |
| suggested_action | text | YES | | What BM should do |
| action_ui_link | text | YES | | Deep link to relevant BM page |
| auto_resolve_payload | jsonb | YES | | Payload for future auto-action |
| last_notified_at | timestamptz | YES | | When last Telegram notification was sent |
| cooldown_hours | integer | NO | 72 | Hours before re-notifying |
| resolved_at | timestamptz | YES | | NULL = active gap |
| resolved_by | text | YES | | 'auto' or user id |
| created_at | timestamptz | NO | NOW() | |
| updated_at | timestamptz | NO | NOW() | |
Indexes: idx_building_gaps_building (building_id), idx_building_gaps_active (partial WHERE resolved_at IS NULL), idx_building_gaps_dedup UNIQUE (building_id, gap_type, entity_id WHERE resolved_at IS NULL)
RLS: enabled. BM/staff read via user_roles membership.

### ai_view_gaps
Columns: id, building_id, gap_type, severity, category, title, description, suggested_action, action_ui_link, is_active (computed: resolved_at IS NULL), created_at, updated_at
Source: building_gaps — active gaps only (no resolved_at filter in view, is_active computed column for filtering)

## §Table-leave_policies
### leave_policies
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | YES | | FK → buildings (NULL for contractor-org policies) |
| contractor_org_id | uuid | YES | | FK → contractor_orgs (NULL for building policies) |
| leave_type | text | NO | | 'annual' / 'sick' / 'emergency' / 'unpaid' / custom |
| annual_entitlement | numeric | NO | 14 | Default entitlement days per year |
| requires_document | boolean | NO | false | If true, document_url required on submission |
| is_active | boolean | NO | true | |
| created_at | timestamptz | NO | NOW() | |
| updated_at | timestamptz | NO | NOW() | |

## §Table-leave_balances
### leave_balances
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK → users |
| building_id | uuid | YES | | FK → buildings |
| contractor_org_id | uuid | YES | | FK → contractor_orgs |
| leave_type | text | NO | | Matches leave_policies.leave_type |
| year | integer | NO | | Calendar year |
| entitled | numeric | NO | | Entitlement days for this year |
| used | numeric | NO | 0 | Approved leave days used |
| carried_forward | numeric | NO | 0 | Days carried from prior year |
| created_at | timestamptz | NO | NOW() | |
| updated_at | timestamptz | NO | NOW() | |

## §Table-leave_requests
### leave_requests
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| user_id | uuid | NO | | FK → users (applicant) |
| building_id | uuid | NO | | FK → buildings |
| contractor_org_id | uuid | YES | | FK → contractor_orgs (NULL for building staff) |
| leave_type | text | NO | | Matches leave_policies.leave_type |
| start_date | date | NO | | |
| end_date | date | NO | | |
| days | numeric | NO | | Workdays (0.5 for half-day) |
| is_half_day | boolean | NO | false | |
| half_day_period | text | YES | | 'am' or 'pm' |
| reason | text | YES | | Applicant's reason |
| document_url | text | YES | | MC or supporting doc URL |
| status | text | NO | 'pending' | CHECK IN ('pending','approved','rejected','cancelled') |
| reviewed_by | uuid | YES | | FK → users (approver) |
| reviewed_at | timestamptz | YES | | |
| rejection_reason | text | YES | | Filled on rejection |
| created_at | timestamptz | NO | NOW() | |
| updated_at | timestamptz | NO | NOW() | |

### ai_view_leave
Columns: id, building_id, leave_type, start_date, end_date, days, is_half_day, reason, status, contractor_org_id, created_at, reviewed_at, applicant_name, reviewer_name
Source: leave_requests LEFT JOIN users (applicant) LEFT JOIN users (reviewer)

## §Table-vehicle_fines
### vehicle_fines
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK → buildings |
| fine_number | text | NO | | Auto-generated FINE-0001+ via trigger+sequence |
| violation_type | text | NO | | CHECK IN ('illegal_parking','double_parking','blocking_access','expired_sticker','no_sticker','reserved_lot','fire_lane','other') |
| vehicle_plate | text | NO | | Normalised uppercase, no spaces |
| vehicle_description | text | YES | | Optional free-text description |
| unit_id | uuid | YES | | FK → units (optional link to unit) |
| location | text | YES | | Free-text location |
| photos | jsonb | NO | '[]' | Array of R2/storage URLs |
| notes | text | YES | | |
| is_clamped | boolean | NO | false | |
| clamped_at | timestamptz | YES | | Set when is_clamped=true |
| fine_amount | numeric(10,2) | NO | 50.00 | Editable; defaults from fine_settings |
| status | text | NO | 'outstanding' | CHECK IN ('outstanding','paid','waived','appealed') |
| payment_method | text | YES | | CHECK IN ('cash','online','waived') |
| paid_at | timestamptz | YES | | |
| paid_to | uuid | YES | | FK → users (guard who collected) |
| unclamped_at | timestamptz | YES | | |
| unclamped_by | uuid | YES | | FK → users (guard who unclamped) |
| created_by | uuid | NO | | FK → users |
| created_at | timestamptz | NO | NOW() | |
| updated_at | timestamptz | NO | NOW() | |
Indexes: idx_vehicle_fines_building (building_id), idx_vehicle_fines_plate (vehicle_plate, building_id), idx_vehicle_fines_status (partial WHERE status='outstanding'), idx_vehicle_fines_number (fine_number, building_id)
RLS: enabled. building members via user_roles.

## §Table-fine_settings
### fine_settings
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK → buildings |
| violation_type | text | NO | | Matches vehicle_fines.violation_type |
| default_amount | numeric(10,2) | NO | | Default fine amount for this violation |
| description | text | YES | | Human-readable description |
| is_clampable | boolean | NO | true | Whether this violation triggers clamping |
| is_active | boolean | NO | true | |
| created_at | timestamptz | NO | NOW() | |
UNIQUE: (building_id, violation_type)
RLS: enabled. building members via user_roles.

### ai_view_fines
Columns: id, building_id, fine_number, violation_type, vehicle_plate, location, is_clamped, fine_amount, status, payment_method, paid_at, unclamped_at, created_at, issued_by_name, unit_number
Source: vehicle_fines JOIN users (guard) LEFT JOIN units


## §Table-building_spaces
### building_spaces
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK → buildings |
| block_id | uuid | YES | | FK → blocks |
| name | text | NO | | e.g. "Lot G-01", "Function Hall A" |
| space_type | text | NO | shoplot | shoplot/common_area/parking/function_hall/office/storage/other |
| floor | text | YES | | |
| area_sqft | numeric | YES | | |
| is_active | boolean | NO | true | |
| notes | text | YES | | |
| created_at | timestamptz | NO | NOW() | |
| updated_at | timestamptz | NO | NOW() | |
Indexes: idx_building_spaces_building (building_id)
RLS: enabled. building_id isolation.

## §Table-building_tenancies
<!-- Migration 077_building_tenancies applied to prod at V32d Batch A. TS types regenerated. -->
### building_tenancies
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK → buildings |
| space_id | uuid | NO | | FK → building_spaces |
| tenant_name | text | NO | | |
| tenant_phone | text | YES | | |
| tenant_email | text | YES | | |
| tenant_ic | text | YES | | NRIC/passport |
| tenant_resident_id | uuid | YES | | FK → residents |
| monthly_rent | numeric | YES | | |
| deposit_amount | numeric | YES | | |
| start_date | date | NO | | |
| end_date | date | YES | | |
| is_active | boolean | NO | true | |
| terminated_at | timestamptz | YES | | |
| termination_reason | text | YES | | |
| notes | text | YES | | |
| created_by | uuid | YES | | |
| created_at | timestamptz | NO | NOW() | |
| updated_at | timestamptz | NO | NOW() | |
Indexes: idx_building_tenancies_building (building_id), idx_building_tenancies_space (space_id), idx_building_tenancies_active (building_id, is_active)
RLS: enabled. building_id isolation.

## §Table-building_tenancy_documents
### building_tenancy_documents
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| tenancy_id | uuid | NO | | FK → building_tenancies ON DELETE CASCADE |
| building_id | uuid | NO | | FK → buildings |
| document_type | text | YES | tenancy_agreement | |
| title | text | NO | | |
| file_url | text | NO | | |
| uploaded_by | uuid | YES | | |
| created_at | timestamptz | NO | NOW() | |
Indexes: idx_building_tenancy_docs_tenancy (tenancy_id)
RLS: enabled. building_id isolation.

### ai_view_building_tenancies
Columns: id, building_id, space_name, space_type, block_name, tenant_name, monthly_rent, deposit_amount, start_date, end_date, is_active, terminated_at, notes, created_at, expiry_status, days_to_expiry
Source: building_tenancies JOIN building_spaces LEFT JOIN blocks

## §Table-resident_data_staging

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | — | FK buildings(id) |
| resident_id | uuid | YES | — | FK residents(id), null if new resident |
| unit_id | uuid | YES | — | FK units(id) |
| full_name | text | YES | — | |
| phone | text | YES | — | |
| email | text | YES | — | |
| ic_number | text | YES | — | |
| role | text | YES | — | CHECK: owner, tenant, family, occupant |
| related_person_name | text | YES | — | Cross-reference (e.g. tenant→owner) |
| related_person_phone | text | YES | — | |
| related_person_role | text | YES | — | CHECK: owner, tenant |
| source_channel | text | NO | — | whatsapp, telegram, email |
| source_message_id | uuid | YES | — | |
| ai_confidence | text | YES | 'medium' | CHECK: high, medium, low |
| status | text | YES | 'pending' | CHECK: pending, approved, rejected, merged |
| reviewed_by | uuid | YES | — | FK users(id) |
| reviewed_at | timestamptz | YES | — | |
| review_notes | text | YES | — | |
| created_at | timestamptz | YES | now() | |
| updated_at | timestamptz | YES | now() | |

Indexes: idx_staging_building_status (building_id, status), idx_staging_resident (resident_id)
RLS: enabled. building_id isolation.
Migration: 083_resident_data_staging.sql

### ai_view_resident_data_staging
Columns: id, building_id, resident_id, unit_id, full_name, role, related_person_name, related_person_role, ai_confidence, status, source_channel, created_at
Source: resident_data_staging (direct select, no joins)

## §Table-import_backfill_log
### import_backfill_log (migration 097, V32a)
```
id (uuid PK DEFAULT gen_random_uuid()),
building_id (uuid NOT NULL FK→buildings),
run_tag (text NOT NULL),        -- e.g. 'v32a_chv_gap'
csv_source (text NOT NULL),     -- file path of source CSV
stats (jsonb NOT NULL),         -- run-level counters (unitsInserted, residentsInserted, etc.)
ran_at (timestamptz DEFAULT now()),
ran_by (text)                   -- 'service_role' for script runs
```
**Index:** idx_import_backfill_log_building on building_id
**Purpose:** Lightweight per-run audit for any building backfill. One row per script execution. No per-row detail — just aggregate stats JSONB.
**CHV V32a run:** run_tag='v32a_chv_gap', unitsInserted=80, residentsInserted=79.
