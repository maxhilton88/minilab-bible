# Minilab — Session SOP (Standard Operating Procedure)
# Version: 1.0
# Created: 2026-03-26
# Rule: This file is READ-ONLY for agents. Only a human + Opus planning session can modify it.

---

## HOW TO USE THIS FILE

This is the PERMANENT operating manual for every Claude Code session working on Minilab.
- **Line 1-80:** Quick Context Card — read EVERY session (30 seconds)
- **Line 85-150:** Session Startup Protocol — follow these steps every time
- **Line 155-250:** Record Keeping Rules — what to update after every task
- **Line 255-350:** Universal Session Prompt — copy-paste template for humans
- **Line 355-450:** Document Maintenance — which docs are living vs frozen
- **Line 455-550:** Hygiene Rules — preventing doc drift, stale data, merge conflicts

---

## QUICK CONTEXT CARD (read every session — 30 seconds)

**What is Minilab?** AI-powered building management SaaS for Malaysian JMB/MC strata operations.

**Stack:** Next.js 14 App Router · TypeScript strict · Supabase PostgreSQL · Vercel · Tailwind CSS v3 + shadcn/ui

**Key URLs:**
- Live: https://minilab.my
- Database: Supabase project nncbsumdmsyjjrgdjjea (Singapore)
- DNS: Cloudflare (nameservers: ignacio.ns.cloudflare.com, selah.ns.cloudflare.com)
- Email: Cloudflare Email Routing (catch-all *@minilab.my)
- Media: Cloudflare R2 bucket minilab-media (media.minilab.my)
- Email sending: Resend (domain verified)

**Git rule:** Push to main ONLY. One task = one commit + push. Vercel auto-deploys. No branches.

**Three core principles:**
1. Phone number = identity (no passwords, no email login)
2. Buildings are sovereign (all data belongs to JMB/MC, RLS enforced)
3. No app downloads (WhatsApp for residents, Telegram for staff)

**AI routing:** DeepSeek V3 for ROUTINE (85%), Claude Sonnet for SENSITIVE/LEGAL (15%)

**Build status:** V1-V3 complete. V4 Phases 37-41 complete. BM portal ~95% complete (54 pages, 22 sidebar items). Other portals are shells.

**Scale:** 126 database tables, 293 API routes, 215 pages across 17 portals, 37 migrations, 170+ decisions logged.

---

## SESSION STARTUP PROTOCOL (follow every time)

### Step 1: Read context files (2 minutes)

Read in this EXACT order:
1. `CLAUDE.md` — master rulebook (architecture, hard rules)
2. `docs/SESSION-SOP.md` — this file (you're reading it)
3. `docs/decisions.log` — LAST 20 ENTRIES ONLY (scroll to bottom)
4. `docs/actual-schema-columns.md` — ONLY if working on database/API
5. `docs/project-inventory.md` — Quick Context Card (first 50 lines) + jump to relevant section

### Step 2: Identify the task

The human will tell you what to do. If they paste a prompt from the Universal Session Prompt template (see below), follow it. If they describe a task verbally, confirm understanding before starting.

### Step 3: Check current state

Before writing code:
- `git log --oneline -5` — see last 5 commits
- `git status` — ensure clean working tree
- `git branch` — ensure you're on main
- Check if there's a `docs/bm-full-audit-session3.md` or newer audit file for current bugs

### Step 4: Do the work

Follow CLAUDE.md rules. One task = one commit + push.

### Step 5: Update records (CRITICAL — never skip)

After EVERY task, update ALL of the following that apply:

| What changed | Update this file | How |
|-------------|-----------------|-----|
| Architecture decision | `docs/decisions.log` | Append new D-XXXX entry at bottom |
| New database table/column | `docs/actual-schema-columns.md` | Add table/columns in alphabetical order |
| New migration applied | Note in decisions.log | Include migration number |
| New API route | No doc update needed | Code is the source of truth |
| New page | No doc update needed | Code is the source of truth |
| New env var | `docs/env-required.md` | Add with purpose and source |
| Bug fix | Note in commit message | Prefix with "fix:" |
| Feature | Note in commit message | Prefix with "feat:" |
| Session complete | `docs/session-context.md` | Update build status section |

### Step 6: Commit and push

```bash
git add .
git commit -m "[prefix]: brief description"
git push origin main
```

Prefixes: `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`

If push fails (conflict): `git pull --rebase origin main && git push origin main`

---

## RECORD KEEPING RULES

### decisions.log — The Institutional Memory

This is the MOST IMPORTANT file in the project. Every architectural decision lives here.

**Format (never change this):**
```
---
DATE: YYYY-MM-DDTHH:MM:SSZ
DECISION_ID: D-XXXX
CATEGORY: architecture | schema | feature | fix | ui | security | scope | tooling | integration
CONTEXT: What situation required a decision
DECISION: What was chosen and why
DECIDED_BY: human | claude | human + claude
TASK_REF: reference to task or fix number
---
```

**Rules:**
- ALWAYS check the last entry's D-XXXX number before adding a new one
- Increment by 1 (if last is D-0160, next is D-0161)
- NEVER edit or delete existing entries
- NEVER duplicate a D-XXXX number
- If you see merge conflict markers (<<<<<<), fix them immediately

### actual-schema-columns.md — The Schema Source of Truth

**When to update:** Every time you:
- Create a new table (via migration or direct SQL)
- Add/remove/rename a column
- Add a new enum type or value

**Format per table:**
```markdown
### table_name
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | NO | gen_random_uuid() | PK |
| building_id | uuid | NO | | FK → buildings |
| ... | ... | ... | ... | ... |
```

### env-required.md — Environment Variable Registry

**When to update:** Every time you add a new `process.env.SOMETHING` reference in code.

**Format:**
```
VAR_NAME — purpose [required/optional] [source: where to get it]
```

### session-context.md — Session Quick Start

**When to update:** At the END of every session (or every 3-4 tasks if long session).

**What to update:**
- Build status line
- Page counts if changed significantly
- Blocked items list
- Last 5 decisions summary

---

## UNIVERSAL SESSION PROMPT

The human copies this template and fills in [TASK] before pasting into a new Claude Code session.

```
=== MINILAB SESSION START ===

You are working on Minilab — an AI-powered building management SaaS.

## STARTUP (do these steps before writing any code)
1. Read CLAUDE.md
2. Read docs/SESSION-SOP.md (this describes your operating rules)
3. Read docs/decisions.log — LAST 20 ENTRIES ONLY
4. Run: git log --oneline -5 (check recent commits)
5. Run: git status (ensure clean tree, on main branch)

## YOUR TASK
[TASK DESCRIPTION — human fills this in]

## RULES (from CLAUDE.md + SESSION-SOP.md)
- Git: push to main only. One task = one commit + push.
- Schema: NEVER guess column names. Check docs/actual-schema-columns.md or query via Supabase MCP.
- Types: NEVER modify database.types.ts manually. It's auto-generated.
- Decisions: After ANY architectural decision, append to docs/decisions.log with next D-XXXX number.
- Schema changes: After ANY table/column change, update docs/actual-schema-columns.md.
- Env vars: After adding ANY new process.env reference, update docs/env-required.md.
- Supabase MCP is connected — use it to verify schemas and apply migrations.

## AFTER COMPLETING THE TASK
1. Update all relevant docs (see SESSION-SOP.md Record Keeping Rules)
2. Commit: git add . && git commit -m "[prefix]: description" && git push origin main
3. If push fails: git pull --rebase origin main && git push origin main
4. Tell me: what was done, what files changed, any decisions made, any issues found

=== END ===
```

### Task-Specific Prompt Extensions

For **audit/scan tasks**, add:
```
## MODE: READ-ONLY AUDIT
Do NOT fix anything. Report only. Save report to docs/[audit-name].md
```

For **multi-fix tasks**, add:
```
## MODE: FIX BATCH
Fix all items listed below. One commit for all fixes.
After each fix, verify TypeScript compiles: npx tsc --noEmit
```

For **new feature tasks**, add:
```
## MODE: NEW FEATURE
Read the relevant phase in v4-build-requirements.md before starting.
Check docs/actual-schema-columns.md for any tables you'll need.
If a table doesn't exist, create a migration via Supabase MCP.
```

---

## DOCUMENT MAINTENANCE

### Living Documents (always keep current)

| File | Updated by | When |
|------|-----------|------|
| `CLAUDE.md` | Human + Opus planning session | When architecture changes |
| `docs/decisions.log` | Every session | After every decision |
| `docs/actual-schema-columns.md` | Every session that touches DB | After schema changes |
| `docs/env-required.md` | Every session that adds env vars | After new env vars |
| `docs/session-context.md` | End of every session | Session summary |
| `docs/SESSION-SOP.md` | Human + Opus planning session ONLY | Rarely — when process changes |
| `docs/project-inventory.md` | Opus scan every ~5 sessions | When major changes accumulate |

### Frozen Documents (never edit)

| File | Why frozen |
|------|-----------|
| `tasks.md` | V1 complete, historical |
| `v2-build-requirements.md` | V2 complete, historical |
| `packages/shared/types/database.types.ts` | Auto-generated, never edit manually |

### Historical Documents (read-only reference)

All files in docs/ with "audit", "report", "status", "completion" in the name. These are point-in-time snapshots. They provide context but are NOT maintained.

### Bible Sections (rarely updated)

Files in `docs/bible-sections/`. These are extracted from the Project Bible (DOCX). Update only when the Bible is updated.

---

## HYGIENE RULES

### Preventing Doc Drift

The #1 problem: code changes but docs don't. These rules prevent that:

1. **Schema changes → immediate doc update.** If you ALTER TABLE or CREATE TABLE, update actual-schema-columns.md in the SAME commit. Not "later". Not "next task". Same commit.

2. **New env vars → immediate doc update.** If you add `process.env.NEW_VAR` anywhere, add it to env-required.md in the SAME commit.

3. **Architecture decisions → immediate log entry.** If you make a choice between two approaches, log it in decisions.log BEFORE implementing. Not after.

### Preventing Merge Conflicts in decisions.log

decisions.log is append-only. Conflicts happen when two sessions append at the same time. Rules:
1. Always `git pull origin main` before appending
2. Always check the last D-XXXX number AFTER pulling (it may have changed)
3. If you see `<<<<<<<` markers in decisions.log, fix them immediately — remove the markers, keep BOTH entries, ensure D-XXXX numbers are sequential

### Preventing Stale Branches

CLAUDE.md says "no branches". Claude Code creates worktree branches (claude/*). Rules:
1. Always merge to main before ending a session
2. Delete the worktree branch after merging: `git branch -d claude/branch-name`
3. Delete remote branch: `git push origin --delete claude/branch-name`
4. If you find stale branches, clean them up: `git branch -D claude/old-branch`

### Preventing Type Drift

`database.types.ts` must match the live database. Rules:
1. After applying migrations, the HUMAN must regenerate types locally:
   ```powershell
   npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea > packages/shared/types/database.types.ts
   ```
2. Never edit this file manually
3. If you see `as any` casts that shouldn't be needed, the types file may be stale — ask the human to regenerate

### Project Inventory Refresh

Every ~5 sessions (or after major work), run a fresh inventory scan:
1. Human pastes the inventory scan prompt (see docs/project-inventory.md header for instructions)
2. Opus generates updated inventory
3. Compare with previous: note what grew, what's stale, what's new

---

## APPENDIX: Quick Reference

### Supabase MCP Commands
```sql
-- Check if table exists
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'TABLE_NAME' ORDER BY ordinal_position;

-- Check enum values
SELECT enumlabel FROM pg_enum JOIN pg_type ON pg_enum.enumtypid = pg_type.oid WHERE typname = 'ENUM_NAME' ORDER BY enumsortorder;

-- Check latest migration
SELECT * FROM supabase_migrations.schema_migrations ORDER BY version DESC LIMIT 5;
```

### Types Regeneration (human runs this)
```powershell
cd C:\Users\USER\Documents\GitHub\minilab
npx supabase gen types typescript --project-id nncbsumdmsyjjrgdjjea > packages/shared/types/database.types.ts
```

### Environment
- Live: minilab.my (Vercel)
- DNS: Cloudflare
- DB: Supabase (Singapore)
- Email send: Resend
- Email receive: Cloudflare Email Routing
- Media storage: Cloudflare R2 (media.minilab.my) with Supabase Storage fallback
- Face recognition: Amazon Rekognition (provider-agnostic interface)
- AI routing: DeepSeek (ROUTINE) / Claude Sonnet (SENSITIVE/LEGAL)
