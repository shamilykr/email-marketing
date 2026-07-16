# Email Marketing Agent — Architecture Guide

> **HTML version:** [`index.html`](index.html) · Host the `docs/email-marketing/` folder for team sharing.

---

## At a glance

| Item | Choice |
|------|--------|
| **New service** | `dextr-marketing-svc` — hotels, campaigns, send, analytics |
| **deepagent** | Scrape + KB markdown; **fills template JSON properties** from hotel KB + campaign details |
| **Auth** | **Option A:** Central auth (preferred) · **Option B:** Marketing login API |
| **Storage** | Postgres + S3 URLs in DB (bucket/credentials in env) |
| **Evolving data** | **`metadata` jsonb** on key tables — avoids rigid columns as requirements change |
| **Email** | AWS SES (From: Dextr AI domain) |

---

## Architecture map

```
Marketing UI  →  dextr-marketing-svc  →  Postgres (+ metadata jsonb)
                      ↓                      S3 (KB.md, images, HTML)
                 deepagent
                   · scrape → onboard APIs
                   · fill campaign_json from KB + template schema
                      ↓
                 AWS SES → recipients
```

---

## What deepagent does (not “campaign copy”)

deepagent does **not** write free-form marketing copy as the primary output.

**Campaign row is created in marketing-svc.** deepagent is called to **fill the properties** in the template’s JSON schema:

1. marketing-svc creates `campaigns` draft (title, description, template_id, hotel_id)
2. marketing-svc sends deepagent: **hotel KB** (markdown URL) + **campaign details** + **template `json_schema`** + **classified assets**
3. deepagent returns **values for each JSON slot** (headline, section bodies, image picks, CTA text, subject, preheader)
4. marketing-svc saves result as `campaign_json` → **renderer** merges template HTML + JSON → `html_url`

---

## Step 1 — Onboarding entry point

Each hotel gets a row in the marketing service — **separate tenant + user tables**, separate onboarding APIs.

### Tables

- **`hotels`** — name, website_url, timezone, onboard_status, `metadata` jsonb
- **`users`** — username, password_hash, role, hotel_id

Product linking (external hotel ids) is **not required at onboard**. Store in `hotels.metadata` later when central auth integration is defined.

### APIs

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/v1/hotels` | Create hotel entry |
| GET | `/api/v1/hotels/:id` | Read status |
| POST | `/api/v1/users` | Create scoped user |
| GET | `/api/v1/assets/:assetId` | Get asset |
| PUT | `/api/v1/assets/:assetId` | Update asset |
| DELETE | `/api/v1/assets/:assetId` | Delete asset |

---

## Step 2 — Onboarding flow

1. Create hotel (`pending`)
2. deepagent scrapes `website_url`
3. deepagent calls marketing APIs:
   - `PUT /hotels/:id/profile`
   - `POST /hotels/:id/kb` — `{ kb_url, version }`
   - `POST /hotels/:id/assets` — `{ url, classification }`
4. Gate: profile + KB + min assets → `ready`

KB = **Markdown** on S3 (`knowledge.md`), format from deepagent skill.

---

## Step 3 — Sign in (two options)

| Option | Mechanism | Notes |
|--------|-----------|--------|
| **A — Central auth** *(preferred)* | SSO / JWT from central auth service | User already logged in elsewhere; marketing-svc validates token + maps to hotel |
| **B — Marketing login** | `POST /api/v1/auth/login` | Standalone username/password against `users` table |

Roles: `SUPERADMIN`, `HOTEL_ADMIN`, `HOTEL_USER` — scoped to `hotel_id`.

---

## Step 4 — Layout templates

Each template: **HTML** (S3 `html_url`), **JSON schema** (DB `json_schema`), **prompt** (DB).

Templates: editorial-split, hero-action-discovery, hero-cards, full-width-stack, side-by-side, Auto.

---

## Step 5 — Campaign creation

1. `POST /campaigns` — draft in marketing-svc
2. `POST /campaigns/:id/generate` — deepagent fills JSON properties
3. Renderer → `html_url`

---

## Steps 6–10

| Step | Topic |
|------|--------|
| 6 | FE preview & editor — fetch `html_url`, `campaign_json`; save or prompt-edit |
| 7 | Recipients — single email or CSV → `campaign_recipients` |
| 8 | Send — SES, tracking links, open pixel |
| 9 | Schedule — `scheduled_at` + cron worker |
| 10 | Analytics — opens/clicks via `tracking_events` |

---

## Service split

| marketing-svc | deepagent |
|---------------|-----------|
| Hotels, users, campaigns CRUD | Website scrape |
| Onboard receive APIs | KB markdown generation |
| Template HTML + JSON schema | **Fill template JSON from KB + campaign context** |
| HTML renderer, SES, tracking | Prompt-edit JSON properties |

---

## Flexible metadata — `jsonb` design

**Assumption:** hotel and campaign data **keeps changing**. Avoid adding columns for every new field.

| Table | Core columns (stable) | `metadata` jsonb (evolving) |
|-------|----------------------|----------------------------|
| `hotels` | id, name, website_url, timezone, onboard_status | Product links, feature flags, custom ids |
| `hotel_profiles` | hotel_id, sender_from_email, sender_from_name | Scraped amenities, dining, rooms, brand — or entire scrape blob |
| `campaigns` | id, hotel_id, template_id, status, scheduled_at | Extra campaign fields without migrations |
| `email_templates` | key, html_url, json_schema, prompt | Layout variants, A/B config later |

**Trade-off:** Querying inside jsonb is slower than indexed columns — fine for v1; promote hot fields to columns when stable.

---

## Database — 11 tables

See [Database design](#database-design) in HTML or sections below in this doc.

1. hotels  
2. hotel_profiles  
3. knowledge_bases  
4. assets  
5. users  
6. email_templates  
7. campaigns  
8. campaign_recipients  
9. campaign_links  
10. tracking_events  
11. jobs  

DDL: [`../email-marketing-db-schema.sql`](../email-marketing-db-schema.sql)

---

# Database design

## `hotels`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | Marketing hotel id |
| name | text | Display name |
| website_url | text | Scrape source |
| timezone | text | IANA — scheduling |
| onboard_status | text | not_started \| pending \| completed \| failed |
| current_kb_id | uuid FK | Active KB version |
| metadata | jsonb | **Evolving** — external ids, flags, custom fields |
| scraped_at | timestamptz | Onboard complete time |
| created_at, updated_at, deleted_at | timestamptz | Audit |

---

## `hotel_profiles`

| Column | Type | Description |
|--------|------|-------------|
| hotel_id | uuid PK FK | 1:1 with hotels |
| sender_from_name | text | SES display name |
| sender_from_email | text | Dextr AI SES sender |
| reply_to | text | Optional |
| metadata | jsonb | **Scraped profile blob** — tagline, address, amenities, dining, rooms, brand |
| scraped_at, updated_at | timestamptz | |

---

## `knowledge_bases`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| hotel_id | uuid FK | |
| version | int | Unique per hotel |
| kb_url | text | HTTPS URL to knowledge.md |
| status | text | building \| ready \| failed |
| created_at | timestamptz | |

---

## `assets`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| hotel_id | uuid FK | |
| url | text | Image URL (unique per hotel) |
| classification | text | hero, room, dining, spa, … |
| alt_text | text | Optional |
| page_source | text | Optional |
| created_at | timestamptz | |

---

## `users`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| username | text UK | |
| password_hash | text | bcrypt (Option B auth) |
| role | text | SUPERADMIN \| HOTEL_ADMIN \| HOTEL_USER |
| hotel_id | uuid FK | Null for SUPERADMIN |
| email | text | |
| active | boolean | |
| metadata | jsonb | Preferences, audit extras |
| last_login_at, created_at, updated_at, deleted_at | timestamptz | |

---

## `email_templates`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| key | text UK | editorial-split, … |
| name, description | text | FE gallery |
| version | int | Layout version |
| prompt | text | LLM instructions |
| html_url | text | S3 layout HTML |
| json_schema | jsonb | Slot definitions deepagent fills |
| thumbnail_url | text | Gallery image |
| metadata | jsonb | Extra layout config |
| active | boolean | |
| created_at, updated_at | timestamptz | |

---

## `campaigns`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| hotel_id | uuid FK | |
| template_id | uuid FK | |
| template_key | text | Snapshot |
| title, description | text | User input |
| campaign_reason | text | spa, dining, seasonal, … |
| subject, preheader | text | Email headers |
| campaign_json | jsonb | **Filled template properties** (deepagent output) |
| html_url | text | Rendered email on S3 |
| status | text | draft \| scheduled \| sending \| sent \| failed \| cancelled |
| scheduled_at, timezone | timestamptz, text | |
| created_by_user_id | uuid FK | |
| metadata | jsonb | **Evolving campaign fields** |
| sent_at, last_error | timestamptz, text | |
| created_at, updated_at | timestamptz | |

---

## `campaign_recipients`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| campaign_id | uuid FK | |
| email | text | Unique per campaign |
| source | text | manual \| csv |
| send_status | text | pending \| sent \| failed |
| ses_message_id, last_error | text | |
| created_at | timestamptz | |

---

## `campaign_links`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| campaign_id | uuid FK | |
| label, destination_url | text | |
| tracking_token | text UK | |
| created_at | timestamptz | |

---

## `tracking_events`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| campaign_id | uuid FK | |
| link_id | uuid FK | Null for opens |
| recipient_email | text | |
| event_type | text | open \| click |
| created_at | timestamptz | |

---

## `jobs`

| Column | Type | Description |
|--------|------|-------------|
| id | uuid PK | |
| hotel_id, campaign_id | uuid FK | Nullable |
| type | text | scrape \| send \| scheduled_send |
| status | text | queued \| running \| succeeded \| failed |
| stage, progress_percent | text, int | |
| payload, result | jsonb | |
| error | text | |
| scheduled_for, started_at, finished_at, created_at | timestamptz | |
