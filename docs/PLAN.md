# LeaseHawk — Build Plan

TDD, one task per focused session. Each task lists files touched, the interfaces/seams it produces, the test to write first, and done-criteria. Mirror the shipped contract-reviewer's file layout unless noted.

## T1 — Repo scaffold + Supabase schema

Files: `package.json`, `tsconfig.json`, `wrangler.toml`, `supabase/migrations/0001_init.sql`, `.dev.vars.example`, `.gitignore`.
Interfaces: none yet (schema only).
Test first: none (infra task) — but write `supabase/migrations/0001_init.sql` to exactly match the LLD's `profiles`/`reviews` tables, RLS policies, `spend_credit(p_user uuid) returns boolean`, `refund_credit(p_user uuid) returns void`, and the `leases` private storage bucket.
Done: `wrangler dev` boots with no errors; migration applies cleanly on a fresh Supabase project via `supabase db push`.

## T2 — Review data types + Supabase client (`src/review.ts`, `src/supa.ts`)

Files: `src/review.ts`, `src/supa.ts`, `test/supa.test.ts`.
Interfaces: `export interface ReviewReport { summary: string; property_type: "residential"|"commercial"; ...; }` (full shape per LLD JSON schema); `export const MODEL = "claude-sonnet-4-6"`; `getReview(id): Promise<Review|null>`, `insertReview(...)`, `updateReview(id, patch)`, `spendCredit(userId): Promise<boolean>`, `refundCredit(userId): Promise<void>` — all plain-fetch PostgREST/RPC calls with injected `fetch`.
Test first: mock `fetch`, assert `spendCredit` calls the RPC endpoint with the right body and returns `false` when the RPC returns `false` (no credits).
Done: `vitest run test/supa.test.ts` green with no live network calls.

## T3 — Anthropic provider (`src/providers/anthropic.ts`)

Files: `src/providers/anthropic.ts`, `test/providers/anthropic.test.ts`.
Interfaces: `reviewWithAnthropic(opts: { text: string; apiKey: string; fetchFn?: typeof fetch }): Promise<{ report: ReviewReport; model: string; inputTokens: number; outputTokens: number }>`. System prompt lives in `src/prompt/system.ts` per the LLD's lease-specific instructions (deposit/lock-in/notice/maintenance/renewal/registration, state-Rent-Control caveat, not-legal-advice).
Test first: mock `fetch` returning a canned Anthropic JSON response; assert the parsed `ReviewReport` matches shape and `model === "claude-sonnet-4-6"`. Add a case for a malformed-JSON response to exercise the repair path.
Done: unit tests green; prompt file reviewed against LLD caveats checklist.

## T4 — DeepSeek fallback provider (`src/providers/deepseek.ts`)

Files: `src/providers/deepseek.ts`, `test/providers/deepseek.test.ts`.
Interfaces: `reviewWithDeepSeek(opts): Promise<{ report; model: "deepseek-chat"; inputTokens; outputTokens }>`, same shape as T3, `json_object` mode + explicit JSON instruction (no raw quotes rule).
Test first: mock `fetch`, assert correct request body (`response_format: {type:"json_object"}`) and correct parsing.
Done: unit tests green.

## T5 — VPS async provider (`src/providers/vps.ts`)

Files: `src/providers/vps.ts`, `test/providers/vps.test.ts`.
Interfaces: `dispatchVpsReview(opts: { baseUrl; sharedSecret; reviewId; text; callbackUrl; fetchFn? }): Promise<void>` — copy the shipped signature exactly (fire-and-forget POST to `/review-async`).
Test first: mock `fetch`, assert POST body/headers; assert it throws `ReviewError` on non-2xx.
Done: unit tests green.

## T6 — PDF/image text extraction (`src/pdf.ts`)

Files: `src/pdf.ts`, `test/pdf.test.ts`.
Interfaces: `extractPdfText(bytes: ArrayBuffer): Promise<string>` (unpdf, per reference); image documents bypass extraction and are passed natively to the anthropic vision path (image leases are rarer than for receipts but do occur — scanned agreements).
Test first: feed a small fixture PDF, assert extracted text contains a known phrase.
Done: unit test green against a checked-in fixture.

## T7 — Route handlers (`src/handlers.ts`, `src/router.ts`)

Files: `src/handlers.ts`, `src/router.ts`, `test/router.test.ts`, `test/handlers.test.ts`.
Interfaces: `route(req, deps): Promise<Response|null>` with the exact route table from the LLD; `handleReview`, `handleReviewCallback`, `handleReport`, `handleMe`, `handleAnnounce` matching the shipped signatures (JWT verify via GoTrue, spend/refund on failure, stale-pending sweep hook).
Test first: for each route, mock `deps` and assert correct dispatch + status codes (401/402/404 cases first — these are the ones easy to get wrong).
Done: `vitest run` green across router + handlers; every failure mode in the LLD's API table has a corresponding test.

## T8 — Worker entrypoint (`src/worker.ts`)

Files: `src/worker.ts`.
Interfaces: default export `{ fetch(req, env, ctx) }` wiring `route()` to real `env` bindings and a scheduled handler for the stale-credit sweep cron.
Test first: none new (integration-level; covered by smoke test in T10).
Done: `wrangler dev` serves `/config.js` and rejects unauthenticated `/api/review` with 401.

## T9 — Frontend (`public/*.html`)

Files: `public/index.html`, `public/app.html`, `public/report.html`, `public/tos.html`, `public/pricing.html`.
Interfaces: none (static, supabase-js CDN + fetch to `/api/*`).
Test first: none (manual/visual) — but every page carrying review output must render the exact "automated review, not legal advice" disclaimer string.
Done: manual click-through against `wrangler dev` completes a full upload → review → report cycle using a test lease PDF.

## T10 — Deploy + live smoke test + launch checklist

Files: `scripts/smoke.ts`, `wrangler.toml` (production vars), secrets set via `wrangler secret put`.
Interfaces: `scripts/smoke.ts` runs end-to-end against the deployed URL (signup/magic-link stub, upload fixture lease, poll `/api/report/:id` to `done`, assert disclaimer present in report).
Done: `wrangler deploy` succeeds; `npm run smoke` passes against production; launch checklist confirmed — not-legal-advice disclaimer live on landing + report + ToS, pricing page shows Rs99/review, all secrets (`ANTHROPIC_API_KEY` or `DEEPSEEK_API_KEYS`, `SUPABASE_SERVICE_ROLE_KEY`, `VPS_SHARED_SECRET` if used) set via `wrangler secret put`, no `.dev.vars` committed.
