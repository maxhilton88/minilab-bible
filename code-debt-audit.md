# Code Debt Audit — V28 (2026-04-13)

> **READ-ONLY AUDIT** — no code was modified. Use this as the fix agenda for Wednesday's cleanup session.

---

## TypeScript Errors (96 total across 21 files)

### Root Cause Summary

| Root Cause | Files Affected | Errors | Fix |
|------------|---------------|--------|-----|
| `database.types.ts` stale (not regenerated after recent migrations) | building-tenancy routes, guard routes, collection/escalation | ~76 | `npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea > database.types.ts` |
| Invalid role strings in `.in('role', [...])` | lib/ai/handoff.ts, lib/crons/permit-expiry.ts | 3 | Remove "building_manager", "admin" — use "bm" etc. |
| Invalid permission string `"settings.view"` | app/api/bm/settings/ai-fallback, bm/whatsapp/template-status | 2 | Change to a valid permission key |
| Missing `await` on Supabase `.notify()` / `.insert()` (PromiseLike .catch) | telegram webhook, whatsapp webhook, email/processor x2, semantic-router | 5 | Add `await` before call |
| Duplicate `ContactListItem` local interface (missing `handoff_case_*` fields) | app/bm/console/page.tsx | 5 | Remove local interface, import from console-shared.tsx |
| Bad type cast via `as` on SelectQueryError | lib/crons/pdpa-deletion.ts | 2 | Use `as unknown as T` or fix the `.select()` |
| Schema mismatch post-migration | lib/crons/pdpa-deletion.ts, collection/escalation-settings | 4 | Fix after type regen |
| Wrong role type values | lib/ai/handoff.ts | 2 | Fix role enum values |
| Architectural type mismatch | lib/moa/dispatch-enforcement.ts, residents/data-quality/staging | 3 | See per-file notes |

---

### Trivial (≈10) — estimate: 15 minutes total

**Single command eliminates 76 errors:**
```bash
npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea > database.types.ts
```
This fixes all `building_tenancies`, `building_spaces`, `pass_number` (visitors), `block_minimum_amount` (collection_settings) type mismatches.

| File | Line | Error | Fix |
|------|------|-------|-----|
| app/api/bm/settings/ai-fallback/route.ts | 8 | `"settings.view"` not valid permission | Change to `"settings.manage"` or valid key |
| app/api/bm/whatsapp/template-status/route.ts | 19 | `"settings.view"` not valid permission | Same fix |
| lib/ai/handoff.ts | 91 | `"building_manager"`, `"admin"` not valid roles | Change to `"bm"`, `"security_admin"` (or remove) |
| lib/crons/permit-expiry.ts | 109 | `"admin"` not valid role | Change to `"bm"` |
| app/bm/console/page.tsx | 42 | Local `ContactListItem` missing `handoff_case_*` fields | Delete local interface, import from `components/bm/console/console-shared.tsx` |

---

### Medium (≈10) — estimate: 25 minutes total

| File | Line | Error | Fix |
|------|------|-------|-----|
| app/api/auth/telegram/webhook/route.ts | 459 | `.catch` on `PromiseLike` — missing await | Add `await` before the Supabase call |
| app/api/webhooks/whatsapp/route.ts | 294 | `.catch` on `PromiseLike` — missing await | Add `await` |
| lib/email/processor.ts | 178, 216 | `.catch` on `PromiseLike` — missing await | Add `await` on both |
| lib/ai/semantic-router.ts | 429 | `.catch` on `PromiseLike` + implicit `any` | Add `await`; add `: unknown` to catch param |
| lib/crons/pdpa-deletion.ts | 34 | Bad `as` cast on `SelectQueryError` | Change to `as unknown as { id: string; ... }` |
| lib/crons/pdpa-deletion.ts | 87 | `.catch` on `PostgrestFilterBuilder` | Add `await` (or use `.then().catch()` pattern) |
| app/api/bm/collection/escalation-settings/route.ts | 63, 82, 94, 108 | Schema mismatch after migration | Fix after type regen; likely `block_minimum_amount` needs regen |
| app/api/bm/renovation/route.ts | 64, 70 | Wrong type on insert result | Destructure properly after `.select()` |

---

### Complex (≈10) — estimate: 30 minutes total

| File | Line | Error | Fix Notes |
|------|------|-------|-----------|
| lib/moa/dispatch-enforcement.ts | 87 | TS2769 no overload match | Review `.from()` call — likely new table post-migration needs regen |
| app/api/bm/residents/data-quality/staging/[id]/route.ts | 77, 94 | TS2769 no overload | Likely stale types; check after regen |
| app/bm/console/page.tsx | 1582–1672 | Type mismatch on props passed to child components | After fixing local interface, these should cascade-resolve |

---

## .single() Triage (525 total across 197 files)

525 `.single()` calls is a lot. `.single()` throws PGRST116 if 0 or 2+ rows returned, crashing the route. Use `.maybeSingle()` for lookups where 0 rows is a valid state.

### Critical — Auth & Webhook Paths (fix first, production crash risk)

| File | Count | Severity | Note |
|------|-------|----------|------|
| app/api/auth/telegram/webhook/route.ts | 22 | HIGH | Auth logic — any lookup can fail on 0 rows |
| app/api/auth/building-login/route.ts | 1 | HIGH | Login crash if slug not found |
| app/api/auth/google/callback/route.ts | 3 | HIGH | OAuth flow crash |
| app/api/auth/vendor-login/route.ts | 1 | HIGH | |
| app/api/auth/resident/check-otp/route.ts | 1 | HIGH | |
| app/api/webhooks/whatsapp/route.ts | 2 | HIGH | Webhook — crash on unknown number |
| app/api/webhooks/stripe/route.ts | 1 | MEDIUM | Billing webhook |

### High — Guard & Kiosk (real-time production operations)

| File | Count | Note |
|------|-------|------|
| app/api/guard/visitors/checkin/route.ts | 2 | Checkin crash if visitor not found |
| app/api/guard/visitors/checkout/route.ts | 1 | |
| app/api/guard/visitors/lookup/route.ts | 2 | |
| app/api/guard/incidents/route.ts | 4 | |
| app/api/guard/init/route.ts | 1 | Gate init crash |
| app/api/kiosk/verify/route.ts | 2 | Kiosk auth crash |
| app/api/kiosk/register/route.ts | 3 | |
| app/api/attendance/clock/route.ts | 4 | Attendance crash |
| app/api/attendance/verify/route.ts | 3 | Face verify crash |

### Low — Internal BM/Admin/Lib (lower traffic, catch blocks usually present)

All other files in app/api/bm/, lib/ai/, lib/crons/ etc. — 450+ calls. Not urgent but a systematic `.maybeSingle()` pass is recommended.

---

## Dead Code

### Files That Reference Unimplemented APIs (TODO markers confirm)

| File | Issue |
|------|-------|
| app/superadmin/billing-settings/page.tsx | UI exists but GET/PATCH API routes are TODO stubs — page is non-functional |
| app/superadmin/supplier-pricing/page.tsx | UI exists but GET API route is TODO — page non-functional |
| app/api/bm/collection/installment/route.ts | Entire route is TODO: "installment_plans table not yet created" |

### SSM Placeholder in Marketing Footer

- `app/(marketing)/_components/marketing-footer.tsx:196` — SSM number is `XXXXXXXXXX` (placeholder, not real registration number)

---

## Code Smells

### Empty Catch Blocks — 319 instances

Most are intentional `} catch { /* ignore */ }` patterns for non-critical paths (SSE stream close, JSON parse fallbacks). A sample of ones to review:

| File | Line | Pattern |
|------|------|---------|
| app/(superadmin-auth)/superadmin/login/page.tsx | 42, 72 | Silent catch — login errors swallowed |
| app/(telegram-forms)/telegram-forms/[action]/page.tsx | 75 | Silent catch |
| app/api/app/cleaning-areas/route.ts | 46 | Silent catch |
| app/api/attendance/clock/route.ts | 44 | Silent catch on clock-in |
| app/api/auth/building-login/route.ts | 22 | Silent catch on login |
| app/api/auth/google/callback/route.ts | 15 | Silent catch on OAuth |

Most in `try { body = await req.json(); } catch { return error }` are fine. The ones in auth/login paths are higher risk.

### console.log in Production — 87 instances

| File | Count | Note |
|------|-------|------|
| app/api/bm/settings/telegram-groups/route.ts | 11 | High noise |
| lib/voice/gemini-live.ts | 10 | Voice pipeline debug |
| lib/email/processor.ts | 10 | Email debug |
| lib/chat/telegram-sync.ts | 5 | Telegram sync |
| lib/telegram/group-service.ts | 4 | |
| lib/ai/generic-read/index.ts | 4 | AI read pipeline |
| lib/ai/ai-pause-check.ts | 4 | |

**Recommendation:** Convert all to `console.error` or remove. The top 3 files alone account for 31 logs.

### `as any` Casts — 1,292 instances

This is the highest debt item by volume. Not all are equally bad — many are in Supabase query results where the type isn't inferrable. However, 1,292 is a sign of systematic type-safety bypass.

**Top offenders to audit first:**
```bash
grep -rn " as any" --include="*.ts" --include="*.tsx" lib/ | sed 's/:.*//' | sort | uniq -c | sort -rn | head -20
```

### TODO/FIXME Comments — Key Items

| File | Comment | Priority |
|------|---------|---------|
| app/api/bm/collection/installment/route.ts | installment_plans table not created | HIGH — feature stub, route returns empty |
| lib/ai/actions/bm.ts:30 | T061: Replace with proper vector similarity query | MEDIUM — AI search using fallback |
| app/api/contact/route.ts | TODO: Send via Resend when RESEND_API_KEY configured | LOW |
| app/api/bm/collection/approve-payment/route.ts | TODO: wire WhatsApp when ready | LOW |
| app/api/developer/dashboard/route.ts | avgRectificationDays = 0 | LOW — metric always zero |
| app/api/setup/wizard/route.ts | TODO: Send Telegram invitation for unregistered providers | LOW |
| Multiple app/api/*.ts | `// TODO: regenerate types` | Resolved by type regen |

### Potential Hardcoded Secrets — NONE FOUND

All grep matches were false positives:
- `mask-image` CSS values (not secrets)
- Dev login passwords (`1234Pass!`) in `app/dev/login/_form.tsx` — acceptable for dev-only route (but confirm `DISABLE_DEV_LOGIN=true` in prod as noted in CLAUDE.md §7 pending fixes)
- `password` state variable names in UI components — legitimate

---

## API Route Health

### Missing try-catch — 227 routes

The majority of routes have no try-catch at all. This is mostly acceptable if Supabase errors are handled via `if (error) return NextResponse.json(...)` pattern (which is idiomatic). However, unhandled promise rejections from other operations (JSON parse, crypto, external calls) will crash these routes.

**Routes that should have try-catch (non-Supabase operations inside):**

| File | Why |
|------|-----|
| app/api/bm/import/upload/route.ts | File processing — crash risk |
| app/api/bm/residents/import/route.ts | CSV parsing — crash risk |
| app/api/face/enroll/route.ts | Face model inference — crash risk |
| app/api/onboard/photo-upload/route.ts | File upload — crash risk |
| app/api/marketplace/quotes-pdf/route.ts | PDF generation — crash risk |

All cron routes lacking try-catch are lower risk as they run server-side with no user impact.

### Missing Auth — Flagged Routes Requiring Review

These routes had no recognizable auth pattern. Many are intentionally unauthenticated (QR polling, models, public endpoints). **Audit these specifically:**

| File | Risk | Note |
|------|------|------|
| app/api/moa/commands/[id]/route.ts | HIGH | MOA command execution — should require internal auth |
| app/api/moa/systems/route.ts | MEDIUM | System listing |
| app/api/superadmin/exit-impersonate/route.ts | MEDIUM | Privilege operation |
| app/api/bm/dashboard/ageing/route.ts | MEDIUM | Financial data |
| app/api/bm/procurement/ai-order/route.ts | MEDIUM | AI ordering |
| app/api/bm/fines/route.ts | MEDIUM | Fine management |
| app/api/kiosk/dashboard/route.ts | LOW | Kiosk uses device auth |
| app/api/guard/init/route.ts | LOW | Guard kiosk init |
| app/api/models/[...path]/route.ts | NONE | Intentional — face models are public |
| app/api/auth/* | NONE | Intentional — auth routes are pre-auth |

### API Routes Using .single() — Already covered above

---

## Schema Drift

### Tables in Code but NOT in Docs — 0 ✓

No undocumented tables found in code. Schema docs are complete.

### Tables in Docs but NOT Referenced via `.from()` in Code

Many tables in docs aren't directly `.from()`'d — they're accessed via RPCs, joins, or supabaseAdmin with different patterns. This is **not** drift — the doc list is authoritative. No action needed.

### Columns Missing from database.types.ts (confirmed by TS errors)

These columns exist in the DB but types haven't been regenerated:

| Table | Missing Column(s) | Evidence |
|-------|------------------|----------|
| `building_tenancies` | entire table missing | TS2769 on `.from('building_tenancies')` |
| `building_spaces` | entire table missing | TS2769 on spaces route |
| `visitors` | `pass_number` | SelectQueryError in guard routes |
| `collection_settings` | `block_minimum_amount` | SelectQueryError in escalation route |

**Fix: Run type regen once, commit `database.types.ts`.**

---

## Recommended Fix Order (Wednesday Session)

### Phase 1 — One Command, 76 Errors Gone (5 minutes)

```bash
npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea > database.types.ts
git add database.types.ts && git commit -m "chore: regenerate Supabase types (building_tenancies, pass_number, spaces)"
git push origin main
```

### Phase 2 — Trivial Fixes (30 minutes)

1. `lib/ai/handoff.ts:91` — remove `"building_manager"`, `"admin"` from role array (2 errors)
2. `lib/crons/permit-expiry.ts:109` — remove `"admin"` from role array (1 error)
3. `app/api/bm/settings/ai-fallback/route.ts:8` — fix permission string (1 error)
4. `app/api/bm/whatsapp/template-status/route.ts:19` — fix permission string (1 error)
5. `app/bm/console/page.tsx:42` — delete local `ContactListItem`, import from console-shared (5 errors)
6. Remove top 30+ `console.log` statements from `telegram-groups/route.ts` and `lib/voice/gemini-live.ts`

### Phase 3 — Medium Fixes (45 minutes)

7. Add `await` to 5 Supabase calls that are missing it (PromiseLike .catch errors)
8. Fix `pdpa-deletion.ts` cast (2 errors)
9. Fix `renovation/route.ts` type inference (2 errors)
10. Check and fix auth on `moa/commands/[id]`, `superadmin/exit-impersonate`

### Phase 4 — .single() Auth Path Hardening (60 minutes, do last)

11. Audit the 22 `.single()` calls in telegram webhook — convert lookups to `.maybeSingle()`
12. Audit 3 `.single()` calls in google/callback
13. Convert guard/visitor checkin/checkout/lookup to `.maybeSingle()`

### Phase 5 — Ongoing / Backlog

- `as any` audit (1,292 instances) — tackle file-by-file over multiple sessions
- `} catch { }` review — focus on auth/login paths
- Complete unimplemented pages: billing-settings, supplier-pricing, installment route
- Replace `console.log` with `console.error` across all lib/ files

---

## Effort Summary

| Phase | Items | Estimated Time |
|-------|-------|---------------|
| Type regen | 1 command | 5 min |
| Trivial fixes | ~15 items | 30 min |
| Medium fixes | ~8 items | 45 min |
| .single() auth hardening | ~30 calls | 60 min |
| **Total Wednesday session** | | **~2.5 hours** |

**Errors eliminated after Phase 1+2+3: ~91 of 96** (remaining 5 are complex architectural issues in moa/dispatch-enforcement and data-quality/staging).
