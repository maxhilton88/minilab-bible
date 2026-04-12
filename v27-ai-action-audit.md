# V27 AI Action Execution Audit
<!-- D-0601 | 2026-04-12 | audit -->

**Scope:** AI action execution layer — pipeline routing vs. actual action execution  
**Auditor:** Claude Sonnet 4.6 (automated audit)  
**Date:** 2026-04-12  
**Status:** Read-only audit — NO code changes made  

---

## Executive Summary

The AI pipeline has a **fundamental architectural split that was never bridged**:

- **Admin pipeline** (BM chat): ✅ Fully wired — `v4-pipeline` → `routeAdminMessage()` → `executeAdminTool()` → 29 real handlers in `admin-handlers.ts`. Actions execute.
- **Resident pipeline** (WhatsApp/Telegram/Email, Tier 1–3): ❌ **Completely broken** — semantic router classifies intent but **generates no response** and **executes no actions**. `specialist-agent.ts` and `executor.ts` are orphaned modules that are never called. Residents receive **silence** for all non-trivial messages.

The two key files that *should* bridge this gap — `lib/ai/specialist-agent.ts` and `lib/ai/actions/executor.ts` — were built but never connected to any webhook.

---

## Part 1: Intent Action Table (23 Intents)

All 23 intents in `lib/ai/intents.ts` have tools defined. None of those tools execute for resident-channel messages.

| # | Intent | Tools in intents.ts | executor.ts handler? | Gets called? | Status |
|---|--------|---------------------|---------------------|--------------|--------|
| 1 | MAINTENANCE_COMPLAINT | create_case, notify_bm, get_resident_unit, get_open_cases | ✅ create_case | ❌ Never | ❌ BROKEN |
| 2 | PAYMENT_ENQUIRY | get_collection_account, get_last_transactions, get_payment_instructions | partial | ❌ Never | ❌ BROKEN |
| 3 | PAYMENT_CLAIM | create_payment_approval, create_payment_claim_case, notify_bm | ✅ (D-0600 handler) | ❌ Never | ❌ BROKEN |
| 4 | FACILITY_BOOKING | get_facilities_list, check_availability, create_booking | ✅ create_booking | ❌ Never | ❌ BROKEN |
| 5 | VISITOR_REGISTER | get_resident_unit, create_visitor_pass, generate_qr | ✅ create_visitor_pass | ❌ Never | ❌ BROKEN |
| 6 | PARCEL_ENQUIRY | get_parcel_status_for_unit | partial | ❌ Never | ❌ BROKEN |
| 7 | RENOVATION_ENQUIRY | get_renovation_rules, get_renovation_status | partial | ❌ Never | ❌ BROKEN |
| 8 | NOISE_COMPLAINT | get_resident_unit, create_case, identify_reported_unit, notify_bm | ✅ create_noise_complaint | ❌ Never | ❌ BROKEN |
| 9 | RENTAL_REQUEST | get_resident_unit, create_rental_listing, notify_agencies | partial | ❌ Never | ❌ BROKEN |
| 10 | BUTLER_REQUEST | create_in_unit_tender, charge_listing_fee, notify_contractors | partial | ❌ Never | ❌ BROKEN |
| 11 | COMPLAINT_FOLLOWUP | find_open_case, get_case_timeline | ✅ find_open_case | ❌ Never | ❌ BROKEN |
| 12 | GENERAL_ENQUIRY | get_building_info, get_facility_hours, get_emergency_contacts | partial | ❌ Never | ❌ BROKEN |
| 13 | LEGAL_SIGNAL | create_legal_record, advance_legal_stage, generate_form28, etc. | ✅ (legal.ts) | ❌ Never | ❌ BROKEN |
| 14 | EMERGENCY | create_urgent_case, notify_bm_immediately, send_emergency_broadcast | ✅ create_urgent_case | ❌ Never | ❌ BROKEN |
| 15 | WORK_COMPLETION | get_work_orders, mark_complete, request_photo, notify_bm | ✅ mark_complete | ❌ Never | ❌ BROKEN |
| 16 | WORK_UPDATE | get_work_orders, update_status | ✅ update_status | ❌ Never | ❌ BROKEN |
| 17 | TENDER_BID | get_open_tenders, charge_bid_fee, create_bid, notify_bm | ✅ create_bid | ❌ Never | ❌ BROKEN |
| 18 | ATTENDANCE_ISSUE | get_staff_record, notify_supervisor, create_flag | partial | ❌ Never | ❌ BROKEN |
| 19 | BM_SEARCH | search_documents, search_media, search_cases, search_residents | ✅ (admin path) | ✅ admin only | ⚠️ ADMIN ONLY |
| 20 | BM_REPORT | get_snapshot, get_performance, get_collection_summary | ✅ (admin path) | ✅ admin only | ⚠️ ADMIN ONLY |
| 21 | BM_SCHEDULE | create_schedule, assign_contractor, set_recurrence | ✅ (admin path) | ✅ admin only | ⚠️ ADMIN ONLY |
| 22 | PROCUREMENT_ORDER | get_inventory, get_supplier, create_po, notify_supplier | partial | ❌ Never | ❌ BROKEN |
| 23 | FORM11_APPROVE | generate_form11, approve_form11, notify_resident | ✅ approve_form11 | ✅ admin only | ⚠️ ADMIN ONLY |

**Root cause for all ❌ BROKEN rows:** `specialist-agent.ts` is never called for resident messages. Even if it were called, `executor.ts` is never imported anywhere outside its own file.

---

## Part 2: Pipeline Trace (The Broken Bridge)

### What should happen (designed architecture)

```
Resident message
  → WhatsApp/TG/Email webhook
  → runV4Pipeline() [routing only]
  → specialistAgent() [LLM generates actions[]]  ← MISSING CALL
  → executeActions() [executor.ts dispatches]    ← MISSING CALL
  → DB writes (create_case, create_booking, etc.)
  → Notify BM
  → Reply to resident
```

### What actually happens

```
Resident message
  → WhatsApp/TG/Email webhook
  → runV4Pipeline() [routing + intent classification]
  → v4Result.routing.response  ← undefined for Tier 1/2/3
  → if (responseText) sendMessage(...)  ← never executes for Tier 1/2/3
  → resident receives SILENCE
```

### Evidence

**`lib/ai/v4-pipeline.ts`** imports:
```typescript
import { executeAdminTool } from './actions/admin-executor';   // ✅ used for admin
import { executeFormSubmission } from './actions/form-executor'; // ✅ used for admin forms
// specialist-agent NOT imported
// executor.ts NOT imported
```

**`lib/ai/semantic-router.ts`** Tier 2/3 return:
```typescript
return {
  tier: 2,
  handler: 'light_ai',
  context: { intent: routerResult.intent_id, ... },
  reason: `Light AI: intent=...`,
  // NO response field
};
```

**`app/api/webhooks/whatsapp/route.ts` line 406:**
```typescript
const responseText = v4Result.routing.response; // undefined for Tier 1/2/3
if (responseText) {  // never true → resident gets nothing
  await sendTextMessage(msg.from, finalMessage, ...);
}
```

**`lib/ai/specialist-agent.ts`** — caller check:
```
grep -rn "specialistAgent" lib/ app/
→ Only found in: lib/ai/providers/index.ts (provider map), lib/ai/specialist-agent.ts (self)
→ NEVER imported by any webhook, pipeline, or route handler
```

**`lib/ai/actions/executor.ts`** — caller check:
```
grep -rn "executeActions" lib/ app/
→ Only found in: lib/ai/actions/executor.ts line 177 (definition)
→ NEVER imported anywhere
```

### Admin pipeline (works correctly)

```
BM chat message
  → app/api/bm/chat/send/route.ts
  → routeAdminMessage() [v4-pipeline, admin branch]
  → selectAdminTool() → executeAdminTool()
  → admin-handlers.ts (29 real handlers)
  → DB write + response returned
```

### specialist-agent.ts capability (built but unused)

`specialist-agent.ts` is fully implemented:
- Takes intent + profile context + fetched data
- Calls DeepSeek (ROUTINE) or Claude (SENSITIVE/LEGAL) with structured JSON prompt
- Parses `actions[]` from LLM output
- Runs hallucination guard on output
- Returns `AgentOutput { message, actions[], requires_bm_approval }`

But `actions[]` from `AgentOutput` are never passed to `executor.ts`.

---

## Part 3: Email Pipeline Status

| Check | Status | Evidence |
|-------|--------|---------|
| Email inbound webhook exists | ✅ YES | `app/api/webhooks/email/route.ts` |
| Calls runV4Pipeline() | ✅ YES | `lib/email/processor.ts:231` — via `processIncomingEmail()` |
| AI classifies intent | ✅ YES | `v4Result.routing.context.intent` stored on message |
| Ticket created when needed | ✅ YES | `v4Result.ticketCreated` → inserts into `tickets` |
| AI-generated reply sent | ❌ NO | Only sends neutral acknowledgment: "We received your email" |
| specialistAgent called | ❌ NO | Same gap as WA/TG — routing only |
| Actions executed | ❌ NO | Same gap |
| sendEmail() function exists | ✅ YES | `lib/email/sender.ts` via Resend API |

**Note:** Email auto-reply (D-0552) is by design a neutral receipt confirmation. But the AI response gap means email senders also get no substantive reply to complaints, booking requests, etc.

---

## Part 4: Risk Verification (R-01 to R-15)

### CRITICAL

**R-01: Telegram webhook secret fail-open**
```
❌ STILL OPEN
File: app/api/auth/telegram/webhook/route.ts:56
Code: if (envSecret && secret !== envSecret) { return 403; }
If TELEGRAM_WEBHOOK_SECRET is unset → envSecret is undefined → check SKIPPED
Any request passes through unauthenticated.
Fix: change to: if (!envSecret || secret !== envSecret)
```

**R-02: Anthropic API key from process.env (not credential vault)**
```
⚠️ PARTIALLY FIXED
File: lib/ai/providers/anthropic.ts:15-19
Code: this.apiKey = apiKey || process.env.ANTHROPIC_API_KEY || '';
      if (!this.apiKey) { throw new Error(...) }
Still uses process.env directly (not getCredential() vault).
Improvement: throws hard at construction time instead of console.warn.
Remaining: no credential vault abstraction, key rotation not supported.
```

**R-03: Hallucination guard fails-open on missing Supabase credentials**
```
❌ STILL OPEN
File: lib/ai/hallucination-guard-v4.ts:51-61
Code: if (!supabaseUrl || !supabaseServiceKey) {
        return { passed: true, hallucinations: [], sanitizedResponse: response, flagForReview: false };
      }
Validation is silently skipped — no log, no flagForReview=true, passes through unchecked.
Should set flagForReview: true and log a warning when credentials are missing.
```

### HIGH

**R-04: Material probe Redis key collision**
```
✅ FIXED
File: lib/ai/material-probe.ts:92
Code: await redis.set(materialProbeKey(user.id, caseData.id), ...)
Key includes both userId AND caseId — no collision between probes for different cases.
```

**R-05: buildingId injection not enforced in execute-query**
```
✅ FIXED
File: lib/ai/generic-read/execute-query.ts:37-40
Code: if (tableConfig.hasBuildingId) { query = query.eq('building_id', buildingId); }
Injection is unconditional for all tables with hasBuildingId: true.
```

**R-06: contacts_temp lacks channel/channel_id columns**
```
❌ STILL OPEN
File: lib/whatsapp/auto-register.ts:156-162
contacts_temp insert: { building_id, phone, full_name, message_preview, review_action }
No channel or channel_id fields inserted.
Auto-registration only supports WhatsApp — Telegram registrations would not be distinguishable.
Schema docs show no channel columns added to contacts_temp.
```

**R-07: No retry for material probe delivery failure**
```
❌ STILL OPEN
File: lib/ai/material-probe.ts:108-110
Code: } catch (err) { console.error('[material-probe] sendMaterialProbe error:', err); }
On send failure: Redis key remains set (probe appears pending), but message was never delivered.
Technician will never see the probe question. Key expires after 24h silently.
No retry, no redis.del cleanup on failure.
```

### MEDIUM

**R-08: auto_reg Redis state not cleared on BM rejection**
```
⚠️ PARTIALLY FIXED
File: lib/whatsapp/auto-register.ts
On human_handoff: redis.del(REGISTRATION_KEY(phone)) — ✅ cleared
On system validation failure (no unit match): redis.set(..., { ex: REGISTRATION_TTL }) — ✅ persists for retry (correct)
On BM rejection (approvals.status = 'rejected'): No webhook/trigger found that clears the Redis key.
After rejection, resident is still in "pending" state in Redis until TTL expires.
```

**R-09: No dedup for Telegram messages**
```
✅ FIXED (D-0585)
File: app/api/auth/telegram/webhook/route.ts:67-79
Code: const alreadySeen = await redis.set(dedupKey, '1', { ex: 86400, nx: true });
update_id stored in Redis with 24h TTL, nx:true prevents re-processing duplicates.
```

**R-10: ANTHROPIC_API_KEY exposure risk**
```
❌ STILL OPEN (same as R-02)
Both anthropic.ts and deepseek.ts read from process.env directly.
No credential vault (getCredential() or similar) implemented.
Risk: key visible in environment dumps, no rotation support.
```

**R-11: Admin category classifier confidence threshold 0.4**
```
✅ FIXED
File: lib/ai/v4-pipeline.ts:409
Code: if (adminRouting.selectedTool && adminRouting.confidence >= 0.6)
Threshold is 0.6 — matches semantic router threshold.
```

**R-12: looksLikeUnitNumber() can reject valid units**
```
✅ N/A — FUNCTION DOES NOT EXIST
lib/whatsapp/auto-register.ts uses DB-driven validation (ILIKE exact match → fuzzy match).
No hardcoded blocklist. Risk is resolved by design.
```

### LOW

**R-13: 2026 holidays hardcoded**
```
❌ STILL OPEN
File: lib/ai/after-hours.ts:42-62
const MY_PUBLIC_HOLIDAYS_2026 = ['2026-01-01', '2026-01-29', ...]  // 19 hardcoded dates
Array is merged with building-level config but has no 2027 entries.
When 2027 arrives, no Malaysian public holidays will be recognized.
```

**R-14: console.warn for missing API keys**
```
✅ FIXED
Both lib/ai/providers/anthropic.ts and lib/ai/providers/deepseek.ts now throw
Error at construction time instead of console.warn. Fail-closed behavior.
```

**R-15: Routing analytics stored in platform_settings**
```
❌ STILL OPEN
File: lib/ai/semantic-router.ts (logRoutingDecision function, bottom of file)
Code: console.log('[SemanticRouter]', JSON.stringify(logEntry));
Still only console.log — no routing_analytics table, no DB insert.
Analytics data is lost on pod restart.
```

### Risk Summary Table

| Risk | Severity | Status | Fix Effort |
|------|----------|--------|------------|
| R-01: Telegram webhook fail-open | CRITICAL | ❌ OPEN | 1 line |
| R-02: Anthropic key process.env | CRITICAL | ⚠️ PARTIAL | medium |
| R-03: Hallucination guard silent pass | CRITICAL | ❌ OPEN | 3 lines |
| R-04: Material probe key collision | HIGH | ✅ FIXED | — |
| R-05: buildingId injection | HIGH | ✅ FIXED | — |
| R-06: contacts_temp no channel | HIGH | ❌ OPEN | migration + code |
| R-07: Material probe no retry | HIGH | ❌ OPEN | small |
| R-08: auto_reg Redis on rejection | MEDIUM | ⚠️ PARTIAL | webhook trigger |
| R-09: Telegram dedup | MEDIUM | ✅ FIXED | — |
| R-10: API key vault | MEDIUM | ❌ OPEN | medium |
| R-11: Admin confidence threshold | MEDIUM | ✅ FIXED | — |
| R-12: looksLikeUnitNumber | MEDIUM | ✅ N/A | — |
| R-13: 2026 holidays hardcoded | LOW | ❌ OPEN | small |
| R-14: console.warn API keys | LOW | ✅ FIXED | — |
| R-15: Routing analytics lost | LOW | ❌ OPEN | migration + 5 lines |

---

## Part 5: Detailed Findings — Broken Action Paths

### F-01: Resident Tier 1/2/3 — Complete Silence (P1)

**What should happen:**  
Resident sends "my sink is leaking" → AI classifies MAINTENANCE_COMPLAINT → specialist agent fetches resident unit → creates case in DB → notifies BM → replies "Case #MC-001 created. BM will contact you within 24 hours."

**What actually happens:**  
Resident sends "my sink is leaking" → semantic router returns `{ tier: 2, handler: 'light_ai', context: {intent: 'MAINTENANCE_COMPLAINT'} }` → WhatsApp webhook: `responseText = v4Result.routing.response` → `responseText` is `undefined` → `if (responseText)` is false → **no message sent**. Resident receives nothing.

**Files to change:**
1. `lib/ai/v4-pipeline.ts` — add specialist agent call for Tier 2/3 resident messages. Import `runSpecialistAgent` from `lib/ai/specialist-agent.ts`.
2. `lib/ai/v4-pipeline.ts` — after specialist agent returns, call `executeActions()` from `lib/ai/actions/executor.ts`.
3. `lib/ai/v4-pipeline.ts` (or webhook) — populate `routing.response` with the specialist agent's `message` field before returning.

**Priority:** P1 — this is the entire resident AI experience.

### F-02: MAINTENANCE_COMPLAINT — Case Not Created (P1)

**What should happen:** `create_case` action → inserts row into `cases` table → returns case ID → notifies BM via Telegram.

**What actually happens:** Action defined in intents.ts tools, handler exists in executor.ts, but `executeActions()` is never called.

**Files:** `lib/ai/actions/executor.ts` (handler exists at `complaints.ts`/`maintenance.ts`) — just needs to be called.

### F-03: PAYMENT_CLAIM — No Approval Created (P1)

**What should happen:** Resident sends payment proof → AI calls `create_payment_approval` → inserts into `approvals` table → BM sees payment pending in Console.

**What actually happens:** Handler `lib/ai/handlers/create-payment-approval.ts` was created in D-0600 and registered in `admin-handlers.ts`, but this is admin-side only. The resident-channel path through `executor.ts` never calls it.

**Files:** `lib/ai/actions/executor.ts` — add `create_payment_approval` case calling `lib/ai/handlers/create-payment-approval.ts`.

### F-04: FACILITY_BOOKING — No Booking Created (P1)

**What should happen:** Resident requests gym booking → AI checks availability → creates booking row → confirms slot to resident.

**What actually happens:** Routed to Tier 1 (`direct_facility_booking`) or Tier 2 → silence.

**Files:** `executor.ts` handler for `create_booking` exists but is unreachable.

### F-05: VISITOR_REGISTER — No Pass Created (P1)

**What should happen:** Resident registers visitor → `create_visitor_pass` action → row in `visitor_passes` → QR sent back to resident.

**What actually happens:** Same gap. Tier 1 handler `direct_visitor_register` classifies intent but nothing executes.

**Files:** `executor.ts` handler for `create_visitor_pass` exists but is unreachable.

### F-06: EMERGENCY — No Urgent Case + No Broadcast (P1)

**What should happen:** Emergency message → `create_urgent_case` + `notify_bm_immediately` + `send_emergency_broadcast`.

**What actually happens:** Tier 0 emergency keywords might catch some patterns. Otherwise → silence.

**Files:** `executor.ts` handler for `create_urgent_case` exists but is unreachable.

### F-07: Email — No AI-Generated Reply (P2)

**What should happen:** Email complaint → AI generates personalized reply with case reference.

**What actually happens:** Auto-reply is neutral "we received your email" (by design per D-0552). Even if D-0552 is revisited, the specialist agent gap means no AI content would be generated anyway.

**Files:** Once F-01 is fixed (specialist agent wired), email pipeline will automatically benefit since `processIncomingEmail()` already calls `runV4Pipeline()`.

---

## Part 6: Recommended Fix Order

### Phase A — Critical 1-liners (do immediately)

1. **R-01 fix** — `app/api/auth/telegram/webhook/route.ts:56`  
   `if (envSecret && ...)` → `if (!envSecret || ...)`  
   **Risk if not fixed:** Unauthenticated Telegram webhook calls accepted in production.

2. **R-03 fix** — `lib/ai/hallucination-guard-v4.ts:57`  
   Change `flagForReview: false` → `flagForReview: true` and add `console.warn('[HallucinationGuard] Supabase creds missing — skipping validation')`  
   **Risk if not fixed:** Hallucinated data passes through silently when DB is unreachable.

### Phase B — Wire the resident action pipeline (high-value, 1-2 days)

3. **F-01/F-02/F-03/F-04/F-05/F-06** — Connect specialist-agent + executor to v4-pipeline resident path.  
   Add to `lib/ai/v4-pipeline.ts` after Tier 2/3 routing for resident messages:
   ```typescript
   import { runSpecialistAgent } from './specialist-agent';
   import { executeActions } from './actions/executor';
   // After semanticRoute() for tier 2/3 non-admin:
   const agentOutput = await runSpecialistAgent({ intent, profile, fetchedData, ... });
   await executeActions(agentOutput.actions, { buildingId, residentId, ... });
   routing.response = agentOutput.message;  // populate response field
   ```
   This single change fixes all 23 resident intents simultaneously.

### Phase C — Data quality fixes (medium, 1-3 hours each)

4. **R-07** — material-probe.ts: add `redis.del(materialProbeKey(user.id, caseData.id))` in catch block.

5. **R-13** — after-hours.ts: move holidays to `platform_settings` table or building_working_hours config. Add a 2027+ fallback.

6. **R-06** — Migration: add `channel TEXT` and `channel_id TEXT` to `contacts_temp`. Update auto-register insert.

### Phase D — Architecture improvements (low urgency)

7. **R-15** — Create `routing_analytics` table. Replace `console.log` in `logRoutingDecision()` with DB insert.

8. **R-08** — Add webhook/trigger on `approvals.status = 'rejected'` for `resident_registration` type → `redis.del(REGISTRATION_KEY(phone))`.

9. **R-10/R-02** — Credential vault: implement `getCredential(key)` abstraction around process.env for API keys (enables rotation, audit trail).

---

## Files Read During This Audit

```
lib/ai/intents.ts
lib/ai/specialist-agent.ts
lib/ai/types.ts
lib/ai/types/ai-response.ts
lib/ai/v4-pipeline.ts
lib/ai/semantic-router.ts
lib/ai/actions/executor.ts
lib/ai/actions/admin-handlers.ts
lib/ai/actions/admin-executor.ts
lib/ai/actions/form-executor.ts
lib/ai/actions/admin-init.ts
lib/ai/providers/anthropic.ts
lib/ai/providers/deepseek.ts
lib/ai/providers/index.ts
lib/ai/hallucination-guard-v4.ts
lib/ai/material-probe.ts
lib/ai/after-hours.ts
lib/ai/generic-read/execute-query.ts
lib/ai/handlers/create-payment-approval.ts
lib/ai/handoff.ts
lib/whatsapp/auto-register.ts
app/api/webhooks/whatsapp/route.ts
app/api/webhooks/email/route.ts (does not exist at this path)
app/api/auth/telegram/webhook/route.ts
lib/email/processor.ts
lib/email/auto-reply.ts
lib/email/sender.ts
docs/startup/actual-schema-columns.md (contacts_temp section)
docs/startup/recent-decisions.md (AI-Pipeline section)
graphify-out/GRAPH_REPORT.md
```
