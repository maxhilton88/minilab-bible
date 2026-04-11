# Cron External Setup (cron-job.org)

**Decision:** D-0518 | Migrated from Vercel Hobby (1 daily cron) to cron-job.org (unlimited schedules).

## Auth

All jobs use the same header:
- Header: `Authorization`
- Value: `Bearer <CRON_SECRET>` (same env var already in Vercel + .env.local)

## Jobs to configure

| Job name | URL | Schedule (UTC) | Method | Failure alert |
|----------|-----|----------------|--------|---------------|
| every-5min | https://minilab.my/api/cron/every-5min | `*/5 * * * *` | GET | Yes |
| every-15min | https://minilab.my/api/cron/every-15min | `*/15 * * * *` | GET | Yes |
| every-30min | https://minilab.my/api/cron/every-30min | `*/30 * * * *` | GET | No |
| morning-ops | https://minilab.my/api/cron/twice-daily | `0 23 * * *` | GET | Yes |
| evening-ops | https://minilab.my/api/cron/twice-daily | `0 10 * * *` | GET | Yes |
| hourly | https://minilab.my/api/cron/hourly | `0 * * * *` | GET | No |
| daily | https://minilab.my/api/cron/daily | `0 1 * * *` | GET | Yes |
| nightly | https://minilab.my/api/cron/nightly | `0 16 * * *` | GET | Yes |
| monthly | https://minilab.my/api/cron/monthly | `0 16 1 * *` | GET | Yes |
| weekly | https://minilab.my/api/cron/weekly | `0 16 * * 0` | GET | No |

**UTC → MYT reference:** UTC+8. `0 23 * * *` = 7AM MYT. `0 10 * * *` = 6PM MYT. `0 16 * * *` = midnight MYT.

## What each job runs

| Job | Tasks | Mode |
|-----|-------|------|
| every-5min | stuck-messages, push-send, moa-cleanup | parallel |
| every-15min | attendance-sync | parallel |
| every-30min | building-state | parallel |
| twice-daily (×2) | daily-ops (morning briefing OR EOD based on MYT hour) | sequential |
| hourly | reminders (24h + 1h bookings, maintenance, parcels) | parallel |
| daily | compliance, inventory-check, document-expiry, credit-reminders, onboarding-chase, gap-engine, billing, org-dunning, trial-expiry, collection-engine, telethon-keepalive, **pdpa-deletion**, **visitor-cleanup** | sequential |
| nightly | snapshot, profile-builder, memory-compiler, nightly-validator, cleanup, **auto-clockout** | sequential |
| monthly | credit-reset, store-billing, **leave-reset** (Jan only, self-gated) | sequential |
| weekly | **permit-expiry**, **pdpa-archival** | sequential |

## Manual trigger (testing)

Each endpoint accepts `?job=<task-name>` to run a single task:
```
https://minilab.my/api/cron/daily?job=compliance
https://minilab.my/api/cron/every-5min?job=stuck-messages
https://minilab.my/api/cron/nightly?job=cleanup
```

## Response format

All routes return:
```json
{
  "results": [
    { "task": "compliance", "status": "ok", "ms": 234 },
    { "task": "billing",    "status": "error", "ms": 12, "error": "DB timeout" }
  ]
}
```

## Timeout warning

Vercel Hobby serverless functions have a **10s execution limit**. Heavy routes (daily, nightly)
may timeout if many buildings are active. Monitor via cron-job.org failure alerts.

If timeouts are frequent, options:
1. Upgrade to Vercel Pro (60s timeout)
2. Break heavy tasks into per-building queue calls

## Idempotency notes

All tasks are idempotent — safe to re-run or double-trigger:
- `billing` / `store-billing` — existing-invoice check + `isFirstOfMonth` guard
- `credit-reset` — monthly RPC handles idempotency
- `stuck-messages` / `moa-cleanup` — status-based, safe to re-run
- `push-send` — deletes queue entry after send
- `snapshot` — upsert on conflict
- `pdpa-deletion` — status check prevents re-processing completed/failed requests
- `visitor-cleanup` — status='active' filter prevents double-update
- `auto-clockout` — IS NULL clock_out_at filter is idempotent
- `permit-expiry` — alert-only, no state written; re-running just re-alerts
- `pdpa-archival` — hard delete; safe to re-run (rows already gone)
- `leave-reset` — year check prevents duplicate balance rows
