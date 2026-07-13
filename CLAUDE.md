# Household Financial Planner — Project Context

## What this is
Single-file HTML financial planning app for Brandon & Minsu. Deployed to GitHub Pages
(public repo). Planning/forecasting tool, NOT a transaction tracker.

## Files
- `index.html` — DEPLOYABLE version. Sanitized demo defaults only. This is the only
  file that ever gets committed/pushed. Contains noindex + PWA meta.
- `financial-planner.html` — personal working copy with reconstructed real defaults.
  NEVER commit (gitignored).
- `finances-data.json` — Brandon's real data blob, imported via the app's
  Overview → Backup & Transfer. NEVER commit (gitignored).

## Architecture (all inside index.html, one <script> block)
- State: single object `S`, persisted via 3-tier `store` abstraction:
  window.storage (Claude artifacts) → localStorage (hosted/standalone) → memory-only.
  Saved under key `finapp-state`; API key under `finapp-apikey` (device-only, never in file).
- Tabs: 6 top-level GROUPS with sub-tabs — Overview | Cash Flow (Income, Expenses,
  Debt) | Position (Net Worth, Portfolio) | Future (Retirement, Simulation, Forecast,
  Goals) | AI Strategy | Settings. Each view is `{html(), after()}`; router in
  `renderContent()` reads `currentGroup()` / `currentView()`.
- Header holds persistent ⬆Push / ⬇Pull buttons (visible on every tab).
- Net Worth shows a Home Equity breakout + "Investable (ex-home)" KPI.
  Charts via Chart.js CDN, destroyed/recreated per render (`charts` map).
- Tax model: 2026 MFJ federal brackets + std deduction, NC 3.99% flat, FICA incl.
  additional Medicare. Minsu's tips % excluded from income tax but not FICA.
  Roth 401k modeled post-tax. See `calcHousehold()`.
- Debt payments auto-drop after their `end` (YYYY-MM) date — see `activeDebtPayment()`.
- Portfolio: stocks (qty × price) and options (contracts × 100 × premium, short = negative).
- Monte Carlo: `runMonteCarlo()`, monthly geometric steps, Box-Muller normals,
  percentile fan (p10/p50/p90) + success % vs 4%-rule target.
- AI Strategy: BYO Anthropic API key, direct browser call with
  `anthropic-dangerous-direct-browser-access` header, model claude-sonnet-4-6,
  web_search tool enabled for live ticker prices. Snapshot built by `buildSnapshot()`.

## Invariants — do not break
1. Real financial data never enters index.html or the git history.
2. API key lives only in storage, never in file or state export.
3. State migrations: when adding keys to DEFAULTS, add a guard in `loadState()`
   so existing saved states don't lose data or crash.
4. Single-file constraint: no build step, no modules — must work as one HTML file
   opened anywhere. CDN deps from cdnjs only.
5. Brandon owns master data; Minsu is import-only. Export blob = sync mechanism.

## Known limitations / roadmap candidates
- Monte Carlo treats portfolio as one lognormal blob — options nonlinearity NOT modeled.
- Holdings lack an account field (taxable/Roth/401k) — tax-placement analysis is blind.
- No shared backend; sync is manual export/import.
- Daycare expense has no end date automation.
- Manual prices go stale; AI can pull live prices but the app itself doesn't.

## Owner preferences (apply to all interactions)
Terse, direct, quantitative. No pandering. Execute first, flag blind spots after.
Direct recommendations over open questions.

## Cloud sync
Private GitHub repo as shared backend via Contents API: `repoPush()`/`repoPull()`
in Overview → Cloud Sync card. Reads AND writes require the token (real access
control, unlike gists). Config (`token`, `repo` as owner/name, `path`, `autoPull`)
stored under `finapp-sync` key — device-only, NEVER in state exports or commits.
PUT requires current file sha (fetched via GET before each push; 404 = create).
Base64 with unicode-safe encode/decode (`b64e`/`b64d`). Fine-grained PAT scoped
to ONLY the data repo, Contents read/write. Last-write-wins; single-master rule
(Brandon pushes, Minsu pulls). Invariant 6: sync credentials never enter S,
exports, or committed files.


## Queued work (in priority order)
1. Rename the "Cash Flow" tab group to "Budget". (Trivial — use it to validate the
   edit → commit → push → live loop.)
2. MORTGAGE RESTRUCTURE (per Brandon, high priority):
   - Add a Mortgage section on the Budget tab: inputs = balance, interest rate, years
     remaining, OR a manually-entered monthly payment (manual entry must be supported —
     Brandon prefers just typing the real number).
   - The mortgage payment rolls into the total EXPENSES figure (not a separate debt
     payment line) so all monthly outflows tie together in one place.
   - REMOVE the mortgage from the Debt tab's line items (Debt keeps student loans,
     personal loan — the ones with rates that matter for payoff decisions).
   - The mortgage BALANCE must still feed Net Worth as a liability so the Home Equity
     card and equity math keep working. Single source of truth: the Mortgage section.
   - Watch for double-counting: after this change, mortgage payment must appear in
     outflows exactly once.
3. EXPENSE CATEGORIES: add a `category` field to each expense line item from a fixed
   taxonomy (Housing, Food, Transport, Childcare, Utilities, Discretionary, Other).
   Prerequisite for actuals tracking. Requires a migration guard in loadState().
4. ACTUALS TRACKING — DEFERRED, do not build yet. Scope when built: bank CSV import,
   map merchants to the fixed category taxonomy, per-category actual-vs-budget chart,
   Claude summarizes variance. Prerequisites: #3 above; decide storage (the single
   state blob is likely insufficient at transaction volume — Supabase free tier is the
   fallback); resolve Minsu's accounts (the import-only ownership model breaks if her
   spending isn't in the feed).

## Deployment
Repo: github.com/BrandonM23/P-Financial-Tool (public) — serves index.html via GitHub
Pages at https://BrandonM23.github.io/P-Financial-Tool/
Data repo: github.com/BrandonM23/P-Financial-Data (private) — holds finances-data.json,
written/read by the app's sync. Never committed from here.
The repo IS the live site: a bad push goes live immediately. git revert is the undo.
