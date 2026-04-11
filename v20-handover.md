# Minilab — V20 Session Handover
# Created: 2026-04-09 | Covers: V19 (D-0443–D-0490)

---

## 1. V19 Summary — Everything Shipped

### V19 Pre-session Audit (D-0443–D-0448)
**Focus:** Telegram groups reliability + Console quick fixes before deeper session work.
- TG group creation: 6 bug fixes — JSON parse safety on non-JSON Telethon responses, trailing slash strip from TELEGRAM_GROUP_SERVICE_URL, webhook race condition fix (match by telegram_group_id first then pending fallback), NaN guard on corrupted delete, sync try-catch per-user, orphan group logging if DB insert fails after Telethon create.
- Console: People section CRUD (add/edit/remove residents from right panel). Internal notes private flag. Direct-message delivery status fallback to `delivery_failed` when no channel path. Channel availability checks building-level credentials (WA phone_number_id + TG bot_token).

### V19-S1 — Console Critical Fixes + AI Human Takeover (D-0449–D-0454)
**Focus:** Fix broken internal notes, 24h WA window, add per-sender AI pause.
- `internal_note` value added to `message_channel_type` enum (migration 067). Console note route changed from `'internal'` (failed silently) to `'internal_note'`. Timeline routes match both values for backwards compatibility.
- `sendWhatsAppDirect` now checks `sender_profiles.last_inbound_at` before sending freeform text. Falls back to `minilab_notification` template if outside 24h window. Returns `sentAsTemplate` flag; console shows toast.
- **AI human takeover system:** `sender_profiles.ai_paused` BOOLEAN DEFAULT false (migration 067). `shouldSkipAI()` in `lib/ai/ai-pause-check.ts` — fail-open on DB errors. Wired into WA webhook + TG resident DM handler.
- `escalate_to_human` AI tool registered in `ai_tools`. `lib/ai/escalation.ts` — regex detection of anger/escalation patterns in resident messages → sets `ai_paused=true` on sender_profile, fires BM in-app notification, replies "connecting you with BM staff".
- Console Pause/Resume AI toggle in middle panel header. Amber "Human Mode" badge when paused. `GET/POST /api/bm/console/ai-pause` — sets `ai_paused` on all unit residents + `ai_can_handle=false` on active tickets.

### V19-S2 — Console 3-View Left Panel Redesign (D-0455–D-0461)
**Focus:** Surface unlinked contacts who were invisible in unit-only view.
- Default console view changed from `'unit'` to `'contact'`.
- Left panel: 3-view tab switcher (Contact / Unit / Case), each with own search, sub-filters, sort.
- **Contact view:** All `sender_profiles` for building. Rows show display_name (→state_json.identity.name→phone fallback), linked unit, classification badge (New/Saved/External), AI Paused badge, channel icon, unread dot, last message timestamp. Sub-filters: All/Unlinked/AI Paused. APIs: `GET /api/bm/console/contacts`, `GET /api/bm/console/contact-messages`.
- **Case view:** All cases with status/priority filters (Open/In Progress/Resolved/All) + sort (Recent/Priority/Oldest). Clicking case loads case-specific messages in middle panel + unit operational hub in right panel. APIs: `GET /api/bm/console/cases-list`, `GET /api/bm/console/case-messages`.
- **Right panel adaptation:** Full 10-section operational hub when unit context; simplified panel (Sender Info card + Link to Unit / Create Case / Mark as External) for unlinked contacts; case-only panel for cases without units.
- `sender_profiles.display_name` (TEXT) + `sender_profiles.classification` (TEXT DEFAULT 'unknown') added (migration 068). `display_name` backfilled from `state_json.identity.name`. `classification='resident'` set on profiles with `resident_id`.
- Link-to-Unit flow: slide-over form → creates resident → updates sender_profile (resident_id + classification='resident' + display_name). Re-associates past messages. `POST /api/bm/console/contacts/link`.
- `POST /api/bm/console/contacts/classify` sets classification='external'.

### V19-S3 — Telegram Groups Reliability (D-0462–D-0467)
**Focus:** Fix 5 crash/data bugs in Telethon integration, add rate limiting.
- Webhook DB update error check: `group linking .update()` destructures `{error}`, returns 500 if fails (was silently swallowed).
- Sync route: `isUserInGroup()` call moved inside try-catch to prevent loop crash on Telegram rate limits.
- Orphan cleanup: if DB insert fails after Telethon group creation, attempts `deleteTelegramGroup()` cleanup. If that also fails → logs with `[ORPHAN]` prefix. User sees "Group creation failed. Please try again."
- Race condition on group removal: now matches by `telegram_group_id` (numeric) + `is_active:true` instead of `group_id` (string) only. Prevents deactivating a recreated group with same chat ID.
- Send-invites generates fresh invite link via Telethon `getGroupInviteLink()` before sending. Updates DB. Validates not null.
- Rate limiting: 100ms delay between users in sync and send-invites loops (~10 req/sec).
- Friendly error messages: `friendlyError()` strips raw DB/service error strings. All UI operations show clean messages.

### V19-S4 — Console Quality + Cleanup (D-0468–D-0473)
**Focus:** Polish console correctness issues found post-S2 deploy.
- `sender_profiles.updated_at` set to `now()` after successful outbound message delivery (WA/TG/email).
- `building_channels` object in unit-details API now includes `email_configured` flag (checks `RESEND_API_KEY`). Compose bar disables email + shows tooltip when not configured.
- Note unit association: when `unit_id` provided but `resident_id` null, looks up primary resident (owner first) from residents table. Prevents orphaned notes.
- TG group null invite link guard: API returns `invite_link_available` flag. UI shows "Link unavailable" with send-invites fallback instead of rendering "null".
- AI system prompt for unregistered senders: `buildContextTypeRules` checks `fetchedData.resident`. Unknown WA users get a prompt asking for unit number before AI answers property questions.
- `D-0468`: Logged future TODOs — Documents section auto-populate from chat uploads; Access Cards section intentional stub.

### V19-S5 — Post-Deploy Fixes (D-0474–D-0479)
**Focus:** Fix 6 bugs found after V19-S1/S2/S3/S4 deployed to production.
- **Internal notes:** Removed `private: true` from insert (column exists but `channel='internal_note'` is sufficient). Added `response.ok` check + error/success toast. Fixed empty catch block with proper error logging.
- **Contacts query:** Console contacts rewritten to join messages via `resident_id` instead of unpopulated `sender_profile_id` column. Unread tracking also uses resident_id path. Unlinked contacts show without message preview.
- **Auto-link bidirectional:** WA webhook upserts sender_profile on inbound + links to resident. BM residents POST auto-links existing sender_profile to new resident. Shared helper at `lib/sender-profiles/auto-link.ts`.
- **TG unknown senders:** Known users (found in users table) with building but not admin/resident get sender_profile created + message stored in console. `resolveUser()` null-return exits early (fixes double-message bug for true unknown users). Phone-share updates sender_profile + auto-links to resident.
- **WA unknown senders:** `auto-register.ts` now creates sender_profile alongside contacts_temp entry. Unknown WA senders immediately visible in Contact view.
- **Documents section:** Query `building_documents` table, render list with title/category/size/date, clickable links to `file_url`. Replaces hardcoded empty array.

### V19-S6 — Contact Management (D-0480–D-0485)
**Focus:** Give BM full control over unlinked/external contacts.
- `sender_profiles.company` column added (migration 069) — company/label for external contacts.
- Classification expanded: unknown / saved / external / resident. 'saved' = BM gave name but no resident link.
- **3 contact actions** in right panel:
  - **Save Contact:** Updates `display_name`, sets `classification='saved'`. Supports cross-channel merge (find duplicate by phone/email/telegram). `POST /api/bm/console/contacts/save`.
  - **Link Internal:** Links to existing resident. `POST /api/bm/console/contacts/link`.
  - **Link External:** Updates `display_name`, `company`, sets `classification='external'`. Supports merge. `POST /api/bm/console/contacts/link-external`.
- **Explicit merge API:** `POST /api/bm/console/contacts/merge` — copies missing fields from secondary to primary, re-associates messages + tickets, deletes secondary.
- **Compose bar for ALL contacts:** Unlinked contacts use sender_profile phone/email/telegram_id. `direct-message` API accepts `sender_profile_id` as alternative to `resident_id`. `handleSend` routes through profile channels.
- **Sub-filters updated:** All / New / External / AI Paused (replaces old Unlinked). Left panel rows show classification badges.

### V19 Telegram Infra (D-0486–D-0490)
**Focus:** Production reliability for Telethon microservice.
- **Keepalive cron (D-0486):** Daily cron pings Telethon `/keepalive` endpoint. If 200 → logs "Telethon session alive". If not → sends Telegram alert to `MASTERAI_CHAT_ID`. Debug logging added to group-service.ts for Vercel diagnosis.
- **Auto-invite on group creation (D-0487):** After group creation API call succeeds, auto-sends invites to all category staff with `telegram_id`. Flow: try `/add-members` Telethon endpoint first (5s timeout), fall back to DM invite links. `getTargetUsersByCategory()` extracted to `lib/telegram/group-targets.ts`. `addUsersToGroup()` in group-service.ts. API response includes `auto_invite` stats (invited/failed/skipped).
- **Force-delete for orphan cleanup (D-0488):** Group delete API calls Telethon DELETE first. If Telethon fails → 502 with error message. UI auto-prompts "Force delete?" on 502. `?force=true` skips Telethon confirmation and deletes DB record only. Fixed `onClick` event-as-arg bug (was passing SyntheticEvent as groupId).
- **Timeout fix (D-0489):** `serviceCall()` now accepts custom `timeout` param. Auto-invite `/add-members` call uses 5s timeout (down from 30s) so DM fallback runs before Vercel 10s function timeout. Per-step console.log for full Vercel log diagnosis.
- **FK ambiguity fix (D-0490):** `getTargetUsersByCategory()` changed `users!inner(...)` to `users!user_roles_user_id_fkey(...)`. `user_roles` has two FK columns to `users` (user_id + assigned_by) — PostgREST was returning empty results silently due to ambiguity.

---

## 2. DB Migrations Applied in V19

| Migration | Date | Summary |
|-----------|------|---------|
| 066 | 2026-04-08 | `console_read_state` table (building_id, user_id, sender_profile_id, last_read_at). Unique: (building_id, user_id, sender_profile_id). |
| 067 | 2026-04-09 | `message_channel_type` enum + `internal_note` value. `sender_profiles.ai_paused` BOOLEAN DEFAULT false. `escalate_to_human` AI tool entry in ai_tools. |
| 068 | 2026-04-09 | `sender_profiles.display_name` TEXT nullable. `sender_profiles.classification` TEXT NOT NULL DEFAULT 'unknown'. Backfill: display_name from state_json.identity.name; classification='resident' for profiles with resident_id. |
| 069 | 2026-04-09 | `sender_profiles.company` TEXT nullable. |

**Note:** Migrations 060–065 were applied in V16–V18 (see recent-decisions.md for D-0388–D-0432).

---

## 3. Pending Code Fixes (Priority Order)

### P1 — Critical (fix before CHV launch)
1. **resolveBuilding login_slug** — `lib/auth/resolve-building.ts` uses `ilike(name, ...)` which is fragile. Switch to `eq(login_slug, ...)` lookup. Buildings need login_slug set in Supabase. Affects all login flows.
2. **Types file stale** — `packages/shared/types/database.types.ts` last regenerated 2026-03-25. Must regenerate before any new DB work: `npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea > packages/shared/types/database.types.ts`.
3. **Uncommitted webhook change** — `app/api/auth/telegram/webhook/route.ts` has an unstaged change: `resolveUser()` gains `skipPrompt = false` parameter (guards the "share phone" prompt behind a flag for unknown senders). Should be committed as part of D-0477 cleanup.

### P2 — Important (before wider rollout)
4. **Console credit widget** — Sidebar shows outdated collection balance widget. Needs redesign per V20 plan.
5. **DISABLE_DEV_LOGIN** — Set `DISABLE_DEV_LOGIN=true` in Vercel Production. Human task (cannot be done via code).
6. **Telethon session** — GCP e2-micro VM Telethon session was re-authorized in V19. If session expires again, requires SSH into VM + `python3 authorize_session.py` (human task).

### P3 — Nice to have
7. **Documents auto-populate** — Console Documents section currently shows `building_documents` table manually uploaded. Future: auto-populate from media files sent in chat (chat uploads → documents). Logged in D-0468.
8. **Access Cards stub** — Console Access Cards section is intentional stub. Backend access_cards table exists; needs UI wiring.
9. **Conversation sub-view** — Console Case view doesn't load conversation history from case_messages when case has no unit. Minor gap.

---

## 4. Pending Human Tasks

| Task | Owner | Priority | Notes |
|------|-------|----------|-------|
| Set `DISABLE_DEV_LOGIN=true` in Vercel | Max | P1-CHV | Prevents demo accounts being used in production |
| E2E test with Seeteng at Lumi Residency | Max | P1-CHV | First real-world building test |
| CHV onboarding — Hoe Zee How | Max | P1 | Cyber Heights Villa, 700+ units. BM account creation + building setup |
| Provision CHV building in superadmin | Max | P1-CHV | Create building record, set login_slug, configure WA + TG channels |
| Telethon keepalive monitoring | Max | P2 | Check daily cron alert fires correctly. SSH: `ssh minilab@34.42.210.102` |
| Regenerate TypeScript types | Max (or Claude) | P2 | `npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea` |
| Commit unstaged TG webhook change | Max/Claude | P2 | `resolveUser(skipPrompt)` param — partial change in `app/api/auth/telegram/webhook/route.ts` |

---

## 5. Future Features (V20 Backlog)

### Console
- Credit widget redesign in sidebar (show per-unit collection balance, not global)
- Console Documents section: auto-populate from chat file uploads
- Console Access Cards: wire to access_cards table
- Console notifications panel: show recent building events

### AI
- MOA Phase 2 — Electron/Python desktop agent for Advelsoft (Seeteng's current software)
- AI proactive pings refinement — smarter timing, better rate limiting per unit
- Multi-building AI context — AI knows about all buildings a BM manages

### Resident Portal
- Resident dashboard (balance, announcements, bookings, tasks)
- Resident mobile app (React Native or enhanced PWA)
- Community feed

### Operations
- Procurement 2.0 — Telegram hardware group auto-tracking (AI reads Telegram group, auto-creates procurement records)
- AGM voting system
- Form 11 auto-generation from collection_accounts

### Infrastructure
- Multi-region Supabase (current: Singapore only)
- Horizontal scaling for Telethon (multiple GCP VMs for large buildings)
- WhatsApp template library management in BM portal

---

## 6. CHV Launch Readiness

| Category | Item | Status | Notes |
|----------|------|--------|-------|
| **Infrastructure** | Vercel hosting | ✅ Live | minilab.my |
| **Infrastructure** | Supabase DB | ✅ Live | nncbsumdmsyjjrgdjjea (Singapore) |
| **Infrastructure** | Cloudflare R2 | ✅ Live | media.minilab.my |
| **Infrastructure** | Telethon VM | ✅ Live | 34.42.210.102:8443 (keepalive cron active) |
| **Security** | DISABLE_DEV_LOGIN | ❌ Pending | Must set in Vercel before CHV |
| **Security** | TOTP 2FA superadmin | ✅ Done | D-0384 |
| **Security** | TG webhook secret | ✅ Done | D-0430 |
| **Security** | WA HMAC validation | ✅ Done | D-0430 |
| **Auth** | BM login (Telegram QR) | ✅ Done | Per-building bot tokens in DB |
| **Auth** | Guard VMS kiosk | ✅ Done | Multi-gate + passcode system (D-0442) |
| **Auth** | Dev login disabled in prod | ❌ Pending | Human task — Vercel env var |
| **Building** | Building provisioning flow | ✅ Done | Superadmin 3-step wizard (D-0383) |
| **Building** | login_slug routing | ⚠️ Partial | resolveBuilding still uses ilike(name) — switch to login_slug before CHV |
| **Communication** | WhatsApp templates | ⚠️ Check | Ensure minilab_notification template approved for CHV WABA |
| **Communication** | Telegram groups | ✅ Done | Auto-create, auto-invite, keepalive, force-delete |
| **AI** | V4 pipeline (WA+TG) | ✅ Done | Async + dedup + 24h window |
| **AI** | Human takeover | ✅ Done | shouldSkipAI() + auto-escalation (D-0451–D-0454) |
| **Console** | BM console all features | ✅ Done | 3-view, contact mgmt, 10 right-panel sections |
| **Onboarding** | BM 8-stage PWA journey | ✅ Done | /app/onboard — all 8 stages live |
| **Reporting** | Basic analytics | ✅ Done | Committee analytics page |
| **E2E test** | Lumi Residency test | ❌ Pending | Seeteng walk-through required |
| **E2E test** | CHV UAT | ❌ Pending | Hoe Zee How sign-off |

**CHV launch blockers:** DISABLE_DEV_LOGIN, login_slug routing fix, CHV building provisioning, E2E test.

---

## 7. Key Architecture Decisions to Carry Forward

| Rule | Source | Never Break |
|------|--------|-------------|
| Portal routing: `lib/auth/get-portal-path.ts` is single source of truth | D-0370/D-0379 | Requires human approval to change |
| Auth gate: PWA routes use `getSession()`, BM portal uses `requirePermission()` | D-0355/D-0356 | Never swap |
| Attendance: ALL reads via Postgres RPCs — never PostgREST from `attendance_logs` | D-0345 | Schema cache errors on PostgREST |
| RPC caller rule: change signature → grep ALL callers → update ALL → NOTIFY pgrst | D-0347/D-0348 | No exceptions |
| AI routing: DeepSeek V3 (`deepseek-chat`) for ROUTINE, Claude Sonnet (`claude-sonnet-4-20250514`) for SENSITIVE | Architecture | Never alias model strings |
| AI human takeover: `shouldSkipAI()` must be checked before runV4Pipeline in ALL channels | D-0451 | Fail-open: if check errors, skip AI |
| Face = identity: no manual clock-in fallback, no name dropdown | D-0338 | Non-negotiable |
| user_roles FK hint: `users!user_roles_user_id_fkey` when joining users | D-0490 | Ambiguous FK returns empty results silently |
| TG group delete: Telethon confirmation required, DB only on ?force=true | D-0488 | Prevents orphaned TG groups |
| V4 pipeline only for WA: V3 removed D-0431 | D-0431 | No regression to V3 |
| WA 24h window: check last_inbound_at, use minilab_notification template outside window | D-0450 | Meta freeform messaging policy |
| Image uploads: compress client-side (lib/image-compress.ts) before R2 upload | D-0337 | All photos, not just some |
| Internal notes: channel='internal_note' (not 'internal') | D-0449 | 'internal' silently fails — enum doesn't include it |
| Sender auto-link: lib/sender-profiles/auto-link.ts on WA inbound + resident creation | D-0476 | Keeps Contact view populated |
| One commit per task, push ONCE at end of session (or when human says push) | CLAUDE.md | No branches. No worktrees. Vercel Hobby: every push = full build. |

---

## 8. Supabase Project Info

| Item | Value |
|------|-------|
| Project ID | `nncbsumdmsyjjrgdjjea` |
| Region | Singapore (ap-southeast-1) |
| URL | See `NEXT_PUBLIC_SUPABASE_URL` in .env.local |
| Edge Functions | `minilab-process-message` (AI pipeline entry point) |
| pgvector | Enabled — 128-dim face descriptors in `face_descriptors` table |
| Migrations applied | 001–069 |
| Types file | `packages/shared/types/database.types.ts` — regen with `npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea` |
| Last types regen | 2026-03-25 (stale — must regen before schema work) |

---

## 9. GCP VM (Telethon Microservice)

| Item | Value |
|------|-------|
| Instance | e2-micro (free tier) |
| IP | 34.42.210.102 |
| Port | 8443 |
| Health endpoint | `GET /keepalive` |
| Keepalive monitoring | Daily cron in lib/crons/ — alerts MASTERAI_CHAT_ID if down |
| Session re-auth | SSH into VM + `python3 authorize_session.py` (human task if session expires) |
| Env vars | `TELEGRAM_GROUP_SERVICE_URL`, `TELEGRAM_GROUP_SERVICE_SECRET` |

---

*Generated: 2026-04-09 at end of V19 session (D-0490)*
