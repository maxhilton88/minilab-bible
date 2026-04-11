# CHV (Cyber Heights Villa) — WhatsApp Go-Live Checklist

## Building Record
- **Building ID:** `d98e6bdc-8dfa-4ec6-89af-ed9272e25beb`
- **Name:** CYBER HEIGHTS VILLA (CHV)
- **Current Status:** `not_connected` (all WA columns NULL)

---

## Step 1: Meta Business Manager Setup

### 1a. Create WABA (WhatsApp Business Account)
- Go to [Meta Business Manager](https://business.facebook.com/) → WhatsApp Accounts
- Create a new WABA under Minilab's Meta Business account
- Note the **WABA ID** (format: `123456789012345`)

### 1b. Add Phone Number
- Add CHV's dedicated phone number to the WABA
- Complete phone number verification (SMS or voice call)
- Note the **Phone Number ID** (format: `123456789012345`)
- The phone number should be in format `+60XXXXXXXXX` (Malaysian)

### 1c. Generate Access Token
- Create a System User under the Meta Business account (or reuse existing)
- Generate a **permanent access token** with `whatsapp_business_messaging` permission
- Store securely — this goes in the buildings table

---

## Step 2: Configure Webhook

### 2a. Webhook URL
Set in Meta Business Manager → WhatsApp → Configuration:
```
URL:     https://minilab.my/api/webhooks/whatsapp
Token:   (same as META_WEBHOOK_VERIFY_TOKEN env var)
```

### 2b. Subscribe to Fields
- `messages` (required)
- `message_deliveries` (optional, for delivery status)
- `message_reads` (optional, for read receipts)

### 2c. Verify Webhook
- Meta sends GET with `hub.verify_token` → our endpoint returns the challenge
- Single webhook URL handles ALL buildings — `resolveSender()` routes by `phone_number_id`

---

## Step 3: Submit Template for Approval

### 3a. Create `minilab_notification` template
In Meta Business Manager → WhatsApp → Message Templates:

```
Template Name:  minilab_notification
Category:       UTILITY
Language:       en (English)
Body:           {{1}}
```
- `{{1}}` is a single text parameter (max 1024 chars)
- This template is used as 24h window fallback for freeform messages

### 3b. Wait for Approval
- Meta reviews templates within 24 hours (usually faster)
- Status must be `APPROVED` before go-live
- If using a custom template name, update `fallback_template_name` in Step 4

### 3c. Optional: Create building-specific templates
If CHV wants custom templates (payment reminders, welcome messages, etc.):
- Create them in Meta Business Manager
- Submit via `/api/bm/whatsapp/setup` or directly in Meta

---

## Step 4: Insert Credentials into Database

Run the following SQL (replace placeholders with actual values):

```sql
UPDATE buildings SET
  whatsapp_phone_number_id = '<PHONE_NUMBER_ID>',
  whatsapp_access_token    = '<PERMANENT_ACCESS_TOKEN>',
  whatsapp_waba_id         = '<WABA_ID>',
  whatsapp_phone_number    = '+60XXXXXXXXX',
  whatsapp_display_name    = 'Cyber Heights Villa',
  whatsapp_status          = 'active',
  whatsapp_verified_at     = NOW(),
  fallback_template_name   = 'minilab_notification'
WHERE id = 'd98e6bdc-8dfa-4ec6-89af-ed9272e25beb';
```

**Column reference:**
| Column | Type | Example Value |
|--------|------|---------------|
| `whatsapp_phone_number_id` | TEXT | `'123456789012345'` |
| `whatsapp_access_token` | TEXT | `'EAAx...'` (long token) |
| `whatsapp_waba_id` | TEXT | `'123456789012345'` |
| `whatsapp_phone_number` | TEXT | `'+60123456789'` |
| `whatsapp_display_name` | TEXT | `'Cyber Heights Villa'` |
| `whatsapp_status` | TEXT | `'active'` |
| `whatsapp_verified_at` | TIMESTAMPTZ | `NOW()` |
| `fallback_template_name` | TEXT | `'minilab_notification'` (default) |

---

## Step 5: Verification Tests

### 5a. Inbound Test
1. Send a WhatsApp message from a test phone to CHV's number
2. Verify webhook receives the message (check Vercel function logs)
3. Confirm `resolveSender()` maps the `phone_number_id` → CHV building
4. Confirm message appears in CHV's console (`/bm/console`)

### 5b. Outbound Test
1. From CHV's console, send a reply to the test contact
2. Verify `getBuildingWhatsAppConfig()` returns CHV credentials (not platform)
3. Confirm message arrives on the test phone from CHV's number

### 5c. 24h Window Fallback Test
1. Wait >24h since last inbound from test phone (or use a fresh number)
2. Send a message from console — should fall back to `minilab_notification` template
3. Verify template name used matches `fallback_template_name` column

### 5d. OTP Unaffected
1. Go to resident login page, enter a phone number
2. Confirm the wa.me link points to `META_WA_DISPLAY_NUMBER` (platform number)
3. NOT CHV's building-specific number

### 5e. Nudge Button Test
1. Open a case with a linked resident in `/bm/cases`
2. Click "Nudge" button — should open wa.me with pre-filled message
3. Message should contain callback link to CHV's API number
4. Resident taps callback link → sends `CASE-{uuid}` → webhook links to case

### 5f. Broadcast Test
1. Create an announcement with `notify_whatsapp: true`
2. Verify `whatsapp_status === 'active'` check passes (Gap 4 fix)
3. Confirm messages sent from CHV's number to residents

---

## Step 6: Platform Fallback Gate (Production)

Once CHV is live with its own WABA, consider setting:
```
MINILAB_WA_PLATFORM_FALLBACK=false
```
in Vercel environment variables. This prevents any building without its own WA credentials from accidentally sending messages through the platform number. Default is `true` (fallback allowed for dev/test).

---

## Architecture Trace Summary

### Inbound Path
```
Meta webhook POST → /api/webhooks/whatsapp
  → HMAC verify (META_APP_SECRET)
  → parseWebhookMessages()
  → resolveSender(phone, phoneNumberId)
      → buildings WHERE whatsapp_phone_number_id = phoneNumberId
      → Returns CHV building ID ✓
  → upsert sender_profiles
  → store message
  → V4 AI pipeline (or CASE- nudge handler)
```

### Outbound Path
```
Console reply / notification / broadcast
  → sendTextMessage() or sendWhatsAppDirect()
      → getBuildingWhatsAppConfig(buildingId)
          → buildings WHERE id = CHV building ID
          → Returns CHV credentials (not platform) ✓
          → Checks whatsapp_status === 'active' ✓
      → callWhatsAppAPI(CHV phoneNumberId, CHV accessToken, ...)
  → Message sent from CHV's number ✓
```

### OTP Path (Unaffected)
```
Resident login → POST /api/auth/whatsapp/request-otp
  → Uses META_WA_DISPLAY_NUMBER (platform env var) ✓
  → No building context at login time ✓
  → CHV credentials never touched ✓
```

---

## Status: READY FOR CREDENTIALS
All code paths are verified. Insert credentials (Step 4) and run tests (Step 5).
