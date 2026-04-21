# CLAUDE.md — SCB Wealth Admin Portal

Project context for Claude (or any dev) picking this up. Read top to bottom before touching the file.

---

## 1. What this is

A clickable prototype for **Standard Chartered Kenya's Wealth Admin Portal** — the internal tool SCB staff use to onboard SME clients and manage their investments in the **Sanlam Shilingi Money Market** fund via the **Vault22** platform.

**Current deliverable:** single-file HTML prototype — `scb-wealth-portal.html` — no build step, opens directly in a browser.

**Not yet built:** PRD, system architecture diagram, sequence diagrams.

---

## 2. Who/what it's for

Three parties in every flow:

| Party | Role |
|---|---|
| **SCB Kenya** | Bank — holds customer CIF, settlement accounts, does the actual debit/credit |
| **Vault22** | Backend platform — order management, reconciliation engine, interest math. The admin portal is the Vault22 UI rebranded as SC. |
| **Sanlam** | Fund manager — executes unit purchase/redemption at NAV, publishes daily rates, confirms month-end interest |

Note: this is **not an omnibus model**. Investments are at individual customer level (different from the Uganda/ABSA version this was adapted from).

---

## 3. Business rules (must-know before touching any flow)

| Parameter | Value |
|---|---|
| Currency | KES only |
| Fund catalogue | Sanlam Shilingi Money Market (`SAN-SHIL-KES`) — single fund |
| Buy cut-off | 07:00 EAT |
| Sell cut-off | 07:15 EAT |
| Sanlam confirmation deadline | 14:00 EAT |
| Settlement | T+0 |
| Daily interest tolerance | ±0.99 KES absolute (below threshold = auto-reconciled, logged only) |
| EOD portfolio match | Exact match required, zero tolerance |
| Cumulative drift | Monitored separately, soft flag when material |
| Month-end confirmation | ~4th of following month (polled via Sanlam API with `is_credited=true`) |
| FWIC | Full-withdraw interest credit — customers who close holdings mid-month still get pro-rata interest paid to their SCB bank account at month-end |
| Maker-checker | **Mandatory** for FWIC batch approval — maker and checker must be different users |

**Gap period (Jan 1–4 before Sanlam confirms December):** provisional EOD records flagged, retroactive adjustment applied when confirmation arrives, per-day holding snapshots drive customer-level redistribution.

---

## 4. Repo / file structure

Everything in one file: **`scb-wealth-portal.html`**.

Top-to-bottom:
1. `<head>` — meta, Google Fonts (Plus Jakarta Sans + JetBrains Mono), Lucide icons CDN
2. `<style>` — design tokens as CSS variables, utility + component classes
3. `<body>` — sidebar + top bar skeleton, `<main id="content">` rendered dynamically
4. `<script>` — four parts:
   - Mock data arrays (`CUSTOMERS`, `FUNDS`, `ORDERS`, `RECURRING`, `RECON`)
   - `app` state object (page + wizard states)
   - Renderer functions (one per screen, returning HTML strings)
   - `attachHandlers()` — rebinds all event listeners after each render

External deps (all CDN): Lucide icons, Google Fonts. No npm, no bundler.

---

## 5. Design system

### Brand
- **SCB blue** `#0473EA` (primary) — darker variant `#0259B8`, light `#E8F1FE`
- **SCB green** `#38D200` (accent) — darker `#2AA800`, light `#E8FBE0`
- Gradient `linear-gradient(135deg, blue, green)` for logo mark, hero avatars, fund tiles

### Type
- **Plus Jakarta Sans** — all UI text
- **JetBrains Mono** — IDs, currency with commas, reference numbers (applied via `.font-mono` class, uses `font-feature-settings: 'tnum'` for aligned digits)

### Structural DNA (matches original Vault22 dashboard reference)
- Left sidebar, 248px, with labeled sections (OVERVIEW / CLIENT MANAGEMENT / INVESTMENT MANAGEMENT / FUND OPERATIONS / GOVERNANCE)
- Top bar with search + notifications + user chip
- Dashboard = greeting + 4 stat cards + 2-column (funds list + pipeline) + quick actions + recent orders table

### Component classes
`.card` · `.card-pad` · `.btn` (+ `.btn-primary` / `.btn-secondary` / `.btn-ghost` / `.btn-dark` / `.btn-success` / `.btn-danger`) · `.badge` (+ tone variants) · `.field` + `.input-wrap` + `.input` · `.select-wrap` + `.select` · `.table` · `.timeline` · `.select-row` (clickable list rows with selected state) · `.stat-card` · `.recon-table` (wide scrollable reconciliation table)

### State colors (consistent across the app)
- Green — Settled / Active / Success / Reconciled
- Blue — In Progress / Executed / Submitted
- Amber — Pending / Awaiting Sanlam / Pending KYC / Drift flagged
- Red — Failed / Rejected / Match X

---

## 6. Data model (mock arrays at top of `<script>`)

```
CUSTOMERS[]   — id, relId, name, type, status, aum, holdings, bankAccounts[]
FUNDS[]       — code, name, currency, yield, risk, aum, nav
ORDERS[]      — id, type (Buy/Sell), customer, fund, amount, status, submitted, units
RECURRING[]   — id, customer, customerId, fund, amount, frequency, nextRun, status, lastStatus, runs
RECON[]       — date, status, rates, flows, SC vs Sanlam EOD/Gross/Net/Cumulative with deltas + match booleans
```

All amounts are **strings** (formatted with commas). Keep consistent — don't mix numbers in.

---

## 7. Screens — what's built

| Screen | Status | Notes |
|---|---|---|
| Dashboard | ✅ Built | Stats, funds list, today's pipeline, quick actions, recent orders |
| Customers list | ✅ Built | Live search by name / ID / RelId |
| Customer onboarding | ✅ Built | 4-step wizard: RelId fetch → enrich → bank select → dual-create (Vault22 + Sanlam) → success with IDs |
| Customer detail | ✅ Built | Header, stats, holdings, contact, transaction history |
| Buy orders list | ✅ Built | Filters, export, status pills |
| New buy order | ✅ Built | 4-step wizard: customer → fund + amount + debit account + optional recurring → review → submit → live lifecycle |
| Sell orders list | ✅ Built | Same shape as buy list |
| New sell order | ✅ Built | 4-step wizard: customer → holding + partial/full (FWIC flagged) → review → submit → lifecycle |
| Order detail | ✅ Built | Works for Buy, Sell, and Failed states; failed state shows retry + red alert card |
| Recurring subscriptions | ✅ Built | List with Active/Paused/Failed states, "How it works" explainer |
| **Reconciliation (MMSP Fund Recon)** | ✅ Built | Wide scrollable table with 7 column groups, filters, export, drift-flagged mock row on 01 Apr |
| Funds | 🟡 Placeholder | Real screen pending |
| Interest | 🟡 Placeholder | Needs maker-checker UI for month-end distribution |
| Reports | 🟡 Placeholder | Buys, sells, recurring, trailer fees, EOD balances with filters/export |
| Audit Log | 🟡 Placeholder | Immutable action log viewer |

---

## 8. Key decisions already made (don't relitigate without reason)

- **No framework, no build step** — pure HTML/CSS/vanilla JS for portability and zero setup
- **No routing library** — navigation is `app.page = x; render()`
- **No `localStorage`** — state is in-memory only (artifact constraint + simpler)
- **CDN for Lucide + fonts** — needs internet on first load (can inline later if required)
- **"Holdings" removed** from sidebar — holdings live inside each customer's detail page
- **Single fund** — Sanlam Fixed Income removed; only Sanlam Shilingi remains
- **Kenya rebrand** — all addresses, phone numbers, emails are Kenyan; "SCB Kenya" in top bar; East Africa Time
- **Vault22 in docs ≠ "Vault22" in UI** — UI always says "SC" (e.g., "SC EOD vs Sanlam EOD" in reconciliation). The backend is still Vault22 but it's invisible to the end user.
- **Wizards are stateful** — each wizard has its own state object on `app` (`app.cw`, `app.bw`, `app.sw`); entering `*-new` route resets the wizard

---

## 9. Reconciliation screen — column structure

Seven column groups spanning the wide table:

| Group | Columns |
|---|---|
| _(no group)_ | Market Close Date, Status |
| RATE | Daily Rate, Rate Status |
| FLOW | Total Invested, Total Redemption |
| EOD BALANCE | SC EOD, Sanlam EOD, Variance, Match, Tol % |
| GROSS INTEREST | SC Gross, Sanlam Gross, Δ, Match |
| NET INTEREST | SC Net, Sanlam Net, Δ, Match |
| CUMULATIVE | SC Cumulative, Sanlam Cumulative, Δ, Match |
| SANLAM | Month-End Int, Sync, [Sync button] |

Match cell = green check / red X / em-dash (for N/A).

Mock drift on `01 Apr 2026` demonstrates the intended partial-mismatch state: EOD matches (strict rule holds) but Gross/Net/Cumulative all show +0.05 delta with red X — exactly the scenario where daily interest has drifted past the ±0.99 KES tolerance.

---

## 10. How to extend

### Add a new screen
1. Write `renderX()` returning a template-literal HTML string
2. Add `"x": renderX` to the renderers map inside `render()`
3. Add a sidebar nav button: `<button class="nav-item" data-nav="x">...</button>`
4. (Optional) Add to `parentMap` in `render()` if it's a sub-page

### Add a wizard step
Wizards store state on `app.cw/bw/sw`. Each renderer generates all steps inside one function — step switching = `app.<w>.step++; render()`.

### Add event handlers
All handlers live inside `attachHandlers()` which runs after every `render()`. Use `data-*` attributes on elements, then wire them up at the bottom of `attachHandlers`.

### Update data
Edit the arrays at the top of the `<script>` block. Keep amounts as strings with comma formatting.

---

## 11. Pending deliverables (agreed earlier)

1. **PRD** — user stories, full data models, API contracts (with labeled assumptions for unknown Sanlam APIs), edge cases, error handling
2. **System architecture diagram** — SCB ↔ Vault22 ↔ Sanlam integration, event flow, API-first, audit logging
3. **Sequence diagrams** — Onboarding, Buy flow, Sell flow (detailed with timings and failure paths)
4. **Real screens** for the 4 placeholders (Funds, Interest, Reports, Audit Log)

When resuming work, check with the user which of these to prioritize first.

---

## 12. Known gotchas

- **Lucide icons only render after `lucide.createIcons()`** — call it after every `render()` (already handled)
- **Template literals with nested backticks** break HTML strings — use single quotes inside template literals or escape carefully
- **Search input re-renders the row body only** (not the whole page) — preserves focus. Don't call full `render()` on keystroke.
- **`data-customer` row click handler** fires on the whole row including recurring subscriptions — the "more" kebab button uses `event.stopPropagation()` to prevent navigation
- **Table width** — recon table has `min-width: 2600px` to force horizontal scroll; if adding columns, bump this
- **Amounts are strings** — arithmetic requires `Number(s.replace(/,/g, ""))` pattern (already used in buy/sell unit estimation)
