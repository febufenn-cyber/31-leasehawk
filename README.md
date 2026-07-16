# LeaseHawk

Upload your Indian rental/lease agreement, get a plain-English risk review of deposit, lock-in, notice, maintenance, and registration traps before you sign.

## The problem

Indian tenancy agreements are drafted by landlords or their brokers and bury one-sided clauses — excessive security deposits, lock-in penalties, silent renewal, maintenance cost dumps — in language most tenants and small landlords never fully parse. Legal review is slow and expensive for a document worth a few thousand rupees a month in rent.

## Target buyer

Tenants and small landlords in Indian metros signing a residential or small-commercial lease, wanting a fast sanity check before signing — not a law firm engagement.

## Pricing hypothesis

Rs99 per review (single document, one-time). Same credit-based model as the shipped reference product: 1 free credit on signup, manual UPI top-up initially.

Status: planned — not yet built (50-SaaS challenge #31)

## Stack

Cloudflare Worker (TypeScript, ESM) serving static frontend + `/api/*` in one deploy. Supabase for magic-link auth, Postgres+RLS, and private document storage. LLM layer is provider-switchable (Anthropic / DeepSeek / VPS `claude -p`) behind the same async-callback pattern as the shipped product.

## How to continue this build

Nothing is built yet. Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered, TDD task list, and execute tasks in order. `CLAUDE.md` points at the private reference implementation to copy patterns from instead of re-deriving them.

## Risks / constraints

- **This is not legal advice.** The review is an automated plain-English summary, not a lawyer's opinion, and creates no lawyer–client relationship. This disclaimer must appear on every review output and on the landing page.
- **Rent Control law varies by state.** India has no single national tenancy law — most states still run pre-liberalization Rent Control Acts alongside the newer Model Tenancy Act (adopted only in a few states/UTs). Clause interpretation must stay generic ("check your state's Rent Control Act") rather than asserting state-specific legal conclusions the model can't verify.
- **Registration and stamp duty rules are state-specific** and commonly ignored in practice (leases under 12 months are often left unregistered) — the reviewer should flag the registration/stamp-duty question, not assert a compliance verdict.
- This repo reuses the shipped contract-reviewer's exact architecture (credits + refund RPCs, provider-switchable LLM layer, async VPS pattern) — deviating from those seams needs a reason, not a rewrite.
