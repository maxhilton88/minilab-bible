# CLAUDE.md — Minilab Platform
<!-- Version: V25 · 2026-04-11 -->
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
| Telegram bot / groups / Telethon | recent-decisions.md | §Telegram |
| Committee portal | recent-decisions.md | §Committee |
| Procurement / hardware store | recent-decisions.md | §Procurement |
| Collection / chase engine | recent-decisions.md | §Collections |
| MOA / office agent / Advelsoft | recent-decisions.md | §MOA |
| Resident portal / WhatsApp OTP | recent-decisions.md | §Resident |
| Supplier / store portal | recent-decisions.md | §Supplier-Store |

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

Face: face-api.js + pgvector, 128-dim descriptors, cosine similarity threshold 0.5, match_face() RPC. No manual clock-in ever. Face IS identity — no name dropdown.

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
7. GRAPH: If major architectural change → update docs/startup/GRAPH_REPORT.md.
8. LIST files changed.
9. COMMIT with prefix (feat: | fix: | docs: | refactor: | chore:) and push to main.
10. SYNC BIBLE: cd docs/startup && git add -A && git commit -m "bible: sync V{XX}" && git push origin main && cd ../..

---

## §7 · Current State

Version: V25 (2026-04-11)
Database: ~165 tables · migrations 001-058+
Pages: ~287 · API routes: ~390 · Decisions: D-0421+ · Portals: 17+

Active priorities:
1. Bug fixes and cleanup pass (pre-MOA)
2. CHV onboarding (Hoe Zee How, 700+ units) — create security+cleaning orgs, assign to CHV, set geofence
3. E2E testing with Seeteng at Lumi Residency (497c38b5-271d-4316-9580-93096d70038e)
4. MOA Phase 2 — command queue, Electron/Python agent, Advelsoft RDP via PyAutoGUI/OpenCV

Pending fixes:
- Guard VMS walk-in: collapse to single-step, white Lucide icons, remove name field
- DISABLE_DEV_LOGIN=true for production
- Superadmin PDPA deletion
- BM staff page dropdown transparency bug

Key contacts:
- Seeteng — BM, Lumi Residency (test user)
- Hoe Zee How — BM, Cyber Heights Villa (next onboarding)

---

## §8 · Doc Inventory

docs/startup/ — Working docs (lazy-loaded via §4 Router)
  actual-schema-columns.md    126KB  Anchored by §Table-{name}
  recent-decisions.md         187KB  Anchored by §Feature-Area
  env-required.md               7KB  Small — OK to read fully
  GRAPH_REPORT.md             ~10KB  Anchored by §Community / §God-Nodes

docs/ — Archive (touch only when Router points here)
  decisions.log               429KB  Raw D-0001+ (not anchored)
  session-context.md           35KB  Legacy (superseded by §7)
  SESSION-SOP.md               13KB  Legacy (superseded by §2+§6)
  minilab-ai-agent-architecture.md  34KB  AI deep-dive (for MOA)
  capabilities-map.md          22KB  Feature overview
  docs/audit/                 ~800KB  V3-V14 audit reports (moved here)
