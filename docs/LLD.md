# LeaseHawk — Low-Level Design

Sibling build of the shipped contract-reviewer. Same Worker+Supabase+credits+provider architecture — this doc only calls out what's different for lease review.

## Architecture

```
Browser (static frontend, Workers Assets)
  │  magic-link auth (supabase-js CDN)
  │  upload PDF/image → POST /api/review
  ▼
Cloudflare Worker (TypeScript, ESM, single deploy)
  ├─ /config.js            → public Supabase URL/anon key
  ├─ /api/review    (POST) → verify JWT, spend_credit, store doc, dispatch review
  ├─ /api/review-callback (POST) → VPS/async provider posts result back, signed
  ├─ /api/report/:id (GET) → fetch stored report, RLS-scoped to owner
  ├─ /api/me        (GET) → profile + credit balance
  └─ /__announce    (POST) → tunnel URL rotation (VPS provider only, per shipped pattern)
  │
  ├─ Supabase (plain fetch, service-role key)
  │    GoTrue /auth/v1/user  → JWT verification
  │    PostgREST             → reviews table CRUD
  │    Storage (private bucket "leases") → uploaded documents
  │
  └─ LLM provider (switchable, see below)
       anthropic (claude-sonnet-4-6) | deepseek (deepseek-chat) | VPS (claude -p, async)
```

Request flow: user uploads → Worker verifies JWT → `spend_credit(user_id)` RPC → PDF text extracted (unpdf) or image passed natively → provider produces structured JSON report → row updated `status='done'` → frontend polls `/api/report/:id`. Any provider failure → `refund_credit(user_id)`, `status='failed'`.

## Data model

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  credits int not null default 1,
  created_at timestamptz not null default now()
);
-- RLS: read own row only. spend_credit/refund_credit are SECURITY DEFINER RPCs,
-- revoked from anon/authenticated, granted to service_role only (Worker calls them).

create table public.reviews (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  filename text not null,
  storage_path text not null,
  status text not null default 'pending' check (status in ('pending','done','failed')),
  report jsonb,
  error text,
  model text,
  input_tokens int,
  output_tokens int,
  created_at timestamptz not null default now()
);
-- RLS: read own reviews only (auth.uid() = user_id). Worker writes via service-role.
-- State machine: pending -> done | failed. No other transitions. A stale-credit
-- sweep cron flips pending rows older than N minutes to failed + refunds.
```

Storage bucket `leases` (private, service-role only writes, signed URLs never issued to the frontend — Worker proxies report data, not raw files, back to the client).

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/config.js` | GET | none | Serves public Supabase config | — |
| `/api/review` | POST | JWT | Spend credit, store doc, dispatch to provider (sync or async) | 401 no/bad JWT; 402 no credits; 500 storage write fails (no spend); provider error → refund + `status=failed` |
| `/api/review-callback` | POST | shared secret | VPS posts async result; updates review row | 401 bad secret; 404 unknown review id; malformed JSON → repair attempt then `failed` + refund |
| `/api/report/:id` | GET | JWT | Returns report JSON, RLS-scoped | 404 not found/not owner; 200 with `status:pending` while in flight |
| `/api/me` | GET | JWT | Profile + credit balance | 401 no JWT |
| `/__announce` | POST | shared secret | VPS tunnel URL rotation on restart (KV-backed) | 401 bad secret |

## LLM strategy

Provider choice per the reference: **anthropic (`claude-sonnet-4-6`)** default for quality, **deepseek (`deepseek-chat`)** as a cheaper text-only fallback, **VPS `claude -p`** for the async/heavy path. Lease documents are almost always short (1-6 pages) and text-native (typed PDF or scanned image) — sync anthropic/deepseek call is the default path; VPS async is the fallback when the sync call would risk Cloudflare's ~100s cap (large scanned multi-page leases run through OCR-heavy extraction) or when API rate limits are hit.

Sync vs async: default to **sync** (anthropic or deepseek) since a typical lease review completes well under 100s. Use the **async VPS pattern** (return `{id}` immediately, VPS POSTs signed callback to `/api/review-callback`, cron sweeps stale `pending` rows) only as an overflow path when sync providers are rate-limited or the document is unusually large — same mechanism as the shipped product, not a new one.

Prompt strategy: system prompt frames the model as a plain-English lease reviewer for Indian tenants/landlords, explicitly instructed to (a) flag deposit amount vs. local norms, lock-in period, notice period, maintenance responsibility, and renewal terms; (b) note that Rent Control Act applicability is state-specific and not to assert which act governs; (c) flag whether registration/stamp duty appears addressed, without asserting compliance; (d) never claim to be legal advice. Output forced to strict JSON (`output_config: json_schema` for anthropic; `json_object` + explicit JSON instruction for deepseek), no raw quotes inside string values (matches the shipped no-raw-quotes prompt rule + Haiku repair fallback).

```json
{
  "summary": "string",
  "property_type": "residential | commercial",
  "parties": ["string"],
  "rent_amount": "string",
  "deposit_amount": "string",
  "lock_in_period": "string",
  "notice_period": "string",
  "overall_risk": "low | medium | high",
  "clauses": [
    { "category": "deposit|lock_in|notice|maintenance|renewal|registration|other",
      "status": "found | missing | risky",
      "risk_level": "green | amber | red",
      "snippet": "string", "plain_english": "string", "india_note": "string" }
  ],
  "top_risks": ["string"],
  "missing_critical": ["string"],
  "questions_to_ask": ["string"]
}
```

Cost per operation: a typical lease (~2k input tokens, ~1.5k output tokens) on `claude-sonnet-4-6` is roughly Rs2-3 at current API pricing; deepseek-chat is roughly a fifth of that. At Rs99/review both providers leave healthy margin; VPS async path is near-zero marginal cost (subscription-billed `claude -p`) but slower.

## Frontend pages

`index.html` (landing + pitch + disclaimer), `app.html` (upload + review list, magic-link gated), `report.html` (single review detail view), `tos.html` (terms incl. not-legal-advice disclaimer), `pricing.html`.

## Error handling / credits flow

Credit spend happens only after the document is durably stored; any downstream failure (provider error, timeout, malformed JSON that survives the Haiku repair fallback) triggers `refund_credit` and marks the review `failed` with a user-facing error string. The stale-credit sweep cron (same as the shipped reference) catches reviews stuck `pending` past a timeout (VPS callback never arrived) and refunds them too.

## Integrations and launch gates

No third-party OAuth or partner API is required for v1 — auth is Supabase magic-link (no launch gate), payments are manual UPI initially (Razorpay checkout is a **post-KYC** gate, not required to launch). No blocking external approvals for v1.

## Security notes

RLS on `profiles` and `reviews` restricts every SELECT to `auth.uid() = user_id`; all writes go through the Worker's service-role key, never exposed to the client. `spend_credit`/`refund_credit` are SECURITY DEFINER and revoked from `anon`/`authenticated` roles. Uploaded lease documents live in a private Storage bucket, never made public. Input validation: file type/size checked before upload accepted; JSON output validated against the schema server-side before being written to `reviews.report`; the `/api/review-callback` and `/__announce` routes require a shared-secret bearer token, not JWT (they're server-to-server).

## Out of scope for v1

Multi-document comparison, negotiation/redline suggestions, e-signature integration, state-specific legal-database lookups (Rent Control Act text), Razorpay checkout (manual UPI only at launch), landlord-side portfolio dashboards, WhatsApp ingest, multi-language output (English only).
