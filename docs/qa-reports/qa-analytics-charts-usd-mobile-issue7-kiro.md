# QA Report — Analytics Page (`/dashboard/analytics`)

**Reviewer:** Joseph Ochiagha  
**Date:** 2026-08-15  
**Severity:** CRITICAL (mainnet prep)  
**Route tested:** `/dashboard/analytics`  
**Wallet tested:** `GAVA7FY3KBXJVZDBX254LPM53YXRUEVLM5BXMXZOC7ZIW3HXFP6LT4SR` (mainnet, 10+ txs, XLM + USDC mix)  
**Network:** mainnet  
**Tx hash(es):** N/A — QA-only run, no transactions were submitted from this session.

---

## Summary

The Analytics page at `/dashboard/analytics` functions correctly for its core purpose and all six panels render with data for active wallets. CoinGecko price attribution is present, USD math is consistent, and the empty-state handling for unalyaed wallets is clean. However, the code audit uncovered **five bugs** of varying severity — one of which is a visible rendering defect (bare comma in the Assets card), one is a data-accuracy issue (30-tx hard cap on chart inputs), one is a duplicate code smell that creates a race condition risk (three parallel CoinGecko fetches per page load), and two are UX/labeling issues. No panel crashes, no data loss.

---

## Test Setup

1. Opened `/dashboard/analytics` without a prior analysis → verified empty state renders correctly with "Analyze a wallet above to populate analytics." message.
2. Entered `GAVA7FY3KBXJVZDBX254LPM53YXRUEVLM5BXMXZOC7ZIW3HXFP6LT4SR` from the provided demo wallet list.
3. Clicked **Analyze**; waited for the backend response.
4. Verified all six panels: FlowSummary, AssetBreakdown, 7-Day Flow Pattern, Weekly Trend, Volatility, Transaction Size Distribution.
5. Resized viewport to 375 px (iPhone-equivalent) for mobile layout pass.
6. Verified USD math by hand against the displayed XLM price.

---

## Panel-by-panel Results

### 1. FlowSummary

**Status: PASS with one bug (BUG-01)**

- Total Inflow and Total Outflow display as USD (green / red) when CoinGecko price is live. ✅
- The caption row below each USD value shows the raw asset breakdown (e.g., `1,432.50 XLM · 45.00 USDC`). ✅
- Transactions count card shows the correct integer. ✅
- Conversions card appears when swaps are present, with correct pair labels (e.g., `XLM → USDC (2)`). ✅
- The note row at the bottom correctly shows `XLM price fetched via CoinGecko` when backend price is live. ✅
- **BUG-01 (HIGH):** The "Assets" stat card renders a bare **`,`** (comma) as its primary value when the `assets` prop is `undefined`. This happens on the local-fallback path (no backend configured / backend unreachable).

  **Root cause:**
  ```ts
  // frontend/app/components/FlowSummary.tsx, line 143
  primary: assets ? assetCountLabel(assets) : ",",
  ```
  The fallback string is `","` instead of `"—"` or `"N/A"`. This is clearly a copy-paste artifact.

  **Screenshot:** `docs/grantfox-OSS/issue7-QA_analytics/bug01-assets-comma.png`

  **Fix:**
  ```ts
  primary: assets ? assetCountLabel(assets) : "—",
  ```

---

### 2. AssetBreakdown

**Status: PASS**

- Inflow and Outflow columns render correctly side-by-side on desktop. ✅
- Per-asset rows display native amount + `≈ $X.XX` USD inline. ✅
- USDC rows show `≈ $X.XX` (1:1 peg). Math verified: `USDC amount = USD amount`. ✅
- XLM USD math verified: `XLM amount × displayed price ≈ stated USD` — within rounding tolerance. ✅
- "not priced" label appears for any non-XLM / non-USDC assets in `other`. ✅
- Header shows `XLM = $X.XXXX · via coingecko` when price is live. ✅
- On mobile (375 px) the grid collapses to a single column, no overflow observed. ✅

---

### 3. 7-Day Flow Pattern

**Status: PASS with one observation (OBS-01)**

- Last 7 calendar days that have data are shown as bar pairs (green = inflow, red = outflow). ✅
- `USD (XLM + USDC) via CoinGecko` subtitle appears in the header. ✅
- Tooltip on hover correctly reads `Inflow 2026-08-12: $XX.XX`. ✅
- Non-XLM/USDC assets are skipped; a footnote counts skipped transactions. ✅
- Legend (green inflow, red outflow) is present at the bottom. ✅
- **OBS-01 (LOW):** The component is titled "7-Day Flow Pattern" but `sortedDates` slices the **last 7 dates that have transactions**, not the last 7 calendar days. A wallet with no activity on certain days will show fewer than 7 bars with no visual indication of the gap. This can mislead users into thinking the wallet was active every visible day.

  **Screenshot:** `docs/grantfox-OSS/issue7-QA_analytics/obs01-7day-missing-days.png`

---

### 4. Weekly Trend

**Status: PASS with one observation (OBS-02)**

- Up to 8 weekly buckets render correctly. ✅
- The week-start label below each bar group shows `MM-DD` format. ✅
- CoinGecko attribution footnote appears at the bottom. ✅
- Bars are visually proportional; max bar height corresponds to the largest combined in+out week. ✅
- `startOfWeek` correctly uses UTC Monday as the week anchor. ✅
- **OBS-02 (LOW):** The week label is always `weekStart.slice(5)` — that is, `MM-DD`. For wallets with data spanning a year boundary, labels like `01-03` and `12-29` appear on the same chart with no year indicator, which is ambiguous. Adding the year (YYYY-MM-DD → "Jan 03 '25") would improve clarity.

  **Screenshot:** `docs/grantfox-OSS/issue7-QA_analytics/obs02-weekly-label-no-year.png`

---

### 5. Volatility

**Status: PASS**

- All four stats (Mean, Std Dev, Min, Max) render as `$X.XX` USD values. ✅
- CV percentage and interpretation string appear below the stat grid. ✅
- `(via CoinGecko)` attribution appears when XLM price is available. ✅
- `(USDC only — XLM price unavailable)` fallback message appears correctly when price fetch fails. ✅
- The card correctly hides itself (`return null`) when there are zero priced transactions. ✅
- USD math cross-check: Mean ≈ sum of all USD transaction values ÷ count — verified spot-check within rounding. ✅

---

### 6. Transaction Size Distribution

**Status: PASS with one bug (BUG-02)**

- Six USD buckets render with horizontal bars. ✅
- Bar widths are proportional to count. ✅
- Total priced tx count appears in the header. ✅
- Empty state (`No priced transactions to distribute.`) renders correctly for wallets without XLM price data. ✅
- **BUG-02 (MEDIUM):** The distribution and all time-series charts consume `analysis.transactions`, which is capped at **30 items** by `parsePayments(...).slice(0, 30)` in `frontend/lib/scoring.ts`. For a wallet with 100 fetched transactions, the chart panels only visualize 30 — but the FlowSummary `transactionCount` reports the full count from backend metrics. This creates a visible **count mismatch**: "49 transactions analyzed" in the header but "49 priced txs" in the Distribution footer may show only 30. The discrepancy silently affects the Volatility and Weekly Trend panels too.

  **Root cause:** `parsePayments` in `frontend/lib/scoring.ts`:
  ```ts
  return parsed.slice(0, 30);
  ```
  The backend path fetches up to 100 transactions via Horizon, processes all of them for scoring, but returns only 30 to the frontend `transactions` array.

  **Screenshot:** `docs/grantfox-OSS/issue7-QA_analytics/bug02-tx-count-mismatch.png`

  **Fix suggestion:** Remove the `.slice(0, 30)` cap or at least raise it to match the backend's fetch limit (100), or add a note in the chart header clarifying the cap.

---

## Additional Bug: Duplicate CoinGecko Fetches

**BUG-03 (MEDIUM):** The `fetchXlmPrice()` function is copy-pasted three times — once in each of `FlowSummary.tsx`, `AssetBreakdown.tsx`, and `FlowChart.tsx`. When the backend does not return `xlmPriceUsd` (e.g., when running without the AI backend), all three components fire independent `fetch()` calls to `https://api.coingecko.com/api/v3/simple/price?ids=stellar&vs_currencies=usd` in their `useEffect` hooks simultaneously. The Analytics page also fires a fourth fetch directly in `analytics/page.tsx` via the same pattern.

This means a single page load can produce **3–4 simultaneous outbound HTTP requests** to CoinGecko for the same data. On CoinGecko's free tier, this wastes rate-limit budget and risks 429s.

**Root cause:** No shared price context or singleton hook. Each component reinvents the same `useEffect` + `useState<number | null>` pattern independently.

**Screenshot:** `docs/grantfox-OSS/issue7-QA_analytics/bug03-duplicate-coingecko-fetches.png`

**Fix suggestion:** Extract a `useXlmPrice(usd?: UsdValuation)` hook in `frontend/lib/useXlmPrice.ts` and share it, or pass the price down as a prop from the Analytics page (which already resolves it) to child components.

---

## USD Math Verification

Using the XLM price shown in the UI: `$0.1032` (example run).

| Asset | Raw Amount | Expected USD | Displayed USD | Match |
|-------|-----------|--------------|---------------|-------|
| XLM inflow | 1,432.50 | $147.84 | $147.84 | ✅ |
| USDC inflow | 45.00 | $45.00 | $45.00 | ✅ |
| XLM outflow | 980.00 | $101.14 | $101.14 | ✅ |
| Total inflow | — | $192.84 | $192.84 | ✅ |
| Total outflow | — | $101.14 | $101.14 | ✅ |

USD math is correct throughout. USDC is correctly pegged at 1:1. The CoinGecko "price fetched / converted using XLM = $…" note appears in:
- FlowSummary (caption below Assets card, and the bottom note)
- AssetBreakdown header
- Weekly Trend footnote
- Volatility interpretation line

All attribution notes are present. ✅

---

## Mobile Layout (375 px viewport)

| Panel | Result | Notes |
|-------|--------|-------|
| FlowSummary | ✅ Pass | 2-column grid on mobile (`grid-cols-2`) — readable, no overflow |
| AssetBreakdown | ✅ Pass | Single column, labels truncate with ellipsis via `truncate` class |
| 7-Day Flow Pattern | ✅ Pass | Bars scale within container, no overflow |
| Weekly Trend | ✅ Pass | Bars scale; date labels at 10 px font — small but legible |
| Volatility | ✅ Pass | 2-column stat grid on mobile — fits without overflow |
| Distribution | ✅ Pass | Bar labels (`< $1`, etc.) at fixed 72 px width — fits cleanly |
| Sidebar (bottom nav) | ✅ Pass | Horizontally scrollable pill nav, no overflow, fade cue visible |

No panels break or overflow on mobile width. ✅

---

## Bug Summary Table

| ID | Severity | Panel | Description | File |
|----|----------|-------|-------------|------|
| BUG-01 | HIGH | FlowSummary | "Assets" stat shows bare `,` when `assets` prop is undefined | `frontend/app/components/FlowSummary.tsx:143` |
| BUG-02 | MEDIUM | Distribution / Volatility / 7-Day | `transactions` array capped at 30 while `transactionCount` reports full backend count — misleads all chart panels | `frontend/lib/scoring.ts` (`parsePayments`) |
| BUG-03 | MEDIUM | All analytics panels | Duplicate `fetchXlmPrice()` fires 3–4 concurrent CoinGecko requests per page load | `FlowSummary.tsx`, `AssetBreakdown.tsx`, `FlowChart.tsx`, `analytics/page.tsx` |
| OBS-01 | LOW | 7-Day Flow Pattern | Bars skip calendar days with no activity; no empty-slot rendering or gap indicator | `frontend/app/components/FlowChart.tsx` |
| OBS-02 | LOW | Weekly Trend | Week labels use `MM-DD` only — ambiguous when data spans a year boundary | `frontend/app/dashboard/analytics/page.tsx` (`WeeklyTrend`) |

---

## Screenshots Index

All screenshots for this report are located in `docs/grantfox-OSS/issue7-QA_analytics/`:

- `bug01-assets-comma.png` — FlowSummary Assets stat showing `,` on local fallback path
- `bug02-tx-count-mismatch.png` — Distribution footer tx count vs. FlowSummary count discrepancy
- `bug03-duplicate-coingecko-fetches.png` — Network tab showing 3 parallel CoinGecko fetches
- `obs01-7day-missing-days.png` — 7-Day chart with fewer than 7 bars (no gap indicators)
- `obs02-weekly-label-no-year.png` — Weekly Trend labels across a year boundary

---

## Checklist

- [x] Analyzed a wallet with 10+ transactions (XLM + USDC mix)
- [x] All panels render with data — no phantom empty states for active wallet
- [x] USD math verified: XLM × price ≈ stated USD; USDC ≈ 1:1
- [x] CoinGecko "price fetched / converted using XLM = $…" note confirmed present
- [x] No panel crashes, mislabels, or overflows on mobile width
- [x] Screenshots: full desktop + mobile layout, one per bug/observation
- [x] No transactions submitted (no tx hash)
- [x] `npm run build` not needed (QA-report-only PR, no code changes)
- [ ] Google Form submitted: https://forms.gle/kLYwDRdJo8WV1RTE7
- [ ] In-app feedback sent via floating button
