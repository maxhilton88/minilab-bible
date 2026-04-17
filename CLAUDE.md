# CLAUDE.md — Minilab Platform
<!-- Version: V35 · 2026-04-18 -->
<!-- THIS IS THE ONLY FILE READ AT STARTUP. Everything else is lazy-loaded via the Router. -->

---

## §1 · What Is Minilab

Minilab (minilab.my) — AI-powered ERP SaaS for the entire Malaysian strata property management industry.
Stack: Next.js 15 App Router · TypeScript strict · Supabase PostgreSQL · Vercel · Tailwind v3 + shadcn/ui
DB: Supabase project `nncbsumdmsyjjrgdjjea` (Singapore region)
Serves: JMB/MC buildings, PMC, security, cleaning, contractor, supplier/store, developer, residents across portfolio-scale operations. NOT a single-building tool.

---

## §2 · Session Protocol

1. Read THIS file (CLAUDE.md) — nothing else yet
2. Run: git log --oneline -10 && git status
3. Check §4 Router below — identify which docs (if any) your task needs
4. Extract only the relevant sections using awk (see §3)
5. Do the work
6. Run Closing Block (§6)

---

## §3 · How To Read Docs (awk, never cat)

Every lazy-loaded doc uses `## §Anchor-Name` headers as semantic markers.

Extract a section (up to the next anchor):

    awk '/^## §Face-Recognition/,/^## §/' docs/startup/recent-decisions.md | head -n -1

List all anchors in a file:

    grep '^## §' docs/startup/recent-decisions.md

Extract a table section from schema:

    awk '/^## §Table-attendance_logs/,/^## §/' docs/startup/actual-schema-columns.md | head -n -1

Rules:
- NEVER cat a full doc over 50 lines. Always awk the section you need.
- If you don't know which anchor to use, grep '^## §' first to see the index.
- If a doc has no anchors yet, read it fully ONCE, then add anchors as part of your closing block.
- Anchors are semantic — they never break when content is added/removed (unlike line numbers).

---

## §4 · Bible Router

Match your task to a row. Read only what's listed. If no row matches, you probably don't need extra docs.

### Schema & Database

| Task | File | Anchor |
|------|------|--------|
| New migration / new table | actual-schema-columns.md | §Table-{name} for related tables |
| Alter column on existing table | actual-schema-columns.md | §Table-{name} |
| Check if column/table exists | actual-schema-columns.md | §Table-{name} |
| New RPC function | actual-schema-columns.md | §RPCs |
| New env var needed | env-required.md | Full file OK (small, ~7KB) |

### Feature Areas → Decision History

| Task | File | Anchor |
|------|------|--------|
| Face recognition / enrollment | recent-decisions.md | §Face-Recognition |
| Attendance / clock-in-out | recent-decisions.md | §Attendance |
| AI agent / pipeline / routing | recent-decisions.md | §AI-Pipeline |
| Auth / login / portal routing | recent-decisions.md | §Auth-Routing |
| Billing / Stripe / credits | recent-decisions.md | §Billing |
| Guard VMS / visitor mgmt | recent-decisions.md | §Guard-VMS |
| PMC portal / multi-building | recent-decisions.md | §PMC-Portal |
| Onboarding PWA / staff reg | recent-decisions.md | §Onboarding |
| Console / inbox / messaging | recent-decisions.md | §Console |
| Unit search / normalization | recent-decisions.md | §Console |
| Telegram bot / groups / Telethon | recent-decisions.md | §Telegram |
| Committee portal | recent-decisions.md | §Committee |
| Procurement / hardware store | recent-decisions.md | §Procurement |
| Collection / chase engine | recent-decisions.md | §Collections |
| MOA / office agent / Advelsoft | recent-decisions.md | §MOA |
| MOA architecture / build spec / skills / connectors | MOA-SPEC.md | Full file OK |
| Resident portal / WhatsApp OTP | recent-decisions.md | §Resident |
| Supplier / store portal | recent-decisions.md | §Supplier-Store |
| Data backfill / re-import gaps | recent-decisions.md | §Data-Backfill |
| TypeScript errors / TS cleanup | recent-decisions.md | §TypeScript-Cleanup |
| Case / task detail sheet / inspection scheduling | recent-decisions.md | §Case-Management |
| Staff management / bulk actions / all-staff page | recent-decisions.md | §Staff-Management |

### Architecture & Code Navigation

| Task | File | Anchor |
|------|------|--------|
| Where is feature X in code? | GRAPH_REPORT.md | §Community-Index |
| What calls what? / dependencies | GRAPH_REPORT.md | §God-Nodes + §Connections |
| Permission / RLS questions | recent-decisions.md | §Permissions |
| Portal routing logic | NONE — see §5 Hard Rules below | — |

### Deep Reference (rarely needed)

| Need | File | When |
|------|------|------|
| Full V3-V12 audit history | docs/audit/*.md | Debugging regression from old code |
| AI agent architecture deep-dive | docs/minilab-ai-agent-architecture.md | MOA build or AI pipeline rewrite |
| Capabilities map | docs/capabilities-map.md | Product-level overview |
| Full raw decision history D-0001+ | docs/decisions.log | Only if recent-decisions.md doesn't cover it |

---

## §5 · Hard Rules (always in memory — never violate)

Billing: JMB/MC always free (no trial, no subscription). Service providers pay per-staff + unit-tier + credit packs. All pricing in billing_config table (superadmin editable, zero hardcoded values).

Hardware Store: NO inventory system. Telegram orders → invoice billing only.

Technician: A building staff member (role=staff, position=Technician), NOT a contractor_worker.

Face: face-api.js + pgvector, 128-dim descriptors, cosine similarity threshold 0.6, match_face() RPC. No manual clock-in ever. Face IS identity — no name dropdown. One photo = one enrollment (autoEnrollFace utility). No separate FaceEnrollModal.

Attendance: Unlimited in/out pairs per day. Late/early/OT calculated from building_working_hours.

Portal routing: Permanent, locked, via lib/auth/get-portal-path.ts. NEVER duplicate this logic anywhere.
- Building staff phone → /app · BM/admin PC → /bm/dashboard
- Guard/cleaner/tech PC → /app · Committee phone → /app PC → /committee/dashboard
- Resident → /resident always · Store = Supplier (same portal)
- ORG admins → their desktop portal always

PWA API routes: Use getSession() not requirePermission().

Types: NEVER modify database.types.ts manually — regenerate:
npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea

RPC Caller Rule: When ANY RPC signature changes → grep -r ".rpc('function_name')" --include="*.ts" --include="*.tsx" → update ALL callers in same commit → NOTIFY pgrst 'reload schema'.

TypeScript discipline: `typescript.ignoreBuildErrors` MUST stay false (or absent) in next.config.js. New TS errors must be fixed before commit. If genuinely unavoidable, use `@ts-expect-error` with a comment explaining why. NEVER `@ts-ignore`. NEVER `as any` (except `.from()` on tables not yet in generated types — always with `@ts-expect-error` noting which table and why). NEVER non-null `!` assertions without a prior null check in the same scope.
Before any commit: `npx tsc --noEmit 2>&1 | grep -c "error TS"` → must return 0. V32d removed ignoreBuildErrors after clearing 108 errors — ZERO errors is the permanent baseline.

Migration discipline: Every migration file MUST be applied to prod in the same session it is created. Verify immediately after apply:
  SELECT name FROM supabase_migrations.schema_migrations WHERE name LIKE '{number}%';
If migration contains CREATE POLICY without IF NOT EXISTS, note in closing block that re-execution is unsafe (register-only on re-apply). Closing block MUST confirm migration applied, not just file created.

PMC endpoint rule: All /api/bm/* handlers that call requirePermission() MUST pass req as second arg. check-api-permission.ts throws in dev if PMC session detected without req. Enforced since D-0730. No exceptions within /api/bm/* except /billing/* and /settings/danger/* (those don't use requirePermission at all).

Git: Push to main only. One task = one commit + push. No branches. If push fails: git pull --rebase origin main && git push origin main.

Infra:
- Telethon microservice: GCP e2-micro 34.42.210.102:8443
- R2 bucket: minilab-media · public URL: https://media.minilab.my · Account ID: 585f5b927632127fe9545602c82c0b09
- Sentry: gated on SENTRY_DSN env var
- Vercel Hobby: 1 daily cron only (/api/cron/daily) · Deploy via Deploy Hook (Vercel webhook broken)
- Face models: /api/models/face-api/
- Service provider registration: Google OAuth + email/password

---

## §6 · Closing Block (every session end — no exceptions)

1. DECISIONS: Append to docs/startup/recent-decisions.md under the correct §Feature-Area anchor. Use next D-XXXX number. If no anchor exists for this feature, CREATE one.
2. SCHEMA: If any table/column changed → update docs/startup/actual-schema-columns.md. Update the relevant §Table-{name} section. If new table → add new §Table-{name} section.
3. ENV: If new env var → update docs/startup/env-required.md
4. NEW TABLE CHECKLIST: Create ai_view_ for it, add to generic_read allowlist, add keywords to ai_tools.
5. ROUTER: If this session introduced a new feature area or pattern → add a row to §4 Router in THIS file.
6. STATE: Update §7 Current State below (version, counts, pending items).
7. GRAPH: If major architectural change → update docs/startup/graph-report.md.
8. LIST files changed.
9. COMMIT with prefix (feat: | fix: | docs: | refactor: | chore:) and push to main.
10. SYNC BIBLE: automatic via .husky/pre-push on git push. Ensure bible §7 Current State is updated when counts change (version, tables, pages, routes, decisions).

---

## §7 · Current State

Version: V35 (open 2026-04-17)
Database: 178 tables · migrations 001-104
Pages: 341 · API routes: 568 · Decisions: D-0740 latest · Portals: 17+
Note: 049, 053 DB-only migrations have no repo files (legacy manual, non-blocking)

V34 accomplishments:
- D-0726: Bible catch-up + auto-sync pre-push hook (no more manual sync)
- D-0727: 23 cleaners unblocked (FIX-1A' backfill + enum + routing fix)
- D-0728: Face-only clock-in enforced per bible §5 (kiosk + PWA + API)
- D-0729: Per-user unread_count wired into console threads
- D-0730: 135-file PMC silent-401 sweep + permission hardening
- D-0731: V34 AI+MOA stack audit — full read-only prep audit for V35 Opus brainstorm
- D-0732: V35 Behavioral Audit — 8 confirmed bugs with exact file:line root causes

V35 status: Behavioral audit ✅ DONE — docs/audits/V35-BEHAVIORAL-AUDIT.md
V35 status: Architecture recon ✅ DONE — docs/audits/V35-ARCHITECTURE-RECON.md (D-0733)
V35 status: W1 RAG wiring ✅ DONE — Tier 3 now actually retrieves (D-0734)
V35 status: W2 stateJson injection ✅ DONE — specialist agent now sees resident profile (D-0735)
V35 status: B1 BM auto-pause ✅ DONE — BM outbound sets 4h TTL, AI silenced (D-0736)
V35 status: H1 PDF email flow ✅ DONE — PDF URL forwarded, Tier 1 inline approval + ACK (D-0737)
V35 status: B4 automated sender filter ✅ DONE — bank/robot emails skip AI, message stored (D-0738)
V35 status: H2B unit hallucination + H2A nudge state ✅ DONE — LLM extracts raw only, substring guard, always-confirm turn, state-aware nudge (D-0739)
V35 status: Night 1 closeout ✅ DONE — docs synced, V35-NIGHT-1-HANDOFF.md created (D-0740)
Next V35 action: Phase 1 complete — CHV re-enable blockers all resolved. Next: deploy + CHV sign-off

V35 active priorities (as of 2026-04-18):

✅ All CHV re-enable blockers cleared (W1, W2, B1, H1, B4, H2B+H2A — D-0734 through D-0739)

Next sessions:
- B3: ai_actions_log table + executor.ts instrumentation (observability)
- A1: Audit loop at minilab.my/__audit/findings — ai_audit_findings + ai_audit_runs tables + 8 starter rules + superadmin-gated page + 6h cron
- R1: Staged CHV re-enable (draft-approval mode first via D-0424 infrastructure; then auto on ROUTINE confidence ≥ 0.75; then full)

Deferred (not P0):
- PDF vision extraction (H1 layer 2 — Gemini can read PDFs)
- Model migration (Haiku 4.5 for auto-reg or specialist agent — pending CHV traffic evidence)
- WhatsApp/Telegram automated sender filter (B4 scope was email-only)
- Orphan file cleanup (5 V3 files + OpenAI embedder.ts + retriever.ts legacy imports)
- MOA Phase 2 recon (CHV walkthrough — access card + car plate vendor names, Advelsoft skill JSON) — blocker for MOA live
- FIX-1B — resolveUser silent 'bm' default hardening (defensive, non-urgent)
- V32b — VMS floor-3 blindspot (needs device screenshot)
- V32c — GQ ground-floor notation (fold into V32b)

Open observations for next session:
- Audit rule design for A1 is a product decision — Opus should design with Max, not ship blind
- R1 is a morning/daytime operation — not a night ship — Max observes first inbound threads on real CHV traffic

Key contacts: Seeteng (Lumi, test user), Hoe Zee How (CHV, 712 units, active)

---

## §8 · Doc Inventory

docs/startup/ — Working docs (lazy-loaded via §4 Router)
  actual-schema-columns.md    148KB  Anchored by §Table-{name}
  recent-decisions.md         351KB  Anchored by §Feature-Area
  env-required.md             6.4KB  Small — OK to read fully
  GRAPH_REPORT.md             9.8KB  Anchored by §Community / §God-Nodes

docs/ — Archive (touch only when Router points here)
  decisions.log               DEPRECATED — historical D-0001 through D-0668 only (see top of file)
  session-context.md           35KB  Legacy (superseded by §7)
  SESSION-SOP.md               13KB  Legacy (superseded by §2+§6)
  minilab-ai-agent-architecture.md  34KB  AI deep-dive (for MOA)
  capabilities-map.md          22KB  Feature overview
  docs/audit/                 ~800KB  V3-V14 audit reports (moved here)
  docs/audits/V34-AI-STACK-AUDIT.md  AI+MOA stack audit for V35 brainstorm (D-0731)
  docs/audits/V35-BEHAVIORAL-AUDIT.md  8-bug behavioral root cause audit, CHV re-enable prerequisites (D-0732)
  docs/audits/V35-ARCHITECTURE-RECON.md  AI infrastructure dead code + RAG verdict (D-0733)
  docs/audits/V35-NIGHT-1-HANDOFF.md  Night 1 session narrative + B3/A1/R1 handoff (D-0740)
