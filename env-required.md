# Minilab — Required Environment Variables

> **How to use:** All these env vars must exist in Vercel environment settings
> (Production + Preview + Development) before any code that uses them will work.
> **NEVER put actual values in this file.** This file is committed to the repo.
> Actual values live in `.env.local` (gitignored) and Vercel dashboard only.

**Total:** 46 unique `process.env` references found in codebase (as of 2026-03-30). Some listed below are documented but not yet referenced in code.

## Database & Infrastructure
| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL — safe for frontend |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon key — safe for frontend |
| `SUPABASE_SERVICE_ROLE_KEY` | Full DB access — SERVER SIDE ONLY, bypasses RLS |
| `DATABASE_URL` | Direct Postgres connection string — for Supabase CLI and migrations |
| `UPSTASH_REDIS_REST_URL` | Redis for rate limiting + conversation state machine |
| `UPSTASH_REDIS_REST_TOKEN` | Redis auth token |

## AI Models
| Variable | Purpose |
|---|---|
| `DEEPSEEK_API_KEY` | DeepSeek V3 — ROUTINE messages (85%), Router classification |
| `ANTHROPIC_API_KEY` | Claude Sonnet 4 — SENSITIVE/LEGAL (15%), ReAct queries. Model: `claude-sonnet-4-20250514` |
| `DASHSCOPE_API_KEY` | Alibaba — Qwen2.5-VL image captioning + text-embedding-v3 |
| `GEMINI_API_KEY` | Gemini Live (voice sales agent) + Gemini Flash-8B (AI vision pre-processor D-0641) |
| `GOOGLE_VISION_KEY` | Google Vision API — OCR for document pipeline |
| `OPENAI_EMBEDDING_KEY` | OpenAI text embeddings — RAG document chunking |


## WhatsApp / Messaging
| Variable | Purpose |
|---|---|
| `META_WABA_ID` | WhatsApp Business Account ID |
| `META_PHONE_NUMBER_ID` | Phone number ID within the WABA |
| `META_ACCESS_TOKEN` | Permanent system user token — never expires |
| `META_WEBHOOK_VERIFY_TOKEN` | Custom string to verify Meta webhook (you set this) |
| `META_APP_SECRET` | HMAC validation of inbound webhook payloads |
| `META_BUSINESS_ID` | Meta Business Manager ID |
| `META_WA_DISPLAY_NUMBER` | WhatsApp display phone number (also used for wa.me deep links in Google OAuth verify-phone) |
| `MINILAB_WA_PLATFORM_FALLBACK` | Gate platform credential fallback for multi-WABA (default `'true'`; set `'false'` in prod to prevent accidental sends from platform number) |
| `TELEGRAM_PLATFORM_BOT_TOKEN` | Platform-level Telegram bot (per-building tokens in DB) |
| `TELEGRAM_BOT_USERNAME` | Telegram bot @username — used in auth login flows |
| `TELEGRAM_WEBHOOK_SECRET` | Telegram webhook signature verification |
| `TELEGRAM_GROUP_SERVICE_URL` | URL of the Telethon microservice on GCP (e.g. http://34.xxx.xxx.xxx:8443) |
| `TELEGRAM_GROUP_SERVICE_SECRET` | API secret for the Telethon group creation microservice |
| `EMAIL_WEBHOOK_SECRET` | Cloudflare email webhook verification |

## Face Verification
| Variable | Purpose |
|---|---|
| `FACE_PROVIDER` | Switches face recognition provider. Values: `rekognition` (default) or `local` (face-api.js + pgvector). Safe to omit — defaults to Rekognition. |
| `AWS_REKOGNITION_ACCESS_KEY_ID` | Amazon Rekognition access key (ap-southeast-1) |
| `AWS_REKOGNITION_SECRET_ACCESS_KEY` | Amazon Rekognition secret key |
| `AWS_REKOGNITION_REGION` | Rekognition region — `ap-southeast-1` |

## Email & Payments
| Variable | Purpose |
|---|---|
| `RESEND_API_KEY` | Transactional email — Form 11, invoices, notifications |
| `STRIPE_SECRET_KEY` | Vendor subscription billing — server side only |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Stripe.js frontend key — safe for frontend |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signature verification (whsec_...) |
| `OPENCLAW_API_KEY` | OpenClaw accounting integration API key |
| `MINILAB_BILLING_EMAIL` | Platform billing contact email |

## Web Push & App Config
| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_VAPID_PUBLIC_KEY` | Web Push public key — needs NEXT_PUBLIC_ prefix for frontend. Generate ONCE with npx web-push generate-vapid-keys |
| `VAPID_PRIVATE_KEY` | Web Push private key — server side only |
| `VAPID_SUBJECT` | Push sender identity — set to mailto:tech@minilab.my |
| `DISABLE_DEV_LOGIN` | Set to `true` to disable the /dev/login page in production. Safe to omit — defaults to enabled. |
| `PREVIEW_BYPASS_SECRET` | Local dev only — allows Claude Code Preview tool to bypass auth via `/api/dev/preview-session`. NEVER set in Vercel. |
| `CRON_SECRET` | Protects cron endpoints from unauthorised triggers |
| `SUPERADMIN_PHONE` | Your personal Malaysian mobile — only number that accesses /superadmin |
| `SUPERADMIN_TOTP_SECRET` | Base32 TOTP secret for superadmin 2FA — generate with any TOTP library |
| `MASTERAI_CHAT_ID` | Your personal Telegram chat ID — for build HITL messages |
| `INTERNAL_API_KEY` | Internal API route authentication |
| `SENTRY_DSN` | Sentry DSN for server-side error tracking (optional — same DSN value as NEXT_PUBLIC_SENTRY_DSN) |
| `NEXT_PUBLIC_SENTRY_DSN` | Sentry DSN for client-side error tracking (optional — errors only captured when set) |
| `SENTRY_AUTH_TOKEN` | Sentry auth token for source map uploads (optional — only needed for readable stack traces in Sentry UI) |
| `MINILAB_URL` | Platform base URL for internal references |

## Google Auth (NextAuth.js)
| Variable | Purpose |
|---|---|
| `GOOGLE_CLIENT_ID` | Google OAuth client ID — from Google Cloud Console |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret — server side only |
| `NEXTAUTH_SECRET` | NextAuth.js session encryption secret — generate with `openssl rand -base64 32` |
| `NEXTAUTH_URL` | NextAuth.js base URL — set to `https://minilab.my` in production |

## Cloudflare R2 Storage
| Variable | Purpose |
|---|---|
| `R2_ACCOUNT_ID` | Cloudflare R2 account ID |
| `R2_ACCESS_KEY_ID` | R2 API access key |
| `R2_SECRET_ACCESS_KEY` | R2 API secret key |
| `R2_BUCKET_NAME` | R2 bucket name — `minilab-media` |
| `R2_PUBLIC_URL` | R2 public URL — `https://media.minilab.my` |

## App URLs (set after domain is configured)
| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_APP_URL` | https://minilab.my |
| `NEXT_PUBLIC_GUARD_URL` | https://guard.minilab.my |
| `NEXT_PUBLIC_CONTRACTOR_URL` | https://contractor.minilab.my |
| `NEXT_PUBLIC_RESIDENT_URL` | https://resident.minilab.my |

*Source: Minilab Project Bible v21.0, Section 22 + Credentials Sheet*
