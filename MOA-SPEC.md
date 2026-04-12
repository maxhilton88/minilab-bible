# MOA-SPEC.md — Minilab Office Agent Build Specification

**Created:** 2026-04-12 (V28 brainstorm session)
**Status:** Architecture locked. Ready for build.

---

## §1 · What Is MOA

MOA (Minilab Office Agent) is Minilab's universal legacy software bridge — a headless browser automation platform that connects Minilab's cloud ERP to any third-party property management software without vendor cooperation or API access.

**The problem MOA solves:** Malaysian strata management relies on dozens of legacy systems (accounting, access cards, car plate recognition, barrier gates, intercom, CCTV) from vendors who won't provide APIs. Every condo uses a different combination. Minilab needs to read from and write to all of them.

**MOA's approach:** If it has a screen, MOA can control it. Browser-based systems get Playwright automation. Desktop-only systems get RDP + keyboard automation. No vendor cooperation required. No APIs begged for.

**MOA does NOT run on the condo's PC.** It runs centrally on Minilab's infrastructure as Playwright-controlled headless Chrome instances that log into vendor portals and drive them via keyboard commands.

---

## §2 · Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  Minilab Cloud (Supabase)                                        │
│                                                                  │
│  moa_commands          moa_skills (JSON)     moa_supported_systems│
│  ┌──────────────┐     ┌────────────────┐     ┌─────────────────┐ │
│  │ building_id   │     │ system: string  │     │ category        │ │
│  │ system_id     │     │ skill steps[]   │     │ name            │ │
│  │ skill_name    │     │ input_schema    │     │ vendor          │ │
│  │ input_data    │     │ output_schema   │     │ connection_type │ │
│  │ status        │     │ version         │     │ (cloud_web /    │ │
│  └──────┬───────┘     └───────┬────────┘     │  cloud_rdp /    │ │
│         │                      │              │  local_desktop) │ │
│         │ Realtime             │ Fetch        └─────────────────┘ │
│         ▼                      ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  MOA Agent (VPS — Node.js + Playwright)                    │  │
│  │                                                            │  │
│  │  System Connectors (parallel browser contexts):            │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │  │
│  │  │ Advelsoft     │ │ Access Card  │ │ Car Plate    │       │  │
│  │  │ cloud04...    │ │ vendor-x.com │ │ vendor-y.com │       │  │
│  │  │ HTML5 RDP     │ │ Web portal   │ │ Web portal   │       │  │
│  │  │ canvas keys   │ │ DOM interact │ │ DOM interact │       │  │
│  │  └──────────────┘ └──────────────┘ └──────────────┘       │  │
│  │                                                            │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │  │
│  │  │ CSS System   │ │ Condo Master │ │ Future...    │       │  │
│  │  │ vendor-z.com │ │ cm-cloud.com │ │              │       │  │
│  │  └──────────────┘ └──────────────┘ └──────────────┘       │  │
│  │                                                            │  │
│  │  Each connector = 1 Playwright BrowserContext               │  │
│  │  Isolated cookies, sessions, state per system               │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## §3 · System Connection Types

Every legacy system falls into one of three categories. MOA handles all three.

### Type 1: Cloud Web Portal (direct DOM)
**Examples:** Most access card systems, car plate systems, newer property management software, CCTV cloud viewers.

The vendor has a web dashboard. Playwright navigates it like a normal website — click buttons, fill forms, read DOM elements, intercept API calls.

```
Playwright → loads vendor portal → interacts with DOM elements
```

**Advantages:** Fastest, most reliable. DOM selectors are stable. Can read structured data directly. Can intercept the vendor's own API calls via network monitoring.

### Type 2: Cloud-Hosted RDP (canvas + keyboard)
**Examples:** Advelsoft, some older accounting systems hosted on cloud Windows servers.

Vendor provides a web login that drops into an HTML5 Remote Desktop session. The actual Windows app renders as a canvas/video stream. Keyboard events sent to the canvas are forwarded to the remote app.

```
Playwright → loads vendor RDP portal → sends keyboard events to canvas → remote Windows app receives them
```

**Advantages:** Keyboard shortcuts are universal across resolutions/window sizes. No pixel coordinates needed.

### Type 3: Local Desktop Only (requires condo PC agent)
**Examples:** Very old systems with no cloud option, hardware-bundled software that only runs on a specific machine.

This is the ONLY case where a local agent on the condo's PC is needed. Python + PyAutoGUI or a lightweight Electron wrapper.

```
Local agent on condo PC → controls desktop app via PyAutoGUI → reports to Minilab via API
```

**Note:** For CHV, ALL three systems are cloud-accessible. Type 3 is documented for future buildings that may have local-only systems.

---

## §4 · Proven Capabilities (tested 2026-04-12 on Advelsoft)

These tests validate the Type 2 (Cloud RDP) approach. Type 1 (Cloud Web) is inherently simpler.

| Capability | Status | Notes |
|------------|--------|-------|
| HTML5 RDP loads in headless browser | ✅ | `cloud04.advelsoft.com.my/software/html5.html` |
| Web portal login (DOM) | ✅ | Standard form: server, username, password fields |
| App login inside RDP (keyboard) | ✅ | VB6 dialog: Tab between fields, Enter to submit |
| Alt+key menu access | ✅ | Alt+R opens Report menu in Advelsoft |
| Arrow key navigation | ✅ | Up/Down moves through menu items |
| Enter to select | ✅ | Activates highlighted menu item |
| Escape to close | ✅ | Closes dialogs and menus |
| Tab between form fields | ✅ | Moves focus in dialogs |
| Universal Printer → browser download | ✅ | Print triggers PDF, HTML5 viewer pushes as Chrome download |
| Clipboard pass-through (Ctrl+C/V) | ✅ | Bidirectional — copy in remote app, read from Playwright |
| Multi-building (phase selector) | ✅ | Same login, different building contexts |

**Key finding:** If these work for Type 2 (the hardest case), Type 1 (direct DOM) is trivially easier.

---

## §5 · Data Extraction Methods

Three proven methods, applicable across all system types:

| Method | How | When to use | System types |
|--------|-----|-------------|-------------|
| **Browser download** | Print/export triggers download, Playwright intercepts | Full reports, PDFs, bulk data | Type 1, Type 2 |
| **Clipboard** | Ctrl+C in vendor app, read via Playwright clipboard API | Quick single-value lookups | Type 2 (RDP apps) |
| **DOM scraping** | Read text/values directly from HTML elements | Structured data from web portals | Type 1 only |
| **Network intercept** | Monitor vendor's own API calls via Playwright | Structured JSON data without UI interaction | Type 1 only |

### Browser Download (Universal Printer pattern — Type 2)
```typescript
const [download] = await Promise.all([
  page.waitForEvent('download'),
  canvas.press('Enter')  // triggers print
]);
const buffer = await download.saveAs('/tmp/report.pdf');
const r2Url = await uploadToR2(buffer, 'moa/reports/...');
```

### Clipboard Extraction (Type 2)
```typescript
await canvas.press('Control+a');
await canvas.press('Control+c');
const text = await page.evaluate(() => navigator.clipboard.readText());
```

### DOM Scraping (Type 1)
```typescript
const balance = await page.locator('.balance-amount').textContent();
const rows = await page.locator('table.records tr').allTextContents();
```

### Network Intercept (Type 1 — best case)
```typescript
const response = await page.waitForResponse(resp => 
  resp.url().includes('/api/residents') && resp.status() === 200
);
const data = await response.json();
```

---

## §6 · Skill JSON Schema

Skills are stored in the `moa_skills` table as JSONB. The agent downloads the skill definition at execution time — **zero deployment for new skills or updates.**

Skills are **system-specific but building-universal** — one skill works across every building that uses that system.

### Example: Type 2 Skill (Advelsoft RDP)

```jsonc
{
  "name": "advelsoft_export_statement",
  "system": "advelsoft_pm",
  "version": "1.0",
  "connection_type": "cloud_rdp",
  "description": "Export billing statement as PDF for a specific unit",
  "category": "accounting",
  "input_schema": {
    "unit_number": "string",
    "period": "string"
  },
  "output_schema": {
    "pdf_url": "string",
    "total_due": "number",
    "status": "string"
  },
  "steps": [
    { "id": 1, "action": "key", "description": "Open Report menu", "keys": ["Alt+r"], "wait_after_ms": 500 },
    { "id": 2, "action": "key", "description": "Navigate to Statement", "keys": ["Down", "Right"], "wait_after_ms": 300 },
    { "id": 3, "action": "key", "description": "Select option", "keys": ["Enter"], "wait_after_ms": 1500 },
    { "id": 4, "action": "type", "description": "Enter unit number", "text": "{{input.unit_number}}", "wait_after_ms": 500 },
    { "id": 5, "action": "key", "description": "Confirm and print", "keys": ["Enter"], "wait_after_ms": 2000 },
    { "id": 6, "action": "download", "description": "Catch PDF", "upload_to": "r2", "r2_path": "moa/statements/{{building.id}}/{{input.unit_number}}/{{input.period}}.pdf", "timeout_ms": 10000 }
  ],
  "error_recovery": { "unexpected_dialog": "screenshot_and_escape", "timeout_ms": 30000, "max_retries": 2 }
}
```

### Example: Type 1 Skill (Web Portal)

```jsonc
{
  "name": "accesscard_get_logs",
  "system": "vendor_x_access_card",
  "version": "1.0",
  "connection_type": "cloud_web",
  "description": "Fetch access card swipe logs for today",
  "category": "access_card",
  "input_schema": {
    "date": "string"
  },
  "output_schema": {
    "logs": "array",
    "status": "string"
  },
  "steps": [
    { "id": 1, "action": "click", "description": "Open logs page", "selector": "a[href='/logs']", "wait_after_ms": 1000 },
    { "id": 2, "action": "click", "description": "Open date picker", "selector": "#date-filter", "wait_after_ms": 300 },
    { "id": 3, "action": "type", "description": "Set date", "selector": "#date-filter input", "text": "{{input.date}}", "clear_first": true, "wait_after_ms": 500 },
    { "id": 4, "action": "click", "description": "Apply filter", "selector": "button.apply-filter", "wait_after_ms": 2000 },
    { "id": 5, "action": "read_dom", "description": "Read log table", "selector": "table.log-entries", "parse_as": "table", "output_key": "logs" }
  ]
}
```

### Action Types Reference

| Action | Description | System types | Key parameters |
|--------|-------------|-------------|------------|
| `key` | Send keyboard shortcut(s) | All | `keys[]` |
| `type` | Type text into field | All | `text`, `selector` (Type 1), `clear_first` |
| `click` | Click DOM element | Type 1 | `selector` |
| `wait` | Pause | All | `ms` |
| `download` | Catch browser download → R2 | All | `upload_to`, `r2_path`, `timeout_ms` |
| `screenshot` | Capture screen | All | `save_as` |
| `verify_screen` | Vision LLM check | Type 2 | `expect` |
| `read_dom` | Read DOM text/table | Type 1 | `selector`, `parse_as`, `output_key` |
| `read_clipboard` | Read clipboard | Type 2 | `parse_as`, `output_key` |
| `intercept_response` | Wait for API response | Type 1 | `url_pattern`, `parse_as`, `output_key` |
| `conditional` | Branch logic | All | `check`, `if_true[]`, `if_false[]` |

### Variable Interpolation

All string fields support:
- `{{input.xxx}}` — from command's input_data
- `{{output.xxx}}` — from a previous step's output_key
- `{{date.today}}` — YYYY-MM-DD
- `{{date.month}}` — YYYY-MM
- `{{building.id}}` — Minilab building UUID
- `{{building.name}}` — building name
- `{{system.config.xxx}}` — from the system's connection config

---

## §7 · Agent Architecture (Node.js + Playwright)

### File Structure

```
moa-agent/
├── package.json
├── tsconfig.json
├── .env                              # SUPABASE_URL, SUPABASE_SERVICE_KEY, R2 creds
├── src/
│   ├── agent.ts                      # Main loop — startup, subscribe, heartbeat, dispatch
│   ├── connector-manager.ts          # Manages multiple system connections per building
│   │
│   ├── connectors/
│   │   ├── base-connector.ts         # Abstract: login(), isAlive(), reconnect()
│   │   ├── cloud-web-connector.ts    # Type 1: direct DOM interaction
│   │   ├── cloud-rdp-connector.ts    # Type 2: HTML5 canvas + keyboard
│   │   └── local-desktop-connector.ts # Type 3: future
│   │
│   ├── systems/                      # System-specific login sequences
│   │   ├── advelsoft/
│   │   │   ├── login.ts              # Web → RDP → app login → phase select
│   │   │   └── helpers.ts            # Advelsoft-specific navigation
│   │   ├── access-card-vendor/
│   │   │   └── login.ts
│   │   ├── car-plate-vendor/
│   │   │   └── login.ts
│   │   └── registry.ts              # system_id → connector type + login module
│   │
│   ├── executor/
│   │   ├── skill-executor.ts         # Parse JSON skill, dispatch actions
│   │   └── actions/
│   │       ├── key.ts
│   │       ├── type.ts
│   │       ├── click.ts              # Type 1 only
│   │       ├── download.ts           # Catch browser download → R2
│   │       ├── screenshot.ts
│   │       ├── verify-screen.ts      # Vision LLM (Type 2)
│   │       ├── read-dom.ts           # Type 1 only
│   │       ├── read-clipboard.ts     # Type 2
│   │       ├── intercept-response.ts # Type 1
│   │       └── conditional.ts
│   │
│   ├── recovery/
│   │   ├── error-handler.ts
│   │   └── vision-fallback.ts
│   │
│   └── lib/
│       ├── supabase.ts
│       ├── r2.ts
│       ├── heartbeat.ts
│       ├── credentials.ts            # Decrypt system credentials
│       └── logger.ts
```

### Main Loop

```
1. Read building→system assignments from moa_agents
2. For each assigned system:
   a. Spawn Playwright BrowserContext
   b. Load system connector from registry
   c. Login
   d. Heartbeat (status: online, system_id: xxx)
3. Subscribe to moa_commands (filter: building_id + status=pending)
4. On new command:
   a. Route to correct connector via command.system_id
   b. Mark "processing"
   c. Fetch skill JSON from moa_skills
   d. Execute steps
   e. Report result
5. Heartbeat every 60s per system
6. Session keepalive — detect disconnect, auto-reconnect
7. Graceful shutdown on SIGTERM
```

---

## §8 · System Configuration (Per Building)

```jsonc
// moa_agents.config for a building with 3 systems
{
  "systems": [
    {
      "system_id": "advelsoft_pm",
      "connection_type": "cloud_rdp",
      "enabled": true,
      "credentials": {
        "portal_url": "cloud04.advelsoft.com.my",
        "web_username": "______",
        "web_password": "encrypted:______",
        "app_username": "priya",
        "app_password": "encrypted:______",
        "phase_number": "01"
      }
    },
    {
      "system_id": "vendor_x_access_card",
      "connection_type": "cloud_web",
      "enabled": true,
      "credentials": {
        "portal_url": "https://access-vendor.com/portal",
        "username": "chv_admin",
        "password": "encrypted:______"
      }
    },
    {
      "system_id": "vendor_y_car_plate",
      "connection_type": "cloud_web",
      "enabled": true,
      "credentials": {
        "portal_url": "https://carplate-vendor.com/dashboard",
        "username": "chv_admin",
        "password": "encrypted:______"
      }
    }
  ]
}
```

### Onboarding a New Building (Same Vendors)
1. Collect credentials → insert moa_agents row → done. Same skills work.

### Onboarding a New Vendor
1. Max records workflows → Claude compiles skill JSONs
2. Create login module in systems/{vendor}/
3. Insert skills in moa_skills
4. All buildings using that vendor get it instantly

---

## §9 · Advelsoft Deep-Dive (First System — Type 2)

### Connection: Cloud-Hosted RDP
- **URL:** `cloud04.advelsoft.com.my/software/html5.html`
- **RDP viewer:** HTML5 canvas-based
- **App:** Advelsoft Property Management V7.6M
- **User:** ch01 (RDP) → priya (app)
- **Multi-building:** Phase selector (01=CHV, 02=Sawtelle, 03=Lakeview)

### Login Sequence
```
1. Browser → cloud04.advelsoft.com.my (web form — DOM)
2. RDP session loads → Windows desktop
3. Open pmw.exe (PM7)
4. App login dialog (keyboard: Tab between fields, Enter to submit)
5. Phase selector (Arrow keys + Enter)
6. Main screen ready
```

### Menu Bar: File | Transaction | E-Invoice | Report | Utilities | Window | Help | Security

### Report Menu (Alt+R)
```
Statement, Reminders, Customer Transactions, Ageing Reports, Billing,
Billing and Collection, Collection, Monthly Balance Analysis,
Outstanding Balance Analysis, Credit Review, Deposits,
Interest Charges Estimation, GL Listing, Audit List, Strata Title, Ad-hoc, Mailing Label
```

### Output: Print (→ browser download) | PDF (→ RDP filesystem) | DATA

### Other Modules: AP7, CB7, GL7, TX7, SM7

---

## §10 · Pending Recon

| System | What we need | Who |
|--------|-------------|-----|
| Access card (CHV) | Vendor name, portal URL, credentials, target workflows | Max / Hoe Zee How |
| Car plate (CHV) | Vendor name, portal URL, credentials, target workflows | Max / Hoe Zee How |
| Advelsoft workflows | Which reports/actions to automate first | Hoe Zee How |

---

## §11 · Schema Changes Needed

```sql
-- Route commands to correct system
ALTER TABLE moa_commands ADD COLUMN system_id TEXT;

-- Skills are per-system
ALTER TABLE moa_skills ADD COLUMN system_id TEXT;
ALTER TABLE moa_skills ADD COLUMN connection_type TEXT;

-- Seed supported systems
INSERT INTO moa_supported_systems (category, name, vendor, is_active) VALUES
  ('accounting', 'Advelsoft Property Management', 'Advelsoft', true),
  ('accounting', 'CSS Property Management', 'CSS', false),
  ('accounting', 'Condo Master', 'Condo Master', false),
  ('access_card', 'TBD', 'TBD', true),
  ('car_plate', 'TBD', 'TBD', true);
```

---

## §12 · Existing Infrastructure (Already Built in Minilab)

| Component | Status |
|-----------|--------|
| moa_commands table + APIs (submit/poll/result) | ✅ |
| moa_agents table + heartbeat API | ✅ |
| moa_skills table + list API | ✅ |
| moa_supported_systems table + API | ✅ |
| integration_requests table + superadmin UI | ✅ |
| BM settings page (/bm/settings/moa) | ✅ |
| AI handlers (check_moa_status, send_moa_command) | ✅ |
| Cron cleanup (timeout stuck, offline stale) | ✅ |

---

## §13 · Error Recovery

1. **Wait + Retry** — timing issues, slow responses
2. **Screenshot + Escape** — unexpected dialogs (Type 2)
3. **Vision LLM** — Claude/DeepSeek identifies unknown screen state (Type 2)
4. **Re-login** — session expired, detect auth redirect (Type 1)
5. **Abort + Report** — screenshot final state, mark failed, alert BM

---

## §14 · Recording New Skills (Puppet Master)

1. **Human records** — Max navigates vendor system, documents every action/keystroke/click
2. **Claude compiles** — converts action log + screenshots into skill JSON
3. **Zero deploy** — insert JSON into moa_skills table via Supabase
4. **Test** — insert test command into moa_commands, watch agent execute

---

## §15 · Build Order

| Session | Task | Effort |
|---------|------|--------|
| 1 | Agent core + connector pattern + registry | 1 session |
| 2 | Advelsoft connector (login flow end-to-end) | 1 session |
| 3 | Skill executor + core actions (key, type, click, download) | 1 session |
| 4 | First Advelsoft skill (statement export) — record + compile + test | 1-2 sessions |
| 5 | Access card recon + web connector + first skill | 1-2 sessions |
| 6 | Car plate recon + web connector + first skill | 1-2 sessions |
| 7 | Error recovery + heartbeat + keepalive | 1 session |
| 8 | More Advelsoft skills (ageing, collection) | 1 each |
| 9 | Multi-building scaling | 1 session |

**Total: 10-14 sessions**
